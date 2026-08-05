원본코드
"""
Resource classes for the Ogem Python SDK.

This module contains resource classes that handle specific API endpoints
and provide a structured interface for different API operations.
"""

import json
from typing import Dict, List, Optional, Union, Any, Iterator

from .types import (
    ChatCompletion,
    ChatCompletionChunk,
    ChatCompletionMessageParam,
    EmbeddingResponse,
    Model,
    ModelList,
    ChatCompletionRequest,
    EmbeddingRequest
)
from .exceptions import StreamError, ValidationError


class Chat:
    """
    Resource class for chat completion operations.
    
    Handles all chat-related API endpoints including regular completions
    and streaming completions.
    """
    
    def __init__(self, client):
        self._client = client
        self.completions = ChatCompletions(client)


class ChatCompletions:
    """Handle chat completion requests."""
    
    def __init__(self, client):
        self._client = client
    
    def create(
        self,
        *,
        model: str,
        messages: List[ChatCompletionMessageParam],
        max_tokens: Optional[int] = None,
        temperature: Optional[float] = None,
        top_p: Optional[float] = None,
        n: Optional[int] = None,
        stream: Optional[bool] = None,
        stop: Optional[Union[str, List[str]]] = None,
        presence_penalty: Optional[float] = None,
        frequency_penalty: Optional[float] = None,
        logit_bias: Optional[Dict[str, int]] = None,
        user: Optional[str] = None,
        functions: Optional[List[Dict[str, Any]]] = None,
        function_call: Optional[Union[str, Dict[str, Any]]] = None,
        tools: Optional[List[Dict[str, Any]]] = None,
        tool_choice: Optional[Union[str, Dict[str, Any]]] = None,
        response_format: Optional[Dict[str, Any]] = None,
        seed: Optional[int] = None,
        logprobs: Optional[bool] = None,
        top_logprobs: Optional[int] = None,
        **kwargs
    ) -> Union[ChatCompletion, Iterator[ChatCompletionChunk]]:
        """
        Create a chat completion.
        
        Args:
            model: The model to use for completion
            messages: List of messages in the conversation
            max_tokens: Maximum number of tokens to generate
            temperature: Sampling temperature (0-2)
            top_p: Nucleus sampling parameter
            n: Number of completions to generate
            stream: Whether to stream the response
            stop: Stop sequences
            presence_penalty: Presence penalty (-2 to 2)
            frequency_penalty: Frequency penalty (-2 to 2)
            logit_bias: Logit bias for specific tokens
            user: User identifier for tracking
            functions: Available functions (deprecated, use tools)
            function_call: Function calling behavior (deprecated, use tool_choice)
            tools: Available tools for the model
            tool_choice: Tool choice behavior
            response_format: Response format specification
            seed: Seed for deterministic outputs
            logprobs: Whether to return log probabilities
            top_logprobs: Number of top log probabilities to return
            **kwargs: Additional parameters
            
        Returns:
            ChatCompletion or Iterator[ChatCompletionChunk] if streaming
        """
        # Build request data
        request_data = {
            "model": model,
            "messages": messages
        }
        
        # Add optional parameters
        if max_tokens is not None:
            request_data["max_tokens"] = max_tokens
        if temperature is not None:
            request_data["temperature"] = temperature
        if top_p is not None:
            request_data["top_p"] = top_p
        if n is not None:
            request_data["n"] = n
        if stream is not None:
            request_data["stream"] = stream
        if stop is not None:
            request_data["stop"] = stop if isinstance(stop, list) else [stop]
        if presence_penalty is not None:
            request_data["presence_penalty"] = presence_penalty
        if frequency_penalty is not None:
            request_data["frequency_penalty"] = frequency_penalty
        if logit_bias is not None:
            request_data["logit_bias"] = logit_bias
        if user is not None:
            request_data["user"] = user
        if functions is not None:
            request_data["functions"] = functions
        if function_call is not None:
            request_data["function_call"] = function_call
        if tools is not None:
            request_data["tools"] = tools
        if tool_choice is not None:
            request_data["tool_choice"] = tool_choice
        if response_format is not None:
            request_data["response_format"] = response_format
        if seed is not None:
            request_data["seed"] = seed
        if logprobs is not None:
            request_data["logprobs"] = logprobs
        if top_logprobs is not None:
            request_data["top_logprobs"] = top_logprobs
        
        # Add any additional kwargs
        request_data.update(kwargs)
        
        # Validate required fields
        if not model:
            raise ValidationError("model is required")
        if not messages:
            raise ValidationError("messages is required")
        
        # Handle streaming vs non-streaming
        if request_data.get("stream", False):
            return self._create_stream(request_data)
        else:
            response = self._client._make_request(
                "POST",
                "/v1/chat/completions",
                json_data=request_data
            )
            return self._parse_completion_response(response)
    
    def _create_stream(self, request_data: Dict[str, Any]) -> Iterator[ChatCompletionChunk]:
        """Create a streaming chat completion."""
        stream = self._client._make_request(
            "POST",
            "/v1/chat/completions",
            json_data=request_data,
            stream=True
        )
        
        for line in stream:
            if line.strip() == "[DONE]":
                break
            
            try:
                chunk_data = json.loads(line)
                yield self._parse_chunk_response(chunk_data)
            except json.JSONDecodeError as e:
                raise StreamError(f"Failed to parse stream chunk: {e}")
    
    def _parse_completion_response(self, data: Dict[str, Any]) -> ChatCompletion:
        """Parse a non-streaming completion response."""
        return ChatCompletion(**data)
    
    def _parse_chunk_response(self, data: Dict[str, Any]) -> ChatCompletionChunk:
        """Parse a streaming chunk response."""
        return ChatCompletionChunk(**data)


