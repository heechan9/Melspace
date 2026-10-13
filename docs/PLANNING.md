# MelSpace — 기획 메모 (2026-10-13)

> 상태: 기획 단계 · 구현 및 실험 미실행. 의존성 설치·데이터 다운로드·학습·웹 구현 없음.

## 1. 확인 근거와 접근 상태

| 자료 | 상태 | 확인한 내용 |
|---|---|---|
| telemanom README, `config.yaml`, `labeled_anomalies.csv`, `channel.py`, `errors.py`, `detector.py` | ✅ 직접 확인 | 데이터 형식·전처리, 기본 파라미터(l_s=250 등), 동적 임계값 계산, 구간 단위 TP/FP/FN 규칙, Apache 2.0 |
| Kaggle 데이터셋 | ✅ 메타데이터·파일 목록 확인 (다운로드 안 함) | 272,154,634 bytes, v1 2022-10-16, 라이선스 표기 "Data files © Original Authors", `data/data/2018-05-19_15.00.10/models/*.h5` 포함 → telemanom `config.yaml`의 `use_id`와 일치 |
| Kaggle의 train/test `.npy` 전체 목록 | ⚠️ 일부만 확인 | 파일 목록 첫 페이지만 조회. train/test 경로 구성은 다운로드 후 확인 |
| 원래 NASA/JPL 직접 배포 경로 (과거 S3) | ❌ 확인 불가 | 현재 telemanom README는 Kaggle만 안내. 공식 대체 경로를 찾지 못함 |
| arXiv 1802.04431 | ✅ 초록·서지 확인 | KDD 2018, DOI 10.1145/3219819.3219845. 본문(라벨 정의 상세)은 미열람 |
| NTRS 20210008188 | ✅ 확인 | JPL, Preprint, Distribution Public, Copyright "Other", 다운로드 파일 없음 |
| NASA PCoE 저장소 | ✅ 확인 | 21개 데이터셋. C-MAPSS(6·17번)는 **시뮬레이션** 데이터. 일부는 현재 직접 다운로드 불가. 첫 범위에서 제외 |
| 위키독스 300834 | ✅ 확인 | VGG16 전이학습 이미지 분류. 구성 순서(데이터→전처리→모델→평가→시각화·혼동행렬)만 차용 |
| 기존 저장소 6개 + 포크 | ✅ README·파일 구조 확인 / ⚠️ 함수 단위 미확인 | 아래 3절 |
| AU Job Atlas | ✅ 비공개 저장소, 이번 세션에서 읽기 전용 클론 | README·파일 구조 확인 |

## 2. 첫 버전 범위

- 채널 1개, 방법 1개 (단순 기준선)
- 평가: 구간 단위 TP/FP/FN, 포착률, 정밀도, 탐지 지연(시점 인덱스)
- 시각화: 정적 그래프 1장
- 제외: LSTM·동적 임계값 재현, 여러 데이터셋, 웹 화면, 배포

채널 후보는 다음 세션에서 함께 고름. 참고용 예시(라벨 CSV에서 확인):
- `A-1` (SMAP): point 이상 1구간 — 첫 그래프로 단순
- `P-1` (SMAP): contextual 이상 3구간 — 단순 기준선이 어려울 수 있음
- `T-1` (SMAP): point 1 + contextual 1 — 유형 비교에 적합

## 3. 기존 프로젝트에서 가져올 것

> 6개 프로젝트 모두 **코드를 직접 열어 확인**함(2026-10-13, git 클론 기준). 재사용 비율은 수치로 만들지 않음.
> GitHub 웹 페이지가 오래된 캐시를 보여준 사례가 있어(bunkering 웹 3커밋 ↔ 실제 301커밋, judicial 29 ↔ 348) 모든 판단은 git 기준.

### 3-1. 첫 버전에 직접 쓰는 것

| 필요한 기능 | 가져올 곳 | 확인한 내용 |
|---|---|---|
| **단순 기준선 탐지기** | FabGuard `src/fabguard/spc.py` | 앞부분(기준 구간)만으로 중심선·이동범위를 계산해 관리한계(±3σ 추정)를 **고정**하고, 이후 값을 `within/above/below/unknown`으로 판정. 기준 구간 자체에 한계 밖 값이 있으면 `baseline_review_required`. 결측은 정상으로 처리하지 않고 `unknown`. 한계 목록을 결과에 동봉 → **첫 기준선 후보로 가장 직접적** |
| 거리 기반 범위 판정 | Bunkering `research/fuel_source_review/condition_diagnostics.py` | train만으로 표준화·k-NN 거리 95% 분위 임계값 설정, test 임계값 조정 없음 → 두 번째 비교 방법 후보 |
| 누수 없는 시간 분할 | Bunkering `train_consumption.py`, FabGuard `data.time_holdout` | 시간순 train/val/test + 경계 간격 제거, 같은 시각이 경계를 넘지 않게 조정, 겹침 assert |
| 검증셋으로만 선택 + 대조군 | Bunkering `train_consumption.py` | 후보는 검증 MAE로만 선택, `train_mean` 대조군과 같이 보고 |
| 결과 보기 전 설정 고정 | FabGuard `config.py`, Judicial `baseline_guard.py` | 실험 설정을 frozen dataclass로 먼저 선언 / 설정 JSON 해시로 고정 후 변경 시 `BASELINE_CHANGED` |
| 원자료 무결성 | FabGuard `data.py`(`OFFICIAL_HASHES`, `validate_raw_files`) | 내려받은 파일 SHA-256을 고정값과 대조 |
| 라벨 이상 처리 | Judicial `quarantine.py` | `P-2` 중복을 지우지 않고 격리·이력 기록 |
| 결과에 근거·한계 동봉 | Bunkering·FabGuard 공통 | 결과 JSON에 입력 해시, 버전, `limitations` 목록 |

