# git clone 받았을 때

```
uv sync
```
오래걸림 

```
uv run jupyter lab
```
주피터 노트북에서 

venv 선택하기 
![alt text](image.png)

(지금 다른 가상환경을 잡혔을때)


```
cd /mnt/c/worksapces/gg1th_qlora_ft_ex
uv run python -m ipykernel install --user --name=gg1th-qlora
```
후 주피터 노트북 재시작 후 가상환경 선택하기

### ollama

```
systemctl status ollama
```


### 허깅페이스 다운로드 했던 모델 리스트 확인 

```
ls ~/.cache/huggingface/hub
```


# lora_finetuning

## GPU(CUDA) 환경 셋팅 (처음부터)

### 0. 전제 조건

- Windows(Host) + NVIDIA GPU (예: RTX 4060), WSL2 Ubuntu 환경
- NVIDIA 드라이버 설치 완료
- [uv](https://docs.astral.sh/uv/) 설치 완료

드라이버가 지원하는 최대 CUDA 버전을 먼저 확인한다.

```powershell
nvidia-smi
```

출력의 `CUDA Version: 13.1` 같은 값이 **드라이버가 지원하는 상한선**이다. 이후 설치하는 torch의 CUDA
빌드 버전(`cu121`, `cu126`, `cu128` 등)은 이 값 이하이면 된다. Blackwell(RTX 50 시리즈) GPU는 최소
`cu128` 이상 빌드가 필요하다.

> **CUDA Toolkit을 시스템에 별도로 설치할 필요는 없다.** pip/uv로 설치하는 `torch`의 CUDA 빌드(예:
> `+cu128`)는 cuBLAS, cuDNN, cuFFT, NCCL 같은 CUDA 런타임 라이브러리를 wheel 안에 이미 포함하고 있다.
> 시스템에 필요한 건 위에서 확인한 **NVIDIA 드라이버뿐**이며, 드라이버가 제공하는 CUDA 드라이버 API
> (`nvcuda.dll`)만 있으면 GPU 연산이 된다.
>
> 예외적으로 CUDA Toolkit(nvcc 컴파일러 포함)이 필요한 경우는 `bitsandbytes`, `triton`, `xformers` 같은
> 패키지를 **소스에서 직접 빌드**해야 할 때뿐이다. 이 프로젝트는 모두 미리 빌드된 wheel을 사용하므로
> 해당하지 않는다.

### 1. 프로젝트 초기화

```powershell
uv init
uv venv
```

### 2. `pyproject.toml`에 CUDA 인덱스 설정

torch/torchvision/torchaudio를 기본 PyPI가 아닌 PyTorch CUDA 전용 인덱스에서 받도록 **처음부터** 지정한다.

```toml
[project]
dependencies = [
    "accelerate>=1.14.0",
    "bitsandbytes>=0.49.2",
    "datasets>=4.3.0",
    "evaluate>=0.4.6",
    "ipykernel>=7.3.0",
    "ipywidgets>=8.1.8",
    "jupyterlab>=4.6.1",
    "matplotlib>=3.11.0",
    "numpy>=2.5.0",
    "ollama>=0.6.2",
    "openai>=2.44.0",
    "pandas>=3.0.3",
    "peft>=0.19.1",
    "protobuf>=7.35.1",
    "python-dotenv>=1.2.3",
    "scikit-learn>=1.9.0",
    "seaborn>=0.13.2",
    "sentencepiece>=0.2.1",
    "tensorboard>=2.21.0",
    "torch>=2.10.0",
    "torchaudio>=2.10.0",
    "torchvision>=0.25.0",
    "tqdm>=4.68.3",
    "trl>=0.24.0",
    "unsloth>=2026.6.9",
]

[tool.uv.sources]
torch = [
    { index = "pytorch-cu128" },
]
torchvision = [
    { index = "pytorch-cu128" },
]
torchaudio = [
    { index = "pytorch-cu128" },
]

[[tool.uv.index]]
name = "pytorch-cu128"
url = "https://download.pytorch.org/whl/cu128"
explicit = true

```

> **중요**: `torch`, `torchvision`, `torchaudio`는 반드시 `[project.dependencies]`에 **직접** 적어야 한다.
> `unsloth`/`accelerate`/`peft` 등이 torch를 간접적으로 끌어오도록 두면, `tool.uv.sources`의 인덱스
> 오버라이드가 적용되지 않아 uv가 기본 PyPI에서 **CPU 전용 torch**를 설치한다. (Windows용 기본 PyPI torch
> 휠은 CUDA를 포함하지 않는다.)

### 3. 의존성 설치

```powershell
uv sync
```

- 실행 전 이 `.venv`를 사용 중인 Jupyter Lab/커널 프로세스가 있으면 종료한다. 켜진 채로 설치하면 DLL(`.pyd`)
  파일이 잠겨 있어 `os error 5 (액세스 거부)`로 설치가 중간에 실패할 수 있고, 이 경우 실패한 패키지가
  **부분 삭제 상태**로 남아 이후 `import` 에러(예: `markupsafe`의 `Markup` 누락)를 일으킬 수 있다.
- 만약 이런 상태가 의심되면 해당 패키지를 강제 재설치한다.

  ```powershell
  uv pip install <패키지명> --reinstall
  uv pip check
  ```

### 4. GPU 인식 확인

```powershell
uv run python -c "import torch; print(torch.__version__); print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
```

정상 출력 예시:

```
2.10.0+cu128
True
NVIDIA GeForce RTX 5070
```

`0gpu_test.ipynb`를 실행해서 행렬 곱 테스트까지 통과하면 최종 확인 완료.

### 5. Jupyter Lab 실행

```powershell
uv run jupyter lab
```

### 참고: GPU/드라이버가 바뀌는 경우

드라이버 CUDA 버전이나 GPU 세대가 바뀌면 `[[tool.uv.index]]`의 `url`을 드라이버가 지원하는 CUDA 버전
이하로 맞춰 조정한다 (`cu121`, `cu124`, `cu129` 등). `nvidia-smi`의 `CUDA Version` 값이 상한선이며,
그 이하의 torch CUDA 빌드는 모두 호환된다.


# 참고 자료
```
https://bob-data.tistory.com/45?category=1150698
```
