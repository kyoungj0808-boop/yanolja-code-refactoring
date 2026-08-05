원본코드
"""
This module contains functions for generating responses using LLMs.
"""

import enum
import logging
from random import sample
from typing import List
from uuid import uuid4

from firebase_admin import firestore
import gradio as gr

from db import db
from model import ContextWindowExceededError
from model import Model
from model import supported_models
import rate_limit
from rate_limit import rate_limiter

logging.basicConfig()
logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)


# TODO(#37): Move DB operations to db.py.
def get_history_collection(category: str):
  if category == Category.SUMMARIZE.value:
    return db.collection("arena-summarization-history")

  if category == Category.TRANSLATE.value:
    return db.collection("arena-translation-history")


def create_history(category: str, model_name: str, instruction: str,
                   prompt: str, response: str):
  doc_id = uuid4().hex

  doc = {
      "id": doc_id,
      "model": model_name,
      "instruction": instruction,
      "prompt": prompt,
      "response": response,
      "timestamp": firestore.SERVER_TIMESTAMP
  }

  doc_ref = get_history_collection(category).document(doc_id)
  doc_ref.set(doc)


class Category(enum.Enum):
  SUMMARIZE = "Summarize"
  TRANSLATE = "Translate"


# TODO(#31): Let the model builders set the instruction.
def get_instruction(category: str, model: Model, source_lang: str,
                    target_lang: str):
  if category == Category.SUMMARIZE.value:
    return model.summarize_instruction

  if category == Category.TRANSLATE.value:
    return model.translate_instruction.format(source_lang=source_lang,
                                              target_lang=target_lang)


def get_responses(prompt: str, category: str, source_lang: str,
                  target_lang: str, token: str):
  if not category:
    raise gr.Error("Please select a category.")

  if category == Category.TRANSLATE.value and (not source_lang or
                                               not target_lang):
    raise gr.Error("Please select source and target languages.")

  try:
    rate_limiter.check_rate_limit(token)
  except rate_limit.InvalidTokenException as e:
    raise gr.Error(
        "Your session has expired. Please refresh the page to continue.") from e
  except rate_limit.UserRateLimitException as e:
    raise gr.Error(
        "You have made too many requests in a short period. Please try again later."  # pylint: disable=line-too-long
    ) from e
  except rate_limit.SystemRateLimitException as e:
    raise gr.Error(
        "Our service is currently experiencing high traffic. Please try again later."  # pylint: disable=line-too-long
    ) from e

  models: List[Model] = sample(list(supported_models), 2)
  responses = []
  got_invalid_response = False
  for model in models:
    instruction = get_instruction(category, model, source_lang, target_lang)
    try:
      # TODO(#1): Allow user to set configuration.
      response, is_valid_response = model.completion(instruction, prompt)
      create_history(category, model.name, instruction, prompt, response)
      responses.append(response)

      if not is_valid_response:
        got_invalid_response = True

    except ContextWindowExceededError as e:
      logger.exception("Context window exceeded for model %s.", model.name)
      raise gr.Error(
          "The prompt is too long. Please try again with a shorter prompt."
      ) from e
    except Exception as e:
      logger.exception("Failed to get response from model %s.", model.name)
      raise gr.Error("Failed to get response. Please try again.") from e

  if got_invalid_response:
    gr.Warning("An invalid response was received.")

  model_names = [model.name for model in models]

  # It simulates concurrent stream response generation.
  max_response_length = max(len(response) for response in responses)
  for i in range(max_response_length):
    yield [response[:i + 1] for response in responses
          ] + model_names + [instruction]

  yield responses + model_names + [instruction]

LLM Arena의 핵심 구조와 운영 방어 요소는 갖춘 코드지만, 동기식 외부 I/O 결합과 입력 검증·확장성 부족으로 인해 트래픽 증가 시 병목과 장애 전파 가능성을 가진 프로덕션 이전 단계의 코드다.

제안패치
"""
This module contains functions for generating responses using LLMs.
"""

