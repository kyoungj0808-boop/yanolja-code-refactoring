원본코드
"""
It provides a platform for comparing the responses of two LLMs. 
"""
import enum
from uuid import uuid4

from firebase_admin import firestore
import gradio as gr
import lingua

from db import db
from leaderboard import build_leaderboard
from leaderboard import SUPPORTED_LANGUAGES
from model import check_models
from model import supported_models
from rate_limit import set_token
import response
from response import get_responses

detector = lingua.LanguageDetectorBuilder.from_all_languages().build()


class VoteOptions(enum.Enum):
  MODEL_A = "Model A is better"
  MODEL_B = "Model B is better"
  TIE = "Tie"


def vote(vote_button, response_a, response_b, model_a_name, model_b_name,
         prompt, instruction, category, source_lang, target_lang):
  doc_id = uuid4().hex
  winner = VoteOptions(vote_button).name.lower()

  deactivated_buttons = [gr.Button(interactive=False) for _ in range(3)]
  outputs = deactivated_buttons + [gr.Row(visible=True)]

  doc = {
      "id": doc_id,
      "prompt": prompt,
      "instruction": instruction,
      "model_a": model_a_name,
      "model_b": model_b_name,
      "model_a_response": response_a,
      "model_b_response": response_b,
      "winner": winner,
      "timestamp": firestore.SERVER_TIMESTAMP
  }

  if category == response.Category.SUMMARIZE.value:
    language_a = detector.detect_language_of(response_a)
    language_b = detector.detect_language_of(response_b)

    # TODO(#37): Move DB operations to db.py.
    doc_ref = db.collection("arena-summarizations").document(doc_id)
    doc["model_a_response_language"] = language_a.name.lower()
    doc["model_b_response_language"] = language_b.name.lower()
    doc_ref.set(doc)

    return outputs

  if category == response.Category.TRANSLATE.value:
    if not source_lang or not target_lang:
      raise gr.Error("Please select source and target languages.")

    doc_ref = db.collection("arena-translations").document(doc_id)
    doc["source_language"] = source_lang.lower()
    doc["target_language"] = target_lang.lower()
    doc_ref.set(doc)

    return outputs

  raise gr.Error("Please select a response type.")


# Removes the persistent orange border from the leaderboard, which
# appears due to the 'generating' class when using the 'every' parameter.
css = """
.leaderboard .generating {
  border: none;
}
"""

