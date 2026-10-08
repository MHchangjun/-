# 화면은 무엇인가에 대한 고찰

## 개요

Agent를 24시간 돌리면서 코드베이스에 뭐라도 기여하게 만들고 싶었다.

아키텍처 검사, UI 검증, 테스트 코드 작성. 뭐든.

근데 특별한 설계 없이 그냥 풀어두면 뭐 하나 수정 못 하고 끝난다.

Agent는 작업을 시작하면 주변 파일을 자꾸 읽어 들인다.

예를 들면 엄청 큰 lint 결과 파일. grep에 자꾸 잡히니까 Agent는 굳이 그걸 열어 본다. 작업과 아무 상관 없는 파일 하나에 context가 폭발한다.

그렇게 token 임계치에 도달하고, 정작 수정 한 줄 못 한 채 session이 종료된다.

## 경계

그래서 필요한 건 **경계(boundary)** 다.

Agent를 특정 범위 안에 가두고, "이 안에서만 검사해라, 이 밖은 읽지 마라"고 제한해야 한다.

생각한 전략은 이렇다.

* 코드베이스를 `List<화면>` 으로 분해한다.
* Agent를 24시간 돌리되, session 단위로 화면 하나씩 맡긴다.
* 각 session의 Agent는 자기 화면 경계 안의 클래스만 읽고, 그 안에서만 작업한다.

context가 화면 단위로 제한되니 token 폭발을 막을 수 있고, 작업 범위가 명확하니 Agent가 끝까지 수정을 완료할 수 있다.

여기서 질문이 하나 나온다.

그래서 Android에서 **화면**이 정확히 뭔데?

이걸 2주 동안 고민했다.

## ViewModelStoreOwner

내가 잡은 기준은 `ViewModelStoreOwner`다.

**ViewModelStoreOwner를 구현했다면 그건 화면이다.**

근거는 ViewModel의 역할에 있다.

ViewModel은 화면의 상태와 비즈니스 로직 접근을 담는 클래스다. 어떤 컴포넌트가 ViewModel을 가질 수 있다는 건, 그 컴포넌트가 유지해야 할 상태와 비즈니스 경계를 가진다는 뜻이다.

그리고 ViewModel을 가지려면 ViewModelStore가 있어야 하고, 그걸 노출하는 게 ViewModelStoreOwner다.

즉 ViewModelStoreOwner를 구현했다는 건 "나는 독립적인 상태와 비즈니스 경계를 가진 단위다"라는 선언처럼 읽힌다.

Activity와 Fragment는 둘 다 ViewModelStoreOwner를 구현한다. 둘 다 화면이다.

깔끔하다.

## 그런데 Compose

Compose에 오면 이 깔끔한 그림이 무너진다.

`@Composable` 함수는 "이렇게 그려라"는 함수일 뿐이다. 자체적인 lifecycle도 없고, 자체적인 ViewModelStoreOwner도 없다.

그럼 Composable 함수는 어떻게 화면이 될까?

Composable 함수 자신은 화면이 아니다. 화면 경계는 항상 **바깥에서 주입되는 ViewModelStoreOwner**가 결정한다.

주입 경로는 크게 두 가지다.

### ComposeView

ComposeView는 ViewTree에 설정된 ViewModelStoreOwner를 가져온다.

결국 host인 Activity나 Fragment의 것을 끌어다 쓰는 거라, 화면 경계는 host Activity/Fragment다. Compose는 거기에 얹혀 있을 뿐이다.

### Compose Navigation

Compose Navigation에서는 `NavBackStackEntry`가 ViewModelStoreOwner를 구현한다.

NavHost는 destination을 그릴 때 해당 NavBackStackEntry를 `LocalViewModelStoreOwner`에 주입한다. androidx 소스를 보면 대략 이런 구조다.

```kotlin
currentEntry?.LocalOwnersProvider(saveableStateHolder) {
    (currentEntry.destination as ComposeNavigator.Destination).content(
        this,
        currentEntry
    )
}
```

`LocalOwnersProvider`가 owner를 주입하고, 그 안에서 우리가 `composable("route") { ... }`에 넘긴 람다가 실행된다.

따라서 화면 경계는 `composable("route") { }` 블록, 즉 그 route의 NavBackStackEntry scope다.

## 문제 : 끊긴 경계

기준은 정했으니, 이제 코드베이스에서 화면을 뽑아내기만 하면 된다.

도구로는 IntelliJ의 Indexing DB를 생각했다. 프로젝트 소스의 선언과 참조 관계를 미리 색인해둔 것으로, 대략 이런 걸 해준다.

* 선언 찾기 : 이름이 X인 클래스/함수가 어디 있나
* 참조 찾기 : 이 심볼을 누가 쓰나
* 관계 따라가기 : 상속 계층, 호출 그래프

Activity, Fragment는 문제없다. ViewModelStoreOwner를 구현한 클래스를 찾으면 끝이다.

문제는 Compose Navigation으로 정의한 화면이다.

`composable {}` 람다는 Indexing DB 상에서 ViewModelStoreOwner와 아무런 정적 관계가 없다. owner가 **런타임에 결정**되기 때문이다.

위 코드처럼 `destination.content(...)`로 dispatch되는 구조에서는, 람다를 등록한 곳과 실행하는 곳을 잇는 심볼 추적이 끊겨 있다.

Indexing DB는 코드에 적힌 심볼과 심볼의 관계만 알 뿐, 프레임워크가 런타임에 뭘 주입하는지는 모른다.

그래서 "이 `composable {}` 안의 `viewModel()`이 어느 owner에 귀속되는가"는 심볼만 따라가서는 답할 수 없다.

## 그냥 composable 람다를 화면이라고 하면 안 되나

```kotlin
composable(TvMenu.HOME.name) {
    TvHomeRoute(
        openContentDetail = openContentDetail,
        play = play,
        ...
    )
}
```

사실 지금 우리 코드베이스에선 그렇게 해도 대부분 맞는다.

하지만 그건 `androidx.navigation.compose.composable`이라는 특정 API 하나에 하드코딩하는 것이다.

Navigation API가 바뀌면? Compose Multiplatform Navigation은? Voyager, Decompose 같은 서드파티 라이브러리는?

그때마다 "이 API의 람다 = 화면"이라는 규칙을 하나씩 새로 박아야 한다.

내가 원한 건 그런 API 매칭의 나열이 아니다.

버전이 다르고, 라이브러리가 다르고, 주입 방식이 달라도 통하는 **단 하나의 기준**으로 질의해서 "여기가 화면이다"가 한 번에 나오는 것이다.

`composable {}` 람다를 화면으로 보는 건 그 기준에서 파생된 한 사례일 뿐이다.

(만드는 사람이 고집불통이라 🤦)

## 결론

하고 싶었던 건, 앱 안의 모든 ViewModelStoreOwner에 대해 경계를 파악하고, 각 owner에 대응하는 UI가 정확히 무엇인지까지 매핑하는 것이었다.

하지만 실패했다.

"ViewModelStoreOwner = 화면"이라는 기준 자체는 여전히 마음에 든다. 다만 그 기준이 Compose에서는 런타임에만 존재하고, 정적 분석으로는 닿지 않았다.

혹시 알고 계시는 불변의 법칙이 있다면 부디!
