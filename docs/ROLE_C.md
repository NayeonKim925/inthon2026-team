# C — 기존 모델 학습 개선

## 목표와 입력/출력

기존 DINOv2+MPNet baseline에 변경 한 가지를 적용하고 동일 검증에서 효과를 비교합니다. 입력은 Train과 정답 설명, 공통 split, 고정 인코더 또는 그 특징 캐시입니다. 출력은 개선된 학습 코드·설정·MLP 가중치와 비교 기록입니다.

## 읽을 순서와 정확한 수정 위치

| 파일·함수 | 이해할 것 | 변경 예 |
| --- | --- | --- |
| `config.json` | hidden_dim=256, lr=0.001, dropout=0.1, epochs=20 | 첫 실험은 learning_rate 하나 변경 |
| `baseline_model.py: SimilarityMLP` | 1536차원 concat → MLP → logit | MLP 크기·구조 |
| `train.py: run_epoch` | BCEWithLogitsLoss, optimizer 갱신 | 학습 loss |
| `train.py: train` | 특징 캐시·학습·BCE 기준 best 저장 | 체크포인트 선택·학습 흐름 |
| `dataset.py: EpisodePairs` | 정답 1개+split 안 무작위 오답 1개 | 오답 수·선택 방식 |
| `submission.py`, `export.py` | 학습 구조와 추론 구조 일치 | 새 모델에 대한 로딩·저장 대응 |

## 지금 A를 기다리지 않고 할 일

1. `exp/mlp-training` 브랜치에서 코드를 읽고 baseline의 입력/출력 shape를 정리합니다.
2. 첫 변경은 config의 학습률 하나처럼 작게 정합니다. 여러 변경을 동시에 섞지 않습니다.
3. `starterkit/config_c_lr.json`을 원본 config와 같은 폴더에 복사하고 학습률을 변경합니다. 같은 폴더에 두어 상대 모델·데이터 경로가 유지되게 합니다.
4. GPU 예약 후 작은 smoke로 실행을 확인합니다. 작은 split 점수는 모델 성능 비교에 사용하지 않습니다.

```bash
cd starterkit
python train.py --config config_c_lr.json --device auto --max-episodes 12 --epochs 3 --output-dir outputs/c_smoke_lr
```

실제 비교는 전체 동일 데이터·공통 split으로 합니다.

```bash
python train.py --config config_c_lr.json --device auto --output-dir outputs/c_lr_v1
python evaluate.py --checkpoint outputs/c_lr_v1/best.pt
```

이 명령은 기존 pair 평가만 합니다. D의 5지선다 평가도 추가로 실행해야 합니다. 새 출력 폴더에서는 원본 코드가 특징을 다시 추출합니다. A와 같은 환경에서 캐시를 안전하게 공유하는 방법을 먼저 협의하면 시간을 줄일 수 있습니다. 캐시 서명 불일치를 무시하지 않습니다.

## 다음 개선 우선순위

1. 동일 특징으로 MLP 학습률·hidden size를 한 번에 하나씩 비교.
2. 같은 split의 실제 설명을 이용한 오답 수 확대 또는 matching/ranking loss 비교.
3. 순서 특징 등 구조 확장은 D와 담당 범위를 합의한 후 진행.

오답 수를 늘리면 `EpisodePairs`, `run_epoch`의 pair_accuracy 계산과 텐서 형태도 함께 바뀝니다. `train.py`만 바꾸고 평가·추론 코드를 방치하지 않습니다. 다른 설명이 실제로도 맞는 false negative 가능성을 기록합니다.

허용 모델 특징을 손실 입력으로 쓰는 것과 teacher의 정답 예측·로짓·순위를 목표로 모사하는 증류를 구분합니다. teacher 출력으로 학습 레이블/목표를 만들지 않습니다. 외부 AI로 학습 설명을 새로 만들지 않습니다.

## 캐시와 fine-tuning 주의

현재 인코더는 requires_grad=False이고 특징 추출은 inference_mode입니다. 인코더 학습을 하려면 고정된 features.pt로 MLP만 학습하는 경로를 바꿔야 합니다. 24시간 대회에서는 먼저 작은 head 학습을 완성합니다. 시간 집계를 바꾸면 과거의 프레임 평균 캐시를 그대로 재사용할 수 없습니다.

## A에게 넘길 결과

- baseline 대비 바꾼 한 가지와 이유.
- 코드 commit·config·공통 split/manifest 버전.
- `best.pt` 위치와 선택 epoch, 학습 기록.
- pair 평가와 5지선다 평가를 구분한 점수.
- 실제 학습 시간·추론 시간·가중치 용량.
- 구조 변경 시 수정된 submission/export/package 코드.

## 첫 완료 기준

- [ ] 기본 baseline과 같은 전체 split으로 비교했다.
- [ ] 한 가지 변경과 결과를 기록했다.
- [ ] 특징 재사용 조건과 경로를 확인했다.
- [ ] 체크포인트를 다시 읽어 평가할 수 있다.
- [ ] A가 같은 commit·설정으로 export할 수 있다.
