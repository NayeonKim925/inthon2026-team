# A — baseline 실행·제출·통합

## 목표와 입력/출력

기존 모델을 먼저 정상 실행·제출하고 팀 전체의 통합 경로를 확보합니다. 입력은 제공 Train, DINOv2·MPNet 로컬 가중치, config입니다. 출력은 공통 split, 학습 체크포인트, 제출 ZIP, 실제 채점 결과와 실행 기록입니다.

## 읽을 순서

1. `starterkit/README.md`: 전체 흐름.
2. `starterkit/config.json`, `common.py`: 경로·환경·seed.
3. `starterkit/train.py`: 생성되는 결과 파일.
4. `starterkit/submission.py`: 서버 인터페이스.
5. `export.py`, `validate.py`, `package.py`: 제출물 생성과 검사.
6. `SERVER_COMPATIBILITY.md`, `VERIFICATION.md`: 과거 검증의 한계. 현재 서버 상태라고 단정하지 않습니다.

## 첫 실행 — 아래 명령은 기존 코드에 있음

모든 명령은 저장소의 `starterkit/` 안에서 실행합니다.

```bash
cd starterkit
python --version
python -c "import torch, transformers; print(torch.__version__, transformers.__version__); print('CUDA:', torch.cuda.is_available())"
```

환경을 운영진 현재 명세와 맞추고 필요한 패키지를 준비합니다. 원본 requirements를 바로 설치하지 않습니다. 데이터는 `../data/Train`에 두거나 아래 CLI로 실제 경로를 지정합니다.

모델 준비는 둘 중 하나만 사용합니다.

```bash
python download_models.py
# 또는 dinov2/, mpnet/ 폴더가 있는 기존 모델 경로에서 복사:
python download_models.py --source-root /실제/모델_폴더
```

작은 실행으로 데이터 로딩·학습·저장이 되는지 확인합니다.

```bash
python train.py --device auto --data-dir ../data/Train --max-episodes 12 --epochs 3 --output-dir outputs/a_smoke
python evaluate.py --checkpoint outputs/a_smoke/best.pt
```

정상 작동하면 전체 데이터를 대상으로 학습합니다. 아래는 기본 20 epoch 및 early stopping 설정을 사용합니다. 같은 결과 폴더는 재사용하지 않습니다.

```bash
python train.py --device auto --data-dir ../data/Train --output-dir outputs/a_baseline_v1
python evaluate.py --checkpoint outputs/a_baseline_v1/best.pt
```

`outputs/a_baseline_v1/splits.json`이 생성되면 바로 D에게 전달합니다. 학습 완료 전이라도 파일 생성 후 공유할 수 있습니다. `features.pt`는 C에게 같은 머신에서 재사용할 수 있는지 알려줍니다. 캐시는 Git에 올리지 않습니다.

제출용으로 내보냅니다.

```bash
python export.py --checkpoint outputs/a_baseline_v1/best.pt --output-dir submission_a_v1
python validate.py --submission-dir submission_a_v1 --device auto
python package.py --submission-dir submission_a_v1 --output a_baseline_v1.zip
```

export 폴더와 ZIP 이름도 실험마다 새로 씁니다. 생성된 ZIP을 채점 사이트에 제출하고 성공/실패·점수·시각을 기록합니다. 사이트 제출은 이 안내 작성 과정에서 수행한 것이 아닙니다.

## 다른 팀원에게 전달

| 받는 사람 | 전달할 것 | 용도 |
| --- | --- | --- |
| 전원 | 실제 환경 버전·데이터 위치·확정 split | 같은 조건에서 개발 |
| C | baseline 점수·features.pt 위치·설정·commit | 중복 특징 추출 최소화 |
| D | 전체 데이터 splits.json | 고정 검증 목록 생성 |
| 전원 | ZIP 생성 방법·서버 결과 | 제출 가능한 경로 확인 |

## 이후 할 일

- B/C/D가 전달한 모델을 공통 검증으로 비교하고 필요하면 결합합니다.
- 기존 export/package는 DINOv2+MPNet 전용입니다. SigLIP·새 모듈에는 필수 파일 목록과 로딩 코드를 변경해야 합니다.
- 제출 용량, 로컬 모델 완결성, 시간·메모리를 측정합니다.
- 학습 commit과 다른 최신 모델 코드를 export가 복사하지 않도록 합니다.
- 첫 baseline 제출과 최종 후보 확보를 우선하고 GPU 예약을 조율합니다.

## 첫 완료 기준

- [ ] smoke와 전체 학습을 구분해 기록했다.
- [ ] 전체 split을 D와 공유했다.
- [ ] 학습 commit·설정·가중치 위치를 기록했다.
- [ ] 로컬 검사를 통과한 ZIP을 만들었다.
- [ ] 실제 서버의 채점 결과 또는 실패 원인을 공유했다.
