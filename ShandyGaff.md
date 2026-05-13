# [Shandy Gaff

> 클라이언트 FA 이벤트 스펙을 KMP로 통합 관리해보려다 폐기한 이야기

## 시작은 Notion 스펙 관리의 한계

이벤트 스펙을 Notion에 정의하고 Web/Android/iOS가 각자 구현하는 구조에는 구조적인 문제가 있었다.

스펙 하나는 대략 이런 모양이다.

**기본 정보**

- Event ID: `250`
- Event Name: `click`
- Event Description: `로그인 버튼 클릭`
- 상태: `파라미터 추가`
- 플랫폼: `web`, `and`, `ios`

**필수/핵심 파라미터**

- `target`: `login_button`

**Other params**

- `method`

**params description**

- `method`: 로그인 방식 (`email` / `kakao` / `apple` / `google`)

**플랫폼별 진행상태**

- web: `Done DataQA1`
- and: `Ready To Do`
- ios: `In Progress`

한 이벤트가 이 정도 면적을 차지하고, 이게 수백 개씩 Notion DB에 쌓인다. 여기서 발생하는 문제들.

- Notion 스펙과 실제 구현이 어긋남. 가이드일 뿐이라 누가 안 지키면 끝. 오타·파라미터 누락이 운영 중에 발견된다.
- 버전 관리 부재. 언제 누가 파라미터를 바꿨는지, 이전 스펙이 뭐였는지 추적이 안 된다.
- 단순 반복. 같은 스펙을 Notion, 세 플랫폼에 각자 옮겨 적는다.

## 떠올린 그림 — 코드가 곧 스펙

코드를 SSOT로 삼고 세 플랫폼이 같은 정의를 공유하면 위 문제는 다 사라진다. KMP의 사용 사례로 어색하지 않은 그림이었다.

화면 단위 interface로 이벤트를 정의하면, export 결과물 자체가 *"이 앱에 어떤 화면이 있고 각 화면에서 뭘 추적하는지"* 의 지도가 된다. interface 이름만 훑어도 앱 구조가 잡히고, 메서드 이름만 봐도 추적 시점이 보인다. 빌드 타임에 오타·파라미터 누락이 막힌다.

```kotlin
interface LoginEvent {
    /**
     * 로그인 화면이 사용자에게 노출되었을 때.
     */
    fun onShow()

    /**
     * 로그인 버튼 클릭.
     *
     * @param method 로그인 방식 ("email", "kakao", "apple", "google")
     */
    fun onClickLogin(method: LoginMethod)
}
```

여기에 AI를 한 층 얹어서, PM이 *"로그인 화면에 '로그인 시도' 이벤트 넣어줘"* 라고 말하면 미리 정의한 Skill이 interface 정의부터 배포까지 처리하는 흐름까지 그렸다. PM 입장에서는 코드를 직접 보지 않아도 되고, 우리 입장에서는 스펙이 PR/커밋 단위로 추적된다.

여기까지가 이상이었다.

## 막상 export 결과를 뜯어보면

### iOS — 바이너리에서 나오는 건 정체불명의 Objective-C 헤더 한 덩어리

`./gradlew assembleXCFramework`로 나오는 결과물은 `shared.xcframework` 하나. 그 안에 있는 건 `shared.h` — 모든 클래스/인터페이스가 합쳐진 Objective-C 헤더 하나다. Swift 코드는 생성되지 않는다. Xcode가 그 헤더를 보고 Swift처럼 보이게 자동 매핑할 뿐.

또 **명시적으로 나눠져 있지 않다.** `LoginEvent.h`, `SignupEvent.h`처럼 화면별로 파일이 분리되지 않는다. 모든 게 한 헤더에 뭉쳐 있어서 iOS 개발자가 헤더를 훑어서 *"이 앱에 어떤 화면이 있구나"* 를 파악할 수 없다. 우리가 핵심으로 노렸던 *interface가 곧 지도* 가 무너진다.

### Swift wrapper로 우회 = 정의를 두 번 작성하는 꼴

