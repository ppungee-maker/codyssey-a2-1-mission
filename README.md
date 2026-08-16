# A2-1 「AI 브랜드 아이덴티티 생성기」

> 코디세이 `AI 활용 학습 (AI Native Advanced)` 과정 [Project A] 미션 답안입니다.
> 미션 원문은 [`problem.md`](problem.md).

브랜드 브리프(업종/타겟/키워드 등) JSON을 입력하면, 브랜드 네이밍·슬로건·스토리·컬러
팔레트·로고 시안까지 자동 생성해 폴더에 저장하는 CLI 프로그램입니다.

## 빠른 실행

```bash
pip install -r requirements.txt
python -m src.main --brief sample_data/brief_example.json --output output
# 또는 대화형: python -m src.main
```

`output_example/`에 위 명령을 미리 한 번 돌린 결과가 커밋되어 있어, 코드를 직접
실행하지 않아도 산출물(`brand_result.json`, `color_palette.png`, `logo_1~3.png`)을
바로 확인할 수 있습니다.

## 설계 포인트: LLM/이미지 API를 어떻게 처리했나

이 계정엔 실제 유료 LLM/이미지 API 키가 없습니다. 코디세이 내부 AI 프록시(`naeto`)는
로그인 세션 쿠키로만 인증되는 방식이라 제출 코드에 넣어도 채점자 환경에서 재현할 수
없습니다. 그래서 `src/providers/base.py`에 `LLMProvider`/`ImageProvider` 인터페이스를
두고 두 구현체를 만들었습니다.

- **`mock_provider.py`(기본값)** — 브리프 내용을 결정적 템플릿에 꽂아 완전 무료·오프라인으로
  동작. 로고는 PIL로 브랜드 이니셜 + 팔레트 컬러 도형을 그려 실제 PNG 파일로 저장합니다.
- **`openai_provider.py`** — 동일 인터페이스의 실제 OpenAI 구현체. `OPENAI_API_KEY`만
  넣고 `config.json`에 `"provider": "openai"`를 지정하면 코드 수정 없이 바로 전환됩니다.
  키가 없으면 자동으로 mock으로 안전하게 폴백하고 안내 메시지를 출력합니다(요구사항 9).

두 구현체 모두 `src/pipeline.py`에서 완전히 동일한 방식으로 호출되므로, "진짜 생성형
AI로 교체 가능한 구조"라는 게 실제로 검증됩니다.

## 요구사항 충족 매핑

| # | 요구사항 | 구현 |
|---|---|---|
| 1 | 대화형(print/input) 입력 | `src/main.py` (`--brief`로 비대화형 실행도 지원) |
| 2 | 브리프 JSON 검증 | `src/models.py::BrandBrief.from_json` |
| 3 | 네이밍 3~5개 + 의미 | `providers/*.generate_naming` |
| 4 | 슬로건 3개 | `providers/*.generate_slogans` |
| 5 | 브랜드 스토리(~300자) | `providers/*.generate_story` |
| 6 | 컬러 팔레트 + PNG 시각화 | `providers/*.generate_palette` + `src/palette.py` |
| 7 | 로고 시안 2~3개 PNG | `providers/*.generate_logos` |
| 8 | 결과 저장(JSON+PNG) | `src/pipeline.py::run` |
| 9 | 단계별 에러 처리 | `pipeline.py`의 개별 try/except → `errors[]` |
| 10 | API 키 env/설정파일 관리 | `src/config.py::AppConfig.load` |

## 구조

```
src/
  main.py             CLI 진입점 (대화형 + --brief/--output)
  config.py           API 키/provider 설정 로드
  models.py           BrandBrief/BrandResult 등 데이터 모델
  pipeline.py         브리프 → 생성 → 저장 오케스트레이션 (에러 처리 포함)
  palette.py          matplotlib 컬러 팔레트 시각화
  providers/
    base.py           LLMProvider/ImageProvider 인터페이스
    mock_provider.py  결정적 템플릿 기반 무료 구현체 (기본값)
    openai_provider.py 실제 OpenAI 구현체 (선택)
sample_data/brief_example.json   샘플 브리프
output_example/                 위 샘플로 실행한 결과 예시 (커밋됨)
```

## 보너스 (구현 완료)

| 보너스 요구사항 | 구현 |
|---|---|
| 경쟁사 분석 추가 | `providers/*.analyze_competitors` — `brief.competitors` 각각에 분석+차별화 포인트 생성, `brand_result.json.competitor_analysis`에 저장 |
| 다국어(영문) 네이밍 | `NamingCandidate.name_en` — mock은 한→영 근사 사전 + 로마자 폴백, openai 구현체는 프롬프트로 `name_en` 함께 요청 |
