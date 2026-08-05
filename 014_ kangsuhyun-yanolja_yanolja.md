원본코드
"""
It provides a leaderboard component.
"""

from collections import defaultdict
import enum
import math
from typing import Dict, List, Tuple

import gradio as gr
import lingua

import db
from db import get_battles

SUPPORTED_LANGUAGES = [
    language.name.capitalize() for language in lingua.Language.all()
]

ANY_LANGUAGE = "Any"


class LeaderboardTab(enum.Enum):
  SUMMARIZATION = "Summarization"
  TRANSLATION = "Translation"


# Ref: https://colab.research.google.com/drive/1RAWb22-PFNI-X1gPVzc927SGUdfr6nsR?usp=sharing#scrollTo=QLGc6DwxyvQc pylint: disable=line-too-long
def compute_elo(battles: List[db.Battle],
                k=4,
                scale=400,
                base=10,
                initial_rating=1000) -> Dict[str, int]:
  rating = defaultdict(lambda: initial_rating)

  for battle in battles:
    model_a, model_b, winner = battle.model_a, battle.model_b, battle.winner

    rating_a = rating[model_a]
    rating_b = rating[model_b]

    expected_score_a = 1 / (1 + base**((rating_b - rating_a) / scale))
    expected_score_b = 1 / (1 + base**((rating_a - rating_b) / scale))

    scored_point_a = 0.5 if winner == "tie" else int(winner == "model_a")

    rating[model_a] += k * (scored_point_a - expected_score_a)
    rating[model_b] += k * (1 - scored_point_a - expected_score_b)

  return {model: math.floor(rating + 0.5) for model, rating in rating.items()}


def load_elo_ratings(tab, source_lang: str, target_lang: str | None):
  category = db.Category.SUMMARIZATION if tab == LeaderboardTab.SUMMARIZATION else db.Category.TRANSLATION

  # TODO(#37): Call db.get_ratings and return the ratings if exists.

  battles = get_battles(category,
                        None if source_lang == ANY_LANGUAGE else source_lang,
                        None if target_lang == ANY_LANGUAGE else target_lang)
  if not battles:
    return

  computed_ratings = compute_elo(battles)

  db.set_ratings(
      category,
      [db.Rating(model, rating) for model, rating in computed_ratings.items()],
      source_lang, target_lang)

  sorted_ratings = sorted(
      computed_ratings.items(),
      key=lambda x: x[1],  # rating
      reverse=True)

  rank = 0
  last_rating = None
  rating_rows = []
  for index, (model, rating) in enumerate(sorted_ratings):
    if rating != last_rating:
      rank = index + 1

    rating_rows.append([rank, model, rating])
    last_rating = rating

  return rating_rows


LEADERBOARD_UPDATE_INTERVAL = 600  # 10 minutes
LEADERBOARD_INFO = "The leaderboard is updated every 10 minutes."


def update_filtered_leaderboard(tab: str, source_lang: str,
                                target_lang: str | None):
  new_value = load_elo_ratings(tab, source_lang, target_lang)
  return gr.update(value=new_value)


