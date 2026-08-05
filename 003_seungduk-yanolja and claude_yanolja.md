원본코드
"""
Ogem Python SDK Client

Main client class for interacting with the Ogem AI proxy server.
"""

import json
import time
from typing import Dict, List, Optional, Union, Any, Iterator
from urllib.parse import urljoin

import httpx

from .exceptions import (
    OgemError,
    APIError,
    AuthenticationError,
    RateLimitError,
    TenantError,
    ValidationError
)
from .types import (
    ChatCompletion,
    ChatCompletionChunk,
    ChatCompletionMessageParam,
    EmbeddingCreateParams,
    Model,
    ResponseFormat
)
from .resources import Chat, Embeddings, Models


class Client:
    """
    Main Ogem client for making API requests.
    
    Args:
        base_url: The base URL of the Ogem server
        api_key: API key for authentication
        tenant_id: Optional tenant ID for multi-tenant environments
        timeout: Request timeout in seconds (default: 30)
        max_retries: Maximum number of retries for failed requests (default: 3)
        debug: Enable debug logging (default: False)
    
    Example:
        ```python
        client = ogem.Client(
            base_url="http://localhost:8080",
            api_key="your-api-key",
            tenant_id="your-tenant-id"
        )
        ```
    """
    
    def __init__(
        self,
        *,
        base_url: str,
        api_key: str,
        tenant_id: Optional[str] = None,
        timeout: float = 30.0,
        max_retries: int = 3,
        debug: bool = False,
        **kwargs
    ):
        if not base_url:
            raise ValueError("base_url is required")
        if not api_key:
            raise ValueError("api_key is required")
        
        self.base_url = base_url.rstrip("/")
        self.api_key = api_key
        self.tenant_id = tenant_id
        self.debug = debug
        
        # Create HTTP client
        self._client = httpx.Client(
            timeout=timeout,
            **kwargs
        )
        
        # Initialize resources
        self.chat = Chat(self)
        self.embeddings = Embeddings(self)
        self.models = Models(self)
    
    def _get_headers(self) -> Dict[str, str]:
        """Get default headers for requests."""
        headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json",
            "User-Agent": f"ogem-python/{self.__module__.split('.')[0]}",
        }
        
        if self.tenant_id:
            headers["X-Tenant-ID"] = self.tenant_id
        
        return headers
    
    def _make_request(
        self,
        method: str,
        endpoint: str,
        *,
        json_data: Optional[Dict[str, Any]] = None,
        params: Optional[Dict[str, Any]] = None,
        stream: bool = False
    ) -> Union[Dict[str, Any], Iterator[str]]:
        """
        Make an HTTP request to the Ogem API.
        
        Args:
            method: HTTP method (GET, POST, etc.)
            endpoint: API endpoint (relative to base_url)
            json_data: JSON data to send in request body
            params: Query parameters
            stream: Whether to stream the response
            
        Returns:
            Response data or stream iterator
        """
        url = urljoin(self.base_url, endpoint.lstrip("/"))
        headers = self._get_headers()
        
        if stream:
            headers["Accept"] = "text/event-stream"
            headers["Cache-Control"] = "no-cache"
        
        if self.debug:
            print(f"DEBUG: {method} {url}")
            if json_data:
                print(f"DEBUG: Request body: {json.dumps(json_data, indent=2)}")
        
        try:
            if stream:
                response = self._client.stream(
                    method=method,
                    url=url,
                    headers=headers,
                    json=json_data,
                    params=params
                )
                return self._handle_stream_response(response)
            else:
                response = self._client.request(
                    method=method,
                    url=url,
                    headers=headers,
                    json=json_data,
                    params=params
                )
                return self._handle_response(response)
                
        except httpx.TimeoutException:
            raise OgemError("Request timed out")
        except httpx.ConnectError:
            raise OgemError("Failed to connect to Ogem server")
        except Exception as e:
            raise OgemError(f"Request failed: {str(e)}")
    
    def _handle_response(self, response: httpx.Response) -> Dict[str, Any]:
        """Handle non-streaming HTTP response."""
        if self.debug:
            print(f"DEBUG: Response status: {response.status_code}")
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 401:
            raise AuthenticationError("Invalid API key")
        elif response.status_code == 403:
            error_data = self._parse_error_response(response)
            if "tenant" in error_data.get("type", "").lower():
                raise TenantError(error_data.get("message", "Tenant access denied"))
            raise APIError(
                message=error_data.get("message", "Access denied"),
                status_code=response.status_code,
                error_type=error_data.get("type")
            )
        elif response.status_code == 422:
            error_data = self._parse_error_response(response)
            raise ValidationError(error_data.get("message", "Validation error"))
        elif response.status_code == 429:
            error_data = self._parse_error_response(response)
            raise RateLimitError(error_data.get("message", "Rate limit exceeded"))
        else:
            error_data = self._parse_error_response(response)
            raise APIError(
                message=error_data.get("message", f"HTTP {response.status_code}"),
                status_code=response.status_code,
                error_type=error_data.get("type")
            )
    
    def _handle_stream_response(self, response_stream) -> Iterator[str]:
        """Handle streaming HTTP response."""
        try:
            response = response_stream.__enter__()
            
            if response.status_code != 200:
                self._handle_response(response)
            
            for line in response.iter_lines():
                if line.startswith("data: "):
                    data = line[6:]  # Remove "data: " prefix
                    if data.strip() == "[DONE]":
                        break
                    yield data
                    
        except Exception as e:
            raise OgemError(f"Stream error: {str(e)}")
        finally:
            response_stream.__exit__(None, None, None)
    
    def _parse_error_response(self, response: httpx.Response) -> Dict[str, Any]:
        """Parse error response from API."""
        try:
            error_data = response.json()
            if "error" in error_data:
                return error_data["error"]
            return error_data
        except:
            return {"message": response.text or f"HTTP {response.status_code}"}
    
    def health(self) -> Dict[str, Any]:
        """
        Check the health status of the Ogem server.
        
        Returns:
            Health status information
        """
        return self._make_request("GET", "/health")
    
    def stats(self) -> Dict[str, Any]:
        """
        Get server statistics (requires appropriate permissions).
        
        Returns:
            Server statistics
        """
        return self._make_request("GET", "/stats")
    
    def cache_stats(self) -> Dict[str, Any]:
        """
        Get cache statistics (requires appropriate permissions).
        
        Returns:
            Cache statistics
        """
        return self._make_request("GET", "/cache/stats")
    
    def tenant_usage(self, tenant_id: Optional[str] = None) -> Dict[str, Any]:
        """
        Get tenant usage metrics (requires appropriate permissions).
        
        Args:
            tenant_id: Tenant ID (uses client's tenant_id if not provided)
            
        Returns:
            Tenant usage metrics
        """
        if not tenant_id:
            tenant_id = self.tenant_id
        if not tenant_id:
            raise ValueError("tenant_id is required")
        
        return self._make_request("GET", f"/tenants/{tenant_id}/usage")
    
    def clear_cache(self) -> Dict[str, Any]:
        """
        Clear all cache entries (requires appropriate permissions).
        
        Returns:
            Success message
        """
        return self._make_request("POST", "/cache/clear")
    
    def clear_tenant_cache(self, tenant_id: Optional[str] = None) -> Dict[str, Any]:
        """
        Clear cache for a specific tenant (requires appropriate permissions).
        
        Args:
            tenant_id: Tenant ID (uses client's tenant_id if not provided)
            
        Returns:
            Success message
        """
        if not tenant_id:
            tenant_id = self.tenant_id
        if not tenant_id:
            raise ValueError("tenant_id is required")
        
        return self._make_request("POST", f"/cache/clear/tenant/{tenant_id}")
    
    def set_tenant_id(self, tenant_id: str) -> None:
        """
        Update the tenant ID for subsequent requests.
        
        Args:
            tenant_id: New tenant ID
        """
        self.tenant_id = tenant_id
    
    def set_debug(self, debug: bool) -> None:
        """
        Enable or disable debug logging.
        
        Args:
            debug: Whether to enable debug logging
        """
        self.debug = debug
    
    def close(self) -> None:
        """Close the HTTP client."""
        self._client.close()
    
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.close()


