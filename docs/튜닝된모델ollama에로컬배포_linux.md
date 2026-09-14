[ 튜닝된 모델 ollama 배포 ]
실행 순서
```
1. cmake 설치
2. llama.cpp 받기		
3. llama.cpp 가상환경 만들기
4. HF 병합 모델을 F16 GGUF로 변환
5. llama.cpp 빌드 (F16/BF16 GGUF)
6. 양자화 실행 파일 확인 및 Q4_K_M 양자화하기
7. Ollama 설치 확인
8. Ollama에 등록하기
9. 등록 모델 테스트
```

0. 작업 디렉토리 확인
- 작업 디렉토리 : ./qlora_ft_ex
- 튜닝이 끝난 모델의 위치 : qlora_ft_ex/outputs/mymodel 

1. cmake 설치 및 확인
- CMake 설치 :
```
sudo apt update
sudo apt install -y cmake
```
- CMake 설치 확인 : cmake --version

2. llama.cpp 받기 또는 업데이트
- 작업 폴더로 이동
```
cd /d qlora_ft_ex
```
- llama.cpp 받기 또는 업데이트
```
sudo apt update
sudo apt install -y git cmake build-essential

git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
```

3. uv 가상환경 만들고 활성화 하기
```
uv sync

source .venv/bin/activate

# transformer 맞춰주기
uv pip install -U "transformers>=5"
```

4. HF 병합 모델을 F16 GGUF로 변환
- 병합 모델 폴더 확인
qlora_ft_ex/outputs/mymodel

대략 아래 파일들이 보여야 정상이다.
	config.json
	model.safetensors
	tokenizer.json
	tokenizer_config.json
	generation_config.json

- 폴더 이동
```
cd qlora_ft_ex

python ./llama.cpp/convert_hf_to_gguf.py ./outputs/mymodel --outfile ./outputs/mymodel-f16.gguf --outtype f16
```

[참고]
```
[Fine-tuning 환경]
transformers 5.x
tokenizers x.x

        ↓ 모델 저장

[GGUF 변환 환경]
transformers 5.x
tokenizers x.x
```

- 변환 결과 확인:
```
ls outputs/mymodel-f16.gguf
```

5. llama.cpp 빌드 (시간 오래 걸림)
```
cd ./qlora_ft_ex/llama.cpp
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel
```

6. 양자화 실행 파일 확인 및 Q4_K_M 양자화하기
- 양자화 실행 파일 확인
```
cd ./qlora_ft_ex/
ls llama.cpp/build/bin/llama-quantize
```

- 양자화 실행
```
llama.cpp/build/bin/llama-quantize \
  ./outputs/mymodel-f16.gguf \
  ./outputs/mymodel-q4_k_m.gguf \
  Q4_K_M
```

- 결과 확인:
```
ls ./outputs/mymodel-q4_k_m.gguf
```

7. 경로가 출력되면 정상.
- 올라마 모델 리스트 확인 : ollama list

8. Ollama에 등록하기
- Ollama Modelfile 만들기
- 폴더 생성:
```
	mkdir outputs/ollama_sageuk
```
- vscode로 Modelfile 생성:
```
code outputs/ollama_sageuk/Modelfile
```

- 아래 내용을 그대로 붙여넣고 저장한다.
- 절대 경로
```
FROM /mnt/e/gg_ai_merbership_1th/qlora_ft_ex/outputs/mymodel-q4_K_M.gguf

TEMPLATE """{{ if .System }}<|im_start|>system
{{ .System }}<|im_end|>
{{ end }}<|im_start|>user
{{ .Prompt }}<|im_end|>
<|im_start|>assistant
{{ .Response }}<|im_end|>"""

PARAMETER stop "<|im_end|>"
PARAMETER stop "<|im_start|>"
PARAMETER temperature 0.7
PARAMETER top_p 0.8
PARAMETER repeat_penalty 1.12
PARAMETER num_ctx 4096
PARAMETER num_predict 160
```
* 중요: 파일 이름이 반드시 Modelfile이어야 한다. Modelfile.txt면 안 된다.

- Ollama에 모델 등록
```
ollama create sageuk-qwen -f /mnt/e/gg_ai_merbership_1th/qlora_ft_ex/outputs/ollama_sageuk/Modelfile
```

9. 등록 확인하기
- Ollama 모델 목록 확인
```
ollama list
```

- Ollama 배포 모델 테스트
```
ollama run sageuk-qwen "안녕하세요!"
```

- 대화 모드:
ollama run sageuk-qwen

>>> 안녕?

그러시다면, 소인이 곧 풀릴 듯하옵니다.

>>> 좋아하는 칼라는 뭐야?
소인의 마음은 항상 청명하옵니다. 그대를 잘 아뢰어 올리겠나이다.

>>> Send a message (/? for help)

