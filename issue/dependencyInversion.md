# interface + Hilt(DI)를 이용해 서로 다른 모듈 간 강한 dependency 풀기 (dependency Inversion), 2021

## 문제
왓챠 앱의 모듈화 작업을 처음 시작하던 2022년, 커다란 벽에 부딪혔다.

새로운 피처(feature) 모듈을 만들고 있었는데, 이 모듈에서 앱 모듈에 있는 특정 클래스를 참조해야만 했다. 하지만 그 클래스는 앱 모듈의 다른 여러 클래스와 너무 복잡하게 얽혀있어, 단순히 가져오려고 하면 거대한 의존성 뭉치가 함께 딸려와 점진적인 모듈화가 불가능해 보였고, 프로젝트는 진퇴양난에 빠졌다.

## RecyclerView.Adapter에서 얻은 영감
해결의 실마리는 의외의 곳에서 찾아왔다. 바로 흔히 사용하던 RecyclerView.Adapter

어댑터의 생성자를 살펴보니, 클릭 이벤트를 처리하는 구체적인 구현체를 직접 받지 않고 onClickListener라는 **Interface**를 받는 것을 보았다💡

이 패턴은 단순하지만 강력했다.

하위 모듈에서 '약속'을 정의한다. 이 모듈은 무엇이 필요한지는 알지만, 어떻게 동작할지는 알 필요가 없다.   
상위 모듈에서 그 '약속'을 '구현'한다. 구체적인 로직은 상위 모듈이 제공한다.

이 방식을 사용하면, 하위 모듈은 더 이상 상위 모듈을 직접적으로 알 필요 없다.   
오직 자신이 정의한 추상적인 인터페이스에만 의존하게 된다.

## 난관: ViewModel interface A 주입하기

피처 모듈에 A interface 정의하고, 앱 모듈에서 그 구현체를 만들었다.   
이제 이 구현체를 피처 모듈에 넘겨주기만 하면 될 것 같았다. 하지만 곧바로 새로운 문제에 부딪혔다.

```kotlin
@HiltViewModel
class FeatureViewModel @Inject constructor(
  private val repository: Repository
  /* 여기에 A를 넣고 싶지만… */
) : ViewModel()
```

FeatureViewModel은 이미 @Inject 생성자 주입을 통해 repository 같은 의존성을 잘 받고 있었는데, 여기에 A를 추가하려고 하자 문제가 생겼다.

생성자에 A를 넣으면 피처 모듈 입장에선 AImpl을 알 수 없어 주입할 방법이 없고, 그렇다고 lateinit이나 setter로 주입을 하면 ViewModel의 불변성이 깨지고 Hilt가 제공하던 편리한 DI 기능을 포기해야 했다.

생성자 주입은 유지하면서 앱 모듈의 구현체를 안전히 연결할 방법이 필요했다.

## 마지막 퍼즐 조각, @Binds

생성자 주입을 포기해야 하나 고민하던 중, 문득 코드 리뷰에서 중 관성적으로 사용하던 @Binds 어노테이션이 봤다.

```kotlin
@Binds
abstract fun bindRepository(
    repositoryImpl: RepositoryImpl,
): Repository
```

보통은 같은 모듈 안에서 인터페이스와 구현체를 묶어주는데 사용했지만, 만약... 앱 모듈에서 @Binds를 정의하면, Hilt가 이걸 알아보고 :feature 모듈에 주입해줄 수 있지 않을까?

떨리는 마음으로 앱 모듈에 Hilt 모듈을 만들고, 피처 모듈의 인터페이스 A와 앱 모듈의 구현체 AImpl을 묶는 코드를 작성하고 빌드에 성공했다.

## 결론

* **의존성 역전 원칙**을 몸소 체감한 순간이었다.
* 구체적인 구현이 아닌 추상적인 인터페이스에 의존함으로써, 모듈 간의 결합을 성공적으로 끊어냈고 진정한 의미의 점진적 모듈화를 가능하게 했다.
* Hilt는 모듈별로 따로 동작하는 게 아니었다. (hilt 동작에 대해 잘 모르고 어쩌다 잘 찍어 맞췄다.)

## 여담
* 세기에 발명인 줄 알았으나 2019년 [square 모듈화](https://www.droidcon.com/2019/11/15/android-at-scale-square/) 발표에서 이미 이 내용에 대해 설명이 있었다.
