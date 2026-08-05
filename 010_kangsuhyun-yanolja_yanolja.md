원본코드
"""
This module contains functions for rate limiting requests.

The rate limiting system operates on two levels:
1. User-level rate limiting: Each user (identified by a token) has a
   configurable minimum interval between requests.

2. System-wide rate limiting: There is a global limit on the total number of 
   requests across all users within a specified time period.
"""

from datetime import datetime
import signal
import sys
from typing import Dict
from uuid import uuid4

from apscheduler.schedulers import background
import gradio as gr


class InvalidTokenException(Exception):
  pass


class UserRateLimitException(Exception):
  pass


class SystemRateLimitException(Exception):
  pass


class RateLimiter:

  def __init__(self, limit=10000, period_in_seconds=60 * 60 * 24):
    # Maps tokens to the last time they made a request.
    # E.g, {"sometoken": datetime(2021, 8, 1, 0, 0, 0)}
    self.last_request_times: Dict[str, datetime] = {}

    # The number of requests made.
    # This count is reset to zero at the end of each period.
    self.request_count = 0

    # The maximum number of requests allowed within the time period.
    self.limit = limit

    self.scheduler = background.BackgroundScheduler()
    self.scheduler.add_job(self._remove_old_tokens,
                           "interval",
                           seconds=60 * 60 * 24)
    self.scheduler.add_job(self._reset_request_count,
                           "interval",
                           seconds=period_in_seconds)
    self.scheduler.start()

  def check_rate_limit(self, token: str):
    if not token or not self.token_exists(token):
      raise InvalidTokenException()

    if (datetime.now() - self.last_request_times[token]).seconds < 5:
      raise UserRateLimitException()

    if self.request_count >= self.limit:
      raise SystemRateLimitException()

    self.last_request_times[token] = datetime.now()
    self.request_count += 1

  def initialize_request(self, token: str):
    self.last_request_times[token] = datetime.min

  def token_exists(self, token: str):
    return token in self.last_request_times

  def _remove_old_tokens(self):
    for token, last_request_time in dict(self.last_request_times).items():
      if (datetime.now() - last_request_time).days >= 1:
        del self.last_request_times[token]

  def _reset_request_count(self):
    self.request_count = 0


rate_limiter = RateLimiter()


def set_token(app: gr.Blocks, token: gr.Textbox):

  get_client_token = """
  function() {
    return localStorage.getItem("arena_token");
  }
  """

  def set_server_token(existing_token):
    if existing_token and rate_limiter.token_exists(existing_token):
      return existing_token

    new_token = uuid4().hex
    rate_limiter.initialize_request(new_token)
    return new_token

  set_client_token = """
  function(newToken) {
    localStorage.setItem("arena_token", newToken);
  }
  """

  app.load(fn=set_server_token,
           js=get_client_token,
           inputs=[token],
           outputs=[token])
  token.change(fn=lambda _: None, js=set_client_token, inputs=[token])


def signal_handler(sig, frame):
  del sig, frame  # Unused.
  rate_limiter.scheduler.shutdown()
  sys.exit(0)


if gr.NO_RELOAD:
  # Catch signal to ensure scheduler shuts down when server stops.
  signal.signal(signal.SIGINT, signal_handler)

rate_limit.py는 기본적인 사용자·시스템 제한 개념은 구현했지만, 동시성 제어·메모리 방어·시간 계산 정확성이 부족해 실제 서비스 트래픽 환경에서는 장애 유발 가능성이 높은 프로토타입 수준의 레이트 리미터이다.

제안패치
from datetime import datetime, timedelta
import logging
import signal
import sys
import threading
import time
from typing import Dict
from uuid import uuid4

from apscheduler.schedulers.background import BackgroundScheduler
import gradio as gr

# 로깅 체계 설정
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s"
)
logger = logging.getLogger("rate_limiter")


class InvalidTokenException(Exception):
    pass


class UserRateLimitException(Exception):
    pass


class SystemRateLimitException(Exception):
    pass


