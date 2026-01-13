# task1-2-report

<Task1>

Process of collecting raw IMU data
1) 수집 장치 및 데이터 형식

IMU 센서로부터 다음 신호를 측정하여 CSV 형태로 저장:
가속도: Acc_X, Acc_Y, Acc_Z (단위: g)
자이로: Gyro_X, Gyro_Y, Gyro_Z (단위: deg/s)
타임스탬프: Timestamp (ms)

2) IMU_SD_serialtrigger.ino 기반 수집 흐름

IMU 초기화 및 센서 설정(샘플링 주기 설정 포함)
시리얼 트리거(버튼/명령 등)로 기록 시작/세션 구분
일정 주기로 (timestamp + accel + gyro)를 읽고 SD/로그 파일에 저장
새로운 세션 시작 시 --- NEW SESSION --- 같은 마커가 들어갈 수 있음

3) 본 분석 코드에서 확인한 샘플링 주파수

raw_imu_log.csv의 timestamp 차이를 이용해 평균 샘플링 주기(avg_dt)를 계산하고,
샘플링 주파수 fs = 1000/avg_dt로 추정
결과: Estimated Sampling Frequency ≈ 72.09 Hz
즉, 평균적으로 약 1/72초 간격으로 IMU 데이터가 기록됨



Why perform smoothing, filtering of raw IMU data?
1) 스무딩(SMA)의 필요성

raw IMU에는 센서 잡음, 미세 떨림, 순간 스파이크가 포함됨
이동평균으로 고주파 잡음을 줄이면:
전체 추세(trend) 파악이 쉬워지고
feature(평균/분산/최대치)가 안정적으로 계산됨

2) 필터링(High-pass)의 필요성: 중력 제거

가속도에는 항상 중력(약 1g)이 포함되어 DC 성분처럼 작용
중력 성분이 크면 움직임에 의한 변화가 상대적으로 묻힐 수 있음
high-pass filter로 중력/저주파 성분을 줄이면:
동작/충격/진동 같은 동적 이벤트가 더 선명해짐
ZCR 같은 “부호 변화 기반 특징”이 의미 있게 계산됨




What is preformed in feature extraction?
1) SMV 기반 특징 (동작 강도/충격 탐지에 유용)

SMV_Mean: 일정 구간의 평균 활동 강도
SMV_Std: 활동 변화량/불규칙성 (활동이 클수록 분산 증가)
Max_SMV: 충격(impact) 후보 지표 (낙상/큰 충격 시 peak 발생 가능)

2) ZCR 기반 특징 (진동/활동성 지표)

중력 제거 신호에서 0-crossing 횟수를 계산
빠른 진동/활발한 움직임일수록 ZCR 증가 가능




Comment on the analysis performed? Use and need?
1) 본 분석의 유용성

raw IMU를 그대로 보면 노이즈/중력 영향 때문에 이벤트를 정량적으로 구분하기 어려움
스무딩/중력 제거 후 특징을 추출하면:
언제 강한 이벤트가 있었는지(Max_SMV 피크), 어느 구간이 활동이 큰지(SMV_Std 상승), 진동/움직임이 얼마나 잦은지(ZCR 변화)를 시간축에서 명확히 비교할 수 있음

2) 활용 예시

낙상/충격 탐지: Max_SMV 피크 + SMV_Std 상승 구간 탐색
활동 인식: SMV 평균/표준편차 + ZCR을 조합하여 동작 상태 분류
후속 ML 입력(feature vector) 생성: window 단위 특징을 학습 데이터로 사용 가능

3) 한계 및 개선 포인트

현재 특징은 윈도우 기반 단순 통계(Mean/Std/Max/ZCR) 중심이므로,
동작 종류가 복잡해지면 추가 특징(에너지, FFT 대역 파워, jerk 등)이 필요할 수 있다.
또한 분석 코드가 train/test 분리 없이 전체 구간을 한 번에 처리하므로, 실제 모델 학습을 한다면 구간 분할 및 라벨링(ground truth 정의)이 필요하다.




## Task 1

### Process of Collecting Raw IMU Data

#### 1) Data acquisition device and format
The IMU sensor measures the following signals and stores them in a CSV file:
- **Acceleration:** `Acc_X`, `Acc_Y`, `Acc_Z` (unit: **g**)
- **Gyroscope:** `Gyro_X`, `Gyro_Y`, `Gyro_Z` (unit: **deg/s**)
- **Timestamp:** `Timestamp` (unit: **ms**)

