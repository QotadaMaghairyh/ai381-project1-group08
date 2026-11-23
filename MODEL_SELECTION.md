# Model Selection & Promotion Gate Decisions (Student 1)

## Candidate Models
- **logreg (`LogisticRegression(max_iter=1000)`)**: Linear baseline with standardized numerical features and one-hot encoded categorical features. Achieved **F1 = 0.495**, **AUC = 0.798**, **p95 latency ~ 3.5 ms**, and **size = 4 KB**.
- **random_forest (`RandomForestClassifier(n_estimators=200, max_depth=8, random_state=42)`)**: Depth-constrained ensemble of 200 trees. Achieved **F1 = 0.454**, **AUC = 0.759**, **p95 latency ~ 13.5 ms**, and **size = 3160 KB**.
- **gradient_boosting (`GradientBoostingClassifier(random_state=42)`)**: Sequential boosted trees candidate. Achieved **F1 = 0.417**, **AUC = 0.748**, **p95 latency ~ 4.2 ms**, and **size = 142 KB**.

## Promotion Gate Thresholds
- **p95 Latency Budget (`LATENCY_BUDGET_MS = 25.0` ms)**: Ensures real-time single-row inference stays well within interactive service latency limits while accommodating tree-based models.
- **Minimum F1 Gain (`MIN_GAIN = 0.005`)**: Prevents noisy or marginal model swaps in production unless there is a meaningful F1 improvement over the current champion.

## Selected Champion (`churn-model` v1)
- Promoted **`logreg`** (`run_id: a10fb18d2f324d4d9f7e84ae33b16ca3`) as the initial champion on `data/churn_v1.csv` (`data_sha256: 76a6447e764b`), offering the strongest F1 and AUC alongside ultra-low latency (~3.5 ms) and compact artifact size (4 KB).

## Data Hash Logging (`data_sha256`)
- Logging `data_sha256` provides exact data lineage and reproducibility so any changes to training data contents—even under the same file name—are traceable to specific model runs.
