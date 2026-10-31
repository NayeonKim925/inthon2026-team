# Inthon 2026 — 4인 팀 시작 안내

이미지 시퀀스를 보고 후보 설명을 적합한 순서로 정렬하는 데이터톤 프로젝트입니다.
각자 아래 자신의 역할 문서를 읽고 시작하세요. 담당자 실명은 팀에서 정합니다.

| 담당 | 역할 | 처음 읽을 문서 | 첫 결과물 | 작업 브랜치 |
| --- | --- | --- | --- | --- |
| A | 기존 baseline 실행·제출·통합 | [A 시작하기](docs/ROLE_A.md) | 전체 학습 baseline ZIP과 첫 채점 결과 | `infra/baseline-submit` |
| B | SigLIP 대안 모델 | [B 시작하기](docs/ROLE_B.md) | 후보별 점수를 반환하는 SigLIP 모델 | `exp/siglip` |
| C | 기존 baseline 학습 개선 | [C 시작하기](docs/ROLE_C.md) | 한 가지 변경을 적용한 학습 결과 | `exp/mlp-training` |
| D | 공통 5지선다 검증·이후 순서 모델 | [D 시작하기](docs/ROLE_D.md) | 고정 검증 후보 목록과 순위 평가 코드 | `eval/ranking` |

## 전원이 먼저 할 일 — 10분 합의

1. A/B/C/D를 한 명씩 정하고 각자의 GitHub 계정을 저장소 Collaborator로 초대합니다. 읽기와 clone은 가능해도 push에는 쓰기 권한이 필요합니다.
2. [공통 환경·입출력·Git·GPU 사용 규칙](docs/WORKFLOW.md)을 읽습니다.
3. seed `2026`, validation fraction `0.2`를 사용합니다. A가 **전체 데이터**로 만든 `splits.json`의 ID 목록을 확정해 공유합니다. smoke 실행의 split을 공통 검증으로 사용하지 않습니다.
4. 각자 별도 폴더에 clone하고 자기 브랜치에서 시작합니다. 같은 서버에서도 작업 폴더는 분리합니다.
5. 데이터 위치, 가중치 공유 위치, GPU 예약 방법을 팀 채팅에 정합니다. 실제 데이터·가중치는 Git에 올리지 않습니다.

## 코드 위치와 현재 구현 범위

- [starterkit/](starterkit/): 첨부받은 원본 텍스트 코드와 노트북을 보관했습니다. 모델 가중치와 샘플 ZIP은 포함하지 않습니다.
- [starterkit/README.md](starterkit/README.md): 원본 사용법과 baseline 구조 설명입니다.
- baseline: DINOv2 프레임 CLS 특징 평균 + MPNet 텍스트 특징 → concat MLP. 인코더는 동결하고 MLP만 학습합니다.
- `train.py`, `evaluate.py`, `export.py`, `validate.py`, `package.py`는 이미 있습니다.
- **SigLIP 모델, 5지선다 검증기, 시간 순서 모델은 아직 없습니다.** 각 역할 문서의 구현 작업이며, 완료된 기능으로 간주하지 않습니다.
- 기존 `evaluate.py`는 정답 1개와 random negative 1개를 비교합니다. 리더보드의 5지선다 순위 평가와 다릅니다.

## 입력과 출력

| 단계 | 입력 | 출력 |
| --- | --- | --- |
| baseline 학습 | Train 이미지 시퀀스·정답 설명 + 허용된 로컬 인코더 | 특징 캐시, split, MLP 체크포인트, 학습 기록 |
| 모델 추론 | 시간순 RGB PIL 이미지 리스트 + `id`, `text`가 있는 후보 리스트 | 후보 입력 순서대로 유한한 점수 리스트 |
| 서버 제출 인터페이스 | `Submission(artifact_dir, device)` 후 `predict(images, choices)` | 모든 후보 ID가 정확히 한 번씩 등장하는 내림차순 `list[str]` |
| 공통 로컬 검증 | 고정된 검증 에피소드·후보 목록 + 모델 | 순위 점수, Top-1, 에피소드별 후보 점수, 시간 |

서버에는 정답이나 다른 평가 에피소드의 정보를 쓰지 않습니다. 평가 중 학습·보정·메모리 갱신을 하지 않습니다.

## 동시에 진행하는 순서

| 시점 | A | B | C | D |
| --- | --- | --- | --- | --- |
| 즉시 | 환경·경로 확인, smoke 실행 | 독립 SigLIP 코드 작성 | 기존 학습 구조 분석, 변경 하나 구현 | split 생성 방식 확인, 평가기 작성 |
| 다음 | 전체 baseline 학습, split 공유 | GPU 예약 후 최소 추론 | baseline 특징 캐시 활용 가능 여부 확인 | A의 전체 split으로 검증 후보 고정 |
| 첫 결과 이후 | baseline 제출, 실패 해결 | 공통 검증으로 성능·속도 측정 | 동일 검증으로 개선 비교 | 공통 평가 코드를 먼저 PR, 이후 순서 모델 실험 |
| 통합 | 좋은 모델 선택·결합·포장 | 코드·가중치·export 전달 | 코드·가중치·학습 설정 전달 | 후보 manifest·결과·시간 전달 |

GPU 한 대면 개발은 동시에, GPU 작업은 예약 순서대로 합니다. 한 사람이 다른 사람의 학습 완료를 기다리며 아무것도 하지 않는 구조로 운영하지 않습니다.

## 결과 공유

각 실험은 [실험 기록 양식](docs/EXPERIMENT_TEMPLATE.md)을 복사해 `experiments/<실험ID>.md`로 저장하고 PR을 만듭니다. [GPU 예약 양식](docs/GPU_QUEUE.md)은 채팅이나 공유 문서에 복사해 운영해도 됩니다.

학습 코드 commit, 검증 manifest 버전, 가중치 경로, 로컬 점수, 속도, 제출 점수를 함께 남깁니다. 가중치만 공유하면 재현·통합할 수 없습니다.

## 규정과 실행 환경

- 제출용 사전학습 모델은 제공된 허용 목록만 사용합니다: `facebook/dinov2-base`, `sentence-transformers/all-mpnet-base-v2`, `google/siglip-base-patch16-224`, `SmolVLM-256M-Instruct`.
- 외부 AI 서비스로 학습 데이터를 직접 생성·증강하지 않습니다. 코딩 지원은 허용됩니다. 증류는 금지됩니다.
- 추론은 오프라인, 전체 테스트 15분 이내입니다. 재현은 T4 16GB 한 대에서 24시간 이내여야 합니다.
- 안내 PDF의 ZIP 최대 용량은 1GB입니다. 실제 업로드·압축 해제 한도도 확인합니다.
- **원본 `requirements.txt`는 과거 환경입니다.** 제공된 현재 규정의 Python 3.12.3 / torch 2.8.0+cu128 / transformers 4.57.6 등과 다릅니다. 기존 GPU 환경에 그대로 설치해 덮어쓰지 않습니다. A가 현재 운영진 환경과 맞추고 정확한 환경 기록을 공유합니다.
- 원본 검증 기록은 CPU의 소규모 smoke 결과입니다. 현재 서버 호환성·전체 데이터 성능·T4 15분 통과를 입증하지 않습니다.
- 20회 제출은 팀의 목표입니다. 실제 제출 횟수 제한과 최종 모델 선택 방식은 채점 사이트에서 확인합니다.

이 저장소의 안내는 제공된 규정·starterkit을 해석한 팀 작업 계획입니다. 운영진의 최신 공식 규정이 우선합니다.