#### 2) Data collection flow based on `IMU_SD_serialtrigger.ino`
- Initialize the IMU and configure sensor settings (including sampling interval)
- Start logging / separate sessions using a serial trigger (e.g., button or command)
- Periodically read `(timestamp + accel + gyro)` and save to SD card / log file
- When a new session starts, a marker such as `--- NEW SESSION ---` may be inserted

#### 3) Sampling frequency verified in this analysis
Using timestamp differences in `raw_imu_log.csv`, the average sampling interval (`avg_dt`) is computed, and the sampling frequency is estimated as:
- **fs = 1000 / avg_dt**
- Result: **Estimated Sampling Frequency ≈ 72.09 Hz**
- In other words, IMU data is recorded approximately every **1/72 seconds** on average


---

### Why Perform Smoothing and Filtering on Raw IMU Data?

#### 1) Why smoothing (SMA) is needed
Raw IMU data typically includes sensor noise, small vibrations, and occasional spikes. Applying a moving average helps reduce high-frequency noise, which:
- makes the overall trend easier to observe
- stabilizes feature computation (e.g., mean/variance/peaks)

#### 2) Why high-pass filtering is needed (gravity removal)
Acceleration signals always include gravity (~1g), which behaves like a DC (low-frequency) component. If gravity dominates, motion-related changes can be less visible. By applying a high-pass filter:
- gravity / low-frequency components are reduced
- dynamic events (movement, impact, vibration) become clearer
- sign-change based features such as **ZCR** become more meaningful


---

### What Is Performed in Feature Extraction?

#### 1) SMV-based features (useful for motion intensity / impact detection)
- **SMV_Mean:** average activity intensity within a window
- **SMV_Std:** variability / irregularity of motion (higher activity → higher variance)
- **Max_SMV:** indicator of potential impacts (e.g., fall or strong shock may produce peaks)

#### 2) ZCR-based features (indicator of vibration / activity)
- Compute the number of **zero crossings** from the gravity-removed signal
- Faster vibration or more active movement can lead to higher ZCR


---

### Comments on the Analysis: Use and Need

#### 1) Usefulness of this analysis
Raw IMU signals are hard to interpret quantitatively due to noise and gravity effects. After smoothing and gravity removal, extracted features allow clearer time-based comparisons such as:
- **when strong events occurred** (peaks in `Max_SMV`)
- **which segments had high activity** (increase in `SMV_Std`)
- **how frequent vibration/movement was** (changes in `ZCR`)

#### 2) Example applications
- **Fall / impact detection:** detect segments with `Max_SMV` peaks and increased `SMV_Std`
- **Activity recognition:** classify motion states using a combination of SMV statistics and ZCR
- **ML input generation:** use window-based features as feature vectors for model training

#### 3) Limitations and possible improvements
Current features are mostly simple window-based statistics (Mean/Std/Max/ZCR). If motion classes become more complex, additional features may be needed (e.g., signal energy, FFT band power, jerk).  
Also, the current analysis processes the full signal without train/test separation. For real model training, proper segmentation and labeling (ground-truth definition) are required.




<Task2>

Process of collecting smooth IMU data

slow-moving action을 수행하며 IMU 데이터를 기록하였다.
로그 파일(imu_smooth.csv)에는 다음 항목이 포함된다.
입력(feature): ax, ay, az, gx, gy, gz
정답(reference): roll, pitch, yaw (Device Fusion 결과)
시간: timestamp

Task 2 모델링에서는 다음과 같이 사용했다.
Roll/Pitch 회귀 입력(feature): 주로 ax, ay, az (가속도 3축)
Target(정답): roll 또는 pitch (Fusion 값)
일부 모델(SVR, FFNN/MLP)은 입력 스케일에 민감하므로 StandardScaler로 표준화를 적용하였다.



What is R² metric?

R²(결정계수)는 회귀 모델이 정답 데이터의 변동을 얼마나 잘 설명하는지 나타내는 지표이다.

y: ground-truth (Device Fusion roll/pitch)
𝑦^: 모델 예측값
𝑦ˉ: 정답 평균
해석:
R² = 1: 완벽한 예측
R² = 0: 평균값 예측과 동일 수준
R² < 0: 평균보다도 나쁨



Description of different regression algorithms (및 추정 방법)
1) Physics-based analytic (Trigonometry) — baseline

느린 동작에서는 가속도가 대부분 중력벡터를 반영하므로 roll/pitch를 삼각함수로 계산할 수 있다.
Roll:

𝑟𝑜𝑙𝑙=arctan⁡2(𝑎𝑦,𝑎𝑧)

Pitch:
𝑝𝑖𝑡𝑐ℎ=artan2(−𝑎𝑥,𝑎𝑦2+𝑎𝑧2)

