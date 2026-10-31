# 공통 작업 방법

## 1. 코드 받기와 브랜치

자기 컴퓨터/Colab/서버의 터미널에서 실행합니다. GPU 실행은 실제 GPU가 연결된 환경에서 합니다. GitHub는 코드 저장·공유 장소입니다.

```bash
git clone https://github.com/NayeonKim925/inthon2026-team.git
cd inthon2026-team
git fetch origin
```

아래에서 자기 역할의 한 줄만 실행합니다. 원격 브랜치를 자동으로 추적합니다.

```bash
git switch --track origin/infra/baseline-submit  # A
git switch --track origin/exp/siglip             # B
git switch --track origin/exp/mlp-training       # C
git switch --track origin/eval/ranking           # D
```

이미 로컬 브랜치가 있으면 `git switch 브랜치명`만 사용합니다. 원격 브랜치가 아직 없으면 `git switch -c 브랜치명`으로 만들고 첫 push에 `git push -u origin 브랜치명`을 사용합니다.

같은 서버의 하나의 clone 폴더를 네 명이 공유하지 않습니다. 브랜치를 바꾸면 폴더 안의 파일도 바뀝니다. 각자 clone하되 원본 데이터는 별도 공용 읽기 전용 경로로 공유할 수 있습니다.

## 2. 로컬 폴더와 환경

권장 데이터 배치입니다. 이미지 이름은 숫자순으로 읽습니다.

```text
inthon2026-team/
  starterkit/
    config.json
    train.py
    models/dinov2/       # 다운로드한 로컬 모델, Git 제외
    models/mpnet/
    outputs/            # 학습 결과, Git 제외
  data/Train/
    episode_1/
      description.txt
      images/1.jpg ...
```

`starterkit/config.json`의 기본 `../data/Train`은 이 구조와 맞습니다. config 안 경로는 config 파일 위치 기준, CLI로 넘기는 경로는 명령 실행 위치 기준입니다. 각자 다른 데이터 경로는 `--data-dir /실제/Train`으로 넘깁니다.

기존 GPU 환경에서 먼저 다음을 확인합니다.

```bash
python --version
python -c "import torch, transformers; print(torch.__version__, transformers.__version__); print('CUDA:', torch.cuda.is_available())"
```

A가 운영진의 현재 환경과 비교해 설치 방법을 공유합니다. 기존 requirements는 그대로 보관한 참고 자료입니다. 새 환경이 필요하면 별도 가상환경을 쓰고, 운영진 GPU의 CUDA 패키지를 임의로 교체하지 않습니다. 모델 다운로드는 인터넷이 있는 준비 단계에서만 하고, 제출에는 필요한 로컬 파일을 모두 포함합니다.

## 3. 공통 모델 계약

서버의 확정 인터페이스는 원본 `starterkit/submission.py`입니다.

```python
Submission(artifact_dir: str, device: str)
predict(images, choices) -> list[str]
```

- `images`: 시간순 RGB PIL 이미지 리스트.
- `choices`: `{"id": str, "text": str}` 리스트. ID를 A~E로 가정하지 않습니다.
- 반환값: 모든 후보 ID를 각각 한 번씩 포함하는 순위 리스트.
- 생성 시 한 번 로딩하고 `eval()`과 추론 모드를 사용합니다.
- `artifact_dir` 안 로컬 모델만 로딩하며 HF 캐시나 인터넷 다운로드에 의존하지 않습니다.

팀 내부에서는 아래 메서드를 추가로 통일합니다. 원본 baseline에 이미 같은 메서드가 있습니다.

```python
score(images, choices) -> list[float]
```

후보 입력 순서대로 점수 하나씩 반환하고 큰 점수가 더 적합하다는 뜻입니다. 모두 유한한 숫자여야 합니다. baseline은 sigmoid 범위이지만 대안 모델까지 0~1로 강제하지 않습니다. 모델별 점수 척도가 다르므로 앙상블에서 그대로 더하지 말고 검증셋으로 결합 방법을 정합니다. 평가 에피소드들로 보정값을 추정하지 않습니다.

B/C/D는 독립 구현·실험을 하고 A가 최종 `submission.py`를 통합합니다. 모델 변경 때 학습·추론·export·package가 같은 구조인지 확인합니다.

## 4. 공통 split과 검증

