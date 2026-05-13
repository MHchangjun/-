# Kanna (2025.08 - 2025.12)

> Android 프로젝트 Refactoring에 특화된 Custom Coding Agent를 만들어 보자

## 개요
왓챠와 피디아에 축적된 레거시 코드를 해결하기 위해 여러 코딩 에이전트를 사용해 보았지만, 수많은 코드 변경에도 불구하고 실제 병합(Merge)으로 이어지는 커밋(Commit)은 드물었다.

기존 에이전트들은 다음과 같은 한계를 보인다.
* 잦은 세션 중단
* 에디터 플러그인 형태가 오히려 작업 흐름을 방해
* 테스트 코드 기반의 점진적 리팩토링이 아닌, 직감에 의존하는 수정으로 인한 빌드 오류 발생
* 쓸데 없는 Token 낭비

Kanna 프로젝트는 이러한 문제들을 Coding Agent를 직접 구현하여 극복하고자 한다.

## 목표
* 강력한 Indexing DB 기반 탐색 + Android Studio Refactor 액션을 그대로 옮긴 Tool 기반 Agent — 임의로 텍스트를 쓰고 지우는 Edit Tool은 제공하지 않는다. IDE의 Refactor 액션(이름 변경, 인터페이스 추출, 이동, 인라인, 함수 추출 등)만을 Tool로 노출
* 토큰이 일정 수치에 도달해 히스토리 압축하는게 아니라 안드로이드에 맞은 똑똑한 히스토리 압축과 목적에 맞은 여러 agent들을 묶어 24시간 무인 동작
* LLM Model 최적화로 비용 최소화

## 사용 기술
* Koog : Kotlin에서 LLM과 Tool를 Graph·Session으로 조합하고 관측·메모리 등 프로덕션 기능을 내장한 AI 에이전트 프레임워크
* kotlin analysis api : Symbol 해석
* Git : Worktree, PR, Commit, Apply
* Ollama(qwen3-coder:30b, gpt-oss:20b), OpenAI api

## 설계

### Plan Agent (Local LLM)

코드베이스 전체를 순회하며 SRP 위반 클래스를 식별하는 진입 단계.

통상의 CLI 기반 Agent는 grep/glob으로 코드를 탐색하기 때문에 "프로젝트 내 모든 클래스의 책임을 분류한다"는 작업 자체가 성립하지 않는다. LLM에게 파일 트리를 직접 돌게 두면 토큰만 낭비되고, 정작 분석이 필요한 파일에 닿기 전에 컨텍스트가 무너진다. 따라서 Plan Agent는 LLM 호출 전에 정적 분석으로 후보를 좁히는 턴(turn) 구조로 설계했다.

1. **클래스 목록 추출** — Kotlin Analysis API(PSI)로 프로젝트 내 모든 클래스를 수집한다.
2. **정적 전처리로 후보 축소** — 책임 분석이 명백히 불필요한 파일을 룰 기반으로 걸러낸다. Hilt `@Module`, `~Response`/`~Dto`처럼 필드만 가진 data class, 구현체가 없는 `interface`/`sealed` 선언, `object` 상수 등이 제외 대상이다. 이 단계까지는 LLM을 호출하지 않는다.
3. **카테고리 분류** — 살아남은 후보만 LLM에 파일 단위로 넘겨 9개 카테고리(UI, Domain, Data, Network, Storage, Contract, Analytics, Content, Platform)로 분류하고 결과를 DB에 기록한다.
4. **SRP 위반 판정** — 한 클래스에서 두 개 이상의 카테고리가 검출되면 SRP 위반으로 마킹한다.
5. **다음 Agent로 전달** — 위반 클래스 목록을 검출된 카테고리 정보와 함께 Test Code Agent에 전달한다.

### Test Code Agent (OpenAi)

리팩토링 전, 동작을 고정시키기 위한 안전망(Safety Net) 구축 단계.

1. Plan Agent가 전달한 클래스에 대해 기존 테스트 코드의 존재 여부와 커버리지 확인
2. 테스트가 없거나 부족하다면, 현재 동작을 그대로 보존하는 Characterization Test 작성
3. 이후 Refactor Agent들이 변경을 가할 때 회귀(Regression) 검증의 기준선(baseline)으로 활용

### Refactor 1 Agent (Local LLM)

처음부터 UseCase / Repository / DataSource를 한 번에 정의하게 하면 LLM이 절대 해내지 못한다. 따라서 "구현 분리"라는 한 가지 작업에만 집중시킨다.

1. 대상 클래스 내부에서 구체적 구현(concrete implementation)만을 식별해 분리. 아키텍처 재배치는 다음 Agent의 책임으로 미룸
2. UI 레이어의 경우, 비즈니스 로직을 우선 ViewModel로 이동시켜 View와 로직의 결합도를 낮춤
3. 분리된 구현체는 interface와 impl로 쪼개 의존성 역전(DIP) 준비를 마침

### Refactor 2 Agent

Refactor 1이 분리해 둔 조각들을 타겟 아키텍처에 맞춰 정렬하는 단계.

1. 분리된 interface/impl을 현재 프로젝트의 타겟 아키텍처(Clean Architecture 기반 UseCase, Repository, DataSource 계층)에 맞게 재배치
2. 모듈 경계와 의존성 방향(Domain ← Data, Domain ← UI)이 깨지지 않도록 패키지/모듈 이동 처리
3. DI 설정(Hilt/Koin)을 갱신하고, Test Code Agent가 작성한 테스트로 회귀 여부 최종 검증

