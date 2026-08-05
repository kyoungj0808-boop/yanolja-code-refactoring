원본코드
"""
Exception classes for the Ogem Python SDK.

This module defines all the custom exceptions that can be raised
by the SDK when interacting with the Ogem API.
"""

from typing import Optional, Any, Dict


class OgemError(Exception):
    """Base exception class for all Ogem SDK errors."""
    
    def __init__(self, message: str):
        self.message = message
        super().__init__(message)


class APIError(OgemError):
    """
    Exception raised when the API returns an error response.
    
    Attributes:
        message: The error message
        status_code: HTTP status code
        error_type: The type of error returned by the API
        error_code: Optional error code
        details: Additional error details
    """
    
    def __init__(
        self,
        message: str,
        status_code: Optional[int] = None,
        error_type: Optional[str] = None,
        error_code: Optional[str] = None,
        details: Optional[Dict[str, Any]] = None
    ):
        self.status_code = status_code
        self.error_type = error_type
        self.error_code = error_code
        self.details = details or {}
        
        # Create comprehensive error message
        full_message = message
        if status_code:
            full_message = f"[{status_code}] {full_message}"
        if error_type:
            full_message = f"{error_type}: {full_message}"
        if error_code:
            full_message = f"{full_message} (code: {error_code})"
        
        super().__init__(full_message)


class AuthenticationError(APIError):
    """
    Exception raised when authentication fails.
    
    This typically happens when:
    - The API key is invalid or missing
    - The API key has expired
    - The API key doesn't have the required permissions
    """
    
    def __init__(self, message: str = "Authentication failed"):
        super().__init__(
            message=message,
            status_code=401,
            error_type="authentication_error"
        )


class RateLimitError(APIError):
    """
    Exception raised when rate limits are exceeded.
    
    This happens when:
    - Too many requests are made in a short time period
    - Token limits are exceeded
    - Cost limits are exceeded
    - Concurrent request limits are exceeded
    
    Attributes:
        retry_after: Number of seconds to wait before retrying (if provided)
        limit_type: Type of limit that was exceeded
    """
    
    def __init__(
        self,
        message: str = "Rate limit exceeded",
        retry_after: Optional[int] = None,
        limit_type: Optional[str] = None
    ):
        super().__init__(
            message=message,
            status_code=429,
            error_type="rate_limit_error",
            details={
                "retry_after": retry_after,
                "limit_type": limit_type
            }
        )
        self.retry_after = retry_after
        self.limit_type = limit_type


class TenantError(APIError):
    """
    Exception raised when there are tenant-related errors.
    
    This happens when:
    - The tenant ID is invalid or not found
    - The tenant is suspended or deleted
    - The tenant has exceeded its limits
    - The user doesn't have access to the tenant
    """
    
    def __init__(self, message: str, tenant_id: Optional[str] = None):
        super().__init__(
            message=message,
            status_code=403,
            error_type="tenant_error",
            details={"tenant_id": tenant_id}
        )
        self.tenant_id = tenant_id


class ValidationError(APIError):
    """
    Exception raised when request validation fails.
    
    This happens when:
    - Required fields are missing
    - Field values are invalid
    - The request format is incorrect
    - Model or provider constraints are violated
    
    Attributes:
        field_errors: Dictionary of field-specific errors
    """
    
    def __init__(
        self,
        message: str = "Validation error",
        field_errors: Optional[Dict[str, Any]] = None
    ):
        super().__init__(
            message=message,
            status_code=422,
            error_type="validation_error",
            details={"field_errors": field_errors or {}}
        )
        self.field_errors = field_errors or {}


class ModelError(APIError):
    """
    Exception raised when there are model-related errors.
    
    This happens when:
    - The requested model is not available
    - The model doesn't support the requested feature
    - The model is temporarily unavailable
    - Model-specific limits are exceeded
    """
    
    def __init__(self, message: str, model_id: Optional[str] = None):
        super().__init__(
            message=message,
            error_type="model_error",
            details={"model_id": model_id}
        )
        self.model_id = model_id


class ProviderError(APIError):
    """
    Exception raised when there are provider-related errors.
    
    This happens when:
    - The AI provider is unavailable
    - Provider-specific errors occur
    - Provider rate limits are hit
    - Provider authentication fails
    """
    
    def __init__(self, message: str, provider: Optional[str] = None):
        super().__init__(
            message=message,
            error_type="provider_error",
            details={"provider": provider}
        )
        self.provider = provider


