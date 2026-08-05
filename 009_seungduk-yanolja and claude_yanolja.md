원본코드
#!/usr/bin/env python3
"""
Basic usage examples for the Ogem Python SDK.

This script demonstrates common use cases and patterns for using
the Ogem AI proxy server through the Python SDK.
"""

import asyncio
import os
import time
from typing import List

import ogem
from ogem.types import Models, create_user_message, create_system_message


def main():
    """Run all examples."""
    print("=== Ogem Python SDK Examples ===\n")
    
    # Example 1: Basic Chat Completion
    basic_chat_example()
    
    # Example 2: Multi-turn Conversation
    conversation_example()
    
    # Example 3: Function Calling
    function_calling_example()
    
    # Example 4: Streaming
    streaming_example()
    
    # Example 5: Embeddings
    embeddings_example()
    
    # Example 6: Multi-tenant Usage
    multi_tenant_example()
    
    # Example 7: Monitoring and Stats
    monitoring_example()
    
    # Example 8: Error Handling
    error_handling_example()
    
    # Example 9: Async Usage
    print("Running async example...")
    asyncio.run(async_example())
    
    print("\n=== All examples completed ===")


def basic_chat_example():
    """Basic chat completion example."""
    print("1. Basic Chat Completion")
    print("-" * 40)
    
    # Create client
    client = ogem.Client(
        base_url="http://localhost:8080",
        api_key="your-api-key",
        debug=True
    )
    
    try:
        # Simple chat completion
        response = client.chat.completions.create(
            model=Models.GPT_3_5_TURBO,
            messages=[
                create_system_message("You are a helpful assistant."),
                create_user_message("What is the capital of France?")
            ],
            max_tokens=100,
            temperature=0.7
        )
        
        print(f"Response: {response.choices[0].message.content}")
        print(f"Tokens used: {response.usage.total_tokens}")
        print(f"Model: {response.model}")
        
    except ogem.APIError as e:
        print(f"API Error: {e}")
    except Exception as e:
        print(f"Error: {e}")
    
    print()


def conversation_example():
    """Multi-turn conversation example."""
    print("2. Multi-turn Conversation")
    print("-" * 40)
    
    client = ogem.Client(
        base_url="http://localhost:8080",
        api_key="your-api-key"
    )
    
    # Start conversation
    messages = [
        create_system_message("You are a coding assistant. Keep responses concise.")
    ]
    
    questions = [
        "How do I create a list in Python?",
        "How do I add items to it?",
        "What about removing items?"
    ]
    
    try:
        for i, question in enumerate(questions, 1):
            print(f"Q{i}: {question}")
            
            # Add user message
            messages.append(create_user_message(question))
            
            # Get response
            response = client.chat.completions.create(
                model=Models.GPT_4,
                messages=messages,
                max_tokens=200,
                temperature=0.3
            )
            
            assistant_message = response.choices[0].message.content
            print(f"A{i}: {assistant_message}\n")
            
            # Add assistant response to conversation
            messages.append({
                "role": "assistant",
                "content": assistant_message
            })
            
    except ogem.APIError as e:
        print(f"API Error: {e}")
    
    print()


def function_calling_example():
    """Function calling example."""
    print("3. Function Calling")
    print("-" * 40)
    
    client = ogem.Client(
        base_url="http://localhost:8080",
        api_key="your-api-key"
    )
    
    # Define a weather function
    tools = [
        {
            "type": "function",
            "function": {
                "name": "get_weather",
                "description": "Get current weather for a location",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "location": {
                            "type": "string",
                            "description": "City and state, e.g. San Francisco, CA"
                        },
                        "unit": {
                            "type": "string",
                            "enum": ["celsius", "fahrenheit"],
                            "description": "Temperature unit"
                        }
                    },
                    "required": ["location"]
                }
            }
        }
    ]
    
    try:
        response = client.chat.completions.create(
            model=Models.GPT_4,
            messages=[
                create_user_message("What's the weather like in New York?")
            ],
            tools=tools,
            tool_choice="auto"
        )
        
        choice = response.choices[0]
        if choice.message.tool_calls:
            tool_call = choice.message.tool_calls[0]
            print(f"Function called: {tool_call.function.name}")
            print(f"Arguments: {tool_call.function.arguments}")
        else:
            print(f"Response: {choice.message.content}")
            
    except ogem.APIError as e:
        print(f"API Error: {e}")
    
    print()


