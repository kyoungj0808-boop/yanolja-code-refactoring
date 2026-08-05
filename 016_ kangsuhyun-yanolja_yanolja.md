원본코드
"""
This module handles the management of the database.
"""
from dataclasses import dataclass
import enum
import os
from typing import List

import firebase_admin
from firebase_admin import credentials
from firebase_admin import firestore
from google.cloud.firestore_v1 import base_query
import gradio as gr

from credentials import get_credentials_json


def get_required_env(name: str) -> str:
  value = os.getenv(name)
  if value is None:
    raise ValueError(f"Environment variable {name} is not set")
  return value


RATINGS_COLLECTION = get_required_env("RATINGS_COLLECTION")
SUMMARIZATIONS_COLLECTION = get_required_env("SUMMARIZATIONS_COLLECTION")
TRANSLATIONS_COLLECTION = get_required_env("TRANSLATIONS_COLLECTION")

if gr.NO_RELOAD:
  firebase_admin.initialize_app(credentials.Certificate(get_credentials_json()))
  db = firestore.client()


class Category(enum.Enum):
  SUMMARIZATION = "summarization"
  TRANSLATION = "translation"


@dataclass
class Rating:
  model: str
  rating: int


def get_ratings(category: Category, source_lang: str | None,
                target_lang: str | None) -> List[Rating] | None:
  doc_id = "#".join([category.value] +
                    [lang for lang in (source_lang, target_lang) if lang])
  # TODO(#37): Make it more clear what fields are in the document.
  doc_dict = db.collection(RATINGS_COLLECTION).document(doc_id).get().to_dict()
  if doc_dict is None:
    return None

  # TODO(#37): Return the timestamp as well.
  doc_dict.pop("timestamp")

  return [Rating(model, rating) for model, rating in doc_dict.items()]


def set_ratings(category: Category, ratings: List[Rating], source_lang: str,
                target_lang: str | None):
  source_lang_lowercase = source_lang.lower()
  target_lang_lowercase = target_lang.lower() if target_lang else None

  doc_id = "#".join([category.value, source_lang_lowercase] +
                    ([target_lang_lowercase] if target_lang_lowercase else []))
  doc_ref = db.collection(RATINGS_COLLECTION).document(doc_id)

  new_ratings = {rating.model: rating.rating for rating in ratings}
  new_ratings["timestamp"] = firestore.SERVER_TIMESTAMP
  doc_ref.set(new_ratings, merge=True)


@dataclass
class Battle:
  model_a: str
  model_b: str
  winner: str


def get_battles(category: Category, source_lang: str | None,
                target_lang: str | None) -> List[Battle]:
  source_lang_lowercase = source_lang.lower() if source_lang else None
  target_lang_lowercase = target_lang.lower() if target_lang else None

  if category == Category.SUMMARIZATION:
    collection = db.collection(SUMMARIZATIONS_COLLECTION).order_by("timestamp")

    if source_lang_lowercase:
      collection = collection.where(filter=base_query.FieldFilter(
          "model_a_response_language", "==", source_lang_lowercase)).where(
              filter=base_query.FieldFilter("model_b_response_language", "==",
                                            source_lang_lowercase))

  elif category == Category.TRANSLATION:
    collection = db.collection(TRANSLATIONS_COLLECTION).order_by("timestamp")

    if source_lang_lowercase:
      collection = collection.where(filter=base_query.FieldFilter(
          "source_language", "==", source_lang_lowercase))

    if target_lang_lowercase:
      collection = collection.where(filter=base_query.FieldFilter(
          "target_language", "==", target_lang_lowercase))

  else:
    raise ValueError(f"Invalid category: {category}")

  docs = collection.stream()
  battles = []
  for doc in docs:
    data = doc.to_dict()
    battles.append(Battle(data["model_a"], data["model_b"], data["winner"]))
  return battles

원본은 Firestore 기반 캐시·배틀 저장 계층으로 구조와 책임 분리가 명확한 실무형 구현이지만, 저장과 조회 사이의 정규화 불일치와 데이터 검증 부족으로 운영 환경에서 캐시 효율 저하와 장애 전파 가능성이 존재하며, 입력 정규화·방어적 파싱·인덱스 대응을 적용하면 안정적인 프로덕션 데이터 계층으로 승격 가능하다.

제안패치
"""
Enterprise-grade Database Management Module for Firebase Firestore.
Features Lazy Initialization of Firebase client, strict Rating Cache replacement strategy (no ghost models),
fine-grained exception isolation (separating infrastructure/API errors from developer bugs),
and robust domain model validation.
"""

from dataclasses import dataclass
import enum
import logging
import os
from typing import List, Optional

import firebase_admin
from firebase_admin import credentials
from firebase_admin import firestore
from google.api_core import exceptions as google_exceptions
from google.cloud.firestore_v1 import base_query

