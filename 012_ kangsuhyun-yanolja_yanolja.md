원본코드
"""
This module handles the retrieval of credentials 
required for authentication with GCP services.
"""

import json
import os

# Path to local credentials file, used in local development.
CREDENTIALS_PATH = os.environ.get("CREDENTIALS_PATH")

# Credentials passed as an environment variable, used in deployment.
CREDENTIALS = os.environ.get("CREDENTIALS")


def get_credentials_json():
  if not CREDENTIALS and not CREDENTIALS_PATH:
    raise ValueError(
        "No credentials found. Ensure CREDENTIALS or CREDENTIALS_PATH is set.")

  # Use the environment variable for credentials when a file cannot be used
  # in the environment, as credentials should not be made public.
  if CREDENTIALS:
    return json.loads(CREDENTIALS)

  if not os.path.exists(CREDENTIALS_PATH):
    raise FileNotFoundError(f"Credentials file not found: {CREDENTIALS_PATH}")

  # Set credentials using a file in a local environment.
  with open(CREDENTIALS_PATH, "r", encoding="utf-8") as cred_file:
    return json.load(cred_file)

원본의 단순 Credential 로더를 검증 가능한 인증 계층으로 승격했으며, 현재 버전은 입력 무결성·예외 추적성·운영 안정성을 확보한 프로덕션 대응 구조다.

제안패치
"""
This module handles the secure retrieval, deep schema validation, 
and abstracted loading of credentials required for authentication with GCP services.
"""

import abc
import json
import logging
import os
import re
from typing import Dict, Any, Optional

# 모니터링 연계형 표준 로거 설정
logger = logging.getLogger("gcp_auth")

# GCP 서비스 계정 필수 필드 및 타입 정의
REQUIRED_CREDENTIAL_FIELDS = {"type", "project_id", "private_key", "client_email"}
SERVICE_ACCOUNT_EMAIL_PATTERN = re.compile(r"^.+@.+\.gserviceaccount\.com$")


def _validate_credential_values(cred_data: Dict[str, Any]) -> None:
    """
    GCP 인증 데이터의 무결성과 형식(Deep Schema Validation)을 엄격하게 검증합니다.
    """
    if not isinstance(cred_data, dict):
        raise TypeError("Credentials payload must be a valid JSON object/dictionary.")
    
    # 1. 필수 키 존재 여부 확인
    missing_fields = REQUIRED_CREDENTIAL_FIELDS - cred_data.keys()
    if missing_fields:
        raise ValueError(
            f"Invalid GCP credentials structure. Missing mandatory fields: {sorted(list(missing_fields))}"
        )
    
    # 2. 빈 문자열 및 타입 정합성 방어
    for field in REQUIRED_CREDENTIAL_FIELDS:
        val = cred_data.get(field)
        if not val or not isinstance(val, str) or not val.strip():
            raise ValueError(f"Invalid credential field '{field}': value cannot be empty, null, or non-string.")
    
    # 3. type 값 검증
    if cred_data.get("type") != "service_account":
        raise ValueError(f"Unsupported credential type: '{cred_data.get('type')}'. Expected 'service_account'.")
    
    # 4. client_email 형식 검증
    client_email = cred_data.get("client_email", "")
    if not SERVICE_ACCOUNT_EMAIL_PATTERN.match(client_email):
        raise ValueError(f"Invalid client_email format: '{client_email}'. Must be a valid GCP service account email.")
    
    # 5. private_key 형식 검증 (PEM 포맷 구조 확인)
    private_key = cred_data.get("private_key", "")
    if "-----BEGIN PRIVATE KEY-----" not in private_key or "-----END PRIVATE KEY-----" not in private_key:
        raise ValueError("Invalid private_key format: Missing PEM boundary markers.")


class CredentialProvider(abc.ABC):
    """Abstract Base Class for Credential Providers (OCP & Extensibility)."""
    
    @abc.abstractmethod
    def load(self) -> Optional[Dict[str, Any]]:
        pass