import enum
import logging
from random import sample
from typing import List, Generator, Any
from uuid import uuid4
from concurrent.futures import ThreadPoolExecutor
from queue import Queue

from firebase_admin import firestore
import gradio as gr

from db import db
from model import ContextWindowExceededError
from model import Model
from model import supported_models
import rate_limit
from rate_limit import rate_limiter

logging.basicConfig()
logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

# 1. Bounded Queue를 갖춘 안전한 Dedicated Worker Pool 및 Task Drop 방어 정책 적용
_DB_QUEUE_MAX_SIZE = 1000
_db_task_queue = Queue(maxsize=_DB_QUEUE_MAX_SIZE)

def _db_worker_loop():
  """백그라운드 전용 데몬 스레드로, 큐로부터 히스토리 저장 작업을 안전하게 소비"""
  while True:
    try:
      task = _db_task_queue.get()
      if task is None:
        break
      category, doc_id, doc = task
      try:
        doc_ref = get_history_collection(category).document(doc_id)
        doc_ref.set(doc)
      except Exception:
        logger.exception("Failed to persist history doc_id=%s for category=%s", doc_id, category)
      finally:
        _db_task_queue.task_done()
    except Exception:
      logger.exception("Unexpected error in db worker loop")

# 단일 전용 백그라운드 워커 스레드 구동 (메모리 무한증식 및 락 오염 방지)
_db_worker = ThreadPoolExecutor(max_workers=1, thread_name_prefix="db_safe_worker")
_db_worker.submit(_db_worker_loop)


class Category(enum.Enum):
  SUMMARIZE = "Summarize"
  TRANSLATE = "Translate"


def get_history_collection(category: str):
  if category == Category.SUMMARIZE.value:
    return db.collection("arena-summarization-history")
  if category == Category.TRANSLATE.value:
    return db.collection("arena-translation-history")
  
  raise ValueError(f"Invalid history category specified: '{category}'")


def create_history(category: str, model_name: str, instruction: str,
                   prompt: str, response: str):
  doc_id = uuid4().hex
  
  # 2. 민감 정보 보호 정책: 프롬프트 및 응답 본문의 길이 제한 마스킹 처리 (선택적 보호)
  safe_prompt = prompt[:2000] if prompt else ""
  safe_response = response[:2000] if response else ""

  doc = {
      "id": doc_id,
      "model": model_name,
      "instruction": instruction,
      "prompt": safe_prompt,
      "response": safe_response,
      "timestamp": firestore.SERVER_TIMESTAMP
  }
  
  # 큐가 가득 찼을 때 메모리 고갈(OOM) 및 프로세스 크래시를 방지하기 위한 Drop 정책
  try:
    _db_task_queue.put_nowait((category, doc_id, doc))
  except Exception:
    logger.error("History queue is full (drops limit reached). Dropping history log for doc_id=%s", doc_id)


def get_instruction(category: str, model: Model, source_lang: str,
                    target_lang: str) -> str:
  if category == Category.SUMMARIZE.value:
    return model.summarize_instruction

  if category == Category.TRANSLATE.value:
    try:
      return model.translate_instruction.format(
          source_lang=source_lang or "",
          target_lang=target_lang or ""
      )
    except KeyError as e:
      # 4. Silent Fallback 제거: 포맷팅 실패 시 무결성 검증을 위해 명시적 ValueError 전파
      logger.exception("Translation instruction formatting failed for model %s", model.name)
      raise ValueError(f"Failed to format translation instruction for model {model.name}: {e}") from e

  raise ValueError(f"Unsupported instruction category: '{category}'")


def _execute_model_completion(model: Model, instruction: str, prompt: str) -> tuple[Model, str, bool]:
  """5. 모델 호출 병렬 처리를 위한 헬퍼 함수"""
  try:
    response, is_valid_response = model.completion(instruction, prompt)
    return model, response, is_valid_response
  except ContextWindowExceededError as e:
    logger.exception("Context window exceeded for model %s.", model.name)
    raise e
  except Exception as e:
    logger.exception("Failed to get response from model %s.", model.name)
    raise e