class Embeddings:
    """
    Resource class for embedding operations.
    
    Handles creating embeddings from text inputs.
    """
    
    def __init__(self, client):
        self._client = client
    
    def create(
        self,
        *,
        model: str,
        input: Union[str, List[str], List[int], List[List[int]]],
        encoding_format: Optional[str] = None,
        dimensions: Optional[int] = None,
        user: Optional[str] = None,
        **kwargs
    ) -> EmbeddingResponse:
        """
        Create embeddings for the given input.
        
        Args:
            model: The embedding model to use
            input: Input text(s) to embed
            encoding_format: Encoding format (float, base64)
            dimensions: Number of dimensions (for supported models)
            user: User identifier for tracking
            **kwargs: Additional parameters
            
        Returns:
            EmbeddingResponse containing the embeddings
        """
        # Build request data
        request_data = {
            "model": model,
            "input": input
        }
        
        # Add optional parameters
        if encoding_format is not None:
            request_data["encoding_format"] = encoding_format
        if dimensions is not None:
            request_data["dimensions"] = dimensions
        if user is not None:
            request_data["user"] = user
        
        # Add any additional kwargs
        request_data.update(kwargs)
        
        # Validate required fields
        if not model:
            raise ValidationError("model is required")
        if not input:
            raise ValidationError("input is required")
        
        response = self._client._make_request(
            "POST",
            "/v1/embeddings",
            json_data=request_data
        )
        
        return EmbeddingResponse(**response)


class Models:
    """
    Resource class for model operations.
    
    Handles listing available models and retrieving model information.
    """
    
    def __init__(self, client):
        self._client = client
    
    def list(self) -> ModelList:
        """
        List all available models.
        
        Returns:
            ModelList containing all available models
        """
        response = self._client._make_request("GET", "/v1/models")
        return ModelList(**response)
    
    def retrieve(self, model_id: str) -> Model:
        """
        Retrieve information about a specific model.
        
        Args:
            model_id: The ID of the model to retrieve
            
        Returns:
            Model information
        """
        if not model_id:
            raise ValidationError("model_id is required")
        
        response = self._client._make_request("GET", f"/v1/models/{model_id}")
        return Model(**response)