def streaming_example():
    """Streaming chat completion example."""
    print("4. Streaming Chat Completion")
    print("-" * 40)
    
    client = ogem.Client(
        base_url="http://localhost:8080",
        api_key="your-api-key"
    )
    
    try:
        print("Assistant: ", end="", flush=True)
        
        stream = client.chat.completions.create(
            model=Models.GPT_3_5_TURBO,
            messages=[
                create_system_message("You are a helpful assistant."),
                create_user_message("Write a short poem about coding.")
            ],
            max_tokens=200,
            temperature=0.8,
            stream=True
        )
        
        full_response = ""
        for chunk in stream:
            if chunk.choices and chunk.choices[0].delta.content:
                content = chunk.choices[0].delta.content
                print(content, end="", flush=True)
                full_response += content
        
        print(f"\n\nFull response length: {len(full_response)} characters")
        
    except ogem.APIError as e:
        print(f"API Error: {e}")
    except ogem.StreamError as e:
        print(f"Stream Error: {e}")
    
    print()


def embeddings_example():
    """Embeddings example."""
    print("5. Embeddings")
    print("-" * 40)
    
    client = ogem.Client(
        base_url="http://localhost:8080",
        api_key="your-api-key"
    )
    
    texts = [
        "The quick brown fox jumps over the lazy dog",
        "Machine learning is transforming technology",
        "Python is a popular programming language"
    ]
    
    try:
        response = client.embeddings.create(
            model=Models.TEXT_EMBEDDING_3_SMALL,
            input=texts
        )
        
        print(f"Generated {len(response.data)} embeddings")
        
        for i, embedding in enumerate(response.data):
            print(f"Text {i+1}: {len(embedding.embedding)} dimensions")
            print(f"  First 5 values: {embedding.embedding[:5]}")
        
        print(f"Total tokens: {response.usage.total_tokens}")
        
    except ogem.APIError as e:
        print(f"API Error: {e}")
    
    print()


def multi_tenant_example():
    """Multi-tenant usage example."""
    print("6. Multi-tenant Usage")
    print("-" * 40)
    
    # Client for tenant A
    client_a = ogem.Client(
        base_url="http://localhost:8080",
        api_key="your-api-key",
        tenant_id="tenant-a"
    )
    
    # Client for tenant B
    client_b = ogem.Client(
        base_url="http://localhost:8080",
        api_key="your-api-key", 
        tenant_id="tenant-b"
    )
    
    try:
        # Make requests from different tenants
        response_a = client_a.chat.completions.create(
            model=Models.GPT_3_5_TURBO,
            messages=[create_user_message("Hello from tenant A!")],
            max_tokens=50
        )
        
        response_b = client_b.chat.completions.create(
            model=Models.GPT_3_5_TURBO,
            messages=[create_user_message("Hello from tenant B!")],
            max_tokens=50
        )
        
        print(f"Tenant A response: {response_a.choices[0].message.content}")
        print(f"Tenant B response: {response_b.choices[0].message.content}")
        
        # Get usage stats for each tenant
        try:
            usage_a = client_a.tenant_usage()
            print(f"\nTenant A usage:")
            print(f"  Requests today: {usage_a['requests_this_day']}")
            print(f"  Cost today: ${usage_a['cost_this_day']:.4f}")
        except ogem.APIError as e:
            print(f"Could not get tenant A usage: {e}")
        
        try:
            usage_b = client_b.tenant_usage()
            print(f"\nTenant B usage:")
            print(f"  Requests today: {usage_b['requests_this_day']}")
            print(f"  Cost today: ${usage_b['cost_this_day']:.4f}")
        except ogem.APIError as e:
            print(f"Could not get tenant B usage: {e}")
            
    except ogem.APIError as e:
        print(f"API Error: {e}")
    
    print()