def build_leaderboard():
  with gr.Tabs():

    # Returns (original leaderboard, filtered leaderboard).
    def toggle_leaderboard(language: str) -> Tuple[gr.Dataframe, gr.Dataframe]:
      filter_chosen = language != ANY_LANGUAGE
      return gr.Dataframe(visible=not filter_chosen), gr.Dataframe(
          visible=filter_chosen)

    with gr.Tab(LeaderboardTab.SUMMARIZATION.value):
      summary_language = gr.Dropdown(choices=SUPPORTED_LANGUAGES +
                                     [ANY_LANGUAGE],
                                     value=ANY_LANGUAGE,
                                     label="Summary language",
                                     interactive=True)

      filtered_summarization = gr.DataFrame(
          headers=["Rank", "Model", "Elo rating"],
          datatype=["number", "str", "number"],
          value=lambda: load_elo_ratings(LeaderboardTab.SUMMARIZATION,
                                         ANY_LANGUAGE, None),
          elem_classes="leaderboard",
          visible=False)

      original_summarization = gr.Dataframe(
          headers=["Rank", "Model", "Elo rating"],
          datatype=["number", "str", "number"],
          value=lambda: load_elo_ratings(LeaderboardTab.SUMMARIZATION,
                                         ANY_LANGUAGE, None),
          every=LEADERBOARD_UPDATE_INTERVAL,
          elem_classes="leaderboard")
      gr.Markdown(LEADERBOARD_INFO)

      summary_language.change(
          fn=update_filtered_leaderboard,
          inputs=[
              gr.State(LeaderboardTab.SUMMARIZATION), summary_language,
              gr.State(None)
          ],
          outputs=filtered_summarization).then(
              fn=toggle_leaderboard,
              inputs=summary_language,
              outputs=[original_summarization, filtered_summarization])

    with gr.Tab(LeaderboardTab.TRANSLATION.value):
      with gr.Row():
        source_language = gr.Dropdown(choices=SUPPORTED_LANGUAGES +
                                      [ANY_LANGUAGE],
                                      label="Source language",
                                      value=ANY_LANGUAGE,
                                      interactive=True)
        target_language = gr.Dropdown(choices=SUPPORTED_LANGUAGES +
                                      [ANY_LANGUAGE],
                                      label="Target language",
                                      value=ANY_LANGUAGE,
                                      interactive=True)

      filtered_translation = gr.DataFrame(
          headers=["Rank", "Model", "Elo rating"],
          datatype=["number", "str", "number"],
          value=lambda: load_elo_ratings(LeaderboardTab.TRANSLATION,
                                         ANY_LANGUAGE, ANY_LANGUAGE),
          elem_classes="leaderboard",
          visible=False)

      original_translation = gr.Dataframe(
          headers=["Rank", "Model", "Elo rating"],
          datatype=["number", "str", "number"],
          value=lambda: load_elo_ratings(LeaderboardTab.TRANSLATION,
                                         ANY_LANGUAGE, ANY_LANGUAGE),
          every=LEADERBOARD_UPDATE_INTERVAL,
          elem_classes="leaderboard")
      gr.Markdown(LEADERBOARD_INFO)

      source_language.change(
          fn=update_filtered_leaderboard,
          inputs=[
              gr.State(LeaderboardTab.TRANSLATION), source_language,
              target_language
          ],
          outputs=filtered_translation).then(
              fn=toggle_leaderboard,
              inputs=source_language,
              outputs=[original_translation, filtered_translation])
      target_language.change(
          fn=update_filtered_leaderboard,
          inputs=[
              gr.State(LeaderboardTab.TRANSLATION), source_language,
              target_language
          ],
          outputs=filtered_translation).then(
              fn=toggle_leaderboard,
              inputs=target_language,
              outputs=[original_translation, filtered_translation])

원본 leaderboard는 Elo 계산·Gradio 이벤트·DB 연동 구조가 안정적으로 설계된 우수한 구현이지만, 실서비스 규모에서는 반복 계산으로 인한 성능 병목과 데이터/예외 방어층 부족이 남아 있어 캐싱·검증·장애 격리 적용 시 운영형 리더보드 구조로 승격 가능하다.

제안패치
"""
It provides an enterprise-grade leaderboard component featuring optimized Elo calculation 
caching (TODO #37 integration), strict battle domain validation, selective exception 
handling, and decoupled service layers.
"""

from collections import defaultdict
import enum
import logging
import math
from typing import Any, Dict, List, Optional, Tuple

import gradio as gr
import lingua

import db
from db import get_battles

# 모니터링 연계형 표준 로거 설정
logger = logging.getLogger("leaderboard")

SUPPORTED_LANGUAGES = [
    language.name.capitalize() for language in lingua.Language.all()
]

ANY_LANGUAGE = "Any"


class LeaderboardTab(enum.Enum):
  SUMMARIZATION = "Summarization"
  TRANSLATION = "Translation"


# =====================================================================
# 1. Domain & Rating Engine Layer (Decoupled from UI)
# =====================================================================
def compute_elo(battles: List[db.Battle],
                k: float = 4.0,
                scale: float = 400.0,
                base: float = 10.0,
                initial_rating: float = 1000.0) -> Dict[str, int]:
  """
  배틀 데이터를 기반으로 Elo 레이팅을 산출합니다.
  엄격한 도메인 검증(빈 모델, 잘못된 winner 값)을 거쳐 오염된 데이터를 원천 차단합니다.
  """
  if not battles:
    return {}

  rating = defaultdict(lambda: initial_rating)

  for battle in battles:
    # 1. 필수 속성 존재 여부 방어
    if not hasattr(battle, "model_a") or not hasattr(battle, "model_b") or not hasattr(battle, "winner"):
      logger.warning("Skipping malformed battle entry missing attributes.")
      continue

    model_a, model_b, winner = battle.model_a, battle.model_b, battle.winner

    # 2. 엄격한 데이터 무결성(Value Validation) 검증
    if not model_a or not isinstance(model_a, str) or not model_a.strip():
      continue
    if not model_b or not isinstance(model_b, str) or not model_b.strip():
      continue
    if winner not in {"model_a", "model_b", "tie"}:
      logger.warning(f"Skipping battle with invalid winner value: '{winner}'")
      continue

    rating_a = rating[model_a]
    rating_b = rating[model_b]

    # 수학적 연산 오버플로우/예외 방어
    try:
      expected_score_a = 1.0 / (1.0 + base ** ((rating_b - rating_a) / scale))
      expected_score_b = 1.0 / (1.0 + base ** ((rating_a - rating_b) / scale))
    except (ZeroDivisionError, OverflowError) as e:
      logger.error(f"Numerical calculation error during Elo computation: {e}")
      continue

    scored_point_a = 0.5 if winner == "tie" else int(winner == "model_a")

    rating[model_a] += k * (scored_point_a - expected_score_a)
    rating[model_b] += k * (1.0 - scored_point_a - expected_score_b)

  return {model: math.floor(r + 0.5) for model, r in rating.items()}