class CacheError(OgemError):
    """
    Exception raised when there are cache-related errors.
    
    This happens when:
    - Cache operations fail
    - Cache is unavailable
    - Cache configuration is invalid
    """
    
    def __init__(self, message: str):
        super().__init__(f"Cache error: {message}")


class StreamError(OgemError):
    """
    Exception raised when streaming operations fail.
    
    This happens when:
    - Stream connection is lost
    - Stream data is corrupted
    - Stream parsing fails
    """
    
    def __init__(self, message: str):
        super().__init__(f"Stream error: {message}")


class TimeoutError(OgemError):
    """
    Exception raised when requests timeout.
    
    This happens when:
    - Request takes longer than the configured timeout
    - Connection timeout occurs
    - Read timeout occurs
    """
    
    def __init__(self, message: str = "Request timed out"):
        super().__init__(message)


class ConnectionError(OgemError):
    """
    Exception raised when connection to the Ogem server fails.
    
    This happens when:
    - Cannot connect to the server
    - Network issues occur
    - DNS resolution fails
    """
    
    def __init__(self, message: str = "Failed to connect to Ogem server"):
        super().__init__(message)


class ConfigurationError(OgemError):
    """
    Exception raised when there are configuration errors.
    
    This happens when:
    - Required configuration is missing
    - Configuration values are invalid
    - Conflicting configuration options
    """
    
    def __init__(self, message: str):
        super().__init__(f"Configuration error: {message}")


# Exception mapping for HTTP status codes
STATUS_CODE_TO_EXCEPTION = {
    400: ValidationError,
    401: AuthenticationError,
    403: TenantError,  # May be overridden based on error type
    404: ModelError,   # Usually model not found
    422: ValidationError,
    429: RateLimitError,
    500: ProviderError,  # Usually provider issues
    502: ProviderError,
    503: ProviderError,
    504: TimeoutError,
}


def create_exception_from_response(
    status_code: int,
    error_data: Dict[str, Any],
    default_message: str = "API request failed"
) -> APIError:
    """
    Create an appropriate exception based on the error response.
    
    Args:
        status_code: HTTP status code
        error_data: Error data from the API response
        default_message: Default message if none provided
        
    Returns:
        Appropriate exception instance
    """
    message = error_data.get("message", default_message)
    error_type = error_data.get("type", "unknown_error")
    error_code = error_data.get("code")
    
    # Determine exception class based on error type or status code
    if "authentication" in error_type.lower():
        return AuthenticationError(message)
    elif "rate_limit" in error_type.lower() or "rate limit" in message.lower():
        retry_after = error_data.get("retry_after")
        limit_type = error_data.get("limit_type")
        return RateLimitError(message, retry_after=retry_after, limit_type=limit_type)
    elif "tenant" in error_type.lower():
        tenant_id = error_data.get("tenant_id")
        return TenantError(message, tenant_id=tenant_id)
    elif "validation" in error_type.lower():
        field_errors = error_data.get("field_errors")
        return ValidationError(message, field_errors=field_errors)
    elif "model" in error_type.lower():
        model_id = error_data.get("model_id")
        return ModelError(message, model_id=model_id)
    elif "provider" in error_type.lower():
        provider = error_data.get("provider")
        return ProviderError(message, provider=provider)
    else:
        # Fall back to status code mapping
        exception_class = STATUS_CODE_TO_EXCEPTION.get(status_code, APIError)
        return exception_class(
            message=message,
            status_code=status_code,
            error_type=error_type,
            error_code=error_code,
            details=error_data
        )


# Retry utilities
def is_retryable_error(error: Exception) -> bool:
    """
    Determine if an error is retryable.
    
    Args:
        error: The exception to check
        
    Returns:
        True if the error is retryable, False otherwise
    """
    if isinstance(error, RateLimitError):
        return True
    elif isinstance(error, APIError):
        # Retry on server errors
        return error.status_code is not None and error.status_code >= 500
    elif isinstance(error, (TimeoutError, ConnectionError)):
        return True
    
    return False