이걸 해결하려고 시도해본 방법들.

- **SKIE.** Swift wrapper를 자동 생성해서 `.swiftmodule`로 묶어준다. 단 SKIE는 Flow/suspend/sealed class처럼 *변환 가치가 있는* 것만 건드리고, 평범한 interface는 사실상 그대로 둔다. 우리 케이스에서는 의미가 없었다.
- **Swift Export.** Kotlin 2.2.20부터 들어간 실험 기능. 진짜 Kotlin → Swift 변환이라 기대하고 시도해봤는데, KDoc 주석은 그대로 손실됐고 생성된 Swift 코드도 깔끔하다고 보기 어려웠다.

결국 도달한 형태는 **Kotlin 정의를 보고 Swift protocol을 한 번 더 작성**해서 SPM 패키지에 끼워 배포하는 것이었다.
 
```swift
import Foundation
 
public protocol LoginEvent {
 
    /// 로그인 화면이 사용자에게 노출되었을 때.
    func onShow()
 
    /// 로그인 버튼 클릭.
    ///
    /// - Parameters:
    ///   - method: 로그인 방식 ("email", "kakao", "apple", "google")
    func onClickLogin(referrer: String, method: String)
}
```
 
보기엔 가장 깔끔하다. 화면별로 파일이 분리돼 있고, KDoc에서 옮긴 주석이 그대로 살아 있다. iOS 개발자가 호출 지점에서 *정의로 가기* 한 번이면 해당 화면의 모든 이벤트 스펙을 한 파일에서 즉시 확인할 수 있다. 우리가 처음에 그렸던 *interface가 곧 스펙* 의 모습에 가장 가까운 형태다.
 
문제는 같은 스펙이 두 쌍으로 존재한다는 점이다. AI가 생성한다 해도 Kotlin과 Swift 두 출력이 일치하는지 매번 검증해야 한다. SSOT 한 줄을 지키자고 시작했는데, 두 정의 사이의 drift를 감시하는 비용이 새로 붙는다.

### JS — 단순 번역, 주석은 손실

Kotlin/JS는 결국 트랜스파일러다. `.d.ts`는 생성되지만 한계가 있다.   
**KDoc → JSDoc 변환이 안 된다.** [KT-44239](https://youtrack.jetbrains.com/issue/KT-44239)는 수년째 미해결. 시그니처만 옮겨지고 `@param` 설명은 다 사라진다. 웹 개발자가 IDE에서 메서드를 호버해도 설명이 보이지 않는다 — interface가 곧 스펙이라는 명분에 직격탄.

대안은 Kotlin 소스를 파싱해 직접 `.d.ts` + 순수 JS 함수를 만드는 codegen인데, iOS의 Swift wrapper와 마찬가지로 결국 *Kotlin 정의에서 한 번 더 떨어져 나온 산출물*이다. 또 두 군데에 정의가 존재한다.

### 생태계 자체의 낮은 관심도

KMP/JS를 프로덕션에서 쓰는 곳을 찾기가 어렵다. TS + React 생태계가 너무 강력해서 *"굳이 Kotlin으로?"* 의 벽이 높다.

## 폐기

SSOT 한 줄을 지키려고 시작했는데, 플랫폼별 wrapper를 한 번씩 더 만들면 SSOT 명분이 옅어진다. 그럴 거면 처음부터 각 플랫폼이 자기 언어로 받는 contract 포맷 — 스키마 기반 codegen, OpenAPI 스타일 IDL 같은 — 이 더 깔끔하다. KMP는 "Kotlin이라는 언어 자체"가 SSOT의 표면이 되어야 가치가 있는데, 외부로 노출되는 순간 그 표면이 깨지는 게 이번 시도의 본질적인 결론이었다.

문제의식 — *"Notion 가이드는 강제력이 없다, 코드가 스펙이어야 한다, AI가 정의를 대신 다뤄야 한다"* — 는 그대로 유효하다. 다만 그 그릇은 KMP가 아니었다.
