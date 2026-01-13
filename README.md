# task1-2-report


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






## Task 2

### Process of Collecting Smoothed IMU Data
To build a regression model under a **slow-moving** condition, IMU data was recorded while performing slow and smooth motions.  
The log file (`imu_smooth.csv`) contains:

- **Inputs (features):** `ax`, `ay`, `az`, `gx`, `gy`, `gz`
- **References (ground-truth):** `roll`, `pitch`, `yaw` (from Device Fusion)
- **Time:** `timestamp`

In Task 2 modeling, the data was used as follows:
- **Roll/Pitch regression inputs:** mainly `ax`, `ay`, `az` (3-axis acceleration)
- **Target (label):** `roll` or `pitch` (Fusion output)
- Some models (e.g., **SVR**, **FFNN/MLP**) are sensitive to feature scaling, so **StandardScaler** normalization was applied.


---

### What Is the R² Metric?
**R² (coefficient of determination)** measures how well a regression model explains the variance of the ground-truth data.

- **y:** ground-truth (Device Fusion roll/pitch)
- **ŷ:** model prediction
- **ȳ:** mean of ground-truth

Interpretation:
- **R² = 1:** perfect prediction
- **R² = 0:** no better than predicting the mean
- **R² < 0:** worse than predicting the mean


---

### Regression Algorithms and Estimation Methods

#### 1) Physics-based Analytic (Trigonometry) — Baseline
For slow motions, acceleration mostly reflects the gravity vector, so roll/pitch can be computed using trigonometric relations.

- **Roll**
  - `roll = atan2(ay, az)`
- **Pitch**
  - `pitch = atan2(-ax, sqrt(ay^2 + az^2))`

**Pros:** no training, minimal computation, physically interpretable  
**Cons:** error can increase during fast motions (large linear acceleration)

#### 2) Linear Regression
Models the relationship between inputs (`ax, ay, az`) and outputs (roll/pitch) as a linear combination.

**Pros:** fast, interpretable (coefficients)  
**Cons:** cannot fully represent nonlinear relations such as `atan2`

#### 3) Polynomial Regression (Non-linear, degree = 3)
Expands inputs into polynomial terms to capture nonlinearity.

**Pros:** more flexible approximation of nonlinear relations  
**Cons:** higher risk of overfitting and reduced interpretability due to feature expansion

#### 4) Decision Tree Regression
Predicts by splitting data using threshold-based rules.

**Pros:** captures nonlinearity and interactions; rule-based interpretability  
**Cons:** outputs often become piecewise-constant (step-like); prone to overfitting

#### 5) Random Forest Regression
Ensembles multiple decision trees and averages their predictions.

**Pros:** more stable and less overfitting than a single tree  
**Cons:** reduced interpretability; increased computation

#### 6) SVM Regression (SVR, RBF Kernel)
Kernel-based nonlinear regression.

**Pros:** strong nonlinear approximation ability  
**Cons:** scaling is essential; requires tuning (`C`, `gamma`, `epsilon`)

#### 7) FeedForward Neural Network (MLP/FFNN)
Learns a nonlinear mapping using a multilayer neural network.

**Pros:** very high expressive power  
**Cons:** requires scaling and hyperparameter tuning; training reproducibility must be managed  
In this experiment, **StandardScaler** was applied and a **3 → 64 → 32 → 1** architecture was used.

#### 8) Accel-only vs Gyro-only vs Complementary Filter (Estimation Comparison)
Although not regression models, these are representative IMU angle estimation approaches:

- **Accel-only:** no drift, but can be corrupted during fast motion
- **Gyro-only:** smooth output, but drift accumulates over time due to integration
- **Complementary filter:** combines gyro smoothness with accel-based drift correction


---

### Performance Comparison of Different Methods

#### 1) Pitch Results
| Method | R² (Pitch) |
|---|---:|
| Analytic (Trigonometry) | 0.9956 |
| Linear Regression | 0.9956 |
| Polynomial Regression (deg 3) | 0.9962 |

Interpretation:
- Under slow-motion conditions, **analytic** and **linear regression** achieved similarly high performance.
- Polynomial regression shows a **small improvement** (0.9956 → 0.9962), but the gain is minor.

#### 2) Roll Results (Physics / Filtering Perspective)
| Method | R² (Roll) |
|---|---:|
| Accel-only (Trigonometry) | 0.9998 |
| Gyro-only (Integration) | 0.7501 |
| Complementary Filter (α = 0.96) | 0.9998 |

Interpretation:
- **Gyro-only** performance drops significantly due to drift (R² = 0.7501).
- **Complementary filtering** corrects drift and maintains top performance comparable to accel-only.

#### 3) Roll Results (ML Regression Models)
| Model | R² (Roll) |
|---|---:|
| Analytic (Trigonometry) | 0.9998 |
| Linear Regression | 0.9975 |
| Polynomial Regression (deg 3) | 0.9998 |
| Decision Tree (max_depth = 3) | 0.9888 |
| Random Forest (n = 100, depth = 5) | 0.9995 |
| SVR (RBF, scaled) | 0.9997 |
| FFNN/MLP (scaled, 64-32) | 0.9998 |

Interpretation:
- In slow-motion conditions, the **analytic baseline** already achieves near-ceiling performance (0.9998), and most models approach it.
- **Linear regression** is lower because it approximates the nonlinear `atan2` relationship linearly.
- **Decision tree** is lower due to step-like approximation and limited depth.
- **RF / SVR / FFNN / Polynomial** perform very well, but do not clearly outperform the analytic baseline.


---

### Which Algorithm Is the Best?

#### Conclusion under this experiment setting (slow-moving)
- **Overall best baseline (most efficient): Analytic formula (Trigonometry)**
  - Roll: **R² = 0.9998** (best)
  - Pitch: **R² = 0.9956** (top-level)
  - No training required, minimal computation, physically interpretable

- **Best sensor-fusion approach: Complementary filter**
  - Solves gyro drift while maintaining 최고 성능 on roll (**R² = 0.9998**)

- **Best learned model (ML perspective)**
  - Roll: **Polynomial (deg 3)** / **FFNN** tie with analytic (**R² = 0.9998**)
  - Pitch: **Polynomial (deg 3)** is best (**R² = 0.9962**)
  - However, the improvement is small and comes with additional training/tuning cost