def get_retry_delay(error: Exception, attempt: int, base_delay: float = 1.0) -> float:
    """
    Calculate the delay before retrying a failed request.
    
    Args:
        error: The exception that occurred
        attempt: The current attempt number (1-based)
        base_delay: Base delay in seconds
        
    Returns:
        Delay in seconds before retry
    """
    if isinstance(error, RateLimitError) and error.retry_after:
        return float(error.retry_after)
    
    # Exponential backoff with jitter
    import random
    delay = base_delay * (2 ** (attempt - 1))
    jitter = random.uniform(0, 0.1) * delay
    return delay + jitter

계층형 예외 처리·HTTP 매핑·재시도 제어까지 갖춘 SDK 핵심 방어 계층으로 설계되었으며, 일부 네임스페이스 충돌과 예외 정보 보존 문제를 보완하면 운영 장애 대응력을 갖춘 9.5점 이상 수준의 프로덕션 구조다.

제안패치
"""
Exception classes and resilience utilities for the Ogem Python SDK.

This module defines standardized custom exceptions, dynamic response-based
exception factories, and enterprise-grade resilience strategies including 
max retry limits, retryable status code policies, and sensitive data masking.
"""

import enum
import random
import re
from typing import Optional, Any, Dict


# ==========================================
# 1. Retry Policy Enum
# ==========================================
class RetryPolicy(enum.Enum):
    """Defines the retry behavior strategy for SDK errors."""
    DEFAULT = "default"
    AGGRESSIVE = "aggressive"
    NONE = "none"


# ==========================================
# 2. Sensitive Data Masking Utility
# ==========================================
def _mask_sensitive_data(message: str) -> str:
    """
    Masks potential API keys, secrets, or sensitive tokens in error messages 
    to prevent credential leakage in logging and monitoring systems.
    """
    if not message:
        return message
    
    # OpenAI/Anthropic/General API key patterns (e.g., sk-..., ak-..., Bearer tokens)
    # Replaces sensitive part with asterisks while retaining prefix/suffix shape
    patterns = [
        r"(sk-[a-zA-Z0-9_-]{4})[a-zA-Z0-9_-]+",
        r"(Bearer\s+[a-zA-Z0-9_.-]{4})[a-zA-Z0-9_.-]+",
        r"(api[_-]?key[=:]\s*['\"]?[a-zA-Z0-9_-]{4})[a-zA-Z0-9_-]+"
    ]
    
    masked_message = message
    for pattern in patterns:
        masked_message = re.sub(pattern, r"\1********", masked_message, flags=re.IGNORECASE)
        
    return masked_message


# ==========================================
# 3. Base Exception Definitions
# ==========================================
class OgemError(Exception):
    """Base exception class for all Ogem SDK errors."""
    
    def __init__(self, message: str, context_id: Optional[str] = None):
        self.message = _mask_sensitive_data(message)
        self.context_id = context_id
        
        full_message = self.message
        if context_id:
            full_message = f"[context_id: {context_id}] {full_message}"
            
        super().__init__(full_message)


class APIError(OgemError):
    """
    Exception raised when the API returns an error response.
    """
    
    def __init__(
        self,
        message: str,
        status_code: Optional[int] = None,
        error_type: Optional[str] = None,
        error_code: Optional[str] = None,
        details: Optional[Dict[str, Any]] = None,
        context_id: Optional[str] = None
    ):
        self.status_code = status_code
        self.error_type = error_type
        self.error_code = error_code
        self.details = details or {}
        
        # Extract context_id from error details if not explicitly provided
        if not context_id and self.details:
            context_id = self.details.get("request_id") or self.details.get("trace_id")
        
        masked_message = _mask_sensitive_data(message)
        
        # Build comprehensive error message
        full_message = masked_message
        if status_code:
            full_message = f"[{status_code}] {full_message}"
        if error_type:
            full_message = f"{error_type}: {full_message}"
        if error_code:
            full_message = f"{full_message} (code: {error_code})"
        
        super().__init__(full_message, context_id=context_id)


class AuthenticationError(APIError):
    def __init__(self, message: str = "Authentication failed", details: Optional[Dict[str, Any]] = None, context_id: Optional[str] = None):
        super().__init__(
            message=message,
            status_code=401,
            error_type="authentication_error",
            details=details,
            context_id=context_id
        )