특징: 학습 불필요, 계산량 최소, 해석 가능. 다만 빠른 동작(선형가속 큼)에서는 오차가 커질 수 있다.

2) Linear Regression

입력(ax, ay, az)과 출력(roll/pitch)의 관계를 선형 결합으로 모델링한다.
장점: 빠르고 해석 가능(계수 확인 가능)
단점: atan2와 같은 비선형 관계를 완벽히 표현하기 어렵다.

3) Polynomial Regression (Non-linear Regression, degree=3)
입력을 다항 항으로 확장하여 비선형성을 모델링한다.
장점: 비선형 관계를 더 유연하게 근사
단점: feature 수 증가로 과적합 위험/해석성 저하

4) Decision Tree Regression

feature 임계값 기반 분기로 데이터를 구간화하여 예측한다.
장점: 비선형/상호작용 표현 가능, 규칙 기반 해석 가능
단점: 출력이 piecewise constant(계단형)으로 나타나기 쉬움, 과적합 가능

5) Random Forest Regression

여러 decision tree를 앙상블로 결합(평균)하여 안정적인 예측을 만든다.
장점: 단일 트리보다 과적합 완화, 성능 안정적
단점: 해석성 감소, 연산량 증가

6) SVM Regression (SVR, RBF kernel)

커널 기반 비선형 회귀.
장점: 비선형 근사 능력 우수
단점: 스케일링 필수, C/gamma/epsilon 튜닝 필요

7) FeedForward Neural Network (MLP/FFNN)

다층 신경망으로 비선형 함수를 학습한다.
장점: 매우 강한 표현력
단점: 스케일링 및 하이퍼파라미터 튜닝 필요, 학습/재현성 관리 필요
(본 실험에서는 StandardScaler를 적용하고 3→64→32→1 구조 사용)

8) Accel-only vs Gyro-only vs Complementary filter (추정 방법 비교)

회귀 모델은 아니지만, IMU 각도 추정의 대표적 접근 비교를 포함했다.

Accel-only: drift 없음, 빠른 동작에서 오염 가능
Gyro-only: 매끄럽지만 적분 drift 누적
Complementary: 자이로의 매끄러움 + 가속도로 drift 보정



Compare performance of different regression algorithms
1) Pitch 결과
Method	R² (Pitch)
Analytic (Trigonometry)	0.9956
Linear Regression	0.9956
Polynomial Regression (deg 3)	0.9962

해석:
slow-moving 조건에서 analytic과 linear가 동일 수준으로 매우 높았다.
Polynomial(deg3)은 **미세한 개선(0.9956→0.9962)**을 보였으나 개선폭이 작다.

2) Roll 결과 (Physics/Filtering 관점)
Method	R² (Roll)
Accel-only (Trigonometry)	0.9998
Gyro-only (Integration)	0.7501
Complementary Filter (α=0.96)	0.9998

해석:
Gyro-only는 drift 누적으로 성능이 크게 하락(R²=0.7501).
Complementary filter는 drift를 보정하여 accel-only 수준의 최고 성능을 유지했다.

3) Roll 결과 (ML 회귀 모델 비교)
Model	R² (Roll)
Analytic (Trigonometry)	0.9998
Linear Regression	0.9975
Polynomial Regression (deg 3)	0.9998
Decision Tree (max_depth=3)	0.9888
Random Forest (n=100, depth=5)	0.9995
SVR (RBF, scaled)	0.9997
FFNN/MLP (scaled, 64-32)	0.9998

해석:
slow-moving 조건에서는 analytic이 이미 최상급 성능(0.9998)을 보여 대부분 모델이 이를 “근접하게” 따라간다.
Linear는 비선형 atan2 관계를 선형으로 근사하여 성능이 상대적으로 낮다(0.9975).
Decision Tree는 계단형 근사와 깊이 제한으로 성능이 더 낮다(0.9888).
RF/SVR/FFNN/Poly는 매우 높지만, analytic 대비 뚜렷한 우위는 없다.




Which algorithm is the best?
1) 본 실험 조건(slow-moving)에서의 결론

Overall best baseline (가장 효율적): Analytic formula (Trigonometry)
Roll: R²=0.9998 (최고)
Pitch: R²=0.9956 (최고 수준)
학습 불필요, 계산량 최소, 물리적으로 해석 가능

Best fusion approach (센서 관점): Complementary filter
Gyro-only drift 문제를 해결하면서 roll에서 최고 성능(R²=0.9998)을 유지

Best learned model(ML 관점):
Roll: Polynomial(deg3) / FFNN가 analytic과 동률(0.9998)
Pitch: Polynomial(deg3)가 최고(0.9962)
단, 향상 폭이 작고 학습/튜닝 비용이 추가됨
