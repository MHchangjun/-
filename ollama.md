# Coding Agent 개발기

작년 중순 즘 여러 AI Agent가 나오면서 피쳐 개발보다 기존 코드를 리팩토링하는데 여러 AI를 사용해 봤지만 결과도 만좁 스럽지 못하고 Agent가 실제 어떻게 돌아가는지 볼 수 없는 black box인 경우가 많았음. 
또 안드로이드에는 정말 다양한 요소를 신경 써야한다 hilt라 던가 databinding, resource 등 하지만 Coding Agent들이 하나같이 모듈을 분리할 때 이런 점을 전혀 신경 쓰지 못하고 느낌대로 모듈을 옮겨 놓고 다 했다고 함.
그래서 아주 강력한 Indexing DB 기반의 리팩토링 전용 Coding Agent를 만들어 보기 위해 Agent Framework인 Koog를 통해 Agent를 만들기 시작

담당하는 Android 프로젝트 3개 중 2개는 레거시와 스파게티 코드 목표하는 아키텍처에 맞지 않는 레거시 코드가 넘쳐나 이 프로젝트들을 클린 아키텍처로 리팩토링 계획
관성적이며 잘 알려진 클린 아키텍처고 우리 아키텍처에 대해 장황하게 AI에게 설명할 필요 없어 Android Studio에서 마우스 한 번 안쓰고 refactor 도구만으로 얼마든지 리팩토링할 수 있은 걸 알았고 리팩토링은 새로운 코드를 생성하기 보다, 
기존 동작을 최대한 유지한 체 코드를 수정하는 거니까 성능 좋은 AI가 굳이 필요 없을꺼라고 생각함
그리고 다른 AI Agent를 사용할 때 결제했던 거 대비 계쏙 추가 크래딧을 사야하는 한계 때문에 로컬에서 돌아갈 수 있는 걸 골랐고 Koog에서 Ollama를 지원했다

그래서 시작된 강력한 Indexing DB와 LSP 기반 안드로이드 전용 리팩토링 Agent

## Model 선택

정말 서버급 컴퓨터가 아닌 이상 30b 이하 모델을 사용해야하고 여러 회사에서 개발한 Model도 이런 니즈에 맞춰 30b 이하로 많이 분포되어 있다.
또 llama.cpp로 바로 돌릴 수 있는 GGUF 타입에 모델이 있고 세부적인 세팅 없이 바로 서빙 가능한 Ollama 모델이 있다.

Ollama 모델 중 Coding에 특화된 모델을 찾으면 된다.

### Tool Call
Agent는 유저에 prompt를 이해하고 목표를 달성하 위해 여러 툴을 호출한다. 그래서 Tool 호출 가능한 모델을 사용해야한다 아무리 코딩을 잘한다고 한들 Agent에서 Tool를 호출할 수 없는 모델은 전처리만 늘어난 짐덩어리다.

### SWE Bench

### 코드 수정
Indexing DB를 이용하거나 LSP를 이용해 정확한 코드만 LLM Session에 올리고자하는 건 당연 context engineering 때문일 거다
그런데 1000줄 짜리 코드 중 일부 함수를 수정하는 Tool를 생각해 보자 그냥 무식하게 덮어쓰기하기엔 그 1000줄짜리 코드 수정 patch가 LLM Session에 그대로 올라갈 것이다.

### 결국 직접 설계해야하는 Turn
Koog Framework에서 다양한 기능이 있고 여러 strategy를 지원하지만 결국 LLM Session에 뭘 넣고 뭘 빼야할지 세부적으로 컨트롤할 수 없다.
또 Streaming는 직접 턴을 제어해야해 결국 Koog를 쓰는 이유가 Ollama client를 사용할 수 있다 정도?
Android 개발자가 좋아 죽은 kotlin인거 빼면 장점이 별로 없은거 같다.

