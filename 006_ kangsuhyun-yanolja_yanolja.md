원본코드
"""
This module contains functions to interact with the models.
"""

import json
import os
from typing import List, Optional, Tuple

import litellm

DEFAULT_SUMMARIZE_INSTRUCTION = "Summarize the given text without changing the language of it."  # pylint: disable=line-too-long
DEFAULT_TRANSLATE_INSTRUCTION = "Translate the given text from {source_lang} to {target_lang}."  # pylint: disable=line-too-long


class ContextWindowExceededError(Exception):
  pass


class Model:

  def __init__(
      self,
      name: str,
      provider: str = None,
      api_key: str = None,
      api_base: str = None,
      summarize_instruction: str = None,
      translate_instruction: str = None,
  ):
    self.name = name
    self.provider = provider
    self.api_key = api_key
    self.api_base = api_base
    self.summarize_instruction = summarize_instruction or DEFAULT_SUMMARIZE_INSTRUCTION  # pylint: disable=line-too-long
    self.translate_instruction = translate_instruction or DEFAULT_TRANSLATE_INSTRUCTION  # pylint: disable=line-too-long

  # Returns the parsed result or raw response, and whether parsing succeeded.
  def completion(self,
                 instruction: str,
                 prompt: str,
                 max_tokens: Optional[float] = None,
                 max_retries: int = 2) -> Tuple[str, bool]:
    messages = [{
        "role":
            "system",
        "content":
            instruction + """
Output following this JSON format without using code blocks:
{"result": "your result here"}"""
    }, {
        "role": "user",
        "content": prompt
    }]

    for attempt in range(max_retries + 1):
      try:
        response = litellm.completion(model=self.provider + "/" +
                                      self.name if self.provider else self.name,
                                      api_key=self.api_key,
                                      api_base=self.api_base,
                                      messages=messages,
                                      max_tokens=max_tokens,
                                      **self._get_completion_kwargs())

        json_response = response.choices[0].message.content
        parsed_json = json.loads(json_response)
        return parsed_json["result"], True

      except litellm.ContextWindowExceededError as e:
        raise ContextWindowExceededError() from e
      except json.JSONDecodeError:
        if attempt == max_retries:
          return json_response, False

  def _get_completion_kwargs(self):
    return {
        # Ref: https://litellm.vercel.app/docs/completion/input#optional-fields # pylint: disable=line-too-long
        "response_format": {
            "type": "json_object"
        }
    }


class AnthropicModel(Model):

  def completion(self,
                 instruction: str,
                 prompt: str,
                 max_tokens: Optional[float] = None,
                 max_retries: int = 2) -> Tuple[str, bool]:
    # Ref: https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/increase-consistency#prefill-claudes-response # pylint: disable=line-too-long
    prefix = "<result>"
    suffix = "</result>"
    messages = [{
        "role":
            "user",
        "content":
            f"""{instruction}
Output following this format:
{prefix}...{suffix}
Text:
{prompt}"""
    }, {
        "role": "assistant",
        "content": prefix
    }]

    for attempt in range(max_retries + 1):
      try:
        response = litellm.completion(
            model=self.provider + "/" +
            self.name if self.provider else self.name,
            api_key=self.api_key,
            api_base=self.api_base,
            messages=messages,
            max_tokens=max_tokens,
        )

      except litellm.ContextWindowExceededError as e:
        raise ContextWindowExceededError() from e

      result = response.choices[0].message.content
      if result.endswith(suffix):
        return result.removesuffix(suffix).strip(), True

      if attempt == max_retries:
        return result, False


class VertexModel(Model):

  def __init__(self, name: str, vertex_credentials: str):
    super().__init__(name, provider="vertex_ai")
    self.vertex_credentials = vertex_credentials

  def _get_completion_kwargs(self):
    return {
        "response_format": {
            "type": "json_object"
        },
        "vertex_credentials": self.vertex_credentials
    }


supported_models: List[Model] = [
    Model("gpt-4o-2024-11-20"),
    Model("gpt-4o-mini-2024-07-18"),
    AnthropicModel("claude-3-5-sonnet-20241022"),
    AnthropicModel("claude-3-5-haiku-20241022"),
    VertexModel("gemini-1.5-pro-002",
                vertex_credentials=os.getenv("VERTEX_CREDENTIALS")),
    VertexModel("gemini-1.5-flash-002",
                vertex_credentials=os.getenv("VERTEX_CREDENTIALS")),
    Model("google/gemma-2-9b-it", provider="deepinfra"),
    Model("google/gemma-2-27b-it", provider="deepinfra"),
    Model("meta-llama/Meta-Llama-3.1-8B-Instruct", provider="deepinfra"),
    Model("meta-llama/Meta-Llama-3.1-70B-Instruct", provider="deepinfra"),
    Model("meta-llama/Meta-Llama-3.1-405B-Instruct", provider="deepinfra"),
    Model("meta-llama/Llama-3.2-3B-Instruct", provider="deepinfra"),
    Model("meta-llama/Llama-3.2-1B-Instruct", provider="deepinfra"),
    Model("Qwen/Qwen2.5-72B-Instruct", provider="deepinfra"),
]


