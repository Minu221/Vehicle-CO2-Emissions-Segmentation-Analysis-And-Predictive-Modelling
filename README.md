# Vehicle CO2 Emissions: Segmentation Analysis And Predictive Modelling

This project looks at vehicle CO2 emissions using technical specs, based on the EPA Fuel Economy Guide dataset. It combines two methods — clustering (to find groups) and regression (to predict a number) — and tests each design choice with data instead of just guessing.

---

## Problem Statement

Before a vehicle gets official EPA emissions testing, manufacturers, regulators, and buyers often want a fast way to guess its CO2 output just from basic specs — engine size, number of cylinders, drivetrain, fuel type, and vehicle class.

This project answers three questions using real vehicle data:

1. **Segmentation:** What natural groups of vehicles exist in the market today, based only on technical specs — and how does CO2 output differ between them?
2. **Prediction:** Given a vehicle's specs, how much CO2 (g/mile) is it likely to emit?
3. **Reliability:** How much can we trust that prediction — and for which vehicles should we *not* trust it?

The project is split into **two separate parts** — clustering and regression — instead of assuming one must feed into the other. Whether combining them actually helps prediction is tested directly (see Appendix), not just assumed.

---

## Dataset

- **Source:** [EPA Fuel Economy Guide](https://www.fueleconomy.gov/feg/download.shtml) (`all_alpha_25.xlsx`), U.S. Environmental Protection Agency.
- **Raw size:** 2,492 rows × 18 columns, one row per certified vehicle setup.
- **Target column:** `Comb CO2` — combined city/highway CO2 emissions (g/mile).

| Feature | Description |
|---|---|
| `Model` | Brand and model name |
| `Displ` | Engine size (liters) |
| `Cyl` | Number of cylinders |
| `Trans` | Transmission type and number of gears |
| `Drive` | Drive type (2WD / 4WD) |
| `Fuel` | Fuel type |
| `Veh Class` | EPA vehicle class |
| `Cert Region`, `Stnd`, `Stnd Description`, `Underhood ID` | Admin/certification info (removed, see below) |
| `Air Pollution Score`, `Greenhouse Gas Score`, `SmartWay` | EPA rating scores (removed, see below) |
| `City/Hwy/Cmb MPG` | Fuel economy numbers (removed, see below) |

---

## Data Cleaning

A few data quality problems were found and fixed before analysis:

- **Duplicate rows:** many rows described the exact same car, just tested under different rules (`Cert Region`/`Stnd`), for example, the same car tested under both Federal and California standards. These duplicate rows were removed, cutting the dataset from 2,492 to 1,217 rows.
- **Fake missing values:** `Displ` and `Cyl` used `-1` as a stand-in value (mostly for electric cars) instead of a real number. These were changed to `NaN`, then filled with `0` — safe to do because the `Fuel = Electricity` column already marks these cars separately.
- **Mixed-up column:** `Trans` (like `"Auto-6"`, `"SCV-7"`) held two pieces of information at once, so it was split into `Trans_Type` and `Num_Gears`. A `Num_Gears = 0` for CVT transmissions is a real value (these gearboxes have no fixed number of gears), not a missing one.
- **Broken target values:** `Comb CO2` used an `"x/y"` format for Flex-Fuel (Ethanol/Gas) cars. Each of these rows was split into two rows — one with the Gasoline number, one with the Ethanol number, instead of being deleted or averaged.

---

## Choosing Features & Checking for Leakage

Instead of using all 18 original columns, features were chosen carefully, for two different reasons:

**Removed — leaks the answer (confirmed with a chart):**
`Greenhouse Gas Score` has an almost perfect straight-line relationship with `Comb CO2` — it's basically CO2 rewritten as a score by EPA, not new information. `SmartWay` is a label built from cutoffs on this same score, so it was removed for the same reason.

**Removed — not a real spec of the car:**
`Air Pollution Score` only has a weak, scattered relationship with CO2 (not leakage in the strict sense), but it's a *rating* for other pollutants (like NOx), not a *spec* of the car itself, so it was left out to keep the feature list focused on pure technical specs.

**Removed — leaks through overlap:**
`City MPG`, `Hwy MPG`, `Cmb MPG` are almost a direct math conversion of CO2, so they were dropped too.

**Final feature list:** `Displ`, `Cyl`, `Num_Gears`, `Trans_Type`, `Drive`, `Fuel`, `Veh Class`.

---

## Method

### Preprocessing (shared by both parts)
1. Train/test split (80/20) done **before** any encoding or scaling.
2. `OneHotEncoder` fit only on the training set (`handle_unknown="ignore"` so rare categories seen only in test don't cause errors).
3. `StandardScaler` fit only on the training set for number-based features.
4. Both tools are only applied (`.transform()`, never re-fit) to the test set, this stops test data from leaking into the preprocessing step, a common mistake when mixing clustering and regression in one pipeline.

### Part 1 — Segmentation (K-means Clustering)
- Clustering only used **technical specs** — `Comb CO2` was left out of the clustering input, and only used afterward to describe what each group means.
- The best number of groups (`k`) was picked using the Elbow method and Silhouette score together. `k=2` had the best silhouette score, but was too simple to be useful; `k=5` was chosen because it's a strong local peak in silhouette score with a clear "elbow" in the inertia chart — a good balance between quality and detail.
- Group assignment: `KMeans.fit()` only on the training set, `.predict()` on the test set
### Part 2 — Prediction (Regression)
- Two models were trained and compared: **Linear Regression** (simple, easy to explain) and **Random Forest** (can capture more complex, non-straight-line patterns).
- The main model does **not** use the cluster label from Part 1 as a feature — this was tested separately as an extra experiment (see Appendix), not assumed to help by default.
- **Checking stability:** 5-fold cross-validation was used to check that the Random Forest's results hold up across different ways of splitting the data, instead of relying on just one train/test split (see Results).
- **On tuning:** a `RandomizedSearchCV` sweep was run to check for a better setup, but since cross-validation already showed strong, stable results, and tuning didn't meaningfully improve things, the simple default setup (`n_estimators=300`) was kept as the final model. Tuning is most useful when results look unstable or clearly need improving

---

## Results

### Part 1 — Five Vehicle Groups Found

| Cluster | Name | What it looks like | Average CO2 |
|:---:|---|---|---:|
| 0 | Compact CVT Vehicles | Small engines, ~81% CVT gearbox | 245.1 |
| 1 | 4WD Utility Vehicles | Mid-size engines, 100% 4WD, mostly SUVs/pickups | 372.8 |
| 2 | Zero-Emission Vehicles | ~99% electric | 0.0 |
| 3 | Compact 2WD Vehicles | Mid-size engines, 100% 2WD, mostly compact cars | 358.6 |
| 4 | Large-Engine High-Emission Vehicles | Biggest engines (~5.0L) and most cylinders (~8.4) | 519.5 |

**Key finding:** Cluster 1 and Cluster 3 have almost the same engine size and cylinder count, but different CO2 levels — a closer look showed the real difference is `Drive` (100% 4WD vs. 100% 2WD). This confirms that drivetrain matters a lot for emissions, even between cars with the same engine size.

### Part 2 — CO2 Prediction

| Model | RMSE (g/mile) | MAE | R² |
|---|---:|---:|---:|
| Linear Regression | 50.00 | 36.09 | 0.9353 |
| **Random Forest** | **36.73** | **22.10** | **0.9651** |

**Chosen model: Random Forest.** Its error (RMSE) is about 26% lower than Linear Regression's, which shows it picks up on patterns that involve more than one feature at once — for example, engine size affects CO2 differently for 2WD vs. 4WD cars — something a simple, additive Linear Regression can't capture.

**Making predictions easier to understand:** a raw CO2 number (like "271 g/mile") doesn't mean much on its own to most people. So each prediction is also labeled with a `Predicted_Emission_Level` — `Zero-Emission`, `Low`, `Moderate`, or `High` — based on where it falls compared to the training data. This gives a quick, plain-language read (e.g. "this car's predicted emissions are in the High group") alongside the exact number.

**Checking stability with cross-validation:** running the model 5 times on different data splits confirmed the result above wasn't just luck from one good split:

| Metric | Values across 5 splits | Average | Spread (std) |
|---|---|---:|---:|
| RMSE | 38.64 / 38.47 / 31.62 / 40.16 / 42.12 | 38.20 | 3.54 |
| R² | 0.961 / 0.954 / 0.974 / 0.959 / 0.959 | 0.9614 | 0.007 |

The average RMSE here (38.20) is close to the original test result (36.73), and the spread between splits is small (about 9% of the average), so the model's performance looks reliable, not just a lucky outcome. 

### Error Analysis — Where the Model Struggles

| Fuel Type | Average Error | Sample Count |
|---|---:|---:|
| **Gasoline/Electricity (PHEV)** | **65.7** | 13 |
| Gasoline | 27.7 | 163 |
| Ethanol | 7.6 | 1 |
| Diesel | 5.8 | 5 |
| Hydrogen | 4.9 | 1 |
| Electricity | 0.0 | 62 |

Plug-in Hybrid vehicles have an average error more than double that of regular Gasoline vehicles — a real pattern, not just random noise. This makes sense: a PHEV's combined CO2 depends heavily on battery size and how far it can drive on electricity alone, and neither of these is in the available specs. (Fuel types with only 1–5 samples are shown for completeness, but the numbers aren't reliable given how few examples there are.)

A related idea was also tested — that regular (non-plug-in) hybrid cars labeled simply as `"Gasoline"` in the EPA data (like the Ford Escape HEV) might also show higher error. This was **not confirmed** by the data (average error 23.5 vs. 22.0 for regular Gasoline cars, n=9). This suggests that regular hybrid assist has too small an effect on CO2 to matter much here, unlike the much bigger, unaccounted-for effect of PHEV electric range.

---

## Why This Matters in Practice

| Question | Answered by | Real-world use |
|---|---|---|
| Which car setups cause the most emissions? | Part 1 (Clustering) | R&D teams can focus engineering or electrification efforts on the highest-emission group (large engines, 4WD) instead of spreading resources evenly. Policymakers could design incentives around specific setups instead of one blanket rule per brand. |
| What will a new car's CO2 be, before it's officially tested? | Part 2 (Regression) | Manufacturers can get a quick, low-cost CO2 estimate straight from a design spec sheet, before paying for full EPA testing. Car comparison websites could show an estimated CO2 number for new models that don't have official ratings yet. |
| When should this estimate not be trusted? | Error Analysis | Any real use of this model should flag PHEV predictions as less reliable, since the error is about 2.4x higher for this group, using the number blindly here could lead to bad decisions, like in tax policy or a company's fleet emissions targets. |

---

## Additional Experiment: Does Clustering Improve Regression?

Based on an earlier idea that clustering could improve regression, the Part 1 cluster label was added as an extra feature to the Part 2 models, as a controlled test:

| Model | RMSE | MAE | R² |
|---|---:|---:|---:|
| Linear Regression (no cluster) | 50.00 | 36.09 | 0.9353 |
| Linear Regression (+ cluster) | 50.04 | 35.66 | 0.9352 |
| Random Forest (no cluster) | **36.73** | **22.10** | **0.9651** |
| Random Forest (+ cluster) | 37.55 | 22.30 | 0.9635 |

**Finding: adding the cluster label doesn't help — it slightly hurts Random Forest, and barely changes Linear Regression.** The cluster label is built entirely from features the models already have (`Displ`, `Cyl`, `Drive`, `Fuel`, `Veh Class`). Random Forest, in particular, already finds equally good (or better) groupings on its own from the raw features, so the cluster label just adds repeated, lower-detail information instead of anything new.

This was checked with real experiments, not just assumed. Clustering is still useful as a **way to explore and describe the data** (Part 1), but it isn't needed as an input for regression (Part 2).

---

## Limitations

- **Sample size:** After removing duplicates, 1,217 rows are left. This is enough for the models used here, but it limits how much we can trust results for rare fuel types (like Hydrogen or Diesel — both have 5 or fewer samples in the test set).
- **Missing powertrain details:** The dataset doesn't include battery size or electric-only driving range, which is the main reason PHEV predictions have higher error. Adding this information would likely reduce that error a lot, but it wasn't available in the EPA file used here.
- **One model year only:** Results are based on a single year of EPA data; emissions technology and the mix of cars on the market change over time, so predictions should be checked again periodically against newer data.

---

## Tools Used

`Python` · `pandas` · `NumPy` · `scikit-learn` (K-means, Linear Regression, Random Forest, PCA, StandardScaler, OneHotEncoder, cross-validation, RandomizedSearchCV) · `matplotlib` · `seaborn`

**Notebook layout:**
```
1. Load Dataset & Data Inspection
2. Data Cleaning
3. EDA (numerical + categorical, leakage check)
4. Feature Preprocessing (shared)
PART 1 — Vehicle Segmentation (Clustering)
PART 2 — CO2 Emission Prediction (Regression)
   - Model comparison, feature importance
   - Test-set predictions by vehicle model, error analysis by fuel type
   - Cross-validation (stability check)
   - Save trained model files
APPENDIX — Clustering + Regression experiment
Overall Conclusion
```