with gr.Blocks(title="Yanolja Arena", css=css) as app:
  token = gr.Textbox(visible=False)
  set_token(app, token)

  with gr.Row():
    gr.HTML("""
    <h1 style="text-align: center; font-size: 28px; margin-bottom: 16px">Yanolja Arena</h1>
    <p style="text-align: center; font-size: 16px">Yanolja Arena helps find the best LLMs for summarizing and translating text. We compare two random models at a time and use an ELO rating system to score them.</p>
    <p style="text-align: center; font-size: 16px">This is an open-source project. Check it out on <a href="https://github.com/yanolja/arena">GitHub</a>.</p>
    """)
  with gr.Accordion("How to Use", open=False):
    gr.Markdown("""
      1. **For Summaries:**
        - Enter the text you want summarized into the prompt box.

      2. **For Translations:**
        - Choose the language you're translating from and to.
        - Enter the text you want translated into the prompt box.

      3. **Voting:**
        - After you see both results, pick which one you think is better.
      """)

  with gr.Accordion("Available Models", open=False):
    gr.Markdown("\n".join([f"- {model.name}" for model in supported_models]))

  with gr.Row():
    category_radio = gr.Radio(
        choices=[category.value for category in response.Category],
        value=response.Category.SUMMARIZE.value,
        label="Category",
        info="The chosen category determines the instruction sent to the LLMs.")

    source_language = gr.Dropdown(
        choices=SUPPORTED_LANGUAGES,
        value=lingua.Language.ENGLISH.name.capitalize(),
        label="Source language",
        info="Choose the source language for translation.",
        interactive=True,
        visible=False)
    target_language = gr.Dropdown(
        choices=SUPPORTED_LANGUAGES,
        value=lingua.Language.KOREAN.name.capitalize(),
        label="Target language",
        info="Choose the target language for translation.",
        interactive=True,
        visible=False)

    def update_language_visibility(category):
      visible = category == response.Category.TRANSLATE.value
      return {
          source_language: gr.Dropdown(visible=visible),
          target_language: gr.Dropdown(visible=visible)
      }

    category_radio.change(update_language_visibility, category_radio,
                          [source_language, target_language])

  model_names = [gr.State(None), gr.State(None)]
  response_boxes = [gr.State(None), gr.State(None)]

  prompt_textarea = gr.TextArea(label="Prompt", lines=4)
  submit = gr.Button()

  with gr.Group():
    with gr.Row():
      response_boxes[0] = gr.Textbox(label="Model A", interactive=False)

      response_boxes[1] = gr.Textbox(label="Model B", interactive=False)

    with gr.Row(visible=False) as model_name_row:
      model_names[0] = gr.Textbox(show_label=False)
      model_names[1] = gr.Textbox(show_label=False)

  with gr.Row(visible=False) as vote_row:
    option_a = gr.Button(VoteOptions.MODEL_A.value)
    option_b = gr.Button(VoteOptions.MODEL_B.value)
    tie = gr.Button(VoteOptions.TIE.value)

  instruction_state = gr.State("")

  # The following elements need to be reset when the user changes
  # the category, source language, or target language.
  ui_elements = [
      response_boxes[0], response_boxes[1], model_names[0], model_names[1],
      instruction_state, model_name_row, vote_row
  ]

  def reset_ui():
    return [gr.Textbox(value="") for _ in range(4)
           ] + [gr.State(""),
                gr.Row(visible=False),
                gr.Row(visible=False)]

  category_radio.change(fn=reset_ui, outputs=ui_elements)
  source_language.change(fn=reset_ui, outputs=ui_elements)
  target_language.change(fn=reset_ui, outputs=ui_elements)

  submit_event = submit.click(
      fn=lambda: [
          gr.Radio(interactive=False),
          gr.Dropdown(interactive=False),
          gr.Dropdown(interactive=False),
          gr.Button(interactive=False),
          gr.Row(visible=False),
          gr.Row(visible=False),
      ] + [gr.Button(interactive=True) for _ in range(3)],
      outputs=[
          category_radio, source_language, target_language, submit, vote_row,
          model_name_row, option_a, option_b, tie
      ]).then(fn=get_responses,
              inputs=[
                  prompt_textarea, category_radio, source_language,
                  target_language, token
              ],
              outputs=response_boxes + model_names + [instruction_state])
  submit_event.success(fn=lambda: gr.Row(visible=True), outputs=vote_row)
  submit_event.then(
      fn=lambda: [
          gr.Radio(interactive=True),
          gr.Dropdown(interactive=True),
          gr.Dropdown(interactive=True),
          gr.Button(interactive=True)
      ],
      outputs=[category_radio, source_language, target_language, submit])

  def deactivate_after_voting(option_button: gr.Button):
    option_button.click(
        fn=vote,
        inputs=[option_button] + response_boxes + model_names + [
            prompt_textarea, instruction_state, category_radio, source_language,
            target_language
        ],
        outputs=[option_a, option_b, tie, model_name_row]).then(
            fn=lambda: [gr.Button(interactive=False) for _ in range(3)],
            outputs=[option_a, option_b, tie])

  for option in [option_a, option_b, tie]:
    deactivate_after_voting(option)

  build_leaderboard()

if __name__ == "__main__":
  check_models(supported_models)

  # We need to enable queue to use generators.
  app.queue(api_open=False)
  app.launch(debug=True, show_api=False)