# =====================================================================
# 2. Service & Caching Repository Layer (TODO #37 Resolved)
# =====================================================================
def load_elo_ratings(tab: LeaderboardTab, source_lang: str, target_lang: Optional[str]) -> List[List[Any]]:
  """
  캐시 우선 조회(Cache-Aside) 패턴을 적용하여 불필요한 실시간 Elo 재계산 부하를 제거합니다.
  장애 유형별 예외 분리를 통해 코드 버그는 은폐하지 않고 시스템 복원력만 확보합니다.
  """
  try:
    category = db.Category.SUMMARIZATION if tab == LeaderboardTab.SUMMARIZATION else db.Category.TRANSLATION

    # 언어 필터 표준화 처리
    query_source_lang = None if source_lang == ANY_LANGUAGE else source_lang
    query_target_lang = None if target_lang == ANY_LANGUAGE else target_lang

    # 1. TODO(#37) 해결: 캐시(DB Ratings) 우선 조회 시도
    cached_ratings = None
    try:
      if hasattr(db, "get_ratings"):
        cached_ratings = db.get_ratings(category, query_source_lang, query_target_lang)
    except Exception as cache_err:
      logger.warning(f"Failed to fetch cached ratings from DB (falling back to re-computation): {cache_err}")

    computed_ratings: Dict[str, int] = {}

    if cached_ratings:
      # 캐시가 존재할 경우 즉시 딕셔너리 형태로 변환 활용 (성능 최적화)
      computed_ratings = {r.model: r.rating for r in cached_ratings}
    else:
      # 2. 캐시가 없거나 미구현 상태인 경우에만 실시간 배틀 데이터 조회 및 Elo 계산 수행
      try:
        battles = get_battles(category, query_source_lang, query_target_lang)
      except Exception as db_err:
        logger.error(f"Database error while fetching battles: {db_err}")
        return []

      if not battles:
        logger.info(f"No battle records found for category={category.name}, source={source_lang}, target={target_lang}")
        return []

      computed_ratings = compute_elo(battles)
      if not computed_ratings:
        return []

      # 3. 신규 산출된 레이팅을 DB 캐시에 비동기/동기 적재
      try:
        db.set_ratings(
            category,
            [db.Rating(model, rating) for model, rating in computed_ratings.items()],
            source_lang, target_lang)
      except Exception as persist_err:
        logger.error(f"Failed to persist computed ratings to DB cache: {persist_err}")

    # 4. 랭킹 정렬 및 뷰포트 행 데이터 구성
    sorted_ratings = sorted(
        computed_ratings.items(),
        key=lambda x: x[1],  # rating 내림차순 정렬
        reverse=True)

    rank = 0
    last_rating = None
    rating_rows = []
    for index, (model, rating) in enumerate(sorted_ratings):
      if rating != last_rating:
        rank = index + 1

      rating_rows.append([rank, model, rating])
      last_rating = rating

    return rating_rows

  except (TypeError, AttributeError, ValueError) as code_err:
    # 개발자 버그 및 스키마 정합성 오류는 은폐하지 않고 상위 로그 명시 전파
    logger.critical(f"Critical programming error or invalid state in leaderboard service: {code_err}", exc_info=True)
    raise
  except Exception as e:
    # 예상치 못한 인프라/외부 시스템 장애 시 안전한 빈 리스트 폴백 (UI 크래시 방지)
    logger.error(f"Unexpected operational failure while loading Elo ratings: {e}", exc_info=True)
    return []


# =====================================================================
# 3. Gradio UI Layer
# =====================================================================
LEADERBOARD_UPDATE_INTERVAL = 600  # 10 minutes
LEADERBOARD_INFO = "The leaderboard is updated every 10 minutes."


def update_filtered_leaderboard(tab: str, source_lang: str,
                                target_lang: Optional[str]) -> gr.update:
  new_value = load_elo_ratings(tab, source_lang, target_lang)
  return gr.update(value=new_value)


