# 이상적인 숏폼 플레이어

## 개요

숏폼 콘텐츠은 빠르고 부드러운 재생 경험이 중요하다.   
특히 사용자가 스크롤할 때 다음 영상의 첫 프레임이 지연 없이 나타나는 속도는 사용성을 좌우하는 요소다.

<img width="189" height="409" alt="image" src="https://github.com/user-attachments/assets/0c49de27-bdab-4c11-a884-e579bccb0edb" />

최근 숏챠 플레이어를 개발하면서 Player 관리 전략과 이유에 대해 공유하려고 한다.

> Note: Dash 기준으로 설명
> Single Player, Player Pool에 대한 설명은 생략

## 들어가기 전에 : Player.prepare의 역활
Player.prepare에서는 생각 보다 많은 일을 한다.

이 함수의 목표는 play가 호출되었을 때, 지체 없이 즉시 재생을 시작할 수 있는 STATE_READY 상태로 만드는 것이다.   
이를 위해 SampleQueue에 최소한의 데이터가 채워져야 한다.

prepare가 수행하는 주요 작업은 다음과 같다.
* TimeLine 갱신
* Dash Manifest 다운로드 및 파싱 
* 트랙 선택
* 선택한 트랙 기반으로 codec init
* 초기 버퍼링

## 첫 프레임 렌더링 속도 비교: 누가 더 빠른가?

### 전제 조건
* offscreenPageLimit 1 이상 — Player Pool 방식은 현재 페이지 외에 다음 페이지의 Player도 미리 준비되어 있어야 의미가 있으므로
* DefaultPreloadManager 제외 — 뒤에서 별도로 다룸

두 방식의 가장 큰 차이는 언제 prepare()를 호출하느냐에 있으며, 이는 첫 프레임 렌더링 속도에 결정적인 영향을 미친다.

### Player Pool
페이지가 생성되는 시점에 즉시 prepare()를 호출할 수 있다.

사용자가 다음 페이지로 스크롤하면, 이미 **STATE_READY 상태로 대기 중인 Player**가 즉시 재생을 시작한다.   
버퍼(SampleQueue)에 채워진 데이터를 바로 MediaCodec으로 보내 디코딩하고 첫 프레임을 렌더링한다.

이처럼 모든 준비가 끝나있기 때문에 지연 시간이 거의 없는 이상적인 사용자 경험을 제공할 수 있다.

### Single Player
사용자가 페이지를 넘기면, 그때서야 setMediaItem()으로 새 영상을 설정하고 prepare()를 호출한다.

이 방식은 위에서 설명한 prepare()의 모든 과정을 거쳐야만 첫 프레임을 렌더링할 수 있다.   
네트워크 상태가 좋지 않다면 이 지연은 더욱 길어져 사용자 경험을 해칠 수 있다.

## 리소스 소모

Player Pool과 Single Player의 리소스 소모는 어떨까?

Player Pool에 n개의 Player가 있고 이 안에는 여러 SampleQueue, Allocator, TrackGroupArray, 각종 Handler, Listener와 n개의 video, audio codec이 init되어 많은 리소스를 사용할 것이다.

반면 Single Player는 가볍다.

다만 "가볍다"가 항상 좋은 것은 아니다. 리소스 소모는 단순히 적게 쓰는 쪽이 우월하다는 의미보다는, 어디에 비용을 지불할 것인가의 문제에 가깝다.

Player Pool은 메모리와 코덱 인스턴스를 미리 점유하는 대신 페이지 전환 시 prepare 비용을 지불하지 않고, Single Player는 인스턴스 비용을 아끼는 대신 전환 시점마다 prepare 비용을 지불한다. 결국 상시 비용 vs 전환 비용의 트레이드오프다.

또한 리소스 소모는 절대량이 아니라 기기의 여유분 기준으로 봐야 한다. 고사양 기기에서 Player 여러 개를 띄우는 비용과, 저사양 기기에서 같은 수를 띄우는 비용은 체감이 다르다. 동일 코드라도 기기에 따라 Pool은 사치가 될 수도, 합리적인 선택이 될 수도 있다.

## Single Player 최적화

### Playlist
[media 3 1.5.0](https://android-developers.googleblog.com/2025/01/media3-150-whats-new.html)에서 다음 항목을 미리 불러오는 Preload 기능이 추가되었다.   
기본적으로 Playlist Preload는 기본적으로 비활성화되어 있으며, 미리 로드할 지속 시간을 설정하여 활성화할 수 있다.

```kotlin
player.preloadConfiguration =
    PreloadConfiguration(/* targetPreloadDurationUs= */ 5_000_000L)
```

Preload는 현재 플레이백에 필요한 로딩 작업이 모두 완료된 시점에만 시작된다.

PlayList를 구성하면 codec 관점에서도 이점이 있다.

매번 setMediaItem 또는 setMediaSource를 호출 후 prepare하면 기존 codec를 release하고 다시 init 한다.   
반면 playlist에서 window를 변경할 때 특정 조건만 충족한다면 기존 codec를 그대로 사용할 수 있다.

판단은 MediaCodecInfo.canReuseCodec 함수에서 format를 비교한다.   
비디오 같은 경우 MimeType, width, height, colorInfo 등을 비교하고 변경이 없으면 SPS/PPS 변경 여부를 보고 REUSE_RESULT_YES_WITHOUT_RECONFIGURATION, REUSE_RESULT_YES_WITH_RECONFIGURATION를 구분한다.

숏챠 같은 경우 시즌 단위로 재생하고 미디어팀에선 시즌 에피소드들을 한 번에 인코딩하기 때문에 트랙에 따라 SPS의 width, height는 다를 수 있지만 같은 해상도의 트랙을 선택한다면 codec reconfiguration 없이 codec를 재사용할 수 있다.

반면 Player Pool 방식에서는 페이지 전환 시 마다 매번 MediaCodec을 release하고 다시 init하기 때문에, 추가적인 오버헤드가 발생한다.

따라서 숏드라마와 같이 순차 재생이 예상되는 콘텐츠에서는, 다음 화 Preload와 코덱 재사용을 통해 Player Pool과 유사한 유저 경험을 구현할 수 있다.

### DefaultPreloadManager
반면 유트브 숏츠나 인스타 릴스 같은 숏폼은 사용자가 다음 영상 Preload 전에 영상을 넘겨버리기 때문에, Preload할 틈이 거의 없어 실효성이 떨어질 수 있다.

대신, 우선순위 기반으로 여러 미디어소스를 동시에 준비 가능한 DefaultPreloadManager를 사용하면 된다.

페이지가 바뀔 때 마다 PreloadManager에서 MediaSource를 가져와 setMediaSource, prepare를 호출해야 하므로, Playlist와 함께 쓰기에는 다소 애매한 부분이 있다.

물론 PreloadManager와 Player가 같은 Cache을 공유한 다면 PreloadMediaSource 사용하지 않고 같은 Url에 PlayList을 사용할 수 있지만,   
이미 PreloadManager에서 가지고 있는 MediaSource를 또 만드는 꼴이라 리소스 낭비다.

## 결론
물론 Player Pool과 DefaultPreloadManager를 같이 사용하는게 가장 빠르겠지만 상황에따라 Single Player에서 얼마든지 최적화 가능하다.
