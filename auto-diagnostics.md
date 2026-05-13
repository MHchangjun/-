# 경량급 모델을 사용하는 Smoker가 코드 스멜을 99% 고치게 된 이유 - auto diagnostics

경량급 Local LLM은 왜 코드 스멜 하나 제대로 수정하지 못 할까? Smoker를 개발하면서 계속 마주친 질문이다.

Smoker는 정적 분석 도구(detekt, IntelliJ Inspection 등)가 보고한 코드 스멜을 자동으로 고치는 Kotlin 에이전트다. 안에서 도는 모델은 GPT나 Claude가 아니라 로컬에 띄운 경량 LLM(Qwen3.6)이다.

그래도 Local LLM을 계속 붙잡는 이유는 분명하다.   
어떨 땐 큰 모델 못지않게 깔끔한 fix를 내놓는다. 수정 방향도 대체로 옳다. 막히는 건 Kotlin/Android의 사소한 문법 벽 — scope, smart cast, generated source 같은 데서 미끄러져 빌드를 깨뜨릴 뿐이다. 의도는 살아 있는데 마지막 한 칸이 어긋난다. 거기만 메우면 쓸 만하다는 감이 있다.

## 문제: 경량 모델은 자주 헛디딘다

큰 모델은 코드 스멜 같은 건 눈 감고도 푼다. 문제는 Smoker가 쓰는 모델이 그렇게 똑똑하지 못하다는 거다. 함수를 정의하지도 않은 채 호출부터 갈아끼운다. import를 빼먹는다. 모르는 타입이 나오면 FQN을 박아두고 import는 다음 턴에 추가하겠다고 한다. 변경 후 자기가 깬 것도 모르고 다음 이슈로 넘어간다.

이런 실수가 빌드를 깨뜨린다. 그리고 빌드가 깨지는지는, 빌드를 돌려보지 않으면 모른다.

## 그런데 빌드를 못 돌리겠다

가장 자연스러운 검증은 "edit 한 번에 빌드 한 번"이다. 깨지면 rollback, 통과하면 다음 이슈.

문제는 Android 빌드가 가볍지 않다는 점이다. Gradle cache 덕분에 두 번째 빌드부터는 짧아지지만, 프로젝트 규모에 따라 cache가 살아 있어도 한 번에 수십 초가 나온다. 코드 스멜이 수백 개라는 점을 곱하면 답이 안 나오고, 그 시간 내내 에이전트는 빌드 결과만 기다리며 놀고 있다.

## 1차 시도: Kotlin LSP

빌드 없이 diagnostics를 받는 가장 표준적인 길은 LSP다. JetBrains가 만든 `kotlin-lsp`는 IntelliJ 엔진 위에 LSP 프로토콜을 얹은 거라서, 이론적으로는 IDE 수준의 분석을 외부에서 받을 수 있다.

세팅 자체는 했다. 에이전트 시작 시 타겟 프로젝트를 Kotlin LSP 서버로 올리고 `textDocument/diagnostic`으로 diagnostics를 받는 구조. 보기엔 깔끔했다.

근데 막상 돌려보니 문제가 두 가지였다.

### **LLM이 LSP tool을 안 부른다.**

diagnostics를 받으려면 LLM이 정의한 `lsp_diagnostics` tool을 호출해야 한다. 그런데 경량 모델한테 "검증하라"고만 시키면 LSP tool로 가지 않고 shell tool로 빌드를 돌리려고 한다. 학습 데이터에 "검증 = 빌드"가 워낙 강하게 박혀 있어서, LSP가 옆에 있어도 그쪽으로 안 간다.

### **Kotlin LSP는 Android를 모른다.**

이게 더 치명적이었다. `kotlin-lsp`가 반환하는 diagnostics는 일반 Kotlin inspection 수준 이슈는 잘 잡지만, Android Gradle Plugin이 관할하는 영역에서 깨지는 진짜 에러는 못 잡는다. Android Studio 안에서는 빨간 줄이 명확히 그어지는 상황인데 LSP는 그걸 "ERROR"로 안 던지는 경우가 많았다. "Android LSP" 같은 것도 시장에 없다.

LSP 카드는 여기서 접었다.

## 방향 전환: IntelliJ Plugin

빌드를 안 돌리고 Android의 diagnostics를 받으려면, 결국 **Android Studio가 이미 하고 있는 분석 결과를 그대로 가져다 쓰는 것**이 가장 빠르다. 그게 IntelliJ Platform 위에서 도는 plugin이다.

CLI 에이전트를 IntelliJ plugin으로 옮기는 건 결정이 무거웠다. CLI 한 줄이 IDE 환경 의존이 되는 거고, headless 운영이 어려워지고, plugin 배포 사이클도 새로 들어온다. 그래도 가치가 컸다.