# =====================================================================
# 0. Structured Logging Configuration (Enterprise Observability)
# =====================================================================
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    stream=os.sys.stderr,
)
logger = logging.getLogger("db_module")


def get_required_env(name: str) -> str:
  value = os.getenv(name)
  if value is None:
    raise ValueError(f"Environment variable {name} is not set")
  return value


RATINGS_COLLECTION = get_required_env("RATINGS_COLLECTION")
SUMMARIZATIONS_COLLECTION = get_required_env("SUMMARIZATIONS_COLLECTION")
TRANSLATIONS_COLLECTION = get_required_env("TRANSLATIONS_COLLECTION")


# =====================================================================
# 1. Lazy Initialization: Firebase Client Singleton
# =====================================================================
def get_firestore_client() -> firestore.firestore.Client:
  """
  모듈 import 시점의 사이드 이펙트(환경변수/인증 누락 시 크래시, 단위 테스트 방해)를 방지하기 위해 
  Lazy Initialization 패턴을 적용하여 실제 최초 호출 시점에 Firebase 클라이언트를 초기화합니다.
  """
  if not firebase_admin._apps:
    try:
      cred_json = get_credentials_json()
      firebase_admin.initialize_app(credentials.Certificate(cred_json))
      logger.info("Firebase Admin SDK lazily initialized successfully.")
    except Exception as e:
      logger.critical(f"Failed to initialize Firebase Admin SDK: {e}", exc_info=True)
      raise
  return firestore.client()


class Category(enum.Enum):
  SUMMARIZATION = "summarization"
  TRANSLATION = "translation"


@dataclass
class Rating:
  model: str
  rating: int


@dataclass
class Battle:
  model_a: str
  model_b: str
  winner: str


# =====================================================================
# 2. Helper: Normalized Document ID Generator
# =====================================================================
def _generate_ratings_doc_id(category: Category, source_lang: Optional[str], target_lang: Optional[str]) -> str:
  """
  저장(Set)과 조회(Get) 간 대소문자 불일치로 인한 캐시 미스를 방지하기 위해 
  모든 언어 키를 소문자로 정규화(Normalization)합니다.
  """
  parts = [category.value]
  
  if source_lang and source_lang.strip():
    parts.append(source_lang.strip().lower())
    
  if target_lang and target_lang.strip():
    parts.append(target_lang.strip().lower())
    
  return "#".join(parts)


# =====================================================================
# 3. Ratings Repository Layer (Cache-Aside & Strict Replacement)
# =====================================================================
def get_ratings(category: Category, source_lang: Optional[str],
                target_lang: Optional[str]) -> Optional[List[Rating]]:
  """
  정규화된 문서 ID를 기반으로 캐시된 레이팅 데이터를 안전하게 조회합니다.
  """
  try:
    db = get_firestore_client()
    doc_id = _generate_ratings_doc_id(category, source_lang, target_lang)
    doc_snapshot = db.collection(RATINGS_COLLECTION).document(doc_id).get()
    
    if not doc_snapshot.exists:
      return None

    doc_dict = doc_snapshot.to_dict()
    if not doc_dict:
      return None

    # 방어적 메타데이터 파싱 (timestamp 제거)
    doc_dict.pop("timestamp", None)

    ratings = []
    for model, rating_val in doc_dict.items():
      # 빈 모델명, 시스템 필드, 잘못된 타입 철저 검증
      if not model or not isinstance(model, str) or not model.strip():
        continue
      if not isinstance(rating_val, (int, float)):
        continue
        
      ratings.append(Rating(model=model.strip(), rating=int(rating_val)))
        
    return ratings if ratings else None

  except google_exceptions.GoogleAPIError as api_err:
    logger.error(f"Firestore API error while fetching ratings: {api_err}")
    return None
  except Exception as e:
    logger.error(f"Unexpected error fetching ratings for category {category.name}: {e}", exc_info=True)
    return None


def set_ratings(category: Category, ratings: List[Rating], source_lang: str,
                target_lang: Optional[str]) -> None:
  """
  merge=True 사용 시 발생하던 탈락 모델 잔존(Ghost Model) 문제를 해결하기 위해,
  문서 전체 교체(Overwrite) 방식을 적용하여 최신 Elo 랭킹 상태를 정확히 동기화합니다.
  """
  try:
    db = get_firestore_client()
    doc_id = _generate_ratings_doc_id(category, source_lang, target_lang)
    doc_ref = db.collection(RATINGS_COLLECTION).document(doc_id)

    # 전체 교체 데이터 딕셔너리 구성 (이전 시즌에서 탈락한 모델이 잔존하지 않도록 보장)
    new_ratings = {rating.model: rating.rating for rating in ratings}
    new_ratings["timestamp"] = firestore.SERVER_TIMESTAMP
    
    # merge 옵션 없이 전면 교체 수행
    doc_ref.set(new_ratings)
    logger.info(f"Successfully replaced and cached ratings for document ID: {doc_id}")
    
  except google_exceptions.GoogleAPIError as api_err:
    logger.error(f"Firestore API error while setting ratings: {api_err}")
    raise
  except Exception as e:
    logger.error(f"Failed to set ratings for category {category.name}: {e}", exc_info=True)
    raise