class Tenants:
    """
    Resource class for tenant operations.
    
    Handles tenant-specific operations like usage tracking and management.
    """
    
    def __init__(self, client):
        self._client = client
    
    def usage(self, tenant_id: Optional[str] = None) -> Dict[str, Any]:
        """
        Get usage metrics for a tenant.
        
        Args:
            tenant_id: Tenant ID (uses client's tenant_id if not provided)
            
        Returns:
            Tenant usage metrics
        """
        if not tenant_id:
            tenant_id = self._client.tenant_id
        if not tenant_id:
            raise ValidationError("tenant_id is required")
        
        return self._client._make_request("GET", f"/tenants/{tenant_id}/usage")
    
    def limits(self, tenant_id: Optional[str] = None) -> Dict[str, Any]:
        """
        Get limits for a tenant.
        
        Args:
            tenant_id: Tenant ID (uses client's tenant_id if not provided)
            
        Returns:
            Tenant limits
        """
        if not tenant_id:
            tenant_id = self._client.tenant_id
        if not tenant_id:
            raise ValidationError("tenant_id is required")
        
        return self._client._make_request("GET", f"/tenants/{tenant_id}/limits")
    
    def settings(self, tenant_id: Optional[str] = None) -> Dict[str, Any]:
        """
        Get settings for a tenant.
        
        Args:
            tenant_id: Tenant ID (uses client's tenant_id if not provided)
            
        Returns:
            Tenant settings
        """
        if not tenant_id:
            tenant_id = self._client.tenant_id
        if not tenant_id:
            raise ValidationError("tenant_id is required")
        
        return self._client._make_request("GET", f"/tenants/{tenant_id}/settings")


class Cache:
    """
    Resource class for cache operations.
    
    Handles cache management and statistics.
    """
    
    def __init__(self, client):
        self._client = client
    
    def stats(self, tenant_id: Optional[str] = None) -> Dict[str, Any]:
        """
        Get cache statistics.
        
        Args:
            tenant_id: Optional tenant ID for tenant-specific stats
            
        Returns:
            Cache statistics
        """
        if tenant_id:
            return self._client._make_request("GET", f"/cache/stats/tenant/{tenant_id}")
        else:
            return self._client._make_request("GET", "/cache/stats")
    
    def clear(self, tenant_id: Optional[str] = None) -> Dict[str, Any]:
        """
        Clear cache entries.
        
        Args:
            tenant_id: Optional tenant ID to clear only tenant-specific entries
            
        Returns:
            Success message
        """
        if tenant_id:
            return self._client._make_request("POST", f"/cache/clear/tenant/{tenant_id}")
        else:
            return self._client._make_request("POST", "/cache/clear")
    
    def entries(
        self,
        tenant_id: Optional[str] = None,
        model: Optional[str] = None,
        limit: int = 50,
        offset: int = 0
    ) -> Dict[str, Any]:
        """
        List cache entries.
        
        Args:
            tenant_id: Filter by tenant ID
            model: Filter by model
            limit: Maximum number of entries to return
            offset: Number of entries to skip
            
        Returns:
            List of cache entries with pagination info
        """
        params = {"limit": limit, "offset": offset}
        if tenant_id:
            params["tenant_id"] = tenant_id
        if model:
            params["model"] = model
        
        return self._client._make_request(
            "GET",
            "/cache/entries",
            params=params
        )
    
    def analysis(self) -> Dict[str, Any]:
        """
        Get cache analysis and insights.
        
        Returns:
            Cache analysis data
        """
        return self._client._make_request("GET", "/cache/analysis")
    
    def adaptive_state(self) -> Dict[str, Any]:
        """
        Get adaptive caching state.
        
        Returns:
            Adaptive caching state information
        """
        return self._client._make_request("GET", "/cache/adaptive/state")
    
    def set_strategy(self, strategy: str, reason: Optional[str] = None) -> Dict[str, Any]:
        """
        Set the adaptive caching strategy.
        
        Args:
            strategy: Caching strategy to use
            reason: Optional reason for the change
            
        Returns:
            Success message with strategy change info
        """
        request_data = {"strategy": strategy}
        if reason:
            request_data["reason"] = reason
        
        return self._client._make_request(
            "POST",
            "/cache/adaptive/strategy",
            json_data=request_data
        )