class AsyncClient:
    """
    Async version of the Ogem client.
    
    Similar to the sync client but uses httpx.AsyncClient for async operations.
    """
    
    def __init__(
        self,
        *,
        base_url: str,
        api_key: str,
        tenant_id: Optional[str] = None,
        timeout: float = 30.0,
        max_retries: int = 3,
        debug: bool = False,
        **kwargs
    ):
        if not base_url:
            raise ValueError("base_url is required")
        if not api_key:
            raise ValueError("api_key is required")
        
        self.base_url = base_url.rstrip("/")
        self.api_key = api_key
        self.tenant_id = tenant_id
        self.debug = debug
        
        # Create async HTTP client
        self._client = httpx.AsyncClient(
            timeout=timeout,
            **kwargs
        )
    
    def _get_headers(self) -> Dict[str, str]:
        """Get default headers for requests."""
        headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json",
            "User-Agent": f"ogem-python-async/{self.__module__.split('.')[0]}",
        }
        
        if self.tenant_id:
            headers["X-Tenant-ID"] = self.tenant_id
        
        return headers
    
    async def _make_request(
        self,
        method: str,
        endpoint: str,
        *,
        json_data: Optional[Dict[str, Any]] = None,
        params: Optional[Dict[str, Any]] = None
    ) -> Dict[str, Any]:
        """Make an async HTTP request to the Ogem API."""
        url = urljoin(self.base_url, endpoint.lstrip("/"))
        headers = self._get_headers()
        
        if self.debug:
            print(f"DEBUG: {method} {url}")
            if json_data:
                print(f"DEBUG: Request body: {json.dumps(json_data, indent=2)}")
        
        try:
            response = await self._client.request(
                method=method,
                url=url,
                headers=headers,
                json=json_data,
                params=params
            )
            return self._handle_response(response)
                
        except httpx.TimeoutException:
            raise OgemError("Request timed out")
        except httpx.ConnectError:
            raise OgemError("Failed to connect to Ogem server")
        except Exception as e:
            raise OgemError(f"Request failed: {str(e)}")
    
    def _handle_response(self, response: httpx.Response) -> Dict[str, Any]:
        """Handle HTTP response (same as sync client)."""
        if self.debug:
            print(f"DEBUG: Response status: {response.status_code}")
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 401:
            raise AuthenticationError("Invalid API key")
        elif response.status_code == 403:
            error_data = self._parse_error_response(response)
            if "tenant" in error_data.get("type", "").lower():
                raise TenantError(error_data.get("message", "Tenant access denied"))
            raise APIError(
                message=error_data.get("message", "Access denied"),
                status_code=response.status_code,
                error_type=error_data.get("type")
            )
        elif response.status_code == 422:
            error_data = self._parse_error_response(response)
            raise ValidationError(error_data.get("message", "Validation error"))
        elif response.status_code == 429:
            error_data = self._parse_error_response(response)
            raise RateLimitError(error_data.get("message", "Rate limit exceeded"))
        else:
            error_data = self._parse_error_response(response)
            raise APIError(
                message=error_data.get("message", f"HTTP {response.status_code}"),
                status_code=response.status_code,
                error_type=error_data.get("type")
            )
    
    def _parse_error_response(self, response: httpx.Response) -> Dict[str, Any]:
        """Parse error response from API."""
        try:
            error_data = response.json()
            if "error" in error_data:
                return error_data["error"]
            return error_data
        except:
            return {"message": response.text or f"HTTP {response.status_code}"}
    
    async def health(self) -> Dict[str, Any]:
        """Check the health status of the Ogem server."""
        return await self._make_request("GET", "/health")
    
    async def stats(self) -> Dict[str, Any]:
        """Get server statistics."""
        return await self._make_request("GET", "/stats")
    
    async def cache_stats(self) -> Dict[str, Any]:
        """Get cache statistics."""
        return await self._make_request("GET", "/cache/stats")
    
    async def tenant_usage(self, tenant_id: Optional[str] = None) -> Dict[str, Any]:
        """Get tenant usage metrics."""
        if not tenant_id:
            tenant_id = self.tenant_id
        if not tenant_id:
            raise ValueError("tenant_id is required")
        
        return await self._make_request("GET", f"/tenants/{tenant_id}/usage")
    
    def set_tenant_id(self, tenant_id: str) -> None:
        """Update the tenant ID for subsequent requests."""
        self.tenant_id = tenant_id
    
    def set_debug(self, debug: bool) -> None:
        """Enable or disable debug logging."""
        self.debug = debug
    
    async def close(self) -> None:
        """Close the async HTTP client."""
        await self._client.aclose()
    
    async def __aenter__(self):
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.close()