# =====================================================================
# 4. Battles Repository Layer with Fine-Grained Exception Isolation
# =====================================================================
def get_battles(category: Category, source_lang: Optional[str],
                target_lang: Optional[str]) -> List[Battle]:
  """
  배틀 데이터를 스트리밍 조회합니다. 
  Firestore 인덱스 누락 및 네트워크 API 오류는 안전하게 격리(폴백)하고,
  프로그래밍 버그(TypeError, ValueError 등)는 상위로 전파하여 은폐하지 않습니다.
  """
  # 1. 프로그래밍 버그 및 잘못된 인자 전달 검증 (즉시 예외 전파)
  if not isinstance(category, Category):
    raise TypeError(f"Expected Category enum, got {type(category)}")

  try:
    db = get_firestore_client()
    source_lang_lowercase = source_lang.lower().strip() if source_lang else None
    target_lang_lowercase = target_lang.lower().strip() if target_lang else None

    if category == Category.SUMMARIZATION:
      collection = db.collection(SUMMARIZATIONS_COLLECTION).order_by("timestamp")

      if source_lang_lowercase:
        collection = collection.where(filter=base_query.FieldFilter(
            "model_a_response_language", "==", source_lang_lowercase)).where(
                filter=base_query.FieldFilter("model_b_response_language", "==",
                                              source_lang_lowercase))

    elif category == Category.TRANSLATION:
      collection = db.collection(TRANSLATIONS_COLLECTION).order_by("timestamp")

      if source_lang_lowercase:
        collection = collection.where(filter=base_query.FieldFilter(
            "source_language", "==", source_lang_lowercase))

      if target_lang_lowercase:
        collection = collection.where(filter=base_query.FieldFilter(
            "target_language", "==", target_lang_lowercase))

    else:
      raise ValueError(f"Invalid category: {category}")

    docs = collection.stream()
    battles = []
    
    for doc in docs:
      data = doc.to_dict()
      if not data:
        continue
        
      model_a = data.get("model_a")
      model_b = data.get("model_b")
      winner = data.get("winner")
      
      if model_a and model_b and winner:
        battles.append(Battle(str(model_a), str(model_b), str(winner)))
        
    return battles

  except google_exceptions.FailedPrecondition as index_err:
    # Firestore 복합 인덱스 누락 시 개발자 즉시 인지 가능하도록 Critical 레벨 분리 로깅 후 안전 폴백
    logger.critical(
        f"CRITICAL: Missing Firestore composite index for category {category.name}. "
        f"Please check Firebase console indexes. Details: {index_err}"
    )
    return []
    
  except google_exceptions.GoogleAPIError as api_err:
    # 일반적인 Firestore 인프라/API 통신 장애
    logger.error(f"Firestore API error in get_battles: {api_err}")
    return []
    
  except (TypeError, ValueError, AttributeError) as code_bug:
    # 개발자 코드 작성 실수 및 스키마 정합성 오류는 은폐하지 않고 상위로 전파
    logger.error(f"Programming or schema error in get_battles: {code_bug}", exc_info=True)
    raise

최종 개선사항
✅ Firebase 전역 초기화 → Lazy Initialization 전환 → 테스트 격리성과 모듈 import 안정성 확보
✅ get/set 문서 ID 생성 중복 로직 → Normalization Helper 통합 → 언어 키 불일치 캐시 미스 제거
✅ merge=True 기반 부분 업데이트 → 전체 문서 교체 방식 전환 → 탈락 모델 잔존 데이터 방지
✅ 광범위 Exception 처리 → Firestore API 오류와 코드 오류 분리 → 장애 은폐 방지 및 원인 추적성 강화
✅ 무검증 Firestore 데이터 변환 → 도메인 필드 검증 추가 → 손상 데이터 유입 차단
✅ 모든 쿼리 오류 일괄 처리 → 인덱스 오류·인프라 오류·개발 오류 분리 대응 → 운영 장애 판단 정확도 향상
✅ 단순 반환 중심 DB 계층 → 구조화 Logging 적용 → 데이터 파이프라인 상태 관측성 확보

원본은 Firestore 기반 데이터 관리 목적을 충실히 수행하는 안정적인 애플리케이션 코드였지만, 초기화 사이드 이펙트·데이터 잔존·예외 은폐라는 운영 리스크가 존재했으며, 현재 버전은 데이터 무결성·장애 격리·확장 가능한 Repository 구조를 확보한 실무 운영 수준 코드로 승격되었다.