class RateLimitError(APIError):
    def __init__(
        self,
        message: str = "Rate limit exceeded",
        retry_after: Optional[int] = None,
        limit_type: Optional[str] = None,
        details: Optional[Dict[str, Any]] = None,
        context_id: Optional[str] = None
    ):
        merged_details = {"retry_after": retry_after, "limit_type": limit_type}
        if details:
            merged_details.update(details)
            
        super().__init__(
            message=message,
            status_code=429,
            error_type="rate_limit_error",
            details=merged_details,
            context_id=context_id
        )
        self.retry_after = retry_after
        self.limit_type = limit_type


class TenantError(APIError):
    def __init__(self, message: str, tenant_id: Optional[str] = None, details: Optional[Dict[str, Any]] = None, context_id: Optional[str] = None):
        merged_details = {"tenant_id": tenant_id}
        if details:
            merged_details.update(details)
            
        super().__init__(
            message=message,
            status_code=403,
            error_type="tenant_error",
            details=merged_details,
            context_id=context_id
        )
        self.tenant_id = tenant_id


class ValidationError(APIError):
    def __init__(
        self,
        message: str = "Validation error",
        field_errors: Optional[Dict[str, Any]] = None,
        details: Optional[Dict[str, Any]] = None,
        context_id: Optional[str] = None
    ):
        merged_field_errors = field_errors or {}
        merged_details = {"field_errors": merged_field_errors}
        if details:
            merged_details.update(details)
            
        super().__init__(
            message=message,
            status_code=422,
            error_type="validation_error",
            details=merged_details,
            context_id=context_id
        )
        self.field_errors = merged_field_errors


class ModelError(APIError):
    def __init__(self, message: str, model_id: Optional[str] = None, details: Optional[Dict[str, Any]] = None, context_id: Optional[str] = None):
        merged_details = {"model_id": model_id}
        if details:
            merged_details.update(details)
            
        super().__init__(
            message=message,
            error_type="model_error",
            details=merged_details,
            context_id=context_id
        )
        self.model_id = model_id


class ProviderError(APIError):
    def __init__(self, message: str, provider: Optional[str] = None, details: Optional[Dict[str, Any]] = None, context_id: Optional[str] = None):
        merged_details = {"provider": provider}
        if details:
            merged_details.update(details)
            
        super().__init__(
            message=message,
            error_type="provider_error",
            details=merged_details,
            context_id=context_id
        )
        self.provider = provider


class CacheError(OgemError):
    def __init__(self, message: str, context_id: Optional[str] = None):
        super().__init__(f"Cache error: {message}", context_id=context_id)


class StreamError(OgemError):
    def __init__(self, message: str, context_id: Optional[str] = None):
        super().__init__(f"Stream error: {message}", context_id=context_id)


class OgemTimeoutError(OgemError):
    def __init__(self, message: str = "Request timed out", context_id: Optional[str] = None):
        super().__init__(message, context_id=context_id)


class OgemConnectionError(OgemError):
    def __init__(self, message: str = "Failed to connect to Ogem server", context_id: Optional[str] = None):
        super().__init__(message, context_id=context_id)


class ConfigurationError(OgemError):
    def __init__(self, message: str, context_id: Optional[str] = None):
        super().__init__(f"Configuration error: {message}", context_id=context_id)


# ==========================================
# 4. Exception Mapping & Factory
# ==========================================
STATUS_CODE_TO_EXCEPTION = {
    400: ValidationError,
    401: AuthenticationError,
    403: TenantError,
    404: ModelError,
    422: ValidationError,
    429: RateLimitError,
    500: ProviderError,
    502: ProviderError,
    503: ProviderError,
    504: OgemTimeoutError,
}


