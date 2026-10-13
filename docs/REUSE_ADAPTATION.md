# 기존 프로젝트 → MelSpace 대입·수정 계획

> 상태: 기획 단계. 아래는 **무엇을 어떻게 바꿔 넣을지** 정리한 것이며, 코드는 아직 작성하지 않았음.
> 원칙: 원본 저장소는 수정하지 않는다. 필요한 로직은 이해한 뒤 새 저장소에 다시 작성하고, 출처(파일·커밋)를 주석과 문서에 남긴다.
> 커밋 기준: FabGuard `519654e`, Bunkering `7884e97`, Judicial `25d15f9`, Adversarial(포크) `9b64b8e`, TriGuard `d0db55a`, AU Job Atlas `6fb24fa`, telemanom `master`(Apache 2.0).

## 먼저 확인할 위험 하나

telemanom README는 값이 **test 구간의 최솟값·최댓값 기준으로** (-1, 1) 스케일링됐다고만 적고 있다. train이 같은 기준으로 스케일링됐는지는 적혀 있지 않다.
train과 test의 스케일 기준이 다르면 **train으로 정한 임계값이 test에서 같은 의미를 갖지 않는다.** 모든 기준선의 전제이므로 첫 세션에서 가장 먼저 확인한다(아래 V1-0).

---

## V1 — 채널 1개, 기준선 1개

| # | 원본 | 그대로 가져올 부분 | 바꿀 부분 | 새 저장소 위치(예정) |
|---|---|---|---|---|
| V1-0 | FabGuard `advanced_evaluation.population_stability_index` | 기준 분포의 분위 구간으로 두 분포 차이를 수치화 | 입력을 SECOM 변수 → 채널 첫 열의 train vs test 값으로. 판정이 아니라 **진단 기록**으로만 사용 | `src/diagnostics.py` |
| V1-1 | FabGuard `config.ExperimentConfig` | `frozen=True` dataclass로 설정을 결과 전에 선언 | 필드를 `channel_id`, `validation_fraction`, `gap`, `sigma_multiplier`, `method`로 교체 | `src/config.py` |
| V1-2 | FabGuard `data.OFFICIAL_HASHES`, `validate_raw_files` | 파일 SHA-256 대조 | 공식 해시가 없으므로 **첫 다운로드 때 계산한 값**을 기록하고 "공식 해시 아님"으로 표기. 대상: 선택 채널 train/test `.npy`, `labeled_anomalies.csv` | `src/data.py`, `data/HASHES.json` |
| V1-3 | Bunkering `train_consumption.py` 분할부 | 시간순 분할 + 경계 간격 + 겹침 `assert` | train/test는 이미 파일로 나뉘어 있음 → **train 끝 일부를 검증 구간**으로 떼고, 간격(`gap`)은 창 길이만큼. test 파일은 건드리지 않음 | `src/data.py` |
| V1-4 | FabGuard `spc.individuals_report` | 기준 구간으로 중심선·이동범위·관리한계를 계산해 **고정**, 이후 값 판정, 결측은 `unknown`, 기준 구간 내 한계 밖 값 → `baseline_review_required` | ① 입력: CSV(timestamp,value) → `.npy` 첫 열 + **시점 인덱스**(실제 시각 없음) ② 기준 구간 = V1-3의 train 부분 ③ 판정 결과를 **연속 구간(시작·끝)**으로 묶는 함수 추가 ④ `baseline_review_required`가 뜨면 멈추지 말고 기록 후 진행할지 규칙 결정 필요 | `src/baselines/spc.py` |
| V1-5 | telemanom `detector.evaluate_sequences` (Apache 2.0) | "한 시점이라도 겹치면 포착, 실제 구간당 1회" 규칙 | 새로 작성하고 출처 명시. **탐지 지연** 추가(시점 인덱스). FP·FN 분모를 결과에 같이 기록 | `src/metrics.py` |
| V1-6 | Judicial `baseline_guard.freeze_baseline` | 설정을 정렬된 JSON으로 만들어 SHA-256 고정, 이후 비교 | 필수 필드를 `config`, `data_hashes`, `label_row`로 교체. **test 평가 직전에** 고정 파일 생성, 평가 후 일치 확인 | `src/freeze.py`, `results/<run>/frozen.json` |
| V1-7 | Judicial `quarantine` | 지우지 않고 격리 + 이력 | 라벨 CSV `P-2` 중복 2행을 격리 기록. V1에서는 P-2를 첫 채널로 고르지 않아 영향 회피 | `data/label_issues.json` |
| V1-8 | Bunkering `train_consumption.py`의 `metadata` | 결과 JSON에 입력 해시·버전·`limitations` 동봉 | 한계 문구를 텔레메트리용으로: 익명 채널, 시각 없음, test 기준 스케일링, 원 연구와 임계값 방식 다름 | `results/<run>/summary.json` |

