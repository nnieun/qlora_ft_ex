# QLoRA 기반 AI 면접관

## 1. 문제정의

- AI 기반 채용·역량검사와 AI 면접 활용이 확대되고 있음
- 취업준비생은 AI 영상면접과 직무 면접을 대비해야 함
- 하지만 개인 이력에 맞는 심층 질문을 준비하기 어려움

**목표: 직무와 경험을 입력하면, 실제 역량을 검증하는 질문 한 개를 생성한다.**

---

## 2. 요구사항

- 입력: 직무 + 프로젝트·기술·문제 해결 경험
- 원본 데이터: 질문 후보 3개
- 출력: 경험과 가장 관련된 Top-1 질문 한 개
- 원본 JSONL 파일은 수정하지 않음
- 외부 API, 사람 수작업 라벨링 없이 처리

---

## 3. 제약조건

- GPU VRAM: 8GB
- 대형 모델 Full Fine-tuning은 어려움
- 학습·서빙 시 GPU 메모리 사용량을 줄여야 함

**선택: Qwen3-1.7B + QLoRA 4-bit 학습**

---

## 4. Top-1 우선순위 기준

**지원자의 경험과 가장 의미적으로 가까운 질문을 선택한다.**

- 이력에 실제로 적힌 기술과 연결되는가?
- 프로젝트의 선택·실험·문제 해결을 묻는가?
- 이력에 없는 기술을 가정하지 않는가?

현재 구현의 핵심 기준은 **경험 관련성**이다.

---

## 5. Top-1 추출 방법

```text
직무 + 경험
       +
후보 질문 3개
       ↓
로컬 BGE 임베딩 변환
       ↓
경험과 각 질문의 cosine similarity 계산
       ↓
가장 높은 점수의 질문 1개 선택
```

- 모델: `BAAI/bge-m3`
- CPU 실행 → QLoRA의 8GB GPU VRAM을 추가로 사용하지 않음
- 외부 API 호출 없음

---

## 6. 학습 데이터 흐름

```text
datas/ai_interview_sft.jsonl (원본 유지)
       ↓
후보 질문 3개 분리
       ↓
BGE로 Top-1 선택
       ↓
prompt: 직무 + 경험
completion: 선택 질문 1개
       ↓
QLoRA SFT 학습
```

---

## 7. QLoRA 설정

| 항목 | 값 |
| --- | --- |
| 기본 모델 | Qwen/Qwen3-1.7B |
| 양자화 | 4-bit NF4 + double quantization |
| LoRA rank | 16 |
| batch size | 1 |
| gradient accumulation | 8 |
| max length | 512 |
| optimizer | paged_adamw_8bit |
| 메모리 절약 | gradient checkpointing |

---

## 8. LoRA와 QLoRA

| 구분 | LoRA | QLoRA |
| --- | --- | --- |
| 기본 모델 | bf16/fp16 | 4-bit NF4 |
| 학습 대상 | LoRA adapter | LoRA adapter |
| VRAM | 상대적으로 큼 | 상대적으로 작음 |
| 8GB 적합성 | OOM 가능성 있음 | 실제 학습 권장 |

비교 조건: 동일 JSONL, 동일 Top-1 데이터, 동일 모델, 동일 학습 step

---

## 9. 테스트 및 평가

- Before / After: 같은 직무·경험 입력에서 질문 한 개가 생성되는지 확인
- 자원 비교: 최고 VRAM, 학습 시간, OOM 여부
- 품질 비교: 경험 관련성, 구체성, 검증 가능성

---

## 10. 개선 방향

- 31개 데이터에서 수백 개 이상의 직무·경험 사례로 확장
- 경험 관련성 외에 질문의 구체성·검증 가능성 점수 추가
- LoRA와 QLoRA를 동일 조건에서 실제 학습 후 수치 비교
- 번호·해설 없이 질문 한 문장만 출력하도록 후처리 검증 추가