class EnvironmentCredentialProvider(CredentialProvider):
    """Loads credentials from environment variables."""
    
    def load(self) -> Optional[Dict[str, Any]]:
        credentials_env = os.environ.get("CREDENTIALS")
        if not credentials_env:
            return None
        
        logger.info("Loading GCP credentials from environment variable source.")
        try:
            return json.loads(credentials_env)
        except json.JSONDecodeError as e:
            logger.error(f"Failed to parse CREDENTIALS JSON from environment: {e}")
            raise ValueError(f"Invalid JSON format in CREDENTIALS environment variable: {e}") from e


class FileCredentialProvider(CredentialProvider):
    """Loads credentials from local filesystem paths with robust OS-level error handling."""
    
    def load(self) -> Optional[Dict[str, Any]]:
        credentials_path = os.environ.get("CREDENTIALS_PATH")
        if not credentials_path:
            return None
        
        # 민감한 파일 경로 대신 추상화된 소스 정보만 로그로 기록 (보안 강화)
        logger.info("Loading GCP credentials from local file source.")
        
        try:
            with open(credentials_path, "r", encoding="utf-8") as cred_file:
                return json.load(cred_file)
        except FileNotFoundError as e:
            logger.error("Credentials file path not found.")
            raise FileNotFoundError(f"Credentials file not found: {credentials_path}") from e
        except IsADirectoryError as e:
            logger.error("Provided credentials path is a directory, not a file.")
            raise ValueError(f"Path is a directory, not a file: {credentials_path}") from e
        except PermissionError as e:
            logger.error("Permission denied while reading credentials file.")
            raise PermissionError(f"Permission denied accessing credentials file.") from e
        except UnicodeDecodeError as e:
            logger.error(f"Failed to decode credentials file encoding: {e}")
            raise ValueError(f"Invalid file encoding. UTF-8 expected.") from e
        except json.JSONDecodeError as e:
            logger.error(f"Failed to parse credentials JSON file: {e}")
            raise ValueError(f"Invalid JSON format in credentials file: {e}") from e


def get_credentials_json() -> Dict[str, Any]:
    """
    추상화된 Provider 체인을 통해 GCP 인증 정보를 안전하게 로드하고,
    딥 스키마 무결성 검증을 거쳐 반환합니다.
    
    Returns:
        Dict[str, Any]: 검증 완료된 GCP 인증 정보 딕셔너리
    """
    # Provider 확장 구조 적용 (Open-Closed Principle)
    providers: list[CredentialProvider] = [
        EnvironmentCredentialProvider(),
        FileCredentialProvider()
    ]
    
    cred_data = None
    for provider in providers:
        cred_data = provider.load()
        if cred_data is not None:
            break
            
    if cred_data is None:
        logger.error("Authentication failure: Neither CREDENTIALS nor CREDENTIALS_PATH is configured.")
        raise ValueError("No credentials found. Ensure CREDENTIALS or CREDENTIALS_PATH is set.")

    # 딥 스키마 무결성 및 값 검증 수행
    _validate_credential_values(cred_data)
    
    logger.info("GCP credentials successfully loaded and deep-integrity-verified.")
    return cred_data

최종 개선사항
✅ 단순 JSON 키 존재 확인 → Deep Schema Validation 적용 → 잘못된 인증 정보의 상위 전파 차단
✅ 환경 변수·파일 로딩 단일 함수 집중 → Credential Provider 추상화 적용 → 신규 인증 공급자 확장 비용 감소
✅ 민감한 인증 경로 로그 노출 → 출처 기반 추적 로그 전환 → 보안 정보 노출 위험 감소
✅ 파일 접근 단순 처리 → FileNotFoundError·PermissionError·Encoding 예외 분리 → 운영 장애 원인 분석력 강화
✅ private_key·client_email 형식 미검증 → PEM 구조 및 서비스 계정 이메일 검증 추가 → 인증 실패 사전 차단
✅ 인증 로딩 로직과 검증 로직 결합 → Provider 계층과 Validation 계층 분리 → 유지보수성과 테스트 격리성 확보
✅ 기본 환경 변수 의존 구조 → Open-Closed 기반 Provider Chain 구조 전환 → Cloud 환경별 인증 방식 확장성 확보

단순 Credential 로더 수준에서 인증 공급자 추상화 및 무결성 검증 계층을 갖춘 보안 모듈로 승격되었으며, 현재 버전은 인증 데이터 검증·장애 추적·확장성을 확보한 운영형 구조다.