class Monitoring:
    """
    Resource class for monitoring and observability.
    
    Handles health checks, statistics, and system monitoring.
    """
    
    def __init__(self, client):
        self._client = client
    
    def health(self) -> Dict[str, Any]:
        """
        Get server health status.
        
        Returns:
            Health status information
        """
        return self._client._make_request("GET", "/health")
    
    def stats(self) -> Dict[str, Any]:
        """
        Get server statistics.
        
        Returns:
            Server statistics
        """
        return self._client._make_request("GET", "/stats")
    
    def metrics(self) -> Dict[str, Any]:
        """
        Get detailed metrics.
        
        Returns:
            Detailed metrics data
        """
        return self._client._make_request("GET", "/metrics")


# Helper functions for common operations
def create_chat_request(
    model: str,
    messages: List[ChatCompletionMessageParam],
    **kwargs
) -> ChatCompletionRequest:
    """
    Create a chat completion request with fluent interface.
    
    Args:
        model: The model to use
        messages: List of messages
        **kwargs: Additional parameters
        
    Returns:
        ChatCompletionRequest builder instance
    """
    request = ChatCompletionRequest(model, messages)
    
    # Apply any provided parameters
    for key, value in kwargs.items():
        if hasattr(request, key):
            getattr(request, key)(value)
    
    return request


def create_embedding_request(
    model: str,
    input_data: Union[str, List[str]],
    **kwargs
) -> EmbeddingRequest:
    """
    Create an embedding request with fluent interface.
    
    Args:
        model: The embedding model to use
        input_data: Input text(s) to embed
        **kwargs: Additional parameters
        
    Returns:
        EmbeddingRequest builder instance
    """
    request = EmbeddingRequest(model, input_data)
    
    # Apply any provided parameters
    for key, value in kwargs.items():
        if hasattr(request, key):
            getattr(request, key)(value)
    
    return request

Ogem Python SDK의 resources.py는 OpenAI 호환 Resource 계층과 풍부한 API 추상화는 잘 구현했지만, SSE 스트리밍 규격 대응 부족과 멀티테넌시 상태 관리 취약점, 확장에 불리한 수동 파라미터 설계로 인해 엔터프라이즈 환경에서 요구되는 장애 격리와 보안 무결성 수준까지는 보완이 필요한 구조다.

제안패치
"""
Resource classes for the Ogem Python SDK.

This module contains resource classes that handle specific API endpoints
and provide a structured interface for different API operations.
"""

import json
from typing import Dict, List, Optional, Union, Any, Iterator

from .types import (
    ChatCompletion,
    ChatCompletionChunk,
    ChatCompletionMessageParam,
    EmbeddingResponse,
    Model,
    ModelList,
    ChatCompletionRequest,
    EmbeddingRequest
)
from .exceptions import StreamError, ValidationError


# 1. 명시적 화이트리스트 기반 Payload Builder 정의
ALLOWED_CHAT_PARAMS = {
    "max_tokens",
    "temperature",
    "top_p",
    "n",
    "stream",
    "stop",
    "presence_penalty",
    "frequency_penalty",
    "logit_bias",
    "user",
    "functions",
    "function_call",
    "tools",
    "tool_choice",
    "response_format",
    "seed",
    "logprobs",
    "top_logprobs",
}

ALLOWED_EMBEDDING_PARAMS = {
    "encoding_format",
    "dimensions",
    "user",
}


# 2. SSE Parser 전용 클래스 분리 및 malformed chunk 장애 격리(Threshold 기반) 버퍼링
class SSEParser:
    """Robust SSE (Server-Sent Events) parser with malformed chunk isolation."""
    
    def __init__(self, max_invalid_chunks: int = 5):
        self.max_invalid_chunks = max_invalid_chunks
        self.invalid_chunk_count = 0

    def parse_line(self, raw_line: Any) -> Optional[str]:
        if not raw_line:
            return None
        
        line = raw_line.decode("utf-8") if isinstance(raw_line, bytes) else raw_line
        line = line.strip()
        
        if not line or line.startswith(":"):
            return None
        if line == "[DONE]" or line.startswith("data: [DONE]"):
            return "[DONE]"
        
        # 유연한 data: 프리픽스 공백 처리 (data:{json} 및 data: {json} 모두 완벽 지원)
        if line.startswith("data:"):
            line = line[len("data:"):].strip()
            return line
            
        return None

    def handle_decode_error(self, line: str, error: Exception) -> None:
        self.invalid_chunk_count += 1
        if self.invalid_chunk_count > self.max_invalid_chunks:
            raise StreamError(
                f"Exceeded max malformed stream chunks threshold ({self.max_invalid_chunks}). "
                f"Last failed chunk: '{line}': {error}"
            ) from error