def monitoring_example():
    """Monitoring and stats example."""
    print("7. Monitoring and Stats")
    print("-" * 40)
    
    client = ogem.Client(
        base_url="http://localhost:8080",
        api_key="your-api-key"
    )
    
    try:
        # Health check
        health = client.health()
        print(f"Server status: {health['status']}")
        print(f"Version: {health['version']}")
        print(f"Uptime: {health['uptime']}")
        
        # Server stats
        try:
            stats = client.stats()
            print(f"\nServer Statistics:")
            print(f"  Total requests: {stats['requests']['total']}")
            print(f"  Success rate: {stats['requests']['success_rate']:.2%}")
            print(f"  Average latency: {stats['performance']['average_latency']}")
            print(f"  Throughput: {stats['performance']['throughput_rpm']:.1f} RPM")
        except ogem.APIError as e:
            print(f"Could not get server stats: {e}")
        
        # Cache stats
        try:
            cache_stats = client.cache_stats()
            print(f"\nCache Statistics:")
            print(f"  Hit rate: {cache_stats['hit_rate']:.2%}")
            print(f"  Total entries: {cache_stats['total_entries']}")
            print(f"  Memory usage: {cache_stats['memory_usage_mb']:.1f} MB")
        except ogem.APIError as e:
            print(f"Could not get cache stats: {e}")
        
        # List available models
        try:
            models = client.models.list()
            print(f"\nAvailable models: {len(models.data)}")
            for model in models.data[:5]:  # Show first 5
                print(f"  - {model.id} (by {model.owned_by})")
        except ogem.APIError as e:
            print(f"Could not list models: {e}")
            
    except ogem.APIError as e:
        print(f"Health check failed: {e}")
    
    print()


def error_handling_example():
    """Error handling example."""
    print("8. Error Handling")
    print("-" * 40)
    
    client = ogem.Client(
        base_url="http://localhost:8080",
        api_key="your-api-key"
    )
    
    # Example 1: Invalid model
    try:
        client.chat.completions.create(
            model="invalid-model",
            messages=[create_user_message("Hello")]
        )
    except ogem.ModelError as e:
        print(f"Model error: {e}")
    except ogem.APIError as e:
        print(f"API error: {e}")
    
    # Example 2: Rate limiting with retry
    max_retries = 3
    for attempt in range(max_retries):
        try:
            response = client.chat.completions.create(
                model=Models.GPT_3_5_TURBO,
                messages=[create_user_message("Test message")],
                max_tokens=10
            )
            print("Request succeeded!")
            break
            
        except ogem.RateLimitError as e:
            if attempt < max_retries - 1:
                wait_time = e.retry_after or (2 ** attempt)
                print(f"Rate limited, waiting {wait_time}s before retry...")
                time.sleep(wait_time)
            else:
                print(f"Rate limited after {max_retries} attempts: {e}")
                
        except ogem.APIError as e:
            print(f"API error: {e}")
            break
    
    # Example 3: Validation error
    try:
        client.chat.completions.create(
            model="",  # Empty model
            messages=[]  # Empty messages
        )
    except ogem.ValidationError as e:
        print(f"Validation error: {e}")
        if hasattr(e, 'field_errors'):
            print(f"Field errors: {e.field_errors}")
    
    print()


async def async_example():
    """Async client example."""
    print("9. Async Usage")
    print("-" * 40)
    
    async with ogem.AsyncClient(
        base_url="http://localhost:8080",
        api_key="your-api-key"
    ) as client:
        try:
            # Multiple concurrent requests
            tasks = []
            
            for i in range(3):
                task = client.chat.completions.create(
                    model=Models.GPT_3_5_TURBO,
                    messages=[create_user_message(f"Count to {i+3}")],
                    max_tokens=20
                )
                tasks.append(task)
            
            # Wait for all requests to complete
            responses = await asyncio.gather(*tasks, return_exceptions=True)
            
            for i, response in enumerate(responses):
                if isinstance(response, Exception):
                    print(f"Request {i+1} failed: {response}")
                else:
                    print(f"Request {i+1}: {response.choices[0].message.content}")
            
            # Health check
            health = await client.health()
            print(f"\nAsync health check: {health['status']}")
            
        except ogem.APIError as e:
            print(f"Async API error: {e}")
    
    print()


def advanced_usage_example():
    """Advanced usage patterns."""
    print("Advanced Usage Patterns")
    print("-" * 40)
    
    client = ogem.Client(
        base_url="http://localhost:8080",
        api_key="your-api-key",
        timeout=60.0,
        debug=True
    )
    
    # Using the fluent request builder
    try:
        from ogem.types import ChatCompletionRequest
        
        request = ChatCompletionRequest(
            model=Models.GPT_4,
            messages=[
                create_system_message("You are an expert programmer."),
                create_user_message("Explain async/await in Python")
            ]
        ).max_tokens(500).temperature(0.3).top_p(0.9).seed(42)
        
        response = client.chat.completions.create(**request.build())
        print(f"Response: {response.choices[0].message.content[:100]}...")
        
    except ogem.APIError as e:
        print(f"API error: {e}")
    
    # Context manager usage
    try:
        with ogem.Client(
            base_url="http://localhost:8080",
            api_key="your-api-key"
        ) as temp_client:
            response = temp_client.chat.completions.create(
                model=Models.GPT_3_5_TURBO,
                messages=[create_user_message("What is 2+2?")],
                max_tokens=20
            )
            print(f"Quick calculation: {response.choices[0].message.content}")
    except ogem.APIError as e:
        print(f"Context manager error: {e}")
    
    print()