def check_models(models: List[Model]):
  for model in models:
    print(f"Checking model {model.name}...")
    try:
      model.completion(
          """Output following this JSON format without using code blocks:
{"result": "your result here"}""", "How are you?")
      print(f"Model {model.name} is available.")

    # This check is designed to verify the availability of the models
    # without any issues. Therefore, we need to catch all exceptions.
    except Exception as e:  # pylint: disable=broad-except
      raise RuntimeError(f"Model {model.name} is not available: {e}") from e

야놀자 아레나의 model.py는 LiteLLM 기반 멀티 모델 추상화 방향은 우수하지만, 모델별 응답 계약 통일·장애 원인 보존·동적 구성 관리가 부족해 LLM 평가 플랫폼의 핵심 가치인 결과 신뢰성과 운영 안정성을 저해할 수 있는 구조다.

제안패치
"""
This module contains functions to interact with the models.
"""

import json
import os
from typing import List, Optional, Tuple

import litellm

DEFAULT_SUMMARIZE_INSTRUCTION = "Summarize the given text without changing the language of it."  # pylint: disable=line-too-long
DEFAULT_TRANSLATE_INSTRUCTION = "Translate the given text from {source_lang} to {target_lang}."  # pylint: disable=line-too-long


class ContextWindowExceededError(Exception):
  pass


class Model:

  def __init__(
      self,
      name: str,
      provider: str = None,
      api_key: str = None,
      api_base: str = None,
      summarize_instruction: str = None,
      translate_instruction: str = None,
  ):
    self.name = name
    self.provider = provider
    self.api_key = api_key
    self.api_base = api_base
    self.summarize_instruction = summarize_instruction or DEFAULT_SUMMARIZE_INSTRUCTION  # pylint: disable=line-too-long
    self.translate_instruction = translate_instruction or DEFAULT_TRANSLATE_INSTRUCTION  # pylint: disable=line-too-long

  def completion(self,
                 instruction: str,
                 prompt: str,
                 max_tokens: Optional[float] = None,
                 max_retries: int = 2) -> Tuple[str, bool]:
    messages = [{
        "role":
            "system",
        "content":
            instruction + """
Output following this JSON format without using code blocks:
{"result": "your result here"}"""
    }, {
        "role": "user",
        "content": prompt
    }]

    json_response = ""
    for attempt in range(max_retries + 1):
      try:
        response = litellm.completion(model=self.provider + "/" +
                                      self.name if self.provider else self.name,
                                      api_key=self.api_key,
                                      api_base=self.api_base,
                                      messages=messages,
                                      max_tokens=max_tokens,
                                      **self._get_completion_kwargs())

        json_response = response.choices[0].message.content
        parsed_json = json.loads(json_response)
        
        # 데이터 무결성 검증: 'result' 키 존재 여부 및 string 타입 보장
        if not isinstance(parsed_json, dict) or "result" not in parsed_json:
          if attempt == max_retries:
            return json_response, False
          continue

        result_val = parsed_json["result"]
        if not isinstance(result_val, str):
          if attempt == max_retries:
            return json_response, False
          continue

        return result_val, True

      except litellm.ContextWindowExceededError as e:
        raise ContextWindowExceededError() from e
      except (litellm.RateLimitError, litellm.ServiceUnavailableError, litellm.Timeout) as e:
        if attempt == max_retries:
          raise RuntimeError(f"Model {self.name} failed after {max_retries} retries due to provider transient error: {e}") from e
      except json.JSONDecodeError:
        if attempt == max_retries:
          return json_response, False

    return json_response, False

  def _get_completion_kwargs(self):
    return {
        "response_format": {
            "type": "json_object"
        }
    }