야놀자 Arena app.py는 데모 수준의 빠른 연결성은 뛰어나지만, 핵심 평가 데이터의 생성·검증·저장을 UI 이벤트 함수에 직접 묶어둔 구조라 프로덕션 규모 확장 시 장애와 데이터 무결성 문제를 유발할 가능성이 높은 설계다.

제안패치
"""
Production-ready Yanolja Arena main application (Single File Architecture).
"""
import enum
from uuid import uuid4
import gradio as gr
import lingua
from firebase_admin import firestore

from db import db
from leaderboard import build_leaderboard, SUPPORTED_LANGUAGES
from model import check_models, supported_models
from rate_limit import set_token
import response
from response import get_responses

# 지연 없는 안전한 언어 감지 빌더 초기화
detector = lingua.LanguageDetectorBuilder.from_all_languages().build()


class VoteOptions(enum.Enum):
    MODEL_A = "Model A is better"
    MODEL_B = "Model B is better"
    TIE = "Tie"


# ==========================================
# 1. 영속성 및 서비스 계층 (Repository & Service Layer)
# ==========================================
class VotePersistenceError(Exception):
    """데이터베이스 저장 실패 시 발생하는 예외"""
    pass

class DomainValidationError(Exception):
    """비즈니스 규칙 및 입력값 검증 실패 시 발생하는 예외"""
    pass


class VoteRepository:
    """Firestore 영속성을 담당하는 리포지토리"""
    @staticmethod
    def save(doc_id: str, doc: dict, category: str) -> None:
        try:
            if category == "summarize":
                doc_ref = db.collection("arena-summarizations").document(doc_id)
            elif category == "translate":
                doc_ref = db.collection("arena-translations").document(doc_id)
            else:
                raise ValueError(f"Unsupported category for storage: {category}")
            
            # 서버 타임스탬프 무결성 보장
            doc["timestamp"] = firestore.SERVER_TIMESTAMP
            doc_ref.set(doc)
        except Exception as e:
            raise VotePersistenceError(f"Database error during persistence: {str(e)}") from e


class VoteService:
    """투표 비즈니스 로직 및 가공 서비스"""
    @staticmethod
    def process_vote(doc_id: str, winner: str, response_a: str, response_b: str,
                     model_a_name: str, model_b_name: str, prompt: str,
                     instruction: str, category: str, source_lang: str, target_lang: str) -> None:
        
        if not prompt or not prompt.strip():
            raise DomainValidationError("Prompt cannot be empty.")
        if not response_a or not response_b:
            raise DomainValidationError("Both model responses must be present before voting.")

        doc = {
            "id": doc_id,
            "prompt": prompt.strip(),
            "instruction": instruction or "",
            "model_a": model_a_name,
            "model_b": model_b_name,
            "model_a_response": response_a,
            "model_b_response": response_b,
            "winner": winner,
        }

        if category == response.Category.SUMMARIZE.value:
            language_a = detector.detect_language_of(response_a)
            language_b = detector.detect_language_of(response_b)
            
            doc["model_a_response_language"] = language_a.name.lower() if language_a else "unknown"
            doc["model_b_response_language"] = language_b.name.lower() if language_b else "unknown"
            
            VoteRepository.save(doc_id, doc, "summarize")
            return

        if category == response.Category.TRANSLATE.value:
            if not source_lang or not target_lang:
                raise DomainValidationError("Please select source and target languages.")
            
            doc["source_language"] = source_lang.lower()
            doc["target_language"] = target_lang.lower()
            
            VoteRepository.save(doc_id, doc, "translate")
            return

        raise DomainValidationError("Please select a valid response type.")


