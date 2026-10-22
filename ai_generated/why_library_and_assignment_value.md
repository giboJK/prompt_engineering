# 추론이 모델에 내장된 시대에 과제를 라이브러리로 만들 이유와 과제의 의미

> 작성: 2026-10-22, Claude Opus 5.5 (`kant-next-prompt-student/AI_USAGE.md`에 기록)
> 자료 확인일: 2026-10-22. 모델과 플랫폼 문서는 자주 바뀌므로 날짜를 기준으로 읽어 주세요.

## 읽는 법: 근거와 추론의 구분

- **[근거]** 외부 자료나 과제 파일에서 확인한 내용입니다. 링크를 달았고, 내용은 원문을 옮기지 않고 요약했습니다. 각 자료를 직접 열람했는지, 검색 요약으로만 봤는지는 [5장 출처 목록](#5-출처-목록)에 적었습니다.
- **[추론]** 근거를 바탕으로 한 Claude의 판단과 의견입니다. 틀릴 수 있고, 반론이 가능한 부분입니다.

## 0. 한 줄 답

1. **"프롬프트 형식만 정해 넘기면 끝 아닌가?"** 모델에 들어간 것은 "단계별로 생각하라"는 **CoT 프롬프트 기법 하나**입니다. 과제 코드 대부분은 CoT와 무관하게, 모델이 원리적으로 대신할 수 없는 일(도구 실행, 출처 실재 확인, 실패 처리, 비용 측정, 품질 측정)을 다룹니다. 다만 그중 일부(JSON 형식, 재시도, 캐시)는 SDK와 API가 흡수하고 있어서, 라이브러리로 만들 대상은 생각보다 **얇아야** 합니다. **[추론]**
2. **과제의 의미**: 정답 코드보다 "모델 바깥에서 신뢰를 만드는 방법"과 "내장 기능이 내 문제에서 실제로 효과가 있는지 측정하는 방법"을 연습하는 데 있습니다. 내장 추론이 강해질수록 이 측정 능력의 가치가 커집니다. **[추론]**

## 1. 전제 점검: "CoT가 모델 내장 기능이 되었다"는 어디까지 맞나

### 1.1 맞는 부분

- **[근거]** CoT는 원래 프롬프트 기법으로 시작했습니다. 예시에 중간 풀이를 넣는 방식([Wei et al., 2022](https://arxiv.org/abs/2201.11903)), "단계별로 생각해 보자" 한 문장을 붙이는 방식([Kojima et al., 2022](https://arxiv.org/abs/2205.11916)), 여러 풀이를 뽑아 다수결하는 Self-Consistency([Wang et al., 2022](https://arxiv.org/abs/2203.11171))가 대표적입니다. 이후 답하기 전에 스스로 추론하도록 학습된 추론 모델이 나왔습니다([OpenAI, 2024-09](https://openai.com/index/learning-to-reason-with-llms/)).
- **[근거]** OpenAI 추론 모델은 답하기 전에 보이지 않는 추론 토큰을 만들고, 그 양은 `reasoning.effort`로 조절합니다. 원본 추론 내용은 API로 공개하지 않습니다([OpenAI Reasoning models](https://developers.openai.com/api/docs/guides/reasoning)).
- **[근거]** OpenAI는 추론 모델에 "단계별로 생각하라"고 요청할 필요가 없고 오히려 품질을 떨어뜨릴 수 있다고 안내합니다([OpenAI Reasoning best practices](https://developers.openai.com/api/docs/guides/reasoning-best-practices)). Anthropic도 생각 기능이 켜진 모델에는 손으로 쓴 단계별 계획보다 일반적인 지시가 낫다고 안내하고, 일부 최신 모델(Claude Opus 5.5 등)은 생각 기능이 항상 켜져 있어 effort와 질문 난이도에 따라 생각의 양이 정해진다고 설명합니다([Claude Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)).
- **[근거]** 연구 결과도 같은 방향입니다. 추론 모델에서는 명시적 CoT 프롬프트의 추가 이득이 작고 처리 시간은 크게 늘었다는 보고가 있고([Meincke et al., 2025](https://arxiv.org/abs/2506.07142)), CoT의 이득이 주로 수학과 기호 추론에 몰려 있다는 메타 분석도 있습니다([Sprague et al., ICLR 2025](https://arxiv.org/abs/2409.12183)).

### 1.2 조심할 부분

- **내장 추론은 설정과 모델에 따라 꺼질 수 있습니다.**
  - **[근거]** OpenAI 문서는 `effort: none`을 추론이 필요 없는 지연 민감 작업용으로 설명합니다([OpenAI Reasoning models](https://developers.openai.com/api/docs/guides/reasoning)).
  - **[근거]** 과제의 baseline 조건이 바로 `effort: none`이고, reasoning 조건은 `low`입니다([support.py:65](../kant-next-prompt-student/support.py#L65), [ASSIGNMENT.md:67](../kant-next-prompt-student/ASSIGNMENT.md#L67)).
  - **[추론]** 그래서 과제의 비교는 "CoT 문장 하나의 효과"가 아니라 "내장 추론을 거의 끈 상태 대 켜고 점검 지시를 더한 상태"의 비교에 가깝습니다. 과제 문서도 두 차이를 하나의 효과로 단정하지 말라고 합니다.
- **과제의 reasoning 지시는 고전적인 CoT와 다릅니다.**
  - **[근거]** 과제는 숨겨진 사고 과정의 출력을 요구하지 말고, 답하기 전에 조건과 근거를 점검하게 하라고 합니다([ASSIGNMENT.md:54-56](../kant-next-prompt-student/ASSIGNMENT.md#L54)).
  - **[근거]** Anthropic은 생각 기능이 있는 모델에서도 "기준에 맞춰 답을 검증하라"는 자기 점검 지시가 오류를 잘 잡는다고 하면서, 모델에 따라서는 이런 지시가 과잉 검증을 일으켜 토큰과 지연만 늘린다고 합니다([Claude Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)).
  - **[추론]** 즉 "점검 지시가 도움이 되는가"는 모델마다 답이 달라서, 측정 없이는 알 수 없는 질문입니다.
- **내장 추론은 보이지 않고, 보이더라도 실제 판단 과정과 일치한다는 보장이 없습니다.**
  - **[근거]** Anthropic 연구에서 추론 모델은 답을 정할 때 실제로 사용한 힌트를 추론 과정에 드물게만 밝혔습니다(Claude 3.7 Sonnet 약 25%, DeepSeek R1 약 39%). 보상을 노린 행동도 추론 과정에서 거의 인정하지 않았습니다([Anthropic, 2025-04](https://www.anthropic.com/research/reasoning-models-dont-say-think)).
  - **[추론]** 따라서 "모델이 알아서 잘 생각했겠지"를 품질 보증 수단으로 쓸 수 없습니다. 검증은 모델 바깥의 결정적인 코드와 평가로 해야 합니다.

## 2. 질문 1: 그럼 라이브러리로 만들 필요가 있나

### 2.1 과제 코드 중 CoT와 관련된 부분은 아주 작다

과제의 일곱 기능을 "누가 대신해 줄 수 있는가"로 나누면 다음과 같습니다.

| 과제 기능 | 내장 추론이 대신하나 | SDK나 API가 제공하나 | 여전히 내 설계로 남는 것 | 근거 |
|---|---|---|---|---|
| 정책 프롬프트의 reasoning 점검 지시 | 대부분 대신함(모델과 설정에 따라 다름) | `effort` 파라미터 | 넣을지 말지를 측정으로 결정 | [OpenAI](https://developers.openai.com/api/docs/guides/reasoning-best-practices), [Anthropic](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices) |
| 정책 프롬프트의 기본 정책(역할, 범위, 제약, 형식, 성공 기준) | 아니오 | 아니오 | 도메인 규칙 자체 | [OpenAI](https://developers.openai.com/api/docs/guides/reasoning) |
| 답변 검증과 수정 | 아니오 | Structured Outputs가 **형식**은 보장 | 출처가 실제로 있는지, 중복인지, 담당자 확인이 필요한지 같은 **업무 규칙**. 거절과 불완전 응답 처리 | [OpenAI Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs) |
| 티켓 조회(도구) | 아니오. 모델은 호출을 **요청**만 함 | 도구 호출 형식 | 도구 실행, 인자 검증, 읽기 전용 권한, `call_id` 연결 | [OpenAI Function calling](https://developers.openai.com/api/docs/guides/function-calling) |
| 오류 화면 입력 | 일부(이미지 이해) | 이미지 입력 형식 | 관찰과 추측을 구분하는 정책 | [ASSIGNMENT.md:123-125](../kant-next-prompt-student/ASSIGNMENT.md#L123) |
| 호출 재시도 | 아니오 | **예.** 공식 Python SDK가 같은 오류 종류를 기본 2회 재시도 | 시도별 지연과 오류를 기록해야 할 때만 직접 구현 | [openai-python README](https://github.com/openai/openai-python) |
| 사용량과 비용 | 아니오 | usage 필드, 자동 캐시 | 비용 계산, 관측하지 못한 값 처리, 가격 갱신, 고정 prefix 설계 | [OpenAI Prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching) |
| 품질 확인(`compare`, `check`) | 아니오 | 평가 도구는 있으나 바뀜 | 내 도메인의 평가 사례와 기준 | [OpenAI Evaluation best practices](https://developers.openai.com/api/docs/guides/evaluation-best-practices) |

표 뒤의 근거를 풀어 쓰면 다음과 같습니다.

- **[근거]** Structured Outputs는 응답이 주어진 JSON Schema를 따르도록 보장하지만, 내용에는 여전히 실수가 있을 수 있다고 문서가 직접 밝힙니다. 거절은 스키마를 따르지 않을 수 있어 별도 필드로 오고, 출력 한도에 걸리면 응답이 `incomplete`로 잘립니다([OpenAI Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs)).
- **[근거]** 함수 호출에서 함수를 실행하는 주체는 모델이 아니라 애플리케이션입니다. 결과를 돌려줄 때는 같은 `call_id`를 써야 하고, 추론 모델이면 함께 받은 추론 항목도 되돌려 보내야 합니다([OpenAI Function calling](https://developers.openai.com/api/docs/guides/function-calling)).
- **[근거]** 추론 토큰은 출력 토큰으로 과금되고, 출력 한도에 추론 토큰도 포함됩니다([OpenAI Reasoning models](https://developers.openai.com/api/docs/guides/reasoning)). 과제가 "출력에 이미 포함된 추론 토큰을 다시 더하지 말라"고 하는 이유입니다([ASSIGNMENT.md:162](../kant-next-prompt-student/ASSIGNMENT.md#L162)).
- **[근거]** OWASP는 프롬프트 주입을 완전히 막는 방법이 있는지 불분명하다고 보고, 출력 형식을 결정적인 코드로 검증하기, 최소 권한, 위험한 동작에 사람 승인 받기 같은 바깥 방어를 권합니다([OWASP LLM01:2025](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)).
- **[추론]** 내장 추론은 "어떻게 생각할지"를 대신해 주지만, "무엇이 맞는 답인지 정하는 기준", "모델이 손댈 수 있는 범위", "틀렸을 때 어떻게 할지", "얼마가 드는지", "정말 나아졌는지"는 대신하지 못합니다. 이것들은 모델 성능이 아니라 **내 서비스의 계약**이기 때문입니다.

### 2.2 "프롬프트 형식만 정하면 끝"이 성립하지 않는 이유

- **[근거]** 추론 모델 문서도 프롬프트에 명확한 목표, 지켜야 할 제약, 출력 형식, 성공 기준을 주라고 안내합니다. 줄어든 것은 중간 단계를 일일이 지시하는 부분입니다([OpenAI Reasoning models](https://developers.openai.com/api/docs/guides/reasoning), [OpenAI Reasoning best practices](https://developers.openai.com/api/docs/guides/reasoning-best-practices)).
- **[추론]** "형식"은 쉬운 부분이고, 어려운 부분은 그 안에 넣을 **내용**(무엇을 지켜야 하고 언제 사람에게 넘기는지)입니다. 직전 질문에서 "base에 뭘 더해야 할지, 사용자 경험 기준이 뭔지 모르겠다"고 하셨던 고민이 바로 그 내용 설계입니다. 이 고민은 모델이 좋아져도 없어지지 않습니다.
- **[근거]** 머신러닝 시스템에서 모델 코드는 전체의 작은 일부이고, 유지보수 비용은 주변의 연결 코드, 설정, 데이터 의존성에서 크게 생긴다는 고전적인 분석이 있습니다([Sculley et al., NeurIPS 2015](https://proceedings.neurips.cc/paper/2015/file/86df7dcfd896fcaf2674f757a2463eba-Paper.pdf)).
- **[추론]** LLM 앱도 같은 구조입니다. 모델 호출 한 줄을 둘러싼 검증, 권한, 실패 처리, 측정이 실제 작업량의 대부분입니다.

### 2.3 그렇다면 무엇을, 얼마나 라이브러리로 만들까

플랫폼 쪽에서 일어나는 변화의 증거부터 봅니다.

- **[근거]** CoT는 프롬프트에서 모델로, JSON 형식 보장은 API로(Structured Outputs), 재시도는 SDK로(기본 2회), 캐시는 자동 기능으로 옮겨 갔습니다(위 표의 각 근거).
- **[근거]** 이 이동은 빠르고 되돌려지기도 합니다.
  - 과제 가격표의 확인일은 2026-09-18인데, 현재 OpenAI 추론 가이드는 대부분의 작업에 `gpt-6-astra`를 권하고 과제가 쓰는 `gpt-5.6-luna`는 가장 저렴하고 빠른 선택지로 소개합니다.
  - 일부 신모델은 `effort: none`을 거부합니다([OpenAI Reasoning models](https://developers.openai.com/api/docs/guides/reasoning)).
  - `prompt_cache_key`의 역할도 모델 세대에 따라 달라졌습니다([OpenAI Prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching)).
  - OpenAI의 Evals 플랫폼은 2026년 11월 종료가 예정되어 있습니다([OpenAI Evaluation best practices](https://developers.openai.com/api/docs/guides/evaluation-best-practices)).
- **[근거]** Anthropic은 많은 팀의 경험상 복잡한 프레임워크보다 단순하고 조합 가능한 패턴이 잘 작동했다고 보고합니다. 처음에는 LLM API를 직접 쓰고, 복잡도는 효과가 측정될 때만 더하라고 권합니다. 프레임워크의 추상화가 프롬프트와 응답을 가려 디버깅을 어렵게 한다는 지적도 있습니다([Anthropic, Building effective agents, 2024-12](https://www.anthropic.com/engineering/building-effective-agents)).

여기서 나오는 판단은 다음과 같습니다. **[추론]**

| 라이브러리로 **소유할 것**(오래 가는 것) | 라이브러리에서 **얇게 감쌀 것**(플랫폼이 흡수 중인 것) | 라이브러리로 **만들지 않을 것** |
|---|---|---|
| 출력 계약과 업무 규칙(출처 실재, 담당자 확인 조건) | JSON 형식 보장(Structured Outputs 설정) | CoT 문구 템플릿 |
| 실패 종류 구분과 대체 답변 정책 | 재시도(SDK 기본 기능, 시도별 기록이 필요할 때만 직접) | 특정 모델 이름과 effort 값 |
| 도구 권한 경계(읽기 전용, 인자 검증) | 도구 호출 형식과 추론 항목 전달 | 범용 에이전트 프레임워크 |
| 평가 사례와 비교 방법(내 데이터) | 캐시와 usage 필드 읽기 | |
| 비용 계산 규칙(None과 0 구분) | | |

- **[추론]** 라이브러리가 필요한지는 다음 기준으로 판단하세요. 셋 다 해당하지 않으면 개인 프로젝트 안의 모듈 하나면 충분합니다.
  1. 같은 계약(검증, 실패 처리, 비용 기록)을 **두 개 이상의 프로젝트나 기능**에서 쓴다.
  2. 프로바이더나 모델을 바꿀 가능성이 있어서, 그 차이를 한 곳에 가두고 싶다.
  3. 기능마다 로그, 비용, 검증 방식이 제각각이면 곤란한 규모다.
- **[추론, 이전 가이드 보정]** 앞서 작성한 [ai_answer/how_to_utilize_practice_artifact.md](../ai_answer/how_to_utilize_practice_artifact.md)는 재시도 모듈을 직접 만드는 구조였습니다. 공식 SDK가 같은 오류 종류를 기본으로 재시도한다는 점을 반영하면, 기본값은 **SDK 재시도를 쓰고, 시도마다 지연과 오류를 기록해야 할 때만 직접 구현**하는 편이 맞습니다. 과제가 SDK 재시도를 끈 것([support.py:156](../kant-next-prompt-student/support.py#L156))도 시도별 관측을 위해서입니다. 같은 이유로 `Dialect` 중립화도 두 번째 프로바이더가 실제로 필요해질 때까지 미루는 편이 낫습니다.

## 3. 질문 2: 이 과제가 가지는 의미와 효용성

### 3.1 과제가 실제로 훈련시키는 것: LLM 앱의 대표적인 위험 대응

- **[근거]** OWASP Top 10 for LLM Applications 2025에는 프롬프트 주입(LLM01), 민감정보 노출(LLM02), 부적절한 출력 처리(LLM05), 과도한 권한(LLM06), 잘못된 정보(LLM09), 무제한 자원 소비(LLM10) 등이 있습니다([OWASP Top 10 for LLM](https://genai.owasp.org/llm-top-10/)).
- **[추론]** 과제 기능을 이 목록에 대응시키면, 과제는 이 중 여섯 항목을 직접 다룹니다.

| OWASP 항목 | 과제에서 대응하는 부분 |
|---|---|
| LLM01 프롬프트 주입 | Q09 "이전 규칙 무시" 질문에도 정책 유지, 출력 검증 |
| LLM02 민감정보 노출 | 비밀정보 요청 거부, 수정 안내에 예외 원문과 비밀값 금지, 트레이스의 이미지 해시화 |
| LLM05 부적절한 출력 처리 | `validate_answer`의 스키마와 업무 규칙 검사, 거절·불완전 응답 구분 |
| LLM06 과도한 권한 | 읽기 전용 `lookup_ticket`, 함수 이름과 인자 검증 |
| LLM09 잘못된 정보 | `source_ids` 근거, 복구 시각을 단정하지 않기, 화면의 관찰과 추측 구분, `needs_human` |
| LLM10 무제한 자원 소비 | 재시도 3회와 도구 4라운드 상한, 요청별 비용 기록 |

### 3.2 내장 추론의 시대라서 더 필요한 능력: 측정

- **[근거]** 같은 입력에도 출력이 달라질 수 있어서 기존 소프트웨어 테스트만으로는 부족하고, 그래서 평가(evals)가 필요하다고 OpenAI는 설명합니다([OpenAI Evaluation best practices](https://developers.openai.com/api/docs/guides/evaluation-best-practices)).
- **[근거]** 추가 지시가 도움이 되는지는 모델과 과제에 따라 다릅니다. 추론 모델에서는 명시적 CoT 이득이 작았고([Meincke et al., 2025](https://arxiv.org/abs/2506.07142)), 자기 점검 지시는 어떤 모델에는 도움이 되지만 다른 모델에는 과잉 검증을 일으킵니다([Claude Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)).
- **[근거]** 과제는 측정의 규율을 명시합니다.
  - 고정 응답(fixture) 결과는 흐름 확인일 뿐 품질 증명이 아닙니다([README.md:90](../kant-next-prompt-student/README.md#L90)).
  - 검사를 통과해도 사실성은 보장되지 않습니다([README.md:109](../kant-next-prompt-student/README.md#L109)).
  - 지연 차이만으로 캐시 효과를 확정하지 않습니다([ASSIGNMENT.md:176](../kant-next-prompt-student/ASSIGNMENT.md#L176)).
  - 일부 사례는 반복 실행합니다([ASSIGNMENT.md:228](../kant-next-prompt-student/ASSIGNMENT.md#L228)).
  - 차이가 없으면 그 사실도 기록합니다([ASSIGNMENT.md:230](../kant-next-prompt-student/ASSIGNMENT.md#L230)).
  - 실행하지 않은 결과를 만들지 않습니다([README.md:139](../kant-next-prompt-student/README.md#L139)).
- **[추론]** "CoT가 내장되었으니 프롬프트 기법은 끝났다"는 결론은 측정 없이 내린 일반화입니다. 이 과제의 baseline과 reasoning 비교가 바로 **"내 문제에서 추가 지시가 내장 추론 위에 값을 더하는가"를 직접 확인하는 장치**입니다. 결과가 "차이 없음"이어도 그 자체가 비용과 지연을 줄일 근거가 되므로 쓸모 있는 결론입니다.

### 3.3 내장 기능을 제대로 설정하고 고장을 알아보는 능력

- **[근거]** 추론 모델에서 도구를 쓰면 함께 받은 추론 항목을 다시 보내야 하고, `store: false`이면 암호화된 추론 내용을 되돌려 보내야 합니다([OpenAI Reasoning models](https://developers.openai.com/api/docs/guides/reasoning)). 과제는 `store: false`와 `reasoning.encrypted_content`를 쓰고([support.py:66-67](../kant-next-prompt-student/support.py#L66)), 도구 왕복에서 "모델 output 전체"를 연결하게 합니다([ASSIGNMENT.md:106](../kant-next-prompt-student/ASSIGNMENT.md#L106)).
- **[근거]** SDK 자체 재시도와 직접 만든 재시도가 겹치면 시도가 곱해집니다. SDK는 기본으로 재시도하므로, 과제는 이를 끄고 직접 시도를 기록합니다([openai-python README](https://github.com/openai/openai-python), [support.py:156](../kant-next-prompt-student/support.py#L156)).
- **[근거]** 캐시는 앞부분(prefix)이 완전히 같을 때만 적중하므로, 고정 내용을 앞에 두고 바뀌는 내용을 뒤에 둬야 합니다([OpenAI Prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching)). 과제가 문서를 정렬하고 질문을 지시문에 섞지 않는 이유입니다.
- **[추론]** 앞으로 이 코드를 다시 직접 짤 일이 없더라도, 플랫폼 기능이 무엇을 대신하고 있고 잘못 설정하면 어떤 증상이 나는지(캐시 미적중, 추론 항목 누락, 이중 재시도, 비용 이중 계산)를 알아보는 눈은 남습니다. 내장 기능이 많아질수록 직접 구현보다 **설정과 진단**의 비중이 커집니다.

### 3.4 과제의 한계: 결과보다 방법을 가져가세요

- **[근거]** 과제 스스로 두 조건이 지시와 설정을 함께 바꾼다고 밝힙니다([ASSIGNMENT.md:67](../kant-next-prompt-student/ASSIGNMENT.md#L67)). 비교 실행은 사례 10개, 기록 26개입니다([ASSIGNMENT.md:228](../kant-next-prompt-student/ASSIGNMENT.md#L228)).
- **[근거]** 모델 이름, effort 값, 캐시 키의 의미, 평가 플랫폼은 이미 바뀌고 있습니다(2.3절 근거).
- **[추론]**
  - 26개 기록으로는 두 조건의 차이를 통계적으로 확정하기 어렵습니다. 두 변수가 함께 바뀌어서 원인을 하나로 돌릴 수도 없습니다.
  - 모델 하나, 프로바이더 하나의 결과입니다.
  - 그래서 이 과제에서 가져갈 것은 "reasoning이 이겼다/졌다"는 결론이나 특정 설정값이 아닙니다. **계약을 정하고, 바깥에서 검증하고, 실패를 구분하고, 측정으로 판단하는 방법**입니다.

| 가져갈 것 | 버려도 될 것 |
|---|---|
| 기본 정책과 점검 지시를 나누는 사고방식 | 특정 CoT 문구 |
| 출력 계약 + 결정적 검증 + 1회 수정 + 대체 답변 구조 | `gpt-5.6-luna`, `effort: none/low` 같은 설정값 |
| 도구는 앱이 실행하고 권한은 최소로 한다는 원칙 | OpenAI Responses API 응답 모양에 맞춘 세부 코드 |
| None과 0을 구분하는 계량 습관 | 이번 비교의 승패 결과 |
| 고정 사례, 반복 실행, 교란 변수를 인정하는 평가 습관 | 과제용 실행기(`support.cli`) |

## 4. 정리

- **[추론]** 내장 추론은 "생각하는 방법"을 모델 안으로 가져갔고, 개발자에게는 "무엇이 맞는지 정하고, 바깥에서 확인하고, 측정으로 판단하는 일"을 남겼습니다.
- **[추론]** 라이브러리로 만들 가치가 있는 것은 그 남은 일, 곧 **계약, 검증, 실패 정책, 평가 사례**입니다. 재시도, 형식 보장, 캐시처럼 플랫폼이 흡수하는 것은 얇게 감싸거나 그대로 쓰세요.
- **[추론]** 이 과제의 효용은 코드보다 판단 기준을 세워 보는 경험에 있습니다. 그 판단 기준은 모델이 바뀌어도 다시 쓸 수 있습니다.

## 5. 출처 목록

확인 방식은 세 가지로 나눴습니다. **직접 열람**은 이번에 원문 페이지를 열어 확인한 것, **검색 요약**은 검색 결과의 요약만 본 것, **서지 인용**은 논문과 글의 존재와 주제만 인용하고 이번에 본문을 열람하지 않은 것입니다.

| 출처 | 종류 | 날짜 | 확인 방식 | 이 문서에서 뒷받침하는 내용 |
|---|---|---|---|---|
| [OpenAI, Reasoning models](https://developers.openai.com/api/docs/guides/reasoning) | 공식 문서 | 2026-10-22 열람 | 직접 열람 | 추론 토큰 과금, effort 값, 추론 항목 전달, 원본 추론 비공개, 권장 모델 변화 |
| [OpenAI, Reasoning best practices](https://developers.openai.com/api/docs/guides/reasoning-best-practices) | 공식 문서 | 2026-10-22 열람 | 직접 열람 | 단계별 사고 지시 불필요, 명확한 제약과 성공 기준 |
| [OpenAI, Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs) | 공식 문서 | 2026-10-22 열람 | 직접 열람 | 형식 보장, 거절, 불완전 응답, 내용 실수 가능성 |
| [OpenAI, Function calling](https://developers.openai.com/api/docs/guides/function-calling) | 공식 문서 | 2026-10-22 열람 | 직접 열람 | 앱이 함수를 실행, `call_id`, 추론 항목 전달 |
| [OpenAI, Prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching) | 공식 문서 | 2026-10-22 열람 | 직접 열람 | prefix 일치, `prompt_cache_key` 역할 변화, `cached_tokens` |
| [OpenAI, Evaluation best practices](https://developers.openai.com/api/docs/guides/evaluation-best-practices) | 공식 문서 | 2026-10-22 | 검색 요약 | 비결정성과 평가 필요성, Evals 플랫폼 종료 예정 |
| [openai/openai-python README](https://github.com/openai/openai-python) | 공식 SDK 문서 | 2026-10-22 열람 | 직접 열람 | 기본 2회 재시도와 대상 오류, `max_retries` |
| [Anthropic, Claude Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices) | 공식 문서 | 2026-10-22 열람 | 직접 열람 | 일반 지시가 단계별 지시보다 나음, 항상 켜진 적응형 생각, 자기 점검 지시와 과잉 검증 |
| [Anthropic, Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) | 엔지니어링 글 | 2024-12-19 | 직접 열람 | 단순한 패턴, API 직접 사용, 프레임워크 추상화의 비용 |
| [Anthropic, Reasoning models don't always say what they think](https://www.anthropic.com/research/reasoning-models-dont-say-think) | 연구 | 2025-04-03 | 직접 열람 | 추론 과정의 불충실성 |
| [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/) | 보안 표준 | 2025판 | 직접 열람 | 위험 항목 목록 |
| [OWASP LLM01:2025 Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/) | 보안 표준 | 2025판 | 직접 열람 | 완전한 방지책 불명확, 바깥 방어 권고 |
| [Meincke et al., Prompting Science Report 2](https://arxiv.org/abs/2506.07142) | 기술 보고서 | 2025-06 | 검색 요약 | 추론 모델에서 CoT 프롬프트 이득 감소와 시간 비용 |
| [Sprague et al., To CoT or not to CoT?](https://arxiv.org/abs/2409.12183) | 논문(ICLR 2025) | 2024-09 | 검색 요약 | CoT 이득이 수학과 기호 추론에 집중 |
| [Sculley et al., Hidden Technical Debt in ML Systems](https://proceedings.neurips.cc/paper/2015/file/86df7dcfd896fcaf2674f757a2463eba-Paper.pdf) | 논문(NeurIPS 2015) | 2015 | 검색 요약 | 모델 주변 시스템의 유지보수 비용 |
| [Wei et al., Chain-of-Thought Prompting](https://arxiv.org/abs/2201.11903) | 논문 | 2022 | 서지 인용 | CoT의 출발점 |
| [Kojima et al., LLMs are Zero-Shot Reasoners](https://arxiv.org/abs/2205.11916) | 논문 | 2022 | 서지 인용 | "단계별로 생각해 보자" 한 문장 CoT |
| [Wang et al., Self-Consistency](https://arxiv.org/abs/2203.11171) | 논문 | 2022 | 서지 인용 | 다수결 방식 |
| [OpenAI, Learning to reason with LLMs](https://openai.com/index/learning-to-reason-with-llms/) | 발표 글 | 2024-09 | 서지 인용 | 추론 모델의 등장 |
| 과제 파일([ASSIGNMENT.md](../kant-next-prompt-student/ASSIGNMENT.md), [README.md](../kant-next-prompt-student/README.md), [support.py](../kant-next-prompt-student/support.py)) | 과제 자료 | 2026-09-18 | 직접 열람 | 과제의 설정값과 측정 규율 |
