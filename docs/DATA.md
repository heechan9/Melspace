# 데이터 · 이용 조건 · 출처

> 상태: 기획 단계. 아래는 공개 자료를 읽고 확인한 내용이며, 이 저장소는 아직 데이터를 내려받거나 실행하지 않았습니다.
> [README로 돌아가기](../README.md)

## 1. 데이터 (확인한 내용)

대상: NASA가 공개한 SMAP 위성·MSL(큐리오시티 로버) 우주선 텔레메트리 시계열.

| 항목 | 확인 결과 | 출처 |
|---|---|---|
| 구성 | SMAP 55채널·MSL 27채널, 라벨된 이상 구간 105개 | telemanom README |
| 파일 형식 | 채널마다 `train/<id>.npy`, `test/<id>.npy`, 형태 `(시점 수, 입력 차원)` | telemanom README |
| 텔레메트리 값 위치 | **첫 번째 열**. 나머지 열은 명령 송수신 여부를 one-hot으로 인코딩한 정보 | telemanom README · `channel.py` |
| 이미 적용된 전처리 | 시간 정보 익명화, 값은 **test 구간의 최솟값·최댓값 기준으로 (-1, 1) 스케일링** 완료 | telemanom README |
| 채널 의미 | 채널 ID 익명화. 첫 글자가 유형을 뜻함(README 예: `P`=power, `R`=radiation) | telemanom README |
| 라벨 | `labeled_anomalies.csv`: `chan_id`, `spacecraft`, `anomaly_sequences`(test 인덱스 기준 시작·끝), `class`(point/contextual), `num_values` | 원본 CSV 직접 확인 |
| 현재 배포 경로 | Kaggle `patrickfleith/nasa-anomaly-detection-dataset-smap-msl` (약 272 MB, 2022-10-16 v1, 사전학습 모델·예측값 포함) | Kaggle 메타데이터 · telemanom README |

**지키는 규칙**
- 익명 채널에 실제 센서 이름·단위·시각을 붙이지 않습니다. 시간은 "시점 인덱스"로만 표기합니다.
- `P`, `R` 외 첫 글자의 의미는 원 자료에서 확인하지 못했으므로 추측하지 않습니다.
- 원본 데이터 파일은 저장소에 커밋하지 않고, 다운로드 방법만 안내합니다(이용 조건은 아래 2절).

**평가상 한계 (데이터에서 이미 정해진 것)**
- 스케일링에 test 구간의 최솟값·최댓값이 쓰였습니다. 즉 test 정보가 값의 범위에 이미 반영돼 있어, 고정 절댓값 임계값의 성능은 다소 낙관적일 수 있습니다.
- train 구간이 이상 없는 정상 구간인지는 README에서 명시적으로 확인하지 못했습니다. 첫 세션에서 확인할 항목으로 둡니다.
- 라벨 CSV에서 `P-2` 채널이 서로 다른 구간으로 **두 번** 등장합니다. 로드 단계에서 처리 규칙(병합 여부)을 정하고 기록합니다.
- 라벨 파일 전체 탐색 결과와 test 센서값 열람 범위 결정은 [라벨 파일 탐색 기록](./LABEL_NOTES.md)에 있습니다.

## 2. 이용 조건 (데이터와 코드는 따로)

| 대상 | 공개 열람·다운로드 | 재배포·상업적 이용 |
|---|---|---|
| telemanom 코드 | 공개 GitHub | **Apache 2.0** (조건부 허용) |
| SMAP/MSL 데이터 (Kaggle 사본) | 공개. 내려받으려면 Kaggle 계정·API 키 필요 | Kaggle 표기 라이선스는 "Data files © Original Authors". 별도 개방 라이선스가 명시되지 않아 **허용 여부 확인 불가** → 데이터 재배포 안 함 |
| 원 연구 논문 (arXiv 1802.04431) | 공개 | 논문 자체 이용 조건은 이 단계에서 따로 확인하지 않음. 인용만 함 |
| NASA NTRS 기록 20210008188 | 서지 정보 공개(Distribution: Public), 첨부 다운로드 없음 | 저작권 항목 "Other" → 인용만 함 |

코드 라이선스(Apache 2.0)가 데이터에도 적용된다고 가정하지 않습니다.

## 3. 원 연구가 공개한 결과 (MelSpace 실행 결과 아님 · 직접 비교 불가)

telemanom README에 따르면 기본 설정의 LSTM + 동적 임계값은 전체 기준 정밀도 87.5%, 재현율 80.0%를 보고했습니다.

- 이 수치는 **원 연구(Hundman et al., 2018)의 결과**이며, MelSpace가 실행한 결과가 아닙니다.
- MelSpace의 기준선은 임계값 방식과 다중 사건 매칭 규칙이 원 연구와 다르므로([평가 기준](./EVALUATION.md) 참고) **이 수치와 직접 비교하거나 우열을 주장하지 않습니다.**
- 이 프로젝트는 이 수치를 재현하거나 넘어서는 것을 목표로 하지 않습니다.

## 4. 출처

- Telemanom 저장소: https://github.com/khundman/telemanom
- Kaggle 데이터: https://www.kaggle.com/datasets/patrickfleith/nasa-anomaly-detection-dataset-smap-msl
- Hundman et al., *Detecting Spacecraft Anomalies Using LSTMs and Nonparametric Dynamic Thresholding*, KDD 2018 — https://arxiv.org/abs/1802.04431
- NASA NTRS 20210008188: https://ntrs.nasa.gov/citations/20210008188
