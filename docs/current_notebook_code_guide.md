# 현재 `5.qlora_-Copy1.ipynb` 코드 정리

이 문서는 현재 저장된 노트북 코드를 기준으로 작성했다. 학습 원본은 `datas/ai_interview_sft.jsonl`이며, 원본 파일을 수정하지 않는다.

## 1. 현재 구현한 목표

지원자의 직무와 경험을 입력받아, 해당 경험을 검증하는 기술면접 질문 **한 개**를 생성한다.

원본 JSONL의 한 행은 다음 구조다.

```json
{
  "instruction": "지원자의 경험을 바탕으로 실제 역량을 검증할 수 있는 심층 기술면접 질문 3개를 만들어라.",
  "input": "직무: ...\n경험: ...",
  "output": "1. ...\n2. ...\n3. ..."
}
```

원본 `output`에는 질문이 세 개이므로, 그대로 SFT를 수행하면 모델도 질문 세 개를 생성하는 방향으로 학습된다. 현재 코드는 학습 전에 세 후보 중 하나를 고르고, 그 질문 한 개만 정답(`completion`)으로 사용한다.

---

## 2. 데이터 읽기

노트북의 데이터 경로는 다음과 같다.

```python
DATA_PATH = "datas/ai_interview_sft.jsonl"
```

노트북에는 `json` 모듈로 `raw_data`를 읽는 셀과 `load_dataset("json", ...)`으로 `raw_interview_dataset`을 읽는 셀이 모두 있다.

- `raw_data`: 마지막의 LoRA용 데이터셋 생성 셀에서 사용한다.
- `raw_interview_dataset`: QLoRA용 Top-1 선택 및 학습 데이터 생성에 사용한다.

둘 다 같은 JSONL 파일을 읽으므로, 비교 실험 시 두 경로에서 선택된 Top-1 결과가 같은지 확인해야 한다.

---

## 3. Top-1 선택 방법

### 사용 모델

```python
EMBEDDING_MODEL_ID = "BAAI/bge-m3"
embedding_model = SentenceTransformer(EMBEDDING_MODEL_ID, device="cpu")
```

`BAAI/bge-m3`는 질문과 경험 텍스트를 벡터로 바꾸는 로컬 임베딩 모델이다. `device="cpu"`로 고정했으므로 QLoRA 학습 GPU VRAM과 분리된다.

### 선택 과정

```text
input의 직무·경험
  + output의 후보 질문 3개
       ↓
각 텍스트를 정규화 임베딩으로 변환
       ↓
경험 벡터와 질문 벡터의 내적 계산
       ↓
가장 높은 cosine similarity의 질문 선택
```

핵심 코드는 다음과 같다.

```python
embeddings = embedding_model.encode(
    [example["input"], *candidates],
    normalize_embeddings=True,
    convert_to_tensor=True,
)

scores = (embeddings[1:] @ embeddings[0]).cpu().tolist()
selected_index = int(torch.tensor(scores).argmax().item())
```

이 방식은 외부 API나 사람 라벨링을 사용하지 않는 자동 pseudo-labeling이다. 다만 현재 점수는 **경험 관련성**만 반영한다. 질문 난이도나 면접 질문의 날카로움은 직접 점수화하지 않는다.

---

## 4. QLoRA 학습 데이터 생성

Top-1으로 선택된 질문은 `selected_question`에 저장된다.

```python
TOP1_INSTRUCTION = (
    "지원자의 경험을 바탕으로 실제 역량을 검증할 수 있는 "
    "심층 기술면접 질문 한 개만 만들어라. "
    "번호, 해설, 추가 질문 없이 질문 한 문장만 출력하라."
)
```

최종 SFT 데이터는 아래 두 컬럼만 가진다.

| 컬럼 | 내용 |
| --- | --- |
| `prompt` | 위 지시문 + JSONL의 `input` |
| `completion` | `selected_question`, 즉 Top-1 질문 한 개 |

```python
full_dataset = selected_dataset.map(format_interview_example)
train_dataset = full_dataset.select_columns(["prompt", "completion"])
```

이 `train_dataset`이 현재 QLoRA 학습에 전달되는 실제 데이터셋이다.

---

## 5. QLoRA 학습 방식

### 모델과 양자화

```python
MODEL_ID = "Qwen/Qwen3-1.7B"

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=COMPUTE_DTYPE,
    bnb_4bit_use_double_quant=True,
)
```

