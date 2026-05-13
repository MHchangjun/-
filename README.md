# 소개

* Name : 송창준
* Github : https://github.com/MHchangjun

# 경력
* [Watcha](https://watcha.com/) 안드로이드 엔지니어 (2021.4 ~ 현재)
* [Hitit](http://hitit.xyz/) 안드로이드 엔지니어 (2020.2 ~ 2021.2)

# Android × AI 🤖

Claude Code, Codex, Gemini — 디자인→코드, 리팩토링, 리뷰까지 매일 쓴다.   
근데 왜 agent는 끝나야 하지? 그냥 계속 돌면서 기여하면 안 되나? 다들 한 번쯤 해본 생각 아닌가. 피쳐 말고도 코드베이스에 쌓인 일은 산더미인데 말이다.

## Custom Agent

안드로이드를 잘 아는 사람이 직접 turn 흐름과 도구를 깎아, 24시간 코드베이스에 기생하며 기능은 그대로 둔 채 정리할 수 있는 모든 것 — 아키텍처·모듈화·코드 스멜·테스트 코드 — 을 알아서 다듬는 구리 골렘([Copper Golem](https://minecraft.wiki/w/Copper_Golem)). 그걸 목표로 개발하고 있다.

### Kanna (2025.08 ~ 2025.12)
2025년 초부터 다양한 Coding Agent를 실무에 도입해봤지만, 왓챠·피디아·숏챠 3개 앱을 적은 인원으로 운영하며 쌓인 레거시를 정리하기엔 실제 머지로 이어지는 변경이 드물었다. GPT와 대화할 땐 모든 걸 아는 것처럼 보였지만, 막상 Coding Agent로 코드베이스에 적용하는 순간 그 괴리가 컸다. 그 간극을 메워보고자 안드로이드 리팩토링 전용 Coding Agent Kanna를 개발을 시작했다.   

[더보기](https://github.com/MHchangjun/-/blob/master/kanna.md)

### Smoker (2026.1 ~ )

Kanna에서 파고들었던 Local LLM을 기반으로 방향을 옮겨 **Smoker**를 개발 중이다. 상용 Agent가 잘 처리하는 단발성 작업이 아닌, 정적 분석 이슈 수정처럼 안드로이드 개발자가 반복적으로 챙겨야 할 유지보수 영역을 자동화한다.

[더보기](https://github.com/MHchangjun/-/blob/master/smoker.md)

## 고찰 🤔
* [Intell Mac Pro에서 Local LLM 서빙하기 — Ollama -> llama.cpp + Vulkan](https://github.com/MHchangjun/-/blob/master/serving.md)
* [Mac Pro(2019)로 122B 모델을 돌릴 수 있을까?](https://github.com/MHchangjun/-/blob/master/serving2.md)
* [Model를 알고 Agent를 알아라](https://github.com/MHchangjun/-/blob/master/model.md)
* 항상 똑같은 system prompt 어떻게 cache할까?
* [경량급 Local LLM을 사용하는 Smoker가 코드 스멜을 99% 고치게 된 이유 - auto diagnostics](https://github.com/MHchangjun/-/blob/master/auto-diagnostics.md)

---

## 회사에선

전사적으로 AI 도입에 적극적이다 — 개발자가 아니어도 누구나 코딩할 수 있도록 'Claude Code Club'이 사내에서 운영되고 있다.   
그 흐름 위에서, AI로 풀 만한 문제들을 직접 발굴해 각 팀과 함께 풀어보고 있다.

### Shandy Gaff

AI 시대 문서와 코드 간에 경계를 깨 보자

[더보기](https://github.com/MHchangjun/-/blob/master/ShandyGaff.md)

---

# 왓챠

우리 팀은 특정 앱이나 기능이 한 사람에게만 의존하지 않도록 Bus factor를 높이는 방식으로 일했다.   
세 개의 Android 앱을 운영하면서도 담당자를 고정하기보다는, 누구든 필요한 앱에 들어가 개발하고 유지보수할 수 있는 구조를 지향했다.

나 역시 여러 앱을 오가며 기능 개발, 버그 수정, 운영 대응을 해왔다.

## [숏챠](https://play.google.com/store/apps/details?id=com.frograms.atom)

> 개발 리드, 2025 구글플레이 ‘올해를 빛낸 숨은 보석 앱’ 선정

### 개발 전
* 왓챠, 왓챠 피디아 아키텍처, 모듈 구조 리뷰 후 숏챠 아키텍처 결정
* Android, Ios 팀과 KMM 논의

### 피쳐
* 숏폼 플레이어
* 콘텐츠 탐색탭
* 다국어 대응

### 개선
* instrumented test
* CI/CD
* 재생 에러 개선

### 개발
* Compose, Coroutine
* 구글 권장 아키텍처
* Ktor
* Media 3

### 문제 해결
* [이상적인 숏폼 플레이어](https://github.com/MHchangjun/-/blob/master/issue/player.md)
* [재생 이슈 : 분석](https://github.com/MHchangjun/-/blob/master/issue/playerCodecIssue.md)
* [재생 이슈 : Deep dive](https://github.com/MHchangjun/-/blob/master/issue/player-codec-issue2.md)

---

## 왓챠 모바일

### 개별 구매
* 소장, 대여할 수 있는 재화 왓챠 캐시 추가
* 자동 구매 복원
* 선물하기

### 왓챠 AIO (영화, 웹툰, 음악 올인원 구독)
* 여러 도메인을 담을 수 있는 탐색탭 개발
* 숏츠 형태로 여러 도메인을 탐색할 수 있는 ForYou
* 웹툰 뷰어

### 그 외 피쳐
* 소식함 추가
* 비로그인 유저 콘텐츠 탐색 지원
* 마이페이지 리디자인
* Player 배속 재생 추가
* 유입 경로 어트리뷰션 / 제휴·리뷰어 링크 트래킹

### 개선
* Single-Activity + Navigation Component로 전환: Activity 분산으로 복잡해진 화면 전환/딥링크/백스택을 통합해 네비게이션 일관성·상태 복원성 확보
* Remote/Local 모듈 분리 제안: 의존성 얽힘으로 커진 변경 영향 범위를 줄이고 빌드·테스트 경계를 명확히 하기 위해 Data 레이어부터 단계 분할
* 재생 정산 Ping 개선 + 테스트 코드 추가: 네트워크 변동/재시도/중복 호출에서 발생 가능한 정산 누락·중복 리스크를 줄이고 회귀 방지
* 탐색 애니메이션 규칙화(MaterialSharedAxis): 화면별 전환 차이로 인한 UX 산만함을 줄여 일관된 탐색 경험 제공
* RxJava → RxJava2 마이그레이션: Single, Completable 등 RxJava2 기반 타입/오퍼레이터 활용
* 주요 화면 접근성(TalkBack) 개선: 스크린리더 사용자를 위해 콘텐츠 설명, 포커스 순서, 터치 타겟 등을 정비

### 문제 해결
* [이상적인 탐색탭](https://github.com/MHchangjun/-/blob/master/issue/realm.md)

---

## 왓챠 TV

### 피쳐
* 개별 구매 탐색탭 추가
* 구독, 개별 구매 콘텐츠를 표현할 수 있는 통합 콘텐츠 상세 페이지 추가
* 개별 구매 성인+ 탭 추가
* 에피소드 선택 페이지 디자인 변경 대응

### 개선
* Leanback 기반 Navigation -> compose navigation migration
* Tv compose 도입으로 유연한 Focus 관리

---

## 왓챠 피디아

### 피쳐
* 로그인 개선, 이메일 인증 추가
* 콘텐츠 평가 카드 공유

## 고찰 🤔

* [suspend와 LSP의 계약에 대한 고찰](https://github.com/MHchangjun/-/blob/master/issue/suspend.md)
* [Compose Layout 속 그라데이션](https://github.com/MHchangjun/-/blob/master/issue/compose1.md)

# Hitit

> 현실성 없는 과도한 편집과 믿을 수 없는 정보가 넘쳐나는 현 소셜 미디어에 맞서 힛잇 HITIT 은 날 것의 생생함과 있는 그대로의 정확한 정보 공유

## 개발

* MVVM + repository
* room를 이용한 로컬 데이터베이스 (with rx)
* 비동기 처리에 Rxjava2 도입
* [motionlayout](https://developer.android.com/training/constraint-layout/motionlayout)를 이용한 애니메이션
* 백그라운드 영상 업로드, m3u8 트랙 캐싱을 위한 [worker](https://developer.android.com/topic/libraries/architecture/workmanager/basics) (rxWorker)
* mock와 robolectric를 이용한 Unit test, espresso를 이용한 UI test
* Camera 2 -> CameraX
* Exoplayer 2 + [단일 플레이어](https://github.com/google/ExoPlayer/issues/867#issuecomment-343781395)를 이용한 RecyclerView 비디오 playback 처리
* 코드 재사용, 테스트, Boilerplate 코드를 줄이기 위해 dagger2
* Epoxy

## DevOps

* Jira로 이슈 트래킹 및 칸반 보드를 이용하여 애자일 개발 프로세스 진행
* Bitbucket를 통해 프로젝트 관리 및 Pull Request 마다 코드 리뷰
* circleci 통한 빌드 자동화
* sonarqube를 통해 코드 퀄리티 유지
* Pull Request는 sonarqube Coverage 80% 이상 가능
* Confluence/Jira/Bitbucket/Sonarqube 등의 개발 통합환경

# 병역사항
병력특례 (2022.7.5 ~ 2025.5.5)