### context engineering도 중요하지만..
LLM에 성능을 늘릴 수 있는 방법은 많다.
Agent를 직접 개발한다고 Context에만 신경 쓰지말고 기존 Coding Agent들은 어덯게 system prompt를 구성했는지 참고하면 좋다.
개발 초기에 system prompt엔 너는 마틴 파울러야~ 클린 아키텍처란, 안드로이드 코틀린 프로젝트고~ 이런 prompt를 넣어 두고 context 줄이기만 실경 쓰다 직접 수정하려고 하기 보다 자꾸 장황한 설명을 하길래 
이리 저리 바꿔보다 Codex cli system prompt를 참고해 내 Agent에 맞게 수정했더니 매우 Agentic하게 잘 돌아갔다.

### LLM는 지능이 아니다.
한창 코드 수정 Tool에 매달려 있을 때 LLM이 git diff를 생성해 주면 아주 쉽게 부분적으로 코드를 수정할 수 있으니 Context를 아낄 수 있겠거니 해서 여러 LLM으로 테스트해 봤지만 결국 다 실패했다.
LLM는 지능이 아니다. Code Line를 세게하면 안된다. 항상 hunk header가 틀린다. 그래서 문자열을 기준으로 patch하는 Tool를 사용해야한다. LLM에게 line를 세게 두지말고 넘기지도 말자

### LLM Session
LLM Session은 마법에 공간이 아니다. 모델이 가지고 있는 메모리가 있는게 아니라 이전 system prompt 부터 user, assistant, tool 기록 뒤에 붙여 다시 LLM를 호출할 뿐이다.

### LLM 특히 낮은 파라미터 일 수록 prompt는 소용 없다
저 프로젝트를 실패한 원인이자 한계라고 생각한다.

LLM는 자기가 학습한 대로 Tool를 호출하는 경우가 많다.
LLM session에 전혀 등록되어 있지 않는 툴을 호출하려는 경우도 대다수다.

대부분에 Coding Agentic를 강조하는 모델은 shell 기반으로 학습되어 있다.
그래서 아무리 Indexing DB를 활용한 Tool를 쥐어 줘도 cat으로 코드를 불러오고 grep으로 코드를 찾고 ls로 파일을 뒤진다.
또 kotlin lsp에서 지원하는 optimize import를 tool로 만들어 등록한들 잘 사용하지 않는다.

또 Model 마다 코드를 수정하는 tool이 다르다.
자기가 학습한 patch tool로만 patch를 생성하려고 한다.

git oss는 codex diff
devstral small 2는 search/replace

단지 이렇게 학습되었다 일뿐 patch를 잘 생성하는 것도 아니다.

모델이 어떻게 학습되었는지 어떻게 알까?
어떤 patch 도구를 학습했고 어떤 tool description이 잘 워킹할까?

정답은 LLM 모델을 개발한 회사 자체 오픈소스 Agent나 Blog를 보면된다.

gpt oss 20b 같은 경우 실제 블로그에서 Codex diff는 gpt oss 개열에서 생성 가능한다.
또 devstral, qwen 같은 경우 오픈 소스 CLI coding agent가 존재하고 어던 tool이 있고 system prompt 등 모두 확인 가능하다.

즉 이렇게 모델이 학습된 방향을 알아야 적절한 tool이 호출되어 agentic한 모델을 사용 가능하다.

오죽하면 codex cli system prompt에서도 제발 apply_patch tool를 사용하라고 여러번 강조한다.

### 자기가 모르는 언어가 있으면 patch를 생성하지 못하고 장황한 설명한 늘어 놓은다.
한국어 주석이 있으면 patch tool 호출을 절대 못 한다.

### 

## 결론
Codex Cli엔 거창한 Tool도 Prompt도 없다 anthropic 블로그를 보면 그들이 얼마나 모델과 Agent에 간에 치밀하게 설계하는 지 알 수 있다.   
처음엔 단지 하나의 Agent가 kotlin도 하고 python도 하고 android도 하고 IOS도 하려면 괜한 비용이 어디선가 발생하고 있겠거니   
안드로이드 환경에 맞은 다른 플랫폼 따윈 모르는 안드로이드 only Agent를 만들 수 있다고 생각했지만

개발하고 테스트하며 지식이 늘어갈 수록 부질 없다는게 느껴진다.
