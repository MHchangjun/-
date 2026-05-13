# 회사 Mac Pro(2019)로 122B 모델을 돌릴 수 있을까?

요즘 KTransformers + MoE 모델로 적은 VRAM으로 큰 LLM을 돌리는 사례가 여기저기서 소개되어, 혹시 Qwen3 Coder 30B나 돌리고 있던 우리 Mac Pro도? 하는 생각에 테스트해봤다. (대충 FP16은 1B당 약 2GB, 8bit는 약 1GB, Q4 계열은 0.5GB 안팎의 weight 메모리가 필요하다.)

## 기기 스펙

```
CPU             : Intel Xeon W (8코어)
GPU             : AMD Radeon Pro Vega II 32GB
RAM             : 196GB DDR4 6채널
추론 프레임워크   : llama.cpp (Vulkan 백엔드)
```

## 먼저 MoE가 뭔지

요즘 모델 스펙을 보면 122B, active 10B 같은 표현이 자주 보인다. 처음 보면 “122B 모델인데 실제로는 10B만 쓰는 거면, 10B 모델처럼 가볍게 돌릴 수 있는 거 아닌가?” 싶을 수 있다.

반은 맞고, 반은 아니다.

MoE는 매 토큰마다 전체 파라미터를 전부 계산하지 않는다. 예를 들어 Qwen3.5-122B-A10B는 256개의 expert 중 일부만 선택해서 계산한다. 공식 스펙 기준으로는 매 토큰마다 8개의 routed expert와 1개의 shared expert가 활성화된다. 그래서 계산량은 dense 122B 모델보다 훨씬 작고, active parameter 규모인 10B급에 가깝다.

하지만 메모리는 다르다.

어떤 expert가 선택될지는 토큰을 생성하는 순간에 결정되기 때문에, 전체 expert weight는 어딘가에는 로드되어 있어야 한다. 즉 active 10B만 메모리에 있으면 되는 게 아니라, 전체 122B 모델을 VRAM이든 RAM이든 들고 있어야 한다. Q4 계열 양자화 기준으로도 전체 로드 용량은 수십 GB, 보통 70GB 안팎까지 갈 수 있어서 Vega II의 VRAM 32GB에 전부 올리기는 어렵다.

그래서 KTransformers 같은 프레임워크가 하는 일은 이 구조를 이용하는 것이다. 작고 자주 쓰이는 attention이나 dense 부분은 GPU에 두고, 크지만 매번 일부만 쓰이는 expert는 CPU RAM에 둔다. GPU와 CPU가 각자 잘하는 일을 나눠서 처리하는 방식이다. 조건이 맞으면 RTX 4090 한 장으로도 대형 MoE 모델을 실사용 가능한 수준으로 돌리는 사례가 나와서 요즘 화제가 되고 있다.

다만 KTransformers는 CUDA GPU와 x86 CPU 최적화, 특히 Intel AMX 같은 구성을 중심으로 발전하고 있어서 AMD Vega II가 달린 이 Mac Pro에서는 그대로 쓰기 어렵다.

대신 llama.cpp도 MoE expert를 CPU 쪽으로 분리하는 방식이 가능해서, 이번에는 그쪽으로 직접 테스트해봤다.

## 어떻게 나눴나

llama.cpp의 `-ot ".ffn_.*_exps.=CPU"` 옵션으로 Expert weight만 CPU RAM에 두고, 나머지는 전부 GPU에 올렸다.

```
VRAM (~5.5GB) : Expert를 제외한 나머지 공통 레이어
RAM  (~66GB)  : Expert FFN 전체
```

같은 컴포넌트 분리지만 KTransformers와 결정적인 차이가 하나 있다. **KTransformers는 GPU와 CPU를 비동기로 동시에 굴리는 반면, llama.cpp는 매 레이어에서 GPU(Attention) → CPU(Expert) 전환 시 한쪽이 끝날 때까지 다른 쪽이 기다린다.** 이 동기적 대기가 뒤에서 보겠지만 성능에 직격탄을 날린다.

## 벤치마크 결과

`llama-bench`로 측정한 실측치다. (`pp` = prompt processing, `tg` = token generation)

### Qwen3.5 35B-A3B (현재 운영 중인 모델)

| 설정 | pp512 (t/s) | tg128 (t/s) |
|------|---:|---:|
| 전부 GPU | 53.83 | **54.31** |
| Expert 절반 CPU | 46.81 | 23.50 |
| Expert 전부 CPU | 41.28 | 16.22 |

VRAM에 전부 올리면 54 t/s. Expert를 CPU로 빼기 시작하면 바로 반토막이 난다.

### Qwen3.5 122B-A10B (이번에 테스트한 모델)

| 설정 | pp512 (t/s) | tg128 (t/s) |
|------|---:|---:|
| Expert 전부 CPU | 10.51 | 7.75 |
| Expert 30/48 CPU, 18/48 GPU | 12.87 | **11.38** |

VRAM 여유분에 18레이어 Expert를 GPU에 같이 올리니까 7.75 → 11.38로 47% 올랐지만, 35B 전부 GPU(54 t/s)와는 **5배 차이**다.

## 왜 이렇게 느릴까

### 1. GPU가 잘하는 일을 CPU가 하고 있다

Expert가 하는 연산은 행렬곱이다. GPU가 가장 잘하는 작업인데, VRAM에 안 들어가니까 CPU가 대신하고 있다. 게다가 이 Xeon W에는 KTransformers가 적극 활용하는 **Intel AMX(행렬곱 가속기)도 없어서** 범용 연산으로 처리된다.

### 2. 매 레이어마다 GPU↔CPU 핑퐁

48레이어에서 레이어마다 Attention(GPU) → Expert(CPU) 전환이 발생하는데, llama.cpp는 이걸 동기적으로 처리한다. 실측 graph splits **146회.** 매 split마다 한쪽이 놀면서 대기한다. KTransformers가 비동기 파이프라인으로 같은 분리 구조에서 훨씬 빠른 이유가 여기 있다.

### 3. DDR4 메모리 대역폭

토큰 1개 생성 시 CPU가 RAM에서 읽어야 하는 Expert weight는 대략 수 GB다. DDR4 6채널 이론 대역폭 ~120GB/s로는 산술적으로 ~34 t/s가 나와야 하지만, 위의 동기 실행 오버헤드와 PCIe 전송 지연이 합쳐져 실측은 이론치의 1/3 수준에 머무른다.

## 결론

**돌릴 수 있나? — 그렇다, 돌아간다.**

**실용적인가? — 아니다.**

tg 11 t/s면 분당 ~600토큰, 코드로 환산하면 대략 20줄 정도다. tool call을 포함한 멀티턴 Agent 워크로드로 쓰기에는 너무 느리다. 한 번 작업 굴리고 결과 보는 데만 몇 분씩 걸리니, 디버깅 사이클 자체가 무너진다.

그래서 이 Mac Pro에서는 122B-A10B를 억지로 돌리는 것보다, 35B-A3B를 VRAM에 최대한 올려 50 t/s 이상으로 굴리는 쪽이 agent 워크로드에는 훨씬 낫다. SWE 벤치마크 상 큰 차이 없고 모델 지능보다 중요한 게 디버깅 사이클과 tool call 왕복 속도였기 때문이다.