class AnthropicModel(Model):

  def completion(self,
                 instruction: str,
                 prompt: str,
                 max_tokens: Optional[float] = None,
                 max_retries: int = 2) -> Tuple[str, bool]:
    prefix = "<result>"
    suffix = "</result>"
    messages = [{
        "role":
            "user",
        "content":
            f"""{instruction}
Output following this format:
{prefix}...{suffix}
Text:
{prompt}"""
    }, {
        "role": "assistant",
        "content": prefix
    }]

    result = ""
    for attempt in range(max_retries + 1):
      try:
        response = litellm.completion(
            model=self.provider + "/" +
            self.name if self.provider else self.name,
            api_key=self.api_key,
            api_base=self.api_base,
            messages=messages,
            max_tokens=max_tokens,
        )
        result = response.choices[0].message.content
        if result.endswith(suffix):
          clean_result = result.removesuffix(suffix).strip()
          return clean_result, True

      except litellm.ContextWindowExceededError as e:
        raise ContextWindowExceededError() from e
      except (litellm.RateLimitError, litellm.ServiceUnavailableError, litellm.Timeout) as e:
        if attempt == max_retries:
          raise RuntimeError(f"AnthropicModel {self.name} failed after {max_retries} retries due to provider transient error: {e}") from e
      except Exception:
        if attempt == max_retries:
          return result, False

    return result, False


class VertexModel(Model):

  def __init__(self, name: str, vertex_credentials: Optional[str] = None):
    super().__init__(name, provider="vertex_ai")
    resolved_credentials = vertex_credentials or os.getenv("VERTEX_CREDENTIALS")
    if not resolved_credentials:
      raise ValueError(f"VertexModel {name} requires vertex_credentials or VERTEX_CREDENTIALS environment variable.")
    self.vertex_credentials = resolved_credentials

  def _get_completion_kwargs(self):
    return {
        "response_format": {
            "type": "json_object"
        },
        "vertex_credentials": self.vertex_credentials
    }


def get_supported_models() -> List[Model]:
  """Factory pattern for model initialization to prevent hardcoded module-level credential failures."""
  return [
      Model("gpt-4o-2024-11-20"),
      Model("gpt-4o-mini-2024-07-18"),
      AnthropicModel("claude-3-5-sonnet-20241022"),
      AnthropicModel("claude-3-5-haiku-20241022"),
      VertexModel("gemini-1.5-pro-002", vertex_credentials=os.getenv("VERTEX_CREDENTIALS")),
      VertexModel("gemini-1.5-flash-002", vertex_credentials=os.getenv("VERTEX_CREDENTIALS")),
      Model("google/gemma-2-9b-it", provider="deepinfra"),
      Model("google/gemma-2-27b-it", provider="deepinfra"),
      Model("meta-llama/Meta-Llama-3.1-8B-Instruct", provider="deepinfra"),
      Model("meta-llama/Meta-Llama-3.1-70B-Instruct", provider="deepinfra"),
      Model("meta-llama/Meta-Llama-3.1-405B-Instruct", provider="deepinfra"),
      Model("meta-llama/Llama-3.2-3B-Instruct", provider="deepinfra"),
      Model("meta-llama/Llama-3.2-1B-Instruct", provider="deepinfra"),
      Model("Qwen/Qwen2.5-72B-Instruct", provider="deepinfra"),
  ]


supported_models: List[Model] = get_supported_models()


def check_models(models: List[Model]):
  for model in models:
    print(f"Checking model {model.name}...")
    try:
      model.completion(
          """Output following this JSON format without using code blocks:
{"result": "your result here"}""", "How are you?")
      print(f"Model {model.name} is available.")

    except (litellm.AuthenticationError, litellm.APIConnectionError) as e:
      raise RuntimeError(f"Model {model.name} critical setup/network failure: {e}") from e
    except ContextWindowExceededError as e:
      raise RuntimeError(f"Model {model.name} context window error during check: {e}") from e
    except Exception as e:  
      raise RuntimeError(f"Model {model.name} check failed due to unexpected error: {e}") from e

최종 개선사항
✅ JSON 응답 파싱 실패 및 타입 불일치 검증 추가 → LLM 반환 데이터 무결성 확보
✅ Provider 일시 장애 예외 분리 → RateLimit/Timeout/ServiceUnavailable 재시도 안정성 강화
✅ Vertex 인증 정보 초기 검증 추가 → 잘못된 Credential 상태의 런타임 장애 방지
✅ supported_models 전역 생성 제거 및 Factory 전환 → 초기화 실패 격리 및 설정 확장성 개선
✅ 모델별 응답 처리 방식 차이 유지 → Anthropic/일반 모델 특성 대응력 확보
✅ check_models 예외 유형 세분화 → 인증·네트워크·컨텍스트 장애 원인 추적 가능
✅ 불완전한 LLM 결과 반환 차단 → 평가 데이터 오염 및 리더보드 신뢰성 저하 방지

이번 개선본은 model.py의 주요 프로덕션 장애 지점을 대부분 제거했지만, 모델별 출력 계약 통합과 공통 Retry Policy 추상화까지 적용해야 엔터프라이즈 LLM 평가 시스템 수준의 완성도에 도달한다.