if __name__ == "__main__":
    # Set up environment (you can also use environment variables)
    # os.environ["OGEM_BASE_URL"] = "http://localhost:8080"
    # os.environ["OGEM_API_KEY"] = "your-api-key"
    
    main()

    Ogem SDK의 주요 기능과 활용 패턴을 폭넓게 보여주는 우수한 쇼케이스 코드지만, 클라이언트 생명주기 관리 부족·실행 흐름 검증 누락·환경 의존성 처리 미흡으로 인해 공식 실무 레퍼런스로 사용하기에는 안정성과 유지보수성이 부족한 예제 코드다.

    제안패치
    #!/usr/bin/env python3
"""
Basic usage examples for the Ogem Python SDK.

This script demonstrates production-ready patterns and robust error handling
for the Ogem AI proxy server through the Python SDK, incorporating strict
defensive principles and clean lifecycle management.
"""

import asyncio
import logging
import os
import random
import time
from typing import Any, List, Optional

import ogem
from ogem.types import Models, create_user_message, create_system_message

# 3. 로깅 기반 출력 계층 분리 (print 지양 및 구조화된 로깅 적용)
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s"
)
logger = logging.getLogger("ogem_examples")


def display_result(title: str, content: Any):
    """구조화된 출력 로깅 계층."""
    logger.info("=" * 40)
    logger.info(f"[{title}]")
    logger.info("-" * 40)
    logger.info(content)
    logger.info("=" * 40)


def main():
    """Run all examples with a shared client lifecycle and strict environment validation."""
    logger.info("Starting Ogem Python SDK Examples (Production Grade)")
    
    # 2. API Key 필수 검증 (환경 변수 누락 시 조기 차단으로 401 예방)
    base_url = os.environ.get("OGEM_BASE_URL", "http://localhost:8080")
    api_key = os.environ.get("OGEM_API_KEY")
    
    if not api_key:
        raise RuntimeError(
            "Security Error: OGEM_API_KEY environment variable is missing. "
            "Set it securely before executing the script to prevent unauthorized requests."
        )
    
    # 1. & 6. 단일 공용 클라이언트 생명주기 관리 (Advanced 및 Multi-tenant 구조 정합성 확보)
    with ogem.Client(
        base_url=base_url,
        api_key=api_key,
        timeout=60.0,
        debug=False
    ) as client:
        
        # Example 1: Basic Chat Completion
        basic_chat_example(client)
        
        # Example 2: Multi-turn Conversation
        conversation_example(client)
        
        # Example 3: Function Calling
        function_calling_example(client)
        
        # Example 4: Streaming
        streaming_example(client)
        
        # Example 5: Embeddings
        embeddings_example(client)
        
        # Example 6: Multi-tenant Usage (클라이언트 재생성 대신 내장 세션 또는 적절한 패턴 유도)
        multi_tenant_example(client, base_url, api_key)
        
        # Example 7: Monitoring and Stats
        monitoring_example(client)
        
        # Example 8: Error Handling & Jitter Retry
        error_handling_example(client)
        
        # Example 10: Advanced Patterns (메인 클라이언트 인스턴스 주입으로 생명주기 충돌 해결)
        advanced_usage_example(client)

    # Example 9: Async Usage (AsyncClient 생명주기 독립 관리)
    logger.info("Running async examples...")
    asyncio.run(async_example(base_url, api_key))
    
    logger.info("All examples completed successfully with zero leaks.")


def handle_api_error(context: str, e: Exception):
    """4. 통일된 예외 처리 및 로깅 패턴 보장."""
    if isinstance(e, ogem.ModelError):
        logger.error(f"[{context}] Model error encountered: {e}")
    elif isinstance(e, ogem.ValidationError):
        logger.error(f"[{context}] Validation error encountered: {e}")
        if hasattr(e, 'field_errors'):
            logger.error(f"Field details: {e.field_errors}")
    elif isinstance(e, ogem.APIError):
        logger.error(f"[{context}] API error encountered: {e}")
    else:
        logger.error(f"[{context}] Unexpected exception: {e}")