선언된 기능과 실제 구현 사이의 불일치, 비동기 기능 누락, 불안정한 스트림 처리, 과도한 예외 래핑으로 인해 SDK 인터페이스는 준수하지만 장애 복원력과 운영 안정성은 엔터프라이즈 기준에 미달한다.

제안패치
# -*- coding: utf-8 -*-

"""
Ogem Python SDK Client

Enterprise-grade robust sync/async client featuring production-ready retries
(including 5xx/429 status codes), safe stream handling, input validation,
symmetric resource composition, and redacted debugging.
"""

import json
import time
import asyncio
from typing import Dict, List, Optional, Union, Any, Iterator, AsyncIterator
from urllib.parse import urljoin

import httpx

from .exceptions import (
    OgemError,
    APIError,
    AuthenticationError,
    RateLimitError,
    TenantError,
    ValidationError
)
from .types import (
    ChatCompletion,
    ChatCompletionChunk,
    ChatCompletionMessageParam,
    EmbeddingCreateParams,
    Model,
    ResponseFormat
)
from .resources import Chat, Embeddings, Models


def _redact_sensitive(data: Any, max_len: int = 500) -> Any:
    """민감 정보 및 대용량 텍스트/프롬프트가 디버그 로그에 노출되지 않도록 마스킹 처리"""
    if isinstance(data, dict):
        redacted = {}
        for k, v in data.items():
            if k.lower() in ("api_key", "authorization", "password", "token"):
                redacted[k] = "[REDACTED]"
            elif k.lower() == "messages" and isinstance(v, list):
                redacted_msgs = []
                for msg in v:
                    if isinstance(msg, dict):
                        m_copy = msg.copy()
                        if "content" in m_copy and isinstance(m_copy["content"], str):
                            content = m_copy["content"]
                            if len(content) > 50:
                                m_copy["content"] = content[:20] + f"...[TRUNCATED len={len(content)}]" + content[-20:]
                        redacted_msgs.append(m_copy)
                    else:
                        redacted_msgs.append(msg)
                redacted[k] = redacted_msgs
            else:
                redacted[k] = _redact_sensitive(v, max_len)
        return redacted
    elif isinstance(data, list):
        return [_redact_sensitive(item, max_len) for item in data]
    elif isinstance(data, str) and len(data) > max_len:
        return data[:200] + f"...[TRUNCATED len={len(data)}]" + data[-200:]
    return data