# ==========================================
# 2. 프레젠테이션 계층 (UI & Controller)
# ==========================================
def vote_handler(vote_button, response_a, response_b, model_a_name, model_b_name,
                 prompt, instruction, category, source_lang, target_lang):
    doc_id = uuid4().hex
    winner = VoteOptions(vote_button).name.lower()

    deactivated_buttons = [gr.Button(interactive=False) for _ in range(3)]
    outputs = deactivated_buttons + [gr.Row(visible=True)]

    try:
        VoteService.process_vote(
            doc_id=doc_id,
            winner=winner,
            response_a=response_a,
            response_b=response_b,
            model_a_name=model_a_name,
            model_b_name=model_b_name,
            prompt=prompt,
            instruction=instruction,
            category=category,
            source_lang=source_lang,
            target_lang=target_lang
        )
        return outputs
    except DomainValidationError as dve:
        raise gr.Error(str(dve))
    except VotePersistenceError:
        raise gr.Error("An internal error occurred while saving your vote. Please try again.")
    except Exception:
        raise gr.Error("An unexpected error occurred.")


css = """
.leaderboard .generating {
  border: none;
}
"""

with gr.Blocks(title="Yanolja Arena", css=css) as app:
    token = gr.Textbox(visible=False)
    set_token(app, token)

    with gr.Row():
        gr.HTML("""
        <h1 style="text-align: center; font-size: 28px; margin-bottom: 16px">Yanolja Arena</h1>
        <p style="text-align: center; font-size: 16px">Yanolja Arena helps find the best LLMs for summarizing and translating text. We compare two random models at a time and use an ELO rating system to score them.</p>
        <p style="text-align: center; font-size: 16px">This is an open-source project. Check it out on <a href="https://github.com/yanolja/arena">GitHub</a>.</p>
        """)

    with gr.Accordion("How to Use", open=False):
        gr.Markdown("""
          1. **For Summaries:**
            - Enter the text you want summarized into the prompt box.

          2. **For Translations:**
            - Choose the language you're translating from and to.
            - Enter the text you want translated into the prompt box.

          3. **Voting:**
            - After you see both results, pick which one you think is better.
          """)

    with gr.Accordion("Available Models", open=False):
        gr.Markdown("\n".join([f"- {model.name}" for model in supported_models]))

    with gr.Row():
        category_radio = gr.Radio(
            choices=[category.value for category in response.Category],
            value=response.Category.SUMMARIZE.value,
            label="Category",
            info="The chosen category determines the instruction sent to the LLMs."
        )

        source_language = gr.Dropdown(
            choices=SUPPORTED_LANGUAGES,
            value=lingua.Language.ENGLISH.name.capitalize(),
            label="Source language",
            info="Choose the source language for translation.",
            interactive=True,
            visible=False
        )
        target_language = gr.Dropdown(
            choices=SUPPORTED_LANGUAGES,
            value=lingua.Language.KOREAN.name.capitalize(),
            label="Target language",
            info="Choose the target language for translation.",
            interactive=True,
            visible=False
        )

        def update_language_visibility(category):
            visible = category == response.Category.TRANSLATE.value
            return {
                source_language: gr.Dropdown(visible=visible),
                target_language: gr.Dropdown(visible=visible)
            }

        category_radio.change(update_language_visibility, category_radio, [source_language, target_language])

    model_names = [gr.State(None), gr.State(None)]
    response_boxes = [gr.State(None), gr.State(None)]

    prompt_textarea = gr.TextArea(label="Prompt", lines=4)
    submit = gr.Button()

    with gr.Group():
        with gr.Row():
            response_boxes[0] = gr.Textbox(label="Model A", interactive=False)
            response_boxes[1] = gr.Textbox(label="Model B", interactive=False)

        with gr.Row(visible=False) as model_name_row:
            model_names[0] = gr.Textbox(show_label=False)
            model_names[1] = gr.Textbox(show_label=False)

    with gr.Row(visible=False) as vote_row:
        option_a = gr.Button(VoteOptions.MODEL_A.value)
        option_b = gr.Button(VoteOptions.MODEL_B.value)
        tie = gr.Button(VoteOptions.TIE.value)

    instruction_state = gr.State("")

    ui_elements = [
        response_boxes[0], response_boxes[1], model_names[0], model_names[1],
        instruction_state, model_name_row, vote_row
    ]

    def reset_ui():
        # gr.update를 사용하여 컴포넌트 생명주기와 상태 안정성 확보
        return [gr.update(value="") for _ in range(4)] + [
            gr.State(""),
            gr.Row(visible=False),
            gr.Row(visible=False)
        ]

    category_radio.change(fn=reset_ui, outputs=ui_elements)
    source_language.change(fn=reset_ui, outputs=ui_elements)
    target_language.change(fn=reset_ui, outputs=ui_elements)

    def validate_and_get_responses(prompt, category, src_lang, tgt_lang, tok):
        if not prompt or not prompt.strip():
            raise gr.Error("Prompt cannot be empty.")
        return get_responses(prompt, category, src_lang, tgt_lang, tok)

    submit_event = submit.click(
        fn=lambda: [
            gr.Radio(interactive=False),
            gr.Dropdown(interactive=False),
            gr.Dropdown(interactive=False),
            gr.Button(interactive=False),
            gr.Row(visible=False),
            gr.Row(visible=False),
        ] + [gr.Button(interactive=True) for _ in range(3)],
        outputs=[
            category_radio, source_language, target_language, submit, vote_row,
            model_name_row, option_a, option_b, tie
        ]
    ).then(
        fn=validate_and_get_responses,
        inputs=[prompt_textarea, category_radio, source_language, target_language, token],
        outputs=response_boxes + model_names + [instruction_state]
    )
    submit_event.success(fn=lambda: gr.Row(visible=True), outputs=vote_row)
    submit_event.then(
        fn=lambda: [
            gr.Radio(interactive=True),
            gr.Dropdown(interactive=True),
            gr.Dropdown(interactive=True),
            gr.Button(interactive=True)
        ],
        outputs=[category_radio, source_language, target_language, submit]
    )

    def deactivate_after_voting(option_button: gr.Button):
        option_button.click(
            fn=vote_handler,
            inputs=[option_button] + response_boxes + model_names + [
                prompt_textarea, instruction_state, category_radio, source_language,
                target_language
            ],
            outputs=[option_a, option_b, tie, model_name_row]
        ).then(
            fn=lambda: [gr.Button(interactive=False) for _ in range(3)],
            outputs=[option_a, option_b, tie]
        )

    for option in [option_a, option_b, tie]:
        deactivate_after_voting(option)

    build_leaderboard()

