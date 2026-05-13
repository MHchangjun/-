# Intell Mac Pro에서 Local LLM 서빙하기 — Ollama에서 llama.cpp + Vulkan으로

<img width="540" height="663" alt="image" src="https://github.com/user-attachments/assets/6345d8bd-4e7b-4c6b-9153-6331574f9f97" />

## 시작하게 된 배경
24시간 돌아가는 Custom Coding Agent를 개발하면서 가장 힘든 건 디버깅이었다. Ollama로 Local LLM을 맥북에 띄우면 램이 부족해서 IDE 하나 띄우기도 버거웠다.

마침 사내에 Android팀 GitHub Action을 돌리던 2019년 Mac Pro가 있었다. Apple Silicon은 아니지만 RAM 196GB, VRAM 32GB(AMD Radeon Pro Vega II) 사양이라 Local LLM을 돌리기에는 충분해 보였다.

## Ollama
맥북에서 쓰던 그대로, Mac Pro에도 Ollama를 올려서 서빙을 시작했다. 잘 돌아가는 것 처럼 보였다.

## 문제 발견: 왜 이렇게 느리지?
쓰다 보니 체감 속도가 이상했다. VRAM 32GB에 196GB RAM이 달린 워크스테이션이 M1 MacBook보다 느렸다. 분명 뭔가 잘못된 것이었다.
확인해 보니 Ollama가 GPU를 전혀 사용하지 않고 있었다. 원인을 추적해 보니 명확했다.

Intel Mac + AMD Radeon Pro Vega II 조합에서는 Ollama가 GPU를 잡지 못한다.

Ollama는 macOS에서 Apple Silicon의 Metal을 우선 가정하고 동작하는데, Intel Mac에 박힌 discrete AMD GPU는 사실상 사각지대다. 결국 196GB RAM이 있어도 모든 연산을 CPU(Xeon)에서 돌리고 있었던 것이다.

## 2차 시도: llama.cpp + Vulkan
해결 방법을 찾던 중 같은 환경(인텔 맥 + AMD GPU)에서 백엔드를 Vulkan으로 두고 llama.cpp를 직접 빌드하면 GPU가 잡힌다는 Reddit 글을 발견했고, 바로 시도했다.

여담으로 Ollama는 사실 llama.cpp의 wrapper다. 내부적으로 같은 추론 엔진을 사용하기 때문에, llama.cpp를 직접 빌드해서 백엔드만 Vulkan으로 바꾸면 동일한 모델을 GPU로 돌릴 수 있다는 것이 핵심이었다.

### 빌드 과정
먼저 Mac Pro에 Vulkan SDK를 설치한 뒤, llama.cpp를 직접 clone해서 Vulkan 백엔드로 빌드했다.
```bash
git pull
rm -rf build
cmake -B build -DGGML_VULKAN=ON
cmake --build build --config Release -j$(nproc)
```

빌드가 끝나고 실행해 보니 device가 정상적으로 잡혔다.
```
ggml_vulkan: Found 1 Vulkan devices:
ggml_vulkan: 0 = AMD Radeon Pro Vega II (MoltenVK)
```

### 서빙
이제 -ngl 옵션으로 GPU에 올릴 레이어 개수를 지정해서 서빙할 수 있게 되었다.

```bash
./build/bin/llama-server \
  -m ~/models/Qwen3.5-35B-A3B-UD-Q4_K_XL.gguf \
  --alias "qwen3.5" \
  --jinja \
  --device Vulkan0 \
  -ngl 99 \
  -fa on \
  --host 0.0.0.0 --port 8080
```

기존에 Ollama 클라이언트를 쓰던 코드는 OpenAI 클라이언트로 바꿔야 했지만 사용하던 Agent Framework에 간단하게 migration이 가능했다.

## llama.cpp로 갈아타고 얻은 것들

Ollama에서 자유로워지면서 얻은 이점이 생각보다 많았다.
* 백엔드 빌드 타임 제어 (-DGGML_VULKAN=ON 같은 것) — Ollama는 사용자가 백엔드를 강제할 수단이 사실상 없음. 이번 케이스의 핵심.
* 추론 파라미터 세밀 제어 — -ot ".ffn_.*_exps.=CPU", -ctk q8_0, -fa on 같은 옵션. Ollama Modelfile의 PARAMETER로 노출되는 건 일부에 불과함.
* 투명한 로그 — 디바이스/메모리/백엔드 상태가 시작 로그에 그대로 찍힘. Ollama는 추상화 너머라 디버깅이 어려움. (이번에 GPU 못 잡고 있던 것도 Ollama로는 한참 후에 알아챘던 점)
* llama.cpp 본체 패치 즉시 반영 — git pull로 최신 패치 바로 받기. Ollama는 wrapper 특성상 시차가 있음.

## 마무리
처음에는 "그냥 Ollama 깔고 모델 올리면 되겠지" 싶었는데, 결국 인텔 맥 + AMD GPU라는 조금 까다로운 조합 덕분에 한 단계 아래로 내려가게 되었다.   
그 과정에서 Ollama가 어떤 추상화를 제공하고 있었는지, 그 추상화가 깨졌을 때 무엇을 잃는지를 명확히 알게 된 것이 가장 큰 수확이었다.

지금은 Mac Pro에 llama.cpp + Vulkan으로 Qwen 계열 모델을 안정적으로 서빙하고 있고, smoker Agent가 매일 밤 이 서버를 두드리며 리팩토링을 처리하고 있다.
