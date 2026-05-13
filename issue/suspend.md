# suspend와 LSP 계약에 대한 고찰

## 개요
```kotlin
// 문제의 코드
override suspend fun populateFtsData() {
    withContext(ioDispatcher) {
        newsResourceFtsDao.insertAll(...)
        topicFtsDao.insertAll(...)
    }
}
```

코드 리뷰 중 Repository 구현체에서 Dispatcher 변경하는 코드에서 미묘한 Code Smell을 느꼈다.

DAO의 insertAll 함수는 이미 suspend로 선언되어 있는데, 왜 호출하는 쪽에서 ioDispatcher를 지정했을까?

이 책임은 과연 호출하는 쪽(Repository)이 져야 할까?   
아니면 실제 작업을 수행하는 suspend 함수가 내부적으로 처리해야 할까?

이 질문에 대한 해답을 찾아가는 과정에서 suspend 키워드의 본질적인 역할과 LSP(리스코프 치환 원칙)까지 되짚어보게 되었다.

## Coroutines의 본질

코루틴의 가장 큰 장점은 **가벼운 동시성**이다.   
수만 개의 동시 작업을 실행해도 스레드를 사용할 때처럼 시스템에 큰 부담을 주지 않는다.
```kotlin
fun main() = runBlocking {
    repeat(50_000) { // launch a lot of coroutines
        launch {
            delay(5000L)
            print(".")
        }
    }
}
```

이것이 어떻게 가능할까?

비밀은 suspend 함수에 있으며, 공식 문서는 둘의 관계를 이렇게 설명하고 있다.

> Coroutines in Kotlin are built on suspending functions, which allow code to pause and resume without blocking a thread.

즉, suspend 함수는 스레드를 차단하는 대신 중단하고 양보함으로써 코루틴의 가벼운 동시성을 가능하게 한다.   
만약 개발자가 suspend 함수 내부에 스레드를 차단하는 코드를 작성한다면, 이는 코루틴의 가장 큰 장점을 스스로 포기하는 행위와 같다.

따라서 suspend 함수는 절대 호출한 Coroutines의 스레드를 차단하지 않도록 구현해야 한다.

## Non-Blocking 책임
사실 `이 책임을 호출자가 져야 할까, suspend 함수 내부가 져야 할까?`라는 고민 자체가 부질없다.

suspend 함수는 `호출 스레드를 차단하지 않는다(non-blocking)`는 계약을 문서로 명확히 약속하고 있는데,   
이를 사용하는 쪽에서 매번 withContext(ioDispatcher) 같은 dispatcher 스위칭을 해준다는 건 애초에 suspend 키워드의 취지를 무너뜨리는 불필요한 중복일 뿐이다.

## suspend 키워드 속 LSP

하지만 suspend 키워드가 함수를 자동으로 non-blocking으로 만들어주는 마법이 아니다.   
suspend 함수가 non-blocking를 보장하는 건 온전히 개발자의 책임이다.

```kotlin
@Dao
interface NewsResourceFtsDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(newsResources: List<NewsResourceFtsEntity>)
}
```

그래서, 이러한 interface를 구현할 때 non-blocking 책임은 LSP 관점에서 반드시 보장해야 할 **부수효과(side-effect) 계약** 중 하나로 볼 수 있다.

### 이 계약이 중요한 이유
만약 NewsResourceFtsDao interface의 여러 구현체 중 단 하나라도 이 Side-Effect 계약을 어기고 호출자의 스레드를 차단한다면 어떻게 될까?

그 순간 LSP가 깨지고 interface, 더 나아가 라이브러리 전체에 대한 신뢰가 붕괴된다.

개발자는 더 이상 NewsResourceFtsDao interface만 보고 안심하고 코드를 짤 수 없다.   
대신 `혹시 이 구현체는 스레드를 막지 않을까?` 의심하며 모든 구현체의 내부를 들여다봐야 한다.

결국 확신이 없기 때문에 모든 호출부를 withContext로 감싸는 방어적인 코드를 작성하게 되고 interface를 통해 구현을 감추려던 원래의 목적은 사라지고, 코드는 불필요하게 복잡해지며 신뢰할 수 없는 추상화만 남게 될 것이다.

## 결론
suspend 함수의 스레드 관리 책임은 호출자가 아닌, 함수 스스로가 져야 한다.

이렇게 책임을 명확히 하면, 호출자는 어떤 구현체가 오더라도 안심하고 함수를 호출할 수 있다.   
이는 결국 interface에 대한 신뢰를 높이고, 더 유연하고 확장성 있는 코드를 만들어줄 것이다.