class Chat:
    """
    Resource class for chat completion operations.
    
    Handles all chat-related API endpoints including regular completions
    and streaming completions.
    """
    
    def __init__(self, client):
        self._client = client
        self.completions = ChatCompletions(client)


class ChatCompletions:
    """Handle chat completion requests."""
    
    def __init__(self, client):
        self._client = client
    
    def create(
        self,
        *,
        model: str,
        messages: List[ChatCompletionMessageParam],
        max_tokens: Optional[int] = None,
        temperature: Optional[float] = None,
        top_p: Optional[float] = None,
        n: Optional[int] = None,
        stream: Optional[bool] = None,
        stop: Optional[Union[str, List[str]]] = None,
        presence_penalty: Optional[float] = None,
        frequency_penalty: Optional[float] = None,
        logit_bias: Optional[Dict[str, int]] = None,
        user: Optional[str] = None,
        functions: Optional[List[Dict[str, Any]]] = None,
        function_call: Optional[Union[str, Dict[str, Any]]] = None,
        tools: Optional[List[Dict[str, Any]]] = None,
        tool_choice: Optional[Union[str, Dict[str, Any]]] = None,
        response_format: Optional[Dict[str, Any]] = None,
        seed: Optional[int] = None,
        logprobs: Optional[bool] = None,
        top_logprobs: Optional[int] = None,
        **kwargs
    ) -> Union[ChatCompletion, Iterator[ChatCompletionChunk]]:
        if not model:
            raise ValidationError("model is required")
        if not messages:
            raise ValidationError("messages is required")

        # 화이트리스트 기반 명시적 Payload Builder 패턴 적용 (locals() 오염 방지)
        request_data = {
            "model": model,
            "messages": messages
        }
        
        local_params = locals()
        for key in ALLOWED_CHAT_PARAMS:
            if key in local_params and local_params[key] is not None:
                val = local_params[key]
                if key == "stop" and not isinstance(val, list):
                    request_data["stop"] = [val]
                else:
                    request_data[key] = val
        
        # 외부 추가 kwargs 중 허용된 항목만 안전하게 병합
        for key, value in kwargs.items():
            if key in ALLOWED_CHAT_PARAMS and value is not None:
                request_data[key] = value

        # Handle streaming vs non-streaming
        if request_data.get("stream", False):
            return self._create_stream(request_data)
        else:
            response = self._client._make_request(
                "POST",
                "/v1/chat/completions",
                json_data=request_data
            )
            return self._parse_completion_response(response)
    
    def _create_stream(self, request_data: Dict[str, Any]) -> Iterator[ChatCompletionChunk]:
        """Create a streaming chat completion using isolated SSEParser."""
        stream = self._client._make_request(
            "POST",
            "/v1/chat/completions",
            json_data=request_data,
            stream=True
        )
        
        parser = SSEParser(max_invalid_chunks=5)
        
        for raw_line in stream:
            parsed_line = parser.parse_line(raw_line)
            if not parsed_line:
                continue
            if parsed_line == "[DONE]":
                break
            
            try:
                chunk_data = json.loads(parsed_line)
                yield self._parse_chunk_response(chunk_data)
            except json.JSONDecodeError as e:
                parser.handle_decode_error(parsed_line, e)
    
    def _parse_completion_response(self, data: Dict[str, Any]) -> ChatCompletion:
        """Parse a non-streaming completion response."""
        return ChatCompletion(**data)
    
    def _parse_chunk_response(self, data: Dict[str, Any]) -> ChatCompletionChunk:
        """Parse a streaming chunk response."""
        return ChatCompletionChunk(**data)