Plugin 안에 들어가면 IDE가 background에서 돌리는 분석을 그대로 받아 쓸 수 있다.   
edit당 수 ms~수 초면 diagnostics이 도착하는데, Gradle 호출 한 번이 5~30초라는 걸 생각하면 차원이 다르다. kotlin lsp엔 없는 Code Action(Optimize Imports, Quickfix...)을 사용할 수도 있다.

## EditTool에 diagnostics 결과를 박아 넣다

Plugin으로 옮기면서 가장 큰 설계 변화는 **diagnostics를 별도 tool로 분리하지 않고 EditTool 응답에 자동으로 합쳐버린 것**이다.

LSP 시절에는 이런 흐름이었다.

```
LLM이 edit_tool 호출
  ↓
파일 수정
  ↓
LLM이 lsp_diagnostics 호출 (← 자주 빼먹음)
  ↓
diagnostics 받음
  ↓
새 에러 있으면 재수정
```

문제는 가운데 단계를 모델이 자주 빼먹는다는 거였다. 별도 단계로 두면 시그널이 약해진다.

Plugin EditTool은 이렇게 바꿨다. 한 번의 edit 호출 안에서 파일을 in-memory로 갱신하고, IDE가 자동으로 재분석할 때까지 기다린 다음, 그 결과를 응답에 함께 담아 돌려준다.

핵심은 세 가지다.

- 디스크를 거치지 않고 in-memory로 바로 갱신해 stale diagnostics 가능성을 없앤다.
- Polling이 아니라 IDE의 분석 완료 이벤트를 구독해 정확히 필요한 만큼만 기다린다. 작은 파일이면 ms 단위, 큰 파일이면 수 초.
- LLM에게 별도 단계로 diagnostics를 시키지 않고, EditTool 응답 끝에 자동으로 붙인다.

시스템 프롬프트에는 한 줄만 들어간다.

> Edit results include auto-injected `[diagnostics]` — ensure no new errors before moving on.

`diagnostics`라는 단어는 LSP, VS Code, TypeScript compiler에서 표준으로 쓰여 경량 모델도 즉시 "구조화된 분석 결과"로 매핑한다. tool 출력 헤더와 prompt 어휘를 일치시키면 모델이 헤매지 않는다.

## 결과: 50% → 99%

auto diagnostics를 붙이기 전엔 검증 자체를 생략하고 갔다. 모델 수정을 그냥 믿고 코드 스멜을 끝까지 다 돌린 다음, 마지막에 빌드를 돌려 깨진 파일은 rollback하는 식이었다. 10개 리팩토링 시도하면 5개는 그렇게 버려졌다. 

지금은 같은 케이스의 99%가 통과한다. 

모델이 갑자기 똑똑해진 게 아니다. 한 가지가 바뀌었을 뿐이다 — edit 직후에 diagnostics를 코앞에 들이밀면, 경량 모델도 자기가 깬 걸 본다.

## Trade-off

공짜는 아니다.

* **무거워졌다.** CLI 한 줄로 끝나던 게 이제 Android Studio가 켜져 있어야 동작한다. headless로 CI에서 돌리는 길이 막혔다. 사용자가 IDE를 띄워두는 시점에 background로 야간 작업을 돌리는 식으로 운영 모델이 바뀐다.
* **IntelliJ Platform API 학습 비용.** `WriteCommandAction`, `PsiDocumentManager`, `DaemonCodeAnalyzer`, `DumbService`, EDT threading — 외부에서는 보이지 않는 규칙이 많다. 한 번 익히면 강력하지만 진입 장벽이 있다.

대신 얻는 게 명확하다. **Android Studio가 가진 모든 분석 능력에 그대로 접근할 수 있다.** Code Inspect, Lint, Quickfix, Refactoring action, Run/Debug configuration — 외부에서는 인터페이스 자체가 없는 것들이다. 지금까지는 진단만 가져다 쓰고 있지만, 같은 통로로 Compose Preview 검증, Hilt graph 검증, Apply Quickfix 자동 invoke까지 확장할 여지가 열려 있다.

## 한 줄 요약

경량 모델로 Kotlin 코드를 안전하게 고치려면, 모델을 키울 게 아니라 모델이 받는 피드백 루프를 짧고 정확하게 만드는 게 답이었다. 빌드를 돌리는 대신 IDE가 이미 하고 있는 분석에 올라타고, diagnostics를 별도 단계로 두는 대신 edit 응답에 자동으로 붙이는 것. 그게 50%를 99%로 바꿨다.   
결국 모델이 Kotlin이나 Android를 깊이 몰라도 된다. 잘 학습된 모델이라면 edit 직후에 받는 compile / diagnostics 결과를 단서로 받아들이고 스스로 복원해낸다. 모델 능력의 부족분은 피드백 루프가 메운다.