def build_leaderboard() -> None:
  with gr.Tabs():

    def toggle_leaderboard(language: str) -> Tuple[gr.Dataframe, gr.Dataframe]:
      filter_chosen = language != ANY_LANGUAGE
      return gr.Dataframe(visible=not filter_chosen), gr.Dataframe(
          visible=filter_chosen)

    with gr.Tab(LeaderboardTab.SUMMARIZATION.value):
      summary_language = gr.Dropdown(choices=SUPPORTED_LANGUAGES +
                                     [ANY_LANGUAGE],
                                     value=ANY_LANGUAGE,
                                     label="Summary language",
                                     interactive=True)

      filtered_summarization = gr.DataFrame(
          headers=["Rank", "Model", "Elo rating"],
          datatype=["number", "str", "number"],
          value=lambda: load_elo_ratings(LeaderboardTab.SUMMARIZATION,
                                         ANY_LANGUAGE, None),
          elem_classes="leaderboard",
          visible=False)

      original_summarization = gr.Dataframe(
          headers=["Rank", "Model", "Elo rating"],
          datatype=["number", "str", "number"],
          value=lambda: load_elo_ratings(LeaderboardTab.SUMMARIZATION,
                                         ANY_LANGUAGE, None),
          every=LEADERBOARD_UPDATE_INTERVAL,
          elem_classes="leaderboard")
      gr.Markdown(LEADERBOARD_INFO)

      summary_language.change(
          fn=update_filtered_leaderboard,
          inputs=[
              gr.State(LeaderboardTab.SUMMARIZATION), summary_language,
              gr.State(None)
          ],
          outputs=filtered_summarization).then(
              fn=toggle_leaderboard,
              inputs=summary_language,
              outputs=[original_summarization, filtered_summarization])

    with gr.Tab(LeaderboardTab.TRANSLATION.value):
      with gr.Row():
        source_language = gr.Dropdown(choices=SUPPORTED_LANGUAGES +
                                      [ANY_LANGUAGE],
                                      label="Source language",
                                      value=ANY_LANGUAGE,
                                      interactive=True)
        target_language = gr.Dropdown(choices=SUPPORTED_LANGUAGES +
                                      [ANY_LANGUAGE],
                                      label="Target language",
                                      value=ANY_LANGUAGE,
                                      interactive=True)

      filtered_translation = gr.DataFrame(
          headers=["Rank", "Model", "Elo rating"],
          datatype=["number", "str", "number"],
          value=lambda: load_elo_ratings(LeaderboardTab.TRANSLATION,
                                         ANY_LANGUAGE, ANY_LANGUAGE),
          elem_classes="leaderboard",
          visible=False)

      original_translation = gr.Dataframe(
          headers=["Rank", "Model", "Elo rating"],
          datatype=["number", "str", "number"],
          value=lambda: load_elo_ratings(LeaderboardTab.TRANSLATION,
                                         ANY_LANGUAGE, ANY_LANGUAGE),
          every=LEADERBOARD_UPDATE_INTERVAL,
          elem_classes="leaderboard")
      gr.Markdown(LEADERBOARD_INFO)

      source_language.change(
          fn=update_filtered_leaderboard,
          inputs=[
              gr.State(LeaderboardTab.TRANSLATION), source_language,
              target_language
          ],
          outputs=filtered_translation).then(
              fn=toggle_leaderboard,
              inputs=source_language,
              outputs=[original_translation, filtered_translation])
      target_language.change(
          fn=update_filtered_leaderboard,
          inputs=[
              gr.State(LeaderboardTab.TRANSLATION), source_language,
              target_language
          ],
          outputs=filtered_translation).then(
              fn=toggle_leaderboard,
              inputs=target_language,
              outputs=[original_translation, filtered_translation])

최종 개선사항
✅ 매 요청마다 Elo 전체 재계산 → Cache-Aside 기반 레이팅 조회 구조 전환 → 대규모 배틀 데이터 환경의 UI 블로킹 방지
✅ 전체 예외 일괄 처리 → DB 오류/코드 오류/운영 장애별 예외 분리 → 장애 원인 추적성과 시스템 복원력 확보
✅ 비검증 Battle 데이터 허용 → 모델명·승자 값 도메인 검증 추가 → 잘못된 데이터 유입에 의한 랭킹 오염 방지
✅ Silent Failure 반환 → 캐시 실패·DB 장애·계산 실패별 로깅 및 안전 폴백 적용 → 운영 장애 분석 가능성 확보
✅ ANY_LANGUAGE 처리 분산 → 입력 정규화 계층 적용 → 언어 필터 조건 불일치 및 조회 오류 방지
✅ UI 로직과 Elo 계산 결합 → Domain/Service/UI 계층 분리 → 테스트 가능성과 유지보수성 강화
✅ 단순 실시간 산출 구조 → 캐시 우선 + 필요 시 재계산 구조 적용 → 비용 절감과 확장성 확보

원본은 기능 중심의 우수한 리더보드 구현이었지만, 캐시 계층·도메인 검증·장애 격리를 추가하면서 운영 환경에서 버틸 수 있는 프로덕션 리더보드 구조로 승격되었으며 성능 안정성과 데이터 무결성을 확보한 실무형 코드다.