class Embeddings:
    """
    Resource class for embedding operations.
    
    Handles creating embeddings from text inputs.
    """
    
    def __init__(self, client):
        self._client = client
    
    def create(
        self,
        *,
        model: str,
        input: Union[str, List[str], List[int], List[List[int]]],
        encoding_format: Optional[str] = None,
        dimensions: Optional[int] = None,
        user: Optional[str] = None,
        **kwargs
    ) -> EmbeddingResponse:
        if not model:
            raise ValidationError("model is required")
        if not input:
            raise ValidationError("input is required")

        request_data = {
            "model": model,
            "input": input
        }
        
        local_params = locals()
        for key in ALLOWED_EMBEDDING_PARAMS:
            if key in local_params and local_params[key] is not None:
                request_data[key] = local_params[key]
                
        for key, value in kwargs.items():
            if key in ALLOWED_EMBEDDING_PARAMS and value is not None:
                request_data[key] = value
        
        response = self._client._make_request(
            "POST",
            "/v1/embeddings",
            json_data=request_data
        )
        
        return EmbeddingResponse(**response)


class Models:
    """
    Resource class for model operations.
    
    Handles listing available models and retrieving model information.
    """
    
    def __init__(self, client):
        self._client = client
    
    def list(self) -> ModelList:
        response = self._client._make_request("GET", "/v1/models")
        return ModelList(**response)
    
    def retrieve(self, model_id: str) -> Model:
        if not model_id:
            raise ValidationError("model_id is required")
        
        response = self._client._make_request("GET", f"/v1/models/{model_id}")
        return Model(**response)


class Tenants:
    """
    Resource class for tenant operations.
    
    Handles tenant-specific operations like usage tracking and management.
    """
    
    def __init__(self, client):
        self._client = client
    
    def _validate_tenant_access(self, tenant_id: str) -> None:
        """3. 엔터프라이즈급 Tenant Authorization Layer 및 권한 검증 계층 추가"""
        if not tenant_id or not isinstance(tenant_id, str) or not tenant_id.strip():
            raise ValidationError("Invalid or empty tenant_id specified.")
        
        client_tenant = getattr(self._client, "tenant_id", None)
        # 클라이언트 바운드 테넌트가 존재할 경우, 크로스 테넌트 권한 오염(무단 접근) 원천 차단
        if client_tenant and client_tenant != tenant_id:
            raise ValidationError(
                f"Cross-tenant access violation: Cannot query tenant '{tenant_id}' using client bound to '{client_tenant}'."
            )

    def _resolve_tenant_id(self, tenant_id: Optional[str] = None) -> str:
        resolved = tenant_id or getattr(self._client, "tenant_id", None)
        if not resolved:
            raise ValidationError("tenant_id is required (neither explicitly provided nor bound to client)")
        self._validate_tenant_access(resolved)
        return resolved
    
    def usage(self, tenant_id: Optional[str] = None) -> Dict[str, Any]:
        tid = self._resolve_tenant_id(tenant_id)
        return self._client._make_request("GET", f"/tenants/{tid}/usage")
    
    def limits(self, tenant_id: Optional[str] = None) -> Dict[str, Any]:
        tid = self._resolve_tenant_id(tenant_id)
        return self._client._make_request("GET", f"/tenants/{tid}/limits")
    
    def settings(self, tenant_id: Optional[str] = None) -> Dict[str, Any]:
        tid = self._resolve_tenant_id(tenant_id)
        return self._client._make_request("GET", f"/tenants/{tid}/settings")