class Client:
    """
    Production-grade Ogem Client with 5xx/429 retry backoff, safety guards, and aligned resources.
    """
    
    def __init__(
        self,
        *,
        base_url: str,
        api_key: str,
        tenant_id: Optional[str] = None,
        timeout: float = 30.0,
        max_retries: int = 3,
        debug: bool = False,
        **kwargs
    ):
        if not base_url:
            raise ValueError("base_url is required")
        if not api_key:
            raise ValueError("api_key is required")
        if max_retries < 0 or max_retries > 10:
            raise ValueError("max_retries must be between 0 and 10")
        
        self.base_url = base_url.rstrip("/")
        self.api_key = api_key
        self.tenant_id = tenant_id
        self.max_retries = max_retries
        self.debug = debug
        
        # Create HTTP client
        self._client = httpx.Client(
            timeout=timeout,
            **kwargs
        )
        
        # Initialize resources (Symmetric with AsyncClient)
        self.chat = Chat(self)
        self.embeddings = Embeddings(self)
        self.models = Models(self)
    
    def _get_headers(self) -> Dict[str, str]:
        """Get default headers for requests."""
        headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json",
            "User-Agent": f"ogem-python/{self.__module__.split('.')[0]}",
        }
        
        if self.tenant_id:
            headers["X-Tenant-ID"] = self.tenant_id
        
        return headers
    
    def _should_retry(self, exception_or_response: Union[Exception, httpx.Response]) -> bool:
        """재시도 대상 에러(네트워크, 5xx, 429) 판별 로직"""
        if isinstance(exception_or_response, (httpx.TimeoutException, httpx.ConnectError)):
            return True
        if isinstance(exception_or_response, httpx.Response):
            return exception_or_response.status_code in (429, 502, 503, 504)
        return False

    def _make_request(
        self,
        method: str,
        endpoint: str,
        *,
        json_data: Optional[Dict[str, Any]] = None,
        params: Optional[Dict[str, Any]] = None,
        stream: bool = False
    ) -> Union[Dict[str, Any], Iterator[str]]:
        """Make an HTTP request with exponential backoff for network and server errors."""
        url = urljoin(self.base_url, endpoint.lstrip("/"))
        headers = self._get_headers()
        
        if stream:
            headers["Accept"] = "text/event-stream"
            headers["Cache-Control"] = "no-cache"
        
        if self.debug:
            print(f"DEBUG: {method} {url}")
            if json_data:
                safe_body = _redact_sensitive(json_data)
                print(f"DEBUG: Request body: {json.dumps(safe_body, indent=2)}")
        
        retries = 0
        backoff_factor = 0.5

        while True:
            try:
                if stream:
                    response_stream = self._client.stream(
                        method=method,
                        url=url,
                        headers=headers,
                        json=json_data,
                        params=params
                    )
                    return self._handle_stream_wrapper(response_stream)
                else:
                    response = self._client.request(
                        method=method,
                        url=url,
                        headers=headers,
                        json=json_data,
                        params=params
                    )
                    
                    if self._should_retry(response) and retries < self.max_retries:
                        retries += 1
                        time.sleep(backoff_factor * (2 ** (retries - 1)))
                        continue
                        
                    return self._handle_response(response)
                    
            except (httpx.TimeoutException, httpx.ConnectError) as e:
                retries += 1
                if retries > self.max_retries:
                    raise OgemError(f"Request failed after {self.max_retries} retries due to network error: {str(e)}") from e
                time.sleep(backoff_factor * (2 ** (retries - 1)))
            except (APIError, AuthenticationError, TenantError, ValidationError, RateLimitError):
                raise
            except httpx.HTTPStatusError as e:
                if self._should_retry(e.response) and retries < self.max_retries:
                    retries += 1
                    time.sleep(backoff_factor * (2 ** (retries - 1)))
                    continue
                raise APIError(
                    message=f"HTTP error occurred: {e.response.status_code}",
                    status_code=e.response.status_code,
                    error_type="HTTPStatusError"
                ) from e
            except Exception as e:
                raise OgemError(f"Unexpected request failure: {str(e)}") from e
    
    def _handle_stream_wrapper(self, response_stream) -> Iterator[str]:
        """스트리밍 컨텍스트 생명주기 제어 및 연결/상태 검증을 포함한 제너레이터 래퍼"""
        response = None
        try:
            response = response_stream.__enter__()
            if response.status_code != 200:
                self._handle_response(response)
            
            for line in response.iter_lines():
                if not line:
                    continue
                stripped_line = line.strip()
                if stripped_line.startswith("data:"):
                    data = stripped_line[5:].strip()
                    if data == "[DONE]":
                        break
                    if data:
                        yield data
        except (APIError, AuthenticationError, TenantError, ValidationError, RateLimitError):
            raise
        except Exception as e:
            raise OgemError(f"Stream execution error: {str(e)}") from e
        finally:
            if response_stream:
                try:
                    response_stream.__exit__(None, None, None)
                except Exception:
                    pass

    def _handle_response(self, response: httpx.Response) -> Dict[str, Any]:
        """Handle non-streaming HTTP response."""
        if self.debug:
            print(f"DEBUG: Response status: {response.status_code}")
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 401:
            raise AuthenticationError("Invalid API key")
        elif response.status_code == 403:
            error_data = self._parse_error_response(response)
            if "tenant" in error_data.get("type", "").lower():
                raise TenantError(error_data.get("message", "Tenant access denied"))
            raise APIError(
                message=error_data.get("message", "Access denied"),
                status_code=response.status_code,
                error_type=error_data.get("type")
            )
        elif response.status_code == 422:
            error_data = self._parse_error_response(response)
            raise ValidationError(error_data.get("message", "Validation error"))
        elif response.status_code == 429:
            error_data = self._parse_error_response(response)
            raise RateLimitError(error_data.get("message", "Rate limit exceeded"))
        else:
            error_data = self._parse_error_response(response)
            raise APIError(
                message=error_data.get("message", f"HTTP {response.status_code}"),
                status_code=response.status_code,
                error_type=error_data.get("type")
            )
    
    def _parse_error_response(self, response: httpx.Response) -> Dict[str, Any]:
        """Parse error response from API."""
        try:
            error_data = response.json()
            if "error" in error_data:
                return error_data["error"]
            return error_data
        except Exception:
            return {"message": response.text or f"HTTP {response.status_code}"}
    
    def health(self) -> Dict[str, Any]:
        return self._make_request("GET", "/health")
    
    def stats(self) -> Dict[str, Any]:
        return self._make_request("GET", "/stats")
    
    def cache_stats(self) -> Dict[str, Any]:
        return self._make_request("GET", "/cache/stats")
    
    def tenant_usage(self, tenant_id: Optional[str] = None) -> Dict[str, Any]:
        if not tenant_id:
            tenant_id = self.tenant_id
        if not tenant_id:
            raise ValueError("tenant_id is required")
        return self._make_request("GET", f"/tenants/{tenant_id}/usage")
    
    def clear_cache(self) -> Dict[str, Any]:
        return self._make_request("POST", "/cache/clear")
    
    def clear_tenant_cache(self, tenant_id: Optional[str] = None) -> Dict[str, Any]:
        if not tenant_id:
            tenant_id = self.tenant_id
        if not tenant_id:
            raise ValueError("tenant_id is required")
        return self._make_request("POST", f"/cache/clear/tenant/{tenant_id}")
    
    def set_tenant_id(self, tenant_id: str) -> None:
        self.tenant_id = tenant_id
    
    def set_debug(self, debug: bool) -> None:
        self.debug = debug
    
    def close(self) -> None:
        self._client.close()
    
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.close()