기본 Qwen3-1.7B 모델을 4-bit NF4로 GPU에 올린다. 기본 모델 가중치는 고정하고 LoRA adapter만 업데이트한다. 이것이 QLoRA다.

### LoRA adapter 설정

```python
LoraConfig(
    r=16,
    lora_alpha=32,
    lora_dropout=0.05,
    target_modules=[
        "q_proj", "k_proj", "v_proj", "o_proj",
        "gate_proj", "up_proj", "down_proj",
    ],
)
```

attention과 MLP projection layer에 adapter를 붙인다. 노트북 출력 기준 학습 가능한 LoRA 파라미터는 약 17.4M개다.

### 학습 설정

| 항목 | 현재 값 |
| --- | --- |
| `max_steps` | 80 |
| batch size | 1 |
| gradient accumulation | 8 |
| max length | 512 |
| learning rate | `2e-4` |
| optimizer | `paged_adamw_8bit` |
| 메모리 절약 | gradient checkpointing |
| loss 범위 | `completion_only_loss=True` |

`completion_only_loss=True`이므로 prompt의 직무·경험 텍스트가 아닌, Top-1 질문 생성 부분에만 loss를 계산한다.

---

## 6. Before / After 테스트

`test_questions`는 학습 데이터가 아니다. QLoRA 학습 전후에 같은 직무·경험을 넣어 생성 질문이 어떻게 변하는지 확인하는 테스트 입력이다.

```python
test_questions = [
    {
        "직무": "백엔드 개발자",
        "경험": "FastAPI와 PostgreSQL로 주문 API를 만들고, Redis 캐시를 적용해 조회 지연 시간을 개선했다.",
    },
]
```

Before 모델은 번호·해설까지 길게 생성할 수 있다. 이는 파인튜닝 전 기본 모델이 “질문 한 문장만” 지시를 안정적으로 따르지 못하는 기준선 결과다. After에서는 Top-1 질문 한 문장에 가까운 결과가 나오는지 확인한다.

---

## 7. LoRA와 QLoRA 비교: 현재 상태

현재 노트북에는 **QLoRA 학습 코드가 완성되어 있다.**

마지막 셀에는 다음 LoRA용 데이터셋 준비 코드만 추가되어 있다.

```python
lora_raw_dataset = Dataset.from_list(raw_data)
lora_selected_dataset = lora_raw_dataset.map(select_most_experience_related)
lora_full_dataset = lora_selected_dataset.map(format_interview_example)
lora_train_dataset = lora_full_dataset.select_columns(["prompt", "completion"])
```

즉, 현재 상태에서는 `lora_train_dataset`까지 만들어지지만, **일반 LoRA 모델을 bf16/fp16으로 로드하고 학습하는 `lora_model`, `lora_trainer` 코드는 아직 노트북에 저장되어 있지 않다.** 따라서 LoRA vs QLoRA의 실제 수치 비교는 아직 수행되지 않았다.

공정 비교를 위해서는 다음 조건을 같게 유지해야 한다.

```text
동일 JSONL → 동일 Top-1 선택 → 동일 train_dataset
동일 모델(Qwen3-1.7B), rank=16, max_steps=80
동일 batch size=1, gradient accumulation=8, max length=512
```

차이는 기본 모델 로딩 방법 하나다.

| 방식 | 기본 모델 |
| --- | --- |
| LoRA | bf16/fp16으로 로드, 양자화하지 않음 |
| QLoRA | 4-bit NF4로 로드 |

비교할 지표는 최고 VRAM, 학습 시간, 최종 train loss, OOM 여부, 그리고 같은 테스트 경험에서 생성한 질문 품질이다.

> 8GB GPU에서는 LoRA를 시작하기 전에 QLoRA 모델을 GPU에서 제거해야 한다. 두 모델을 동시에 올리면 VRAM이 합산되고, LoRA의 실제 메모리 사용량도 측정할 수 없다.

---

## 8. 실행 순서

1. GPU·패키지 확인
2. Qwen3-1.7B와 tokenizer 로드
3. `datas/ai_interview_sft.jsonl` 로드
4. BGE로 세 후보 중 Top-1 선택
5. `train_dataset` 생성
6. QLoRA adapter 설정 및 QLoRA 학습
7. Before/After 출력 확인
8. QLoRA 결과(VRAM, 시간, loss)를 기록
9. QLoRA 모델을 GPU에서 제거
10. 같은 `train_dataset`으로 일반 LoRA 학습 코드를 별도 실행
11. LoRA 결과를 기록하고 비교