### 3-2. 비교·확장 단계에서 쓸 것

| 필요한 기능 | 가져올 곳 | 확인한 내용 |
|---|---|---|
| 방법 간 공정 비교 강제 | Bunkering `evaluation/contract.py` | 방법마다 사례(seed·설정)가 다르면 실행 거부 |
| 평가 대상 채널 고정 | Judicial `cohort_freeze.py` | 대상 목록을 결과 전에 해시로 잠그고, 바꾸려면 새 버전·사유 필요 |
| 원 연구 수치와 비교 구분 | Judicial `comparison_guard.py` | "비교 불가"와 "비교 가능하지만 다름" 분리 |
| 분포 변화 점검 | FabGuard `advanced_evaluation.py` (`population_stability_index`, `walk_forward_slices`, `bootstrap_top_k_interval`) | train↔test 분포 차이(PSI), 확장 창 walk-forward, 부트스트랩 구간 |
| 부분 실행을 완료로 보고하지 않기 | Adversarial `verification/iterative_result_audit.py` | 일부만 실행된 결과는 전체 PASS가 될 수 없음 → 82채널 중 일부 실패 시 |
| 중복·누수 감사 | Adversarial `verification/local_split_audit.py` | 분할 간 완전 중복 탐지(읽기 전용) |
| 로컬 재현 vs 외부 검증 구분 | Adversarial `verification/status_registry.py`, `stage_b_run.py` | 해시로 묶인 근거에서 상태 생성, 환경 간 저장 라벨 비교 |
| 문서 수치 교차 감사 | Adversarial `src/adversarial_ai/audit/cross_doc.py`, `paper_claims.py`; Judicial `claim_audit.py` | README 수치가 결과 CSV·JSON과 다르면 실패 |
| 원본 변경 감지 | TriGuard `scripts/source_manifest.py` | 검토한 기준 대비 파일 추가·삭제·변경 판정 |
| 업로드 방어 | TriGuard `modules/upload_security.py` | 웹에서 사용자 CSV를 받을 경우 크기·바이너리 차단 |
| 웹 데이터 생성·검증 | AU Job Atlas `scripts/build-app-data.ts`, `src/data/schema.ts`(zod), `npm run check` | 결과 JSON을 스키마로 검증하고, 생성 파일이 원본과 다르면 실패 |
| 정적 웹 | FabGuard `scripts/build_web_evidence.py` + `web/`, AU Job Atlas Vite+TS | 저장된 결과만 보여주고 모델을 다시 돌리지 않는 구조 |

### 3-3. 쓰지 않는 것 (확인 후 제외)

- Bunkering의 DQN·강화학습 환경, Adversarial의 공격·방어 코드, Judicial의 법률 커넥터·전문가 평가 — 문제 유형이 다름
- TriGuard `ml_engine.detect_anomaly_regions`: z-score 이상 탐지지만 **평가 대상 자신의 평균·표준편차로 기준을 잡음**(14개 지역 단면). 시계열에 그대로 쓰면 test 정보가 기준에 섞이므로 반면교사로만 참고
- FabGuard의 Fledge·태양광 커넥터: 수집 계약 설계는 참고할 만하나 이번 데이터(정적 `.npy`)에는 불필요

### 적대적 AI 저장소 반영 상태 (git 참조 기준, 2026-10-13 조회)

| 저장소 | main 최신 | 비고 |
|---|---|---|
| 공식 `th0oel/AdversarialAI_Security` | `2e9bf4a` (2026-10-04, PR #22 병합) | |
| 개인 포크 `heechan9/AdversarialAI_Security` | `9b64b8e` (2026-10-07, PR #80 병합) | 공식 main에 없는 커밋 97개, 포크 main에 없는 공식 커밋 49개 → **갈라진 상태** |

- 포크에만 있는 최근 작업 예: black-box Square·SurFree 결과(PR #76~77), CPU 회귀·SHA 결합 리뷰(PR #79), post-merge 감사 보완(PR #80).
- 공식에만 있는 49개는 PR 병합 커밋일 가능성이 높으나 내용 비교는 하지 않음.
- GitHub 웹 페이지 조회 결과는 포크를 "1 commit, README만"으로 보여줬지만 git 참조와 다름 → 웹 캐시로 판단하고 git 참조를 기준으로 함.

## 4. 결정이 필요한 것 (다음 세션)

1. Kaggle 계정·API 키로 받을지 (전체 약 272 MB)
2. 첫 채널 선택
3. 라벨 CSV `P-2` 중복 처리 규칙
4. 검증 구간 비율 (train 끝 몇 %)
5. 저장소 이름 확정과 공개 여부, 코드 라이선스