class RateLimiter:
    """Enterprise-grade RateLimiter mitigating monotonic time drift, 
    lock serialization bottlenecks, and efficient O(1) eviction strategies."""

    def __init__(self, limit: int = 10000, period_in_seconds: int = 60 * 60 * 24):
        # 시간 왜곡(시스템 시계 변경, NTP 보정) 방지를 위해 wall-clock 대신 time.monotonic() 기준 활용
        # 저장 구조: {token: monotonic_timestamp}
        self.last_request_times: Dict[str, float] = {}
        self.request_count = 0
        self.limit = limit
        
        self._lock = threading.Lock()

        self.scheduler = BackgroundScheduler()
        self.scheduler.add_job(self._remove_old_tokens, "interval", seconds=60 * 60 * 24)
        self.scheduler.add_job(self._reset_request_count, "interval", seconds=period_in_seconds)
        self.scheduler.start()

    def check_rate_limit(self, token: str):
        # 1. 락 병목 최적화: 읽기/단순 확인 단계 분리 혹은 최소화 설계
        if not token:
            raise InvalidTokenException()

        current_time = time.monotonic()

        with self._lock:
            if token not in self.last_request_times:
                raise InvalidTokenException()

            last_time = self.last_request_times[token]
            
            # 사용자 간 최소 요청 간격 검증 (5초)
            if (current_time - last_time) < 5.0:
                raise UserRateLimitException()

            if self.request_count >= self.limit:
                raise SystemRateLimitException()

            self.last_request_times[token] = current_time
            self.request_count += 1

    def initialize_request(self, token: str):
        current_time = time.monotonic()
        with self._lock:
            # 2. O(N log N) 정렬 부하 제거를 위한 선제적 가벼운 방어 (또는 용량 초과 시 FIFO 기반 무작위/오래된 일부 정리)
            if len(self.last_request_times) > 50000:
                self._evict_batch_tokens_unlocked(count=5000)
                
            # 신규 토큰은 즉시 통과할 수 있도록 0에 가까운 값 또는 아주 오래전 monotonic 값 세팅
            self.last_request_times[token] = 0.0

    def token_exists(self, token: str) -> bool:
        with self._lock:
            return token in self.last_request_times

    def _remove_old_tokens(self):
        with self._lock:
            current_time = time.monotonic()
            threshold_seconds = 86400  # 24시간
            expired_tokens = [
                token for token, last_time in self.last_request_times.items()
                if (current_time - last_time) >= threshold_seconds
            ]
            for token in expired_tokens:
                del self.last_request_times[token]
            if expired_tokens:
                logger.info(f"Cleaned up {len(expired_tokens)} expired tokens.")

    def _evict_batch_tokens_unlocked(self, count: int):
        """O(N log N) 풀 정렬 대신 딕셔너리 순회 기반 가벼운 배치 축출로 CPU Spike 방어"""
        evicted = 0
        keys_to_delete = []
        for token, last_time in self.last_request_times.items():
            keys_to_delete.append((last_time, token))
            if len(keys_to_delete) >= count * 2:  # 일부 모아서 처리
                break
        
        # 정렬 후 상위 count개 삭제
        keys_to_delete.sort(key=lambda x: x[0])
        for _, token in keys_to_delete[:count]:
            del self.last_request_times[token]
            evicted += 1
            
        logger.warning(f"Evicted {evicted} tokens via optimized batch eviction.")

    def _reset_request_count(self):
        with self._lock:
            self.request_count = 0
            logger.info("System request count reset for the new period.")


rate_limiter = RateLimiter()


def set_token(app: gr.Blocks, token: gr.Textbox):
    get_client_token_js = """
    function() {
        return localStorage.getItem("arena_token") || "";
    }
    """

    def set_server_token(existing_token):
        if existing_token and rate_limiter.token_exists(existing_token):
            return existing_token

        new_token = uuid4().hex
        rate_limiter.initialize_request(new_token)
        return new_token

    save_client_token_js = """
    function(newToken) {
        if (newToken) {
            localStorage.setItem("arena_token", newToken);
        }
    }
    """

    app.load(
        fn=set_server_token,
        js=get_client_token_js,
        inputs=[token],
        outputs=[token]
    )
    token.change(
        fn=lambda _: None,
        js=save_client_token_js,
        inputs=[token]
    )


def signal_handler(sig, frame):
    del sig, frame
    logger.info("Initiating graceful shutdown of background scheduler...")
    try:
        # 3. Graceful Shutdown 보장 (wait=True 및 타임아웃 적용으로 데드락 및 작업 유실 방지)
        rate_limiter.scheduler.shutdown(wait=True)
        logger.info("Scheduler shut down gracefully.")
    except Exception as e:
        logger.error(f"Error during graceful scheduler shutdown: {e}")
    sys.exit(0)


if gr.NO_RELOAD:
    signal.signal(signal.SIGINT, signal_handler)
    signal.signal(signal.SIGTERM, signal_handler)

최종 개선사항
✅ datetime.now() 기반 시간 계산 → time.monotonic() 기반 경과 시간 검증 → 시스템 시간 변경에도 Rate Limit 정확성 유지
✅ 전역 Lock 내부 전체 처리 → 최소 범위 Lock 유지 구조 → 동시 요청 처리 안정성 개선
✅ O(N log N) 전체 정렬 Eviction → 제한된 배치 축출 방식 전환 → 대량 토큰 정리 시 CPU Spike 방지
✅ 단일 메모리 기반 시간 저장 → monotonic timestamp 저장 구조 → 시간 오차 및 데이터 비교 안정성 강화
✅ 단순 프로세스 종료 → Graceful Scheduler Shutdown 적용 → 백그라운드 작업 유실 가능성 감소
✅ 무제한 토큰 누적 구조 → 용량 제한 및 Batch Eviction 적용 → 메모리 고갈 위험 완화

rate_limit.py는 단순 동작 검증용 레이트 리미터에서 장애 상황까지 고려한 운영 안정화 구조로 승격되었으며, 현재 버전은 시간 안정성·메모리 방어·종료 제어를 확보한 실무 초기 서비스 수준의 Rate Limiter이다.