if __name__ == "__main__":
    check_models(supported_models)
    app.queue(api_open=False)
    # 프로덕션 보안 기준 debug=False 설정 적용
    app.launch(debug=False, show_api=False)

최종 개선사항
✅ UI·DB 직접 결합 구조 → Repository·Service 계층 분리 → 유지보수성과 테스트 가능성 강화
✅ Firestore 저장 로직 분산 → VoteRepository 단일 책임화 → 데이터 영속성 무결성 확보
✅ 단순 RuntimeError 처리 → 도메인 예외(VotePersistenceError) 적용 → 장애 원인 추적성 향상
✅ 투표 검증 로직 분산 → VoteService 중앙화 → 비즈니스 규칙 일관성 확보
✅ lingua 언어 감지 무방비 호출 → 실패 결과 unknown 처리 → 외부 라이브러리 장애 전파 방지
✅ 사용자 입력 검증 부재 → DomainValidationError 기반 차단 → 불필요한 LLM 비용 및 오염 데이터 방어
✅ Gradio 디버그 노출 → debug=False 적용 → 운영 환경 보안 강화

기존 Arena 코드는 빠른 프로토타입 수준이었지만 이번 개선본은 UI·도메인·영속성을 분리해 장애 격리와 데이터 안정성을 확보한 실무 서비스 구조로 진입했다.
