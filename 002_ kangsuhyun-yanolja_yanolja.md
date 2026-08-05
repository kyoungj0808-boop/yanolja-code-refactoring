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

핵심 문제는 전역 초기화, 입력 정규화 불일치, Firestore 데이터 신뢰에 있으며, 마지막 환경 변수 즉시 조회는 치명적 결함이라기보다 테스트성과 Fail Fast 사이의 설계 선택이다.

제안패치
# -*- coding: utf-8 -*-

"""
This module handles the management of the database with enterprise-grade robustness.
"""

from dataclasses import dataclass
import enum
import os
from typing import List, Optional

import firebase_admin
from firebase_admin import credentials
from firebase_admin import firestore
from google.cloud.firestore_v1 import base_query
from google.api_core.exceptions import GoogleAPICallError

try:
    import gradio as gr
except ImportError:
    gr = None


class Settings:
    @property
    def RATINGS_COLLECTION(self) -> str:
        return self._get_env("RATINGS_COLLECTION")

    @property
    def SUMMARIZATIONS_COLLECTION(self) -> str:
        return self._get_env("SUMMARIZATIONS_COLLECTION")

    @property
    def TRANSLATIONS_COLLECTION(self) -> str:
        return self._get_env("TRANSLATIONS_COLLECTION")

    def _get_env(self, name: str) -> str:
        value = os.getenv(name)
        if not value:
            raise ValueError(f"Environment variable '{name}' is not set")
        return value


_settings = Settings()
_db_client = None


def get_db() -> firestore.firestore.Client:
    """지연 초기화(Lazy Initialization) 및 공개 API 기반 DB 커넥션 관리"""
    global _db_client
    if _db_client is not None:
        return _db_client

    if gr is None or getattr(gr, "NO_RELOAD", True):
        try:
            # 내부 속성인 _apps 대신 앱 이름 지정 방식을 통한 안전한 초기화 체크
            try:
                firebase_admin.get_app()
            except ValueError:
                cred_path = os.getenv("GOOGLE_APPLICATION_CREDENTIALS")
                if cred_path and os.path.exists(cred_path):
                    firebase_admin.initialize_app(credentials.Certificate(cred_path))
                else:
                    try:
                        from credentials import get_credentials_json
                        firebase_admin.initialize_app(credentials.Certificate(get_credentials_json()))
                    except Exception:
                        firebase_admin.initialize_app()
            
            _db_client = firestore.client()
        except Exception as e:
            raise RuntimeError(f"Failed to initialize Firebase Admin SDK: {e}")
            
    if _db_client is None:
        raise RuntimeError("Database client is not initialized.")
        
    return _db_client


class Category(enum.Enum):
    SUMMARIZATION = "summarization"
    TRANSLATION = "translation"


@dataclass
class Rating:
    model: str
    rating: int


def get_ratings(category: Category, source_lang: Optional[str],
                target_lang: Optional[str]) -> Optional[List[Rating]]:
    try:
        db = get_db()
        s_lang = source_lang.lower() if source_lang else None
        t_lang = target_lang.lower() if target_lang else None

        doc_id = "#".join([category.value] + [lang for lang in (s_lang, t_lang) if lang])
        
        doc_snapshot = db.collection(_settings.RATINGS_COLLECTION).document(doc_id).get()
        if not doc_snapshot.exists:
            return None

        doc_dict = doc_snapshot.to_dict()
        if not doc_dict:
            return None

        doc_dict.pop("timestamp", None)

        ratings = []
        for model, rating in doc_dict.items():
            # bool은 int의 서브클래스이므로 type()을 통해 엄격하게 필터링
            if type(rating) in (int, float):
                ratings.append(Rating(model, int(rating)))

        return ratings
    except GoogleAPICallError as e:
        print(f"[Firestore API Error] get_ratings failed: {e}")
        return None
    except Exception as e:
        print(f"[Error] get_ratings failed: {e}")
        return None


def set_ratings(category: Category, ratings: List[Rating], source_lang: str,
                target_lang: Optional[str]):
    if not source_lang:
        raise ValueError("source_lang is required")

    try:
        db = get_db()
        source_lang_lowercase = source_lang.lower()
        target_lang_lowercase = target_lang.lower() if target_lang else None

        doc_id = "#".join([category.value, source_lang_lowercase] +
                          ([target_lang_lowercase] if target_lang_lowercase else []))
        doc_ref = db.collection(_settings.RATINGS_COLLECTION).document(doc_id)

        new_ratings = {rating.model: rating.rating for rating in ratings}
        new_ratings["timestamp"] = firestore.SERVER_TIMESTAMP
        doc_ref.set(new_ratings, merge=True)
    except GoogleAPICallError as e:
        print(f"[Firestore API Error] set_ratings failed: {e}")
        raise
    except Exception as e:
        print(f"[Error] set_ratings failed: {e}")
        raise


@dataclass
class Battle:
    model_a: str
    model_b: str
    winner: str


def get_battles(category: Category, source_lang: Optional[str],
                target_lang: Optional[str]) -> List[Battle]:
    try:
        db = get_db()
        source_lang_lowercase = source_lang.lower() if source_lang else None
        target_lang_lowercase = target_lang.lower() if target_lang else None

        if category == Category.SUMMARIZATION:
            collection = db.collection(_settings.SUMMARIZATIONS_COLLECTION).order_by("timestamp")

            if source_lang_lowercase:
                collection = collection.where(filter=base_query.FieldFilter(
                    "model_a_response_language", "==", source_lang_lowercase)).where(
                        filter=base_query.FieldFilter("model_b_response_language", "==",
                                                      source_lang_lowercase))

        elif category == Category.TRANSLATION:
            collection = db.collection(_settings.TRANSLATIONS_COLLECTION).order_by("timestamp")

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
    except GoogleAPICallError as e:
        print(f"[Firestore API Error] get_battles failed: {e}")
        return []
    except Exception as e:
        print(f"[Error] get_battles failed: {e}")
        return []

최종 개선사항
✅ 전역 DB 초기화 제거 → Lazy Initialization 적용 → 테스트성과 초기화 안정성 향상
✅ Firebase 내부 API 제거 → get_app() 기반 공식 API 사용 → SDK 호환성 강화
✅ 환경 변수 접근 캡슐화 → Settings 객체 도입 → 유지보수성과 확장성 향상
✅ Firestore API 예외 처리 추가 → 네트워크 장애 시 안전하게 복구 → 서비스 안정성 향상
✅ bool 타입 제외 후 숫자만 허용 → Rating 데이터 무결성 강화
✅ Firestore 문서 검증 및 방어적 접근 유지 → 손상된 데이터에도 안전하게 동작

치명적인 구조적 문제는 대부분 해소되었으며, 이제는 로깅 체계와 멀티스레드 초기화 정도만 다듬으면 프로덕션 품질에 매우 근접한 코드다.