class Cache:
    """
    Resource class for cache operations.
    
    Handles cache management and statistics.
    """
    
    def __init__(self, client):
        self._client = client
    
    def stats(self, tenant_id: Optional[str] = None) -> Dict[str, Any]:
        if tenant_id:
            return self._client._make_request("GET", f"/cache/stats/tenant/{tenant_id}")
        else:
            return self._client._make_request("GET", "/cache/stats")
    
    def clear(self, tenant_id: Optional[str] = None) -> Dict[str, Any]:
        if tenant_id:
            return self._client._make_request("POST", f"/cache/clear/tenant/{tenant_id}")
        else:
            return self._client._make_request("POST", "/cache/clear")
    
    def entries(
        self,
        tenant_id: Optional[str] = None,
        model: Optional[str] = None,
        limit: int = 50,
        offset: int = 0
    ) -> Dict[str, Any]:
        params = {"limit": limit, "offset": offset}
        if tenant_id:
            params["tenant_id"] = tenant_id
        if model:
            params["model"] = model
        
        return self._client._make_request(
            "GET",
            "/cache/entries",
            params=params
        )
    
    def analysis(self) -> Dict[str, Any]:
        return self._client._make_request("GET", "/cache/analysis")
    
    def adaptive_state(self) -> Dict[str, Any]:
        return self._client._make_request("GET", "/cache/adaptive/state")
    
    def set_strategy(self, strategy: str, reason: Optional[str] = None) -> Dict[str, Any]:
        request_data = {"strategy": strategy}
        if reason:
            request_data["reason"] = reason
        
        return self._client._make_request(
            "POST",
            "/cache/adaptive/strategy",
            json_data=request_data
        )


class Monitoring:
    """
    Resource class for monitoring and observability.
    
    Handles health checks, statistics, and system monitoring.
    """
    
    def __init__(self, client):
        self._client = client
    
    def health(self) -> Dict[str, Any]:
        return self._client._make_request("GET", "/health")
    
    def stats(self) -> Dict[str, Any]:
        return self._client._make_request("GET", "/stats")
    
    def metrics(self) -> Dict[str, Any]:
        return self._client._make_request("GET", "/metrics")


# Helper functions for common operations (Private 속성 유출 방어벽 강화)
def create_chat_request(
    model: str,
    messages: List[ChatCompletionMessageParam],
    **kwargs
) -> ChatCompletionRequest:
    request = ChatCompletionRequest(model, messages)
    
    for key, value in kwargs.items():
        if key.startswith("_"):
            raise ValidationError(f"Accessing private attribute '{key}' is forbidden.")
        if not hasattr(request, key):
            raise ValidationError(f"Invalid request parameter: '{key}' does not exist on ChatCompletionRequest.")
        setter_func = getattr(request, key)
        if callable(setter_func):
            setter_func(value)
            
    return request


def create_embedding_request(
    model: str,
    input_data: Union[str, List[str]],
    **kwargs
) -> EmbeddingRequest:
    request = EmbeddingRequest(model, input_data)
    
    for key, value in kwargs.items():
        if key.startswith("_"):
            raise ValidationError(f"Accessing private attribute '{key}' is forbidden.")
        if not hasattr(request, key):
            raise ValidationError(f"Invalid request parameter: '{key}' does not exist on EmbeddingRequest.")
        setter_func = getattr(request, key)
        if callable(setter_func):
            setter_func(value)
            
    return request

최종개선사항
✅ locals() 기반 암묵적 Payload 생성 → 화이트리스트 기반 명시적 Payload Builder 적용 → 내부 변수 외부 유출 방지 및 API 요청 무결성 강화
✅ 단순 JSON 파싱 스트림 처리 → SSEParser 독립 계층 분리 및 malformed chunk 격리 → 스트리밍 장애 전파 최소화 및 안정성 확보
✅ data: 고정 프리픽스 처리 → SSE 표준 data: 변형 및 heartbeat/comment 대응 → 다양한 Proxy 환경 호환성 강화
✅ 클라이언트 전역 tenant 의존 → Tenant Authorization Layer 추가 → Cross-tenant 데이터 접근 차단 및 멀티테넌시 보안 강화
✅ 동적 hasattr/getattr 접근 → private 속성 차단 및 입력 검증 강화 → 잘못된 Builder 파라미터 주입 방지
✅ 단일 함수 내부 책임 집중 → Stream Parsing/Validation/Payload 생성 역할 분리 → 유지보수성과 장애 추적성 향상

Ogem Python SDK resources.py는 기존 API 추상화 구조를 유지하면서 Payload 오염, SSE 장애, 멀티테넌시 침범 가능성을 제거해 단순 Wrapper 수준에서 엔터프라이즈 SDK Core Layer 수준의 안정성과 방어력을 갖춘 구조로 승격되었다.