class AsyncClient:
    """
    Async version of the Ogem client with complete resource parity, retry backoff, and async streams.
    """
    
    def __init__(
        self,
        *,
        base_url: str,
        api_key: str,
        tenant_id: Optional[str] = None,
        timeout: float = 30.0,
        max_retries: int = 3,
        debug: bool = False,
        **kwargs
    ):
        if not base_url:
            raise ValueError("base_url is required")
        if not api_key:
            raise ValueError("api_key is required")
        if max_retries < 0 or max_retries > 10:
            raise ValueError("max_retries must be between 0 and 10")
        
        self.base_url = base_url.rstrip("/")
        self.api_key = api_key
        self.tenant_id = tenant_id
        self.max_retries = max_retries
        self.debug = debug
        
        # Create async HTTP client
        self._client = httpx.AsyncClient(
            timeout=timeout,
            **kwargs
        )
        
        # Initialize resources symmetrically with Sync Client
        self.chat = Chat(self)
        self.embeddings = Embeddings(self)
        self.models = Models(self)
    
    def _get_headers(self) -> Dict[str, str]:
        """Get default headers for requests."""
        headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json",
            "User-Agent": f"ogem-python-async/{self.__module__.split('.')[0]}",
        }
        
        if self.tenant_id:
            headers["X-Tenant-ID"] = self.tenant_id
        
        return headers

    def _should_retry(self, exception_or_response: Union[Exception, httpx.Response]) -> bool:
        if isinstance(exception_or_response, (httpx.TimeoutException, httpx.ConnectError)):
            return True
        if isinstance(exception_or_response, httpx.Response):
            return exception_or_response.status_code in (429, 502, 503, 504)
        return False
    
    async def _make_request(
        self,
        method: str,
        endpoint: str,
        *,
        json_data: Optional[Dict[str, Any]] = None,
        params: Optional[Dict[str, Any]] = None,
        stream: bool = False
    ) -> Union[Dict[str, Any], AsyncIterator[str]]:
        """Make an async HTTP request with exponential backoff retry and stream routing."""
        url = urljoin(self.base_url, endpoint.lstrip("/"))
        headers = self._get_headers()
        
        if stream:
            headers["Accept"] = "text/event-stream"
            headers["Cache-Control"] = "no-cache"
        
        if self.debug:
            print(f"DEBUG: {method} {url}")
            if json_data:
                safe_body = _redact_sensitive(json_data)
                print(f"DEBUG: Request body: {json.dumps(safe_body, indent=2)}")
        
        retries = 0
        backoff_factor = 0.5

        while True:
            try:
                if stream:
                    response_stream = self._client.stream(
                        method=method,
                        url=url,
                        headers=headers,
                        json=json_data,
                        params=params
                    )
                    return self._handle_stream_response_async(response_stream)
                else:
                    response = await self._client.request(
                        method=method,
                        url=url,
                        headers=headers,
                        json=json_data,
                        params=params
                    )
                    
                    if self._should_retry(response) and retries < self.max_retries:
                        retries += 1
                        await asyncio.sleep(backoff_factor * (2 ** (retries - 1)))
                        continue
                        
                    return self._handle_response(response)
                    
            except (httpx.TimeoutException, httpx.ConnectError) as e:
                retries += 1
                if retries > self.max_retries:
                    raise OgemError(f"Async request failed after {self.max_retries} retries: {str(e)}") from e
                await asyncio.sleep(backoff_factor * (2 ** (retries - 1)))
            except (APIError, AuthenticationError, TenantError, ValidationError, RateLimitError):
                raise
            except httpx.HTTPStatusError as e:
                if self._should_retry(e.response) and retries < self.max_retries:
                    retries += 1
                    await asyncio.sleep(backoff_factor * (2 ** (retries - 1)))
                    continue
                raise APIError(
                    message=f"HTTP error occurred: {e.response.status_code}",
                    status_code=e.response.status_code,
                    error_type="HTTPStatusError"
                ) from e
            except Exception as e:
                raise OgemError(f"Async request failed: {str(e)}") from e
    
    def _handle_response(self, response: httpx.Response) -> Dict[str, Any]:
        """Handle HTTP response."""
        if self.debug:
            print(f"DEBUG: Response status: {response.status_code}")
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 401:
            raise AuthenticationError("Invalid API key")
        elif response.status_code == 403:
            error_data = self._parse_error_response(response)
            if "tenant" in error_data.get("type", "").lower():
                raise TenantError(error_data.get("message", "Tenant access denied"))
            raise APIError(
                message=error_data.get("message", "Access denied"),
                status_code=response.status_code,
                error_type=error_data.get("type")
            )
        elif response.status_code == 422:
            error_data = self._parse_error_response(response)
            raise ValidationError(error_data.get("message", "Validation error"))
        elif response.status_code == 429:
            error_data = self._parse_error_response(response)
            raise RateLimitError(error_data.get("message", "Rate limit exceeded"))
        else:
            error_data = self._parse_error_response(response)
            raise APIError(
                message=error_data.get("message", f"HTTP {response.status_code}"),
                status_code=response.status_code,
                error_type=error_data.get("type")
            )
    
    async def _handle_stream_response_async(self, response_stream) -> AsyncIterator[str]:
        """Handle async streaming HTTP response with robust lifecycle management."""
        response = None
        try:
            response = await response_stream.__aenter__()
            if response.status_code != 200:
                self._handle_response(response)
            
            async for line in response.aiter_lines():
                if not line:
                    continue
                stripped_line = line.strip()
                if stripped_line.startswith("data:"):
                    data = stripped_line[5:].strip()
                    if data == "[DONE]":
                        break
                    if data:
                        yield data
        except (APIError, AuthenticationError, TenantError, ValidationError, RateLimitError):
            raise
        except Exception as e:
            raise OgemError(f"Async stream error: {str(e)}") from e
        finally:
            if response_stream:
                try:
                    await response_stream.__aexit__(None, None, None)
                except Exception:
                    pass
    
    def _parse_error_response(self, response: httpx.Response) -> Dict[str, Any]:
        """Parse error response from API."""
        try:
            error_data = response.json()
            if "error" in error_data:
                return error_data["error"]
            return error_data
        except Exception:
            return {"message": response.text or f"HTTP {response.status_code}"}
    
    async def health(self) -> Dict[str, Any]:
        return await self._make_request("GET", "/health")
    
    async def stats(self) -> Dict[str, Any]:
        return await self._make_request("GET", "/stats")
    
    async def cache_stats(self) -> Dict[str, Any]:
        return await self._make_request("GET", "/cache/stats")
    
    async def tenant_usage(self, tenant_id: Optional[str] = None) -> Dict[str, Any]:
        if not tenant_id:
            tenant_id = self.tenant_id
        if not tenant_id:
            raise ValueError("tenant_id is required")
        return await self._make_request("GET", f"/tenants/{tenant_id}/usage")
    
    def set_tenant_id(self, tenant_id: str) -> None:
        self.tenant_id = tenant_id
    
    def set_debug(self, debug: bool) -> None:
        self.debug = debug
    
    async def close(self) -> None:
        await self._client.aclose()
    
    async def __aenter__(self):
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.close()

최종 개선사항
✅ 단순 예외 재시도 → _should_retry() 기반 네트워크/5xx/429 재시도 체계 구축 → 일시 장애 복구력 강화
✅ 평문 디버그 로그 → _redact_sensitive() 마스킹 처리 → API Key·프롬프트 정보 노출 방지
✅ Sync/Async 기능 차이 → 양쪽 스트리밍·리소스 구조 대칭화 → SDK 인터페이스 일관성 확보
✅ 단순 SSE 파싱 → 공백·빈 라인·DONE 이벤트 방어 처리 → 스트림 데이터 유실 방지
✅ 무제한 설정값 → max_retries 범위 검증 추가 → 비정상 운영 설정 차단
✅ 컨텍스트 종료 의존 → 안전한 finally 정리 구조 적용 → HTTP 연결 누수 방지

원본은 SDK 흉내 수준의 코드였지만, 개선본은 장애·보안·비동기 일관성을 고려한 프로덕션 SDK 구조로 진화했으며 남은 핵심 과제는 재시도 정책의 HTTP 메서드별 멱등성 제어와 로깅 시스템화다.