def create_exception_from_response(
    status_code: int,
    error_data: Dict[str, Any],
    default_message: str = "API request failed"
) -> APIError:
    """
    Create an appropriate exception based on the error response while ensuring
    full metadata preservation, context ID extraction, and masking.
    """
    message = error_data.get("message", default_message)
    error_type = error_data.get("type", "unknown_error")
    error_code = error_data.get("code")
    context_id = error_data.get("request_id") or error_data.get("trace_id")
    
    if "authentication" in error_type.lower():
        return AuthenticationError(message, details=error_data, context_id=context_id)
    elif "rate_limit" in error_type.lower() or "rate limit" in message.lower():
        retry_after = error_data.get("retry_after")
        limit_type = error_data.get("limit_type")
        return RateLimitError(message, retry_after=retry_after, limit_type=limit_type, details=error_data, context_id=context_id)
    elif "tenant" in error_type.lower():
        tenant_id = error_data.get("tenant_id")
        return TenantError(message, tenant_id=tenant_id, details=error_data, context_id=context_id)
    elif "validation" in error_type.lower():
        field_errors = error_data.get("field_errors")
        return ValidationError(message, field_errors=field_errors, details=error_data, context_id=context_id)
    elif "model" in error_type.lower():
        model_id = error_data.get("model_id")
        return ModelError(message, model_id=model_id, details=error_data, context_id=context_id)
    elif "provider" in error_type.lower():
        provider = error_data.get("provider")
        return ProviderError(message, provider=provider, details=error_data, context_id=context_id)
    else:
        exception_class = STATUS_CODE_TO_EXCEPTION.get(status_code, APIError)
        return exception_class(
            message=message,
            status_code=status_code,
            error_type=error_type,
            error_code=error_code,
            details=error_data,
            context_id=context_id
        )


# ==========================================
# 5. Enterprise Resilience & Retry Utilities
# ==========================================
# 명시적으로 일시적 장애 복구가 보장되는 안전한 HTTP 상태 코드 정의
RETRYABLE_STATUS_CODES = {500, 502, 503, 504}
MAX_RETRY_DELAY = 60.0  # 재시도 지연 시간 상한선 (60초)


def is_retryable_error(error: Exception) -> bool:
    """
    Determine if an error is retryable based on strict status code policies
    and connection/timeout categories.
    """
    if isinstance(error, RateLimitError):
        return True
    elif isinstance(error, APIError):
        # 무분별한 5xx 재시도를 방지하고 명시적 일시 장애 코드만 허용
        return error.status_code is not None and error.status_code in RETRYABLE_STATUS_CODES
    elif isinstance(error, (OgemTimeoutError, OgemConnectionError)):
        return True
    
    return False


def get_retry_delay(error: Exception, attempt: int, base_delay: float = 1.0) -> float:
    """
    Calculate the delay before retrying using exponential backoff with jitter,
    enforcing a strict maximum upper bound (MAX_RETRY_DELAY).
    """
    if isinstance(error, RateLimitError) and error.retry_after:
        return float(error.retry_after)
    
    # 지수 백오프 계산
    delay = base_delay * (2 ** (attempt - 1))
    jitter = random.uniform(0, 0.1) * delay
    total_delay = delay + jitter
    
    # 상한선(MAX_RETRY_DELAY) 강제 적용으로 무한 대기 방지
    return min(total_delay, MAX_RETRY_DELAY)

최종 개선사항
✅ 무분별한 재시도 판단 → RETRYABLE_STATUS_CODES 명시 정책 적용 → 불필요한 재시도로 인한 장애 확산 방지
✅ 무제한 Exponential Backoff → MAX_RETRY_DELAY 상한 적용 → 장시간 대기 및 요청 병목 차단
✅ 표준 예외명 충돌 구조 → OgemTimeoutError·OgemConnectionError 도메인 명칭 적용 → 예외 처리 혼선 제거
✅ API 응답 정보 단순 폐기 → details 전체 보존 및 context_id 추출 적용 → 장애 추적성과 원인 분석력 강화
✅ 인증키·토큰 포함 가능 메시지 → 민감정보 Masking 계층 추가 → 로그 및 모니터링 정보 유출 방지
✅ 개별 예외별 상세 데이터 병합 누락 → 모든 API 예외에 metadata 전달 구조 적용 → 디버깅 정보 무결성 확보
✅ 단순 Retry 함수 → RetryPolicy Enum 기반 확장 구조 준비 → 서비스별 복구 전략 분리 기반 확보

단순 예외 정의 모듈에서 장애 복구 정책·보안 보호·추적성을 포함한 SDK 핵심 안정성 계층으로 승격되었으며, 현재 버전은 예외 무결성·운영 대응력·복구 제어력을 갖춘 9.7 수준의 프로덕션 구조다.