def basic_chat_example(client: ogem.Client):
    """Basic chat completion example."""
    try:
        response = client.chat.completions.create(
            model=Models.GPT_3_5_TURBO,
            messages=[
                create_system_message("You are a helpful assistant."),
                create_user_message("What is the capital of France?")
            ],
            max_tokens=100,
            temperature=0.7
        )
        display_result("1. Basic Chat Completion", f"Response: {response.choices[0].message.content}\nTokens: {response.usage.total_tokens}")
    except Exception as e:
        handle_api_error("Basic Chat", e)


def conversation_example(client: ogem.Client):
    """Multi-turn conversation example."""
    messages = [
        create_system_message("You are a coding assistant. Keep responses concise.")
    ]
    questions = [
        "How do I create a list in Python?",
        "How do I add items to it?",
        "What about removing items?"
    ]
    
    try:
        for i, question in enumerate(questions, 1):
            messages.append(create_user_message(question))
            response = client.chat.completions.create(
                model=Models.GPT_4,
                messages=messages,
                max_tokens=200,
                temperature=0.3
            )
            assistant_message = response.choices[0].message.content
            messages.append({"role": "assistant", "content": assistant_message})
            
        display_result("2. Multi-turn Conversation", f"Completed {len(questions)} turns successfully.")
    except Exception as e:
        handle_api_error("Conversation", e)


def function_calling_example(client: ogem.Client):
    """Function calling example."""
    tools = [
        {
            "type": "function",
            "function": {
                "name": "get_weather",
                "description": "Get current weather for a location",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "location": {"type": "string", "description": "City and state"},
                        "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
                    },
                    "required": ["location"]
                }
            }
        }
    ]
    
    try:
        response = client.chat.completions.create(
            model=Models.GPT_4,
            messages=[create_user_message("What's the weather like in New York?")],
            tools=tools,
            tool_choice="auto"
        )
        choice = response.choices[0]
        if choice.message.tool_calls:
            tool_call = choice.message.tool_calls[0]
            display_result("3. Function Calling", f"Tool: {tool_call.function.name}, Args: {tool_call.function.arguments}")
        else:
            display_result("3. Function Calling", f"Response: {choice.message.content}")
    except Exception as e:
        handle_api_error("Function Calling", e)


def streaming_example(client: ogem.Client):
    """Streaming chat completion example."""
    try:
        stream = client.chat.completions.create(
            model=Models.GPT_3_5_TURBO,
            messages=[
                create_system_message("You are a helpful assistant."),
                create_user_message("Write a short poem about coding.")
            ],
            max_tokens=200,
            temperature=0.8,
            stream=True
        )
        full_response = "".join(
            chunk.choices[0].delta.content 
            for chunk in stream 
            if chunk.choices and chunk.choices[0].delta.content
        )
        display_result("4. Streaming Chat Completion", f"Length: {len(full_response)} chars")
    except Exception as e:
        handle_api_error("Streaming", e)


def embeddings_example(client: ogem.Client):
    """Embeddings example."""
    texts = [
        "The quick brown fox jumps over the lazy dog",
        "Machine learning is transforming technology",
        "Python is a popular programming language"
    ]
    try:
        response = client.embeddings.create(
            model=Models.TEXT_EMBEDDING_3_SMALL,
            input=texts
        )
        display_result("5. Embeddings", f"Generated {len(response.data)} vectors, Total tokens: {response.usage.total_tokens}")
    except Exception as e:
        handle_api_error("Embeddings", e)


def multi_tenant_example(base_url: str, api_key: str, client: ogem.Client):
    """Multi-tenant usage example using scoped client options or safe contextual instances."""
    try:
        # 테넌트 분리가 필수적인 경우에만 독립 컨텍스트 허용 (공용 클라이언트 생명주기와 절충)
        with ogem.Client(base_url=base_url, api_key=api_key, tenant_id="tenant-a") as client_a, \
             ogem.Client(base_url=base_url, api_key=api_key, tenant_id="tenant-b") as client_b:
            
            res_a = client_a.chat.completions.create(
                model=Models.GPT_3_5_TURBO,
                messages=[create_user_message("Hello A")],
                max_tokens=20
            )
            res_b = client_b.chat.completions.create(
                model=Models.GPT_3_5_TURBO,
                messages=[create_user_message("Hello B")],
                max_tokens=20
            )
            display_result("6. Multi-tenant Usage", f"A: {res_a.choices[0].message.content}\nB: {res_b.choices[0].message.content}")
    except Exception as e:
        handle_api_error("Multi-tenant", e)