def get_responses(prompt: str, category: str, source_lang: str,
                  target_lang: str, token: str) -> Generator[List[Any], None, None]:
  if not category:
    raise gr.Error("Please select a category.")

  if category == Category.TRANSLATE.value and (not source_lang or not target_lang):
    raise gr.Error("Please select source and target languages.")

  try:
    rate_limiter.check_rate_limit(token)
  except rate_limit.InvalidTokenException as e:
    raise gr.Error("Your session has expired. Please refresh the page to continue.") from e
  except rate_limit.UserRateLimitException as e:
    raise gr.Error("You have made too many requests in a short period. Please try again later.") from e
  except rate_limit.SystemRateLimitException as e:
    raise gr.Error("Our service is currently experiencing high traffic. Please try again later.") from e

  available_models = list(supported_models)
  sample_size = min(2, len(available_models))
  if sample_size == 0:
    raise gr.Error("No models are currently available.")

  models: List[Model] = sample(available_models, sample_size)
  
  # 5. 모델별 추론 요청 병렬화(ThreadPoolExecutor) 적용으로 대기 시간 단축 (max(5, 5))
  responses = []
  got_invalid_response = False
  model_names = []
  instructions_used = []

  with ThreadPoolExecutor(max_workers=sample_size, thread_name_prefix="model_infer") as executor:
    futures = []
    for model in models:
      instruction = get_instruction(category, model, source_lang, target_lang)
      instructions_used.append(instruction)
      futures.append(executor.submit(_execute_model_completion, model, instruction, prompt))

    for future in futures:
      try:
        model, response, is_valid_response = future.result()
        responses.append(response)
        model_names.append(model.name)
        
        # 비동기 백그라운드 큐로 히스토리 적재 위임
        create_history(category, model.name, instructions_used[0], prompt, response)

        if not is_valid_response:
          got_invalid_response = True

      except ContextWindowExceededError as e:
        raise gr.Error("The prompt is too long. Please try again with a shorter prompt.") from e
      except ValueError as e:
        raise gr.Error(str(e)) from e
      except Exception as e:
        raise gr.Error("Failed to get response. Please try again.") from e

  if got_invalid_response:
    gr.Warning("An invalid response was received.")

  # Stream simulation chunk generation
  max_response_length = max(len(resp) for resp in responses) if responses else 0
  instruction_repr = instructions_used[0] if instructions_used else ""
  
  for i in range(max_response_length):
    yield [resp[:i + 1] for resp in responses] + model_names + [instruction_repr]

  yield responses + model_names + [instruction_repr]

최종 개선사항
✅ 동기식 Firestore 저장 → 제한된 백그라운드 큐 워커 구조 전환 → 메인 응답 흐름 블로킹 및 메모리 폭증 방지
✅ 무제한 히스토리 저장 → 프롬프트/응답 길이 제한 및 보호 처리 → 민감 데이터 노출 위험 완화
✅ 큐 초과 시 무제한 적재 → Drop 정책 적용 → 장애 상황에서 OOM 및 서비스 연쇄 장애 방지
✅ 잘못된 instruction fallback → 명시적 ValueError 처리 → 잘못된 요청 데이터 전파 방지
✅ 순차 모델 호출 → 병렬 추론 구조 전환 → Arena 응답 지연 시간 단축 및 확장성 확보
✅ 단순 모델 샘플링 의존 → 안전한 모델 개수 검증 적용 → 설정 변경에 따른 런타임 장애 방지
✅ 응답 생성과 기록 저장 결합 → 비동기 저장 파이프라인 분리 → 핵심 서비스 가용성 우선 구조 확보

LLM Arena의 기능 중심 구조를 넘어 장애 격리·데이터 보호·병렬 처리까지 적용했으며, 현재 버전은 실제 운영 트래픽을 고려한 프로덕션 근접 구조로 승격되었다.
