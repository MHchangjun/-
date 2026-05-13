# 숏챠 재생 이슈 분석 (MediaCodec의 배신)

## 문제 발생 : 0x80000000/UNKNOWN_ERROR
특정 기기에서 플레이어에서 DRM 걸린 콘텐츠 재생 시 ExoPlaybackException : MediaCodecVideoRenderer error 발생했다.

Logcat에는 Codec이 시작된 후 원인을 알 수 없는 에러(0x80000000)를 보고했다는 로그만 덩그러니 남아 있었다.
```
MediaCodec | Codec reported err 0x80000000/UNKNOWN_ERROR, actionCode 0, while in state 6/STARTED
```

이상한 점은 한 둘이 아니었다.
* DRM 콘텐츠에서만 발생
* 대부분 중저가 휴대폰에서 주로 보고된다.
* 로그상 재생 시도한 track는 Supported: true
* 하지만 **낮은 해상도로 재생을 시작한 후 에러 발생 트랙으로 전환하면 정상 재생된다**
* 문제가 되는 트랙은 720x1280 해상도, 이 보다 낮은 540x960 해상도 트랙은 재생이 잘 된다.

## 인코딩 이슈?

각 트랙간 차이는 Scale 밖에 없다.   
하지만 **특정 Scale에서 디코딩 못하는 옵션이 있는게 아닐까?**

인프라팀에 매번 새로운 인코딩을 요청하는 것은 비효율적이었다.   
그래서 빠르고 독립적인 검증을 위해, 직접 인코딩부터 DRM 패키징까지 수행하는 로컬 테스트 환경을 구축하기로 했다.

### ffmpeg
> 다양한 비디오·오디오 포맷과 코덱을 커맨드라인 하나로 다룰 수 있는 오픈소스 멀티미디어 툴킷

문제가 되는 720x1280 트랙 Format에 맞춰 테스트할 mp4를 인코딩했다.

### Shaka Packager
인코딩한 콘텐츠를 Widevine DRM 적용을 위해 Shaka Packager를 사용했다.

[Test Key Server](https://shaka-project.github.io/shaka-packager/html/tutorials/widevine.html#widevine-test-credential)로 암호화

### 인코딩 옵션 테스트
기존 재생 환경에서 MediaItem만 변경해 재생 테스트 했다.   
720x1280 트랙 Format 그대로 인코딩한 영상은 역시 MediaCodecError가 발생했다.

그래서 기존 MediaFormat를 보고 의심되는 값들을 변경해 봤지만 같은 에러가 발생했다.
* 프레임
* H.264 level
* pix_fmt
* colorspace, color_trc, color_primaries

## Scale
그럼 decoder에서 특정 scale 이상을 받을 수 없는 걸까?

기기 마다 scale range가 다르겠지만 720x1280이 decoding 불가했다면 track supported가 falase여야 한다.   
만약 true라도 MediaCodec.configure 단계에서 에러가 발생했어야 했을 텐데...

특이한 점은 첫 재생 트랙으로 540x960을 선택하고 다음 segment로 720x1280 트랙을 재생할 땐 문제 없이 재생되었다.

그럼 **첫 재생에서 특정 scale 이상 재생하지 못 하는 건가?**

### 경계 테스트
그럼 이 기기에서 재생 가능한 first segment의 max scale는 어디까지 일까?

* 가로 세로를 swap한 1280x720 👌
* 극단적인 10000x10000 ✋
* 흔한 1920 x 1080 👌
* 100x1000, 100x1080 👌
* 100x1090 ✋
* 100x1088 👌

> Note: Scale는 항상 짝수

1920 x 1080가 재생이 되는데, 그보다 훨씬 작은 100x1090는 재생이 안된다고?   
단순 픽셀 수 문제가 아니구나

그럼 두 번째 Segment에선 왜 720x1280가 재생이 되는 거지?   
Decoding할 수 있는 Scale이 회전하기라도 하나 ㅋㅋ

## 바늘 찾기
클라 문제도 인코딩 문제도 아니면 왜 이런 이슈가 발생할까?

더 이상 확인할 가설이 떠오르지 않아 에러 StackTrace에 찍히는 MediaCodecRenderer 전체에 breakpoint를 찍어 가며 디버깅 했다.   
720x1280 트랙 재생을 시도할 때 first frame도 render하지 못한 거 보면 Codec 생성 후 혹은 Codec에 데이터를 넣을 때 에러가 발생하고 있는 거 같아 init 위주로 확인했다.

### MediaCodecRenderer.initCodec
MediaCodecRenderer.initCodec 메서드에서 MediaCodecInfo를 확인할 수 있다.   

```
mediaCodec.capabilities.mCapabilitiesinfo.mMap
    * performance-point-1920x1088-range" -> "30-30
    * "size-range" -> "64x64-2048x1088"
    * "feature-adaptive-playback" -> {Integer@48944} 0
    * "max-concurrent-instances" -> "1"
    * "feature-can-swap-width-height" -> {Integer@48948} 1
    * "feature-secure-playback" -> {Integer@48950} 1
```

codec의 정확한 size range와 feature-can-swap-width-height라는 의미 심장한 flag를 확인할 수 있다!

### MediaCodecInfo
feature-can-swap-width-height를 참조하는 곳은 한 곳 밖에 없다.

MediaCodecInfo.VideoCapabilities.parseFromInfo에서 feature-can-swap-width-height가 true일 때 mWidthRange와 mHeightRange를 extend한다.   
Media 3는 단지 mWidthRange와, mHeightRange을 보고 트랙 supported 여부를 판단하고 있다.

그래서 MediaCodec size range가 64x64-2048x1088이더라도 720x1280 트랙이 재생 가능하다고 판단하고 있던 것이다.

### 직접 Swap

그럼 첫 재생 트랙으로 swap이 필요한 Scale을 선택해도, **MediaCodec configure 전에 Codec width, height를 swap해 버리면 되지 않을까?**

하지만 Media 3, Android native Media framework 어디에도 Codec를 직접 swap하는 메서드는 없었다.

## 중간 해결책
TrackSelector에서 첫 재생 트랙을 선택할 때 feature-can-swap-width-height를 고려하지 않고 선택하면 베스트   
하지만 저 flag도 Codec의 size range도 직접 참조할 수 없다.

그래서 MediaCodec Error가 발생하면 720x1280 트랙을 더 이상 선택하지 못하게 처리했다.

```kotlin
if (error.errorCode == PlaybackException.ERROR_CODE_DECODING_FAILED && error.cause?.cause?.message?.contains("0x80000000") == true)
```

```kotlin
trackSelectionParameters = trackSelectionParameters
     .buildUpon()
     .setMaxVideoSize(719, 1279)
     .build()
```

## Media 3 문제?
Media 3에서 MediaCodec width, height를 명시적으로 바꾸는 함수는 없다.   
또 문제의 feature-can-swap-width-height를 직접 참조하는 곳도 없다.   
단지 MediaCodecInfo에서 Supported 여부만 받아 온다.

그래도 정확한 원인과 다른 해결 방법을 알고 싶어 최근에 [이슈](https://github.com/androidx/media/issues/2281) 등록해 뒀다.

## 결론
* 첫 재생 트랙에서 codec width, height swap이 필요한 트랙 선택 시 decoding 과정에서 에러가 난다.
* codec 에러나면 maxVideoSize 조절하는 식으로 해결해 둔 상태

## 다음 스토리
[Deep dive](https://github.com/MHchangjun/-/blob/master/issue/player-codec-issue2.md)