**V1에서 대조군:** Bunkering의 `train_mean` 대조군처럼, "아무 구간도 탐지하지 않음"과 "test 전체를 하나의 이상 구간으로 탐지" 두 극단을 같은 지표로 함께 보고한다. 그래야 지표가 어느 쪽으로 쉽게 부풀려지는지 보인다.

---

## V2 — 비교 방법 추가, 여러 채널

| # | 원본 | 그대로 | 바꿀 부분 | 위치(예정) |
|---|---|---|---|---|
| V2-1 | Bunkering `condition_diagnostics.support` | train만으로 표준화 → k-NN 거리 → 95% 분위 임계값, 참조는 6개마다 1개로 솎아 자기상관 완화 | 특징을 선박 변수 → **최근 w개 값의 창 벡터**로. 임계값 분위는 검증 구간 오경보로만 조정 | `src/baselines/knn_window.py` |
| V2-2 | Bunkering `evaluation/contract.validate_fair_evaluation_plan` | 방법마다 사례가 다르면 실행 거부 | 사례 서명을 `(seed, episode, env_config)` → `(channel_id, split_config_hash)`로 | `src/plan.py` |
| V2-3 | Judicial `cohort_freeze` | 대상 목록을 결과 전에 잠그고, 바꾸면 새 버전·사유 | 대상: 평가할 채널 ID 목록과 선정 규칙(예: "SMAP 전체" 또는 "point 1구간 채널") | `results/cohort.json` |
| V2-4 | Adversarial `verification/iterative_result_audit` | 일부만 실행된 결과는 전체 PASS 불가 | 채널 일부 실패 시 "N/82 완료"로만 보고 | `src/audit.py` |
| V2-5 | FabGuard `advanced_evaluation.bootstrap_top_k_interval` | 부트스트랩 구간 | 표본 단위를 생산 건 → **채널** 단위로(채널 내부 시점은 자기상관 때문에 재표집 안 함) | `src/metrics.py` |
| V2-6 | TriGuard `scripts/source_manifest` | 검토한 기준 대비 파일 추가·삭제·변경 판정 | 대상을 `data/*.csv` → 내려받은 `.npy`·라벨 파일 | `src/data.py` |

---

## V3 — 웹 화면과 결과 공개

| # | 원본 | 그대로 | 바꿀 부분 |
|---|---|---|---|
| V3-1 | FabGuard `scripts/build_web_evidence.py` + `web/` | 저장된 결과만 읽어 웹 데이터 생성, 모델 재실행 없음 | 채널·방법 선택, 시계열 + 실제 구간 음영 + 탐지 구간 |
| V3-2 | AU Job Atlas `src/data/schema.ts`(zod), `npm run check` | 웹 데이터 스키마 검증, 생성 파일 불일치 시 실패 | 스키마를 공고 → 채널 결과로 |
| V3-3 | Judicial `comparison_guard` | "비교 불가"와 "비교 가능하지만 다름" 분리 | 원 연구 수치(정밀도 87.5% 등)는 "임계값 방식이 달라 직접 비교 불가"로 표시 |
| V3-4 | Adversarial `audit/cross_doc`, Judicial `claim_audit` | 문서 수치 ↔ 결과 파일 교차 확인 | README·웹의 수치를 `summary.json`과 대조 |

---

## 가져오지 않는 것

- TriGuard `ml_engine.detect_anomaly_regions`: 평가 대상 자신의 평균·표준편차로 기준을 잡음 → 시계열에선 test 정보 누수. **반면교사**로만 기록
- TriGuard `upload_security`: 웹에서 사용자 업로드를 받지 않는 한 불필요
- Bunkering DQN, Adversarial 공격·방어, Judicial 법률 커넥터·전문가 평가, FabGuard Fledge·태양광 수집기

## 첫 세션 전에 정할 것

1. V1-4의 `baseline_review_required` 처리 규칙 (train 안에 한계 밖 값이 있을 때)
2. 검증 구간 비율과 `gap` 길이
3. 탐지 구간을 만들 때 최소 길이를 둘지 (두면 오경보는 줄지만 짧은 point 이상을 놓침)