A가 전체 Train 실행에서 만든 `splits.json`을 D에게 전달합니다. D가 확정본을 `validation/splits.json`으로 PR합니다. smoke split, 파일 목록이 다른 데이터, 다른 seed의 분할은 섞지 않습니다. D 완료 전에는 같은 전체 데이터·seed·비율로 개발할 수 있지만 결과 비교 시 ID 목록을 대조합니다.

D는 validation split 안의 설명만 후보로 사용하고, 정답 1개와 고정 오답 4개를 묶습니다. 평가에 사용하는 설명은 수정·생성하지 않습니다. 동일 원본 영상의 여러 에피소드라는 메타데이터가 있다면 D가 누수를 확인하고, 분할을 변경할 때는 전원이 새 분할로 다시 비교합니다.

검증 manifest에는 에피소드 ID·후보 ID·설명 소스 에피소드 ID·정답 ID를 넣습니다. 텍스트/이미지 원문은 Git에 넣지 않습니다. 정답 정보는 로컬 평가기만 읽고 모델에는 images/choices만 전달합니다. 실제 테스트 정답은 사용하지 않습니다.

고정 오답은 실제 테스트의 행동·순서가 비슷한 어려운 오답을 대표하지 못합니다. 로컬 점수는 비교용 대리 지표입니다.

## 5. 변경 저장·PR·공통 코드 받기

작은 기능이 동작하면 자신의 실제 변경 파일만 선택합니다.

```bash
git status
git add 실제_수정한_파일
git commit -m "Describe the implemented change"
git push
```

첫 push에서 upstream이 없다면 `git push -u origin 자신의_브랜치`를 사용합니다. GitHub에서 자신의 브랜치 → Compare & pull request → base `main`으로 만듭니다. PR에 변경 목적, 실행 명령, 실험 기록 링크, 알려진 제한을 적고 팀원 한 명이 확인한 뒤 합칩니다.

공통 코드를 받기 전에는 자신의 변경을 commit합니다.

```bash
git fetch origin
git merge origin/main
```

충돌이 나면 담당자와 함께 해결하고, 다른 사람 변경을 무조건 삭제하지 않습니다. 공유 브랜치 force push는 하지 않습니다. 실험이 실패해도 기록은 남기고, 실행되지 않는 코드로 main을 교체하지 않습니다.

## 6. GPU와 캐시

- 한 대의 T4를 네 명이 무계획으로 동시에 쓰지 않습니다. 작업명·담당·예상 종료·사용 메모리를 예약합니다.
- A의 baseline 특징 추출과 B의 SigLIP 특징 추출은 다릅니다. 서로 대체하지 못합니다.
- C가 인코더·전처리·프레임 집계를 바꾸지 않으면 baseline 특징을 재사용할 수 있습니다. 기존 코드는 새 output 폴더마다 `features.pt`를 찾으므로 자동 공유가 아닙니다. 파일 복사 또는 공유 캐시 옵션 구현이 필요합니다.
- 기존 cache signature는 절대 경로·파일 시간과 모델 파일 정보를 포함합니다. 다른 컴퓨터로 복사하면 검사가 실패할 수 있습니다. 검사를 끄지 말고 동일 환경에서 사용하거나 재추출합니다.
- 인코더 fine-tuning에는 동결 특징 캐시 기반 학습을 그대로 쓸 수 없습니다. 별도 학습 경로가 필요합니다.
- 평가 입력·예측·통계를 축적해 다음 평가 에피소드를 보정하거나 검색 메모리를 갱신하지 않습니다.

## 7. 전달 및 제출

모델 담당자는 A에게 코드 commit, 가중치 위치, 모델 다운로드 ID, 설정, 검증 결과, export 방법을 전달합니다. 수정한 코드가 필요한데 가중치만 전달하는 상태를 피합니다.

A는 정확한 학습 commit에서 export하고, 빈 캐시·오프라인 로딩, 반환 ID, ZIP 최상위 구조, 용량, T4 메모리와 실행 시간을 확인합니다. 서버 코드가 없다면 `--server-dir` 검사는 수행할 수 없습니다. 로컬 validate만으로 전체 테스트 15분 통과를 주장하지 않습니다.

전체 테스트는 안내상 1,910개입니다. 900초/1,910 ≈ 0.471초/에피소드가 로딩 등을 제외하기 전 단순 평균 예산입니다. 대표 validation으로 전체 시간을 추정하되 프레임 수 차이·모델 로딩·I/O·실행 환경 때문에 오차가 있음을 기록합니다.

최종 후보의 학습·증강 코드와 환경·seed·설정도 별도 보존합니다. 추론 ZIP에 학습 데이터를 넣지 않습니다.
