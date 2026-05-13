# Deep dive

## 개요 : 올바른 화질을 제공하기 위한 발버둥

이전 분석을 통해 단순한 클라이언트 이슈가 아님을 확인했다.   
하지만 Media 3 이슈가 아니라면 클라 개발자가 무엇을 할 수 있을까?

고민과 무기력함을 새로운 가설들로 해결해 보고자 한다.

"낮은 트랙에서 720x1280로 변경할 수 있는 이유가 뭘까?"   
"정말 Media 3 이슈가 아닐까?"

**"MediaCodec를 직접 Init해 볼까?"**

## MediaExtractor

Media3를 거치지 않고 MediaCodec에 비디오 데이터를 직접 전달하기 위해 MediaExtractor를 사용했다.

```kotlin
val mediaExtractor = MediaExtractor().apply {
   val afd = assets.openFd("video.mp4")
   setDataSource(afd.fileDescriptor, afd.startOffset, afd.length)

   selectTrack(0) // mp4에 video track 밖에 없어 0으로 고정
}
```

## DRM
MediaDrm API를 사용해 Widevine 라이선스를 요청하고, 이를 바탕으로 MediaCrypto 객체를 생성하여 Secure 디코딩 환경을 재현했다.

```kotlin
val drm = MediaDrm(C.WIDEVINE_UUID)
val sessionId = initializeDrmSession(drm, mediaExtractor.drmInitData)
val crypto = MediaCrypto(C.WIDEVINE_UUID, sessionId)
```

> ClearKey DRM을 시도했지만, 문제가 발생하는 A32 기기에서는 보안 디코더가 ClearKey를 지원하지 않아 Widevine으로 테스트를 진행

## MediaCodec

MediaExtractor에서 얻은 MediaFormat과 MediaCrypto로 secure decoder 직접 초기화 했다.   

```kotlin
val codec = MediaCodec.createByCodecName("c2.mtk.avc.decoder.secure")

codec.setCallback(object : MediaCodec.Callback() {
   ...
}

codec.configure(format, surface, crypto, 0)
codec.start()
```

## 가설 검증 : Media3 framework 문제인가?

예상 했듯 Media 3를 사용할 때와 동일한 0x80000000/UNKNOWN_ERROR가 발생했다.   

역시 Media 3 이슈가 아니었다.   
문제는 안드로이드 미디어 framework나 칩셋에 있는 거 같다.

## 에러 발생 지점 추적
Stack Trace는 onInputBufferAvailable에서 MediaCodec.getInputBuffer를 호출할 때 에러가 시작된 것처럼 보였지만, 이를 주석처리해도 codec 시작 후 codec를 참조하는 곳에서 에러가 발생한다.

이는 MediaCodec에 데이터를 넣기 전, 즉 codec.start() 직후 문제가 발생하고 있었다.   
MediaCodec.configure는 throwable인데?

## 원인 발견: csd-0 (SPS) 데이터

configure 단계에서 문제를 일으키는 원인을 찾기 위해 MediaFormat의 파라미터를 하나씩 변경하며 테스트했다.

* width, height 수정 -> 실패💥
* display-width, display-height 수정 -> 실패💥
* csd-0 버퍼 제거 -> 성공✅

configure 시점에 csd-0 데이터에 포함된 SPS를 파싱하여 너비, 높이 등 영상의 속성을 설정한다.   
문제가 된 720x1280 해상도의 SPS 데이터가 특정 secure decoder에서 초기화 오류를 일으키는 것이었다.

이렇게 해상도 설정 이슈라는 가설도 증명되었다.   
근데 해결책은??

## 다음 가설 : Codec 재설정

그렇다면 왜 낮은 해상도(540x960)로 재생을 시작한 후 720x1280으로 전환하면 문제가 없었을까?   

Media 3에서도 input format이 변경될 때 codec reconfigure하는 것을 재현해 봤다.

```kotlin
format.setByteBuffer("csd-0", csd_0_540_960)

codec.configure(format, surface, crypto, 0)
codec.start()

format.setByteBuffer("csd-0", csd_0_720_1280)
format.setInteger("width", 720)
format.setInteger("height", 1280)

codec.stop()
codec.configure(format, surface, crypto, 0)
codec.start()
```

720x1280 csd-0으로 변경 후 여전히 에러가 난다.

이렇게 reconfigure하는 거 아닌가?   
media 3는 어떻게 하지?

## Adaptive Reconfiguration

Media3 코드를 따라가 보니 그 해답은 Adaptive Reconfiguration에 있었습니다.   
이는 코덱을 완전히 멈추고 재시작하는 대신, 재생 중에 스트림에 새로운 설정 정보를 흘려보내 포맷을 바꾼다.

이 동작을 직접 구현해 봤다.
1. 540x960 format으로 Codec configure -> start
2. codec input buffer에 sampleData queue하기 전에, 720x1280 mp4의 csd-0, csd-1를 buffer에 넣고
3. 이 버퍼에 MediaCodec.BUFFER_FLAG_CODEC_CONFIG flag를 달아, Codec Reconfiguration 데이터임을 알린다.
```kotlin
// 처음 onInputBufferAvailable가 불릴 때
buf.put(secondCsd0)
buf.put(secondCsd1)

val pts = System.nanoTime() / 1_000
c.queueSecureInputBuffer(
    index, 0,
    cryptoInfo,
    pts,
    MediaCodec.BUFFER_FLAG_CODEC_CONFIG
)
```

결과는 완벽했다.   
첫 frame render 전에 성공적으로 720x1280 해상도로 전환되었고, 영상은 끊김 없이 재생되었다!

## 결론
* 문제의 본질 : 특정 decoder(c2.mtk.avc.decoder.secure)은 feature-can-swap-width-height 플래그를 지원함에도 불구하고, configure 단계에서 width/height가 뒤바뀐 해상도의 SPS 데이터로 초기화하는 것을 허용하지 않는다.
* 해결책 : 안전한 해상도로 Codec을 먼저 초기화한 후, BUFFER_FLAG_CODEC_CONFIG flag를 이용해 Adaptive Reconfiguration 방식으로 목표 해상도로 전환하면 문제를 우회할 수 있다.
* 이 flag를 보고 재생 가능 range를 늘려 버리는 Android Media Framework나 제대로된 에러를 뱉지도 않는 MediaTek이나 문제가 있는 거 같다..