## 실패의 원인 1 : 흔들이는 기준

Plan Agent는 동작했지만, 정작 그 판정 기준인 SRP 자체가 끊임없이 흔들렸다.

"이유가 오직 하나뿐이어야 한다." 문제는 이 기준을 들고 코드베이스를 들여다보니, 한 줄로 판정할 수 있는 케이스가 거의 없었다는 점이다.

ViewHolder가 `UserHelper.setUser(user)`를 호출하는 코드를 예로 들어 보자. 겉보기엔 헬퍼 API 한 줄을 호출할 뿐이라 깔끔해 보인다. 그런데 `setUser` 내부를 열면 `SharedPreferences`에 직접 쓰고 있다.

- UI가 저장소를 직접 참조하지는 않으니 SRP 위배가 아닌가?
- 헬퍼라는 이름의 얇은 wrapper 뒤에 저장소가 숨어 있을 뿐이니, 본질적으로 UI가 영속성을 알고 있다고 봐야 하나?
- 애초에 이 호출 자체가 ViewModel에 있어야 하고, SRP 이전에 MVVM 적용이 선행되어야 하는 문제 아닌가?

세 답 모두 일리가 있다. 그리고 코드베이스 전체에 이런 모호한 케이스가 도처에 깔려 있었다.

궁극적으로 가고 싶은 그림은 분명했다. `UserHelper.setUser`를 ViewModel로 끌어올리고, `UserHelper` 대신 `LocalDataSource`를 정의하고, 그 위에 Repository와 UseCase를 올려 ViewModel이 추상화를 참조하게 만드는 것.   
문제는 이걸 SRP 위배라는 이름으로 후보에 넣는 게 정당한가였다.

## 실패의 원인 2 : SOLID 원칙에 매몰된 사고

이 모호함을 단번에 걷어낼 답이 SRP 안에 있을 거라 믿었다.

"이유가 오직 하나"의 "이유"가 정확히 뭔가? Robert C. Martin은 이걸 **변경의 주체(Actor)**로 정의했다. 한 모듈은 단 하나의 Actor에 대해서만 책임을 져야 한다.

여기서부터 무너지기 시작했다. Kanna는 여러 프로젝트에 배포될 범용 도구였다. Actor 판정을 하려면 팀마다 조직도와 스펙 문서를 Agent에게 주입해야 하는데, 그건 리팩토링 도구가 요구할 만한 게 아니었다. 결국 Kanna 어디에도 Actor라는 단어를 쓰지 못했다.

거칠게라도 Actor를 "콘텐츠팀", "결제팀"으로 나눠 본다 한들 마찬가지였다. UseCase에서야 깔끔하게 적용되지만, UI는 본질적으로 여러 Actor를 한 화면에 모으는 일이다. 홈 화면을 Actor 단위로 Fragment 쪼개는 게 답일 리 없다.

매몰되어 있던 건 SOLID가 아니라, **SOLID 안에 답이 있을 거라는 믿음**이었다.

## 실패의 원인 3 : Local LLM에 한계

* 2025년 기준 Tool Call을 지원하는 Local LLM 자체가 드물고, 지원하더라도 Agentic한 추론이 거의 불가능하고 장황한 설명한 늘어 논다.
* Agentic 모델들은 CLI 환경의 grep/glob 기반 탐색에 학습되어 있어 내가 설계한 Tool을 제공해도 호출하지 못 했다.
* 모델마다 학습된 방향이 달라 LLM을 교체할 때마다 Tool 정의를 모델에 맞춰 다시 작성해야 했다.

## 개발을 중단한 이유
1. 요구사항 정확도 — 2026년 기준 상용 Coding Agent는 리팩토링 의도를 정확히 파악하고, 빌드가 깨지지 않는 수준의 변경을 안정적으로 수행한다. Kanna가 목표로 했던 "merge 가능한 커밋"이 이미 기본값에 가까워졌다.
2. 압도적인 발전 속도 — 결국 자체 LLM을 가진 회사가 이 시장을 끌고 간다. Anthropic 기술 블로그만 따라 읽어도 그들이 어디까지 와 있는지 보인다. grep/glob에 머물던 모델이 이제는 LSP 설치를 능동적으로 요구한다.
3. SKILL이라는 추상화의 등장 — Kanna가 "Agent 묶음 + 도메인 지식 주입"으로 풀려던 부분이 SKILL이라는 공식적인 형태로 제공된다. 즉, Plan / Test / Refactor Agent를 직접 그래프로 엮지 않아도 동등한 결과를 얻을 수 있는 길이 열렸다.
4. 합리적인 비용 — 직접 Local LLM을 운영해 절감하려 했던 비용 이점이, 상용 Agent의 구독 가격 안에서 충분히 흡수 가능한 수준이 됐다.
5. 환경 독립성 — Android Studio / IntelliJ Plugin에 종속되지 않고 CLI에서 git worktree를 자유롭게 오가며 동작하는 워크플로우가 상용 Agent에서 이미 자연스럽게 지원된다.

## 배운점
* Coding Agent는 내 코드 베이스를 학습한 전지 전능한 도구가 아니다.
* 단일책임이란 무엇인가
* LLM turn과 session