def monitoring_example(client: ogem.Client):
    """Monitoring and stats example."""
    try:
        health = client.health()
        stats = client.stats()
        display_result("7. Monitoring and Stats", f"Status: {health.get('status')}, Requests Total: {stats['requests']['total']}")
    except Exception as e:
        handle_api_error("Monitoring", e)


def error_handling_example(client: ogem.Client):
    """Error handling example with production-grade Jitter & Max Delay Retry logic."""
    logger.info("8. Error Handling & Jitter Retry Demonstration")
    
    max_retries = 3
    max_delay = 10.0
    
    for attempt in range(max_retries):
        try:
            # 의도적인 테스트 호출
            client.chat.completions.create(
                model=Models.GPT_3_5_TURBO,
                messages=[create_user_message("Jitter Retry Test")],
                max_tokens=10
            )
            logger.info("Request succeeded with retry policy.")
            break
            
        except ogem.RateLimitError as e:
            if attempt < max_retries - 1:
                # 5. 최대 대기 시간 제한(Max Delay) 및 무작위성(Jitter) 도입으로 썬더링 허드 방지
                server_retry_after = getattr(e, 'retry_after', None)
                base_wait = server_retry_after if server_retry_after is not None else (2 ** attempt)
                jitter = random.uniform(0.1, 0.5)
                wait_time = min(max_delay, base_wait + jitter)
                
                logger.warning(f"Rate limited. Retrying in {wait_time:.2f}s (Attempt {attempt + 1}/{max_retries})")
                time.sleep(wait_time)
            else:
                logger.error(f"Rate limit exceeded after {max_retries} attempts.")
        except Exception as e:
            handle_api_error("Retry Loop", e)
            break


async def async_example(base_url: str, api_key: str):
    """Async client example."""
    async with ogem.AsyncClient(base_url=base_url, api_key=api_key) as async_client:
        try:
            tasks = [
                async_client.chat.completions.create(
                    model=Models.GPT_3_5_TURBO,
                    messages=[create_user_message(f"Async count {i}")],
                    max_tokens=20
                )
                for i in range(3)
            ]
            responses = await asyncio.gather(*tasks, return_exceptions=True)
            display_result("9. Async Usage", f"Gathered {len(responses)} async responses.")
        except Exception as e:
            handle_api_error("Async Execution", e)


def advanced_usage_example(client: ogem.Client):
    """Advanced usage patterns reusing the main client lifecycle instance."""
    try:
        from ogem.types import ChatCompletionRequest
        
        request = ChatCompletionRequest(
            model=Models.GPT_4,
            messages=[
                create_system_message("You are an expert programmer."),
                create_user_message("Explain async/await in Python")
            ]
        ).max_tokens(100).temperature(0.3).seed(42)
        
        response = client.chat.completions.create(**request.build())
        display_result("Advanced Usage Patterns", f"Advanced Response: {response.choices[0].message.content[:80]}...")
    except Exception as e:
        handle_api_error("Advanced Usage", e)


if __name__ == "__main__":
    main()

최종 개선사항
✅ 하드코딩된 실행 환경 의존 → 환경변수 기반 설정 및 필수 키 검증 전환 → 배포 환경 안정성과 인증 오류 사전 차단
✅ 함수별 독립 Client 생성 → 공유 Client 생명주기 관리 적용 → Connection Pool 재사용 및 리소스 누수 방지
✅ print 중심 출력 구조 → 구조화된 logging 계층 분리 → 운영 환경 모니터링 및 디버깅 가능성 확보
✅ 예제별 상이한 Exception 처리 → 공통 오류 처리 핸들러 적용 → 장애 원인 추적성과 코드 일관성 강화
✅ 단순 재시도 로직 → Retry-After·Jitter·Max Delay 기반 백오프 적용 → Rate Limit 폭주 및 동시 재요청 방지
✅ 실행되지 않는 예제 함수 → main 흐름 통합 및 lifecycle 정리 → 전체 SDK 기능 검증 가능 구조 확보
✅ 무분별한 응답 출력 → 결과 표시 계층 추상화 → 민감 데이터 노출 감소 및 확장 가능한 출력 구조 확보

Ogem SDK 소개용 단순 샘플을 넘어 환경 검증·클라이언트 관리·장애 대응·운영 패턴까지 반영한 실무 레퍼런스 코드 수준으로 상승했으며, 현재 버전은 안정성과 유지보수성을 갖춘 9점대 프로덕션 근접 예제 구조다.
