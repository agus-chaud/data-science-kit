# MLflow cookbook

Loaded from `SKILL.md` when the task touches MLflow: experiment tracking, model registry, deployment, batch inference.

> **Registry model:** classic stage-based registry (`Staging`/`Production` + `transition_model_version_stage`) is **deprecated** since MLflow 2.9. Use the **Unity Catalog model registry** — models get three-level names (`catalog.schema.model`) and versions are promoted with **aliases** (`@champion`, `@challenger`) instead of stages.

## Experiment tracking

```python
import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, f1_score

# Register models in Unity Catalog (default on recent DBR; set explicitly to be safe)
mlflow.set_registry_uri("databricks-uc")

# Set experiment
mlflow.set_experiment("/Users/data-science/customer-churn")

# Start run
with mlflow.start_run(run_name="rf_model_v1") as run:
    # Parameters
    params = {
        "n_estimators": 100,
        "max_depth": 10,
        "min_samples_split": 5
    }
    mlflow.log_params(params)

    # Train model
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
    model = RandomForestClassifier(**params)
    model.fit(X_train, y_train)

    # Evaluate
    y_pred = model.predict(X_test)
    accuracy = accuracy_score(y_test, y_pred)
    f1 = f1_score(y_test, y_pred)

    # Log metrics
    mlflow.log_metric("accuracy", accuracy)
    mlflow.log_metric("f1_score", f1)

    # Log model with signature (required for UC registry)
    mlflow.sklearn.log_model(model, "model", input_example=X_train[:5])

    # Log artifacts
    mlflow.log_artifact("feature_importance.png")

    # Add tags
    mlflow.set_tags({
        "team": "data-science",
        "model_type": "classification"
    })

# Load model from run
run_id = run.info.run_id
model = mlflow.sklearn.load_model(f"runs:/{run_id}/model")

# Register model under a UC three-level name
model_uri = f"runs:/{run_id}/model"
mlflow.register_model(model_uri, "prod.ml.customer_churn_model")
```

## Model registry (UC) and deployment

```python
from mlflow.tracking import MlflowClient

client = MlflowClient()

# Promote a version with an alias (replaces stage transitions)
client.set_registered_model_alias(
    name="prod.ml.customer_churn_model",
    alias="champion",
    version=1,
)

# Add model version description
client.update_model_version(
    name="prod.ml.customer_churn_model",
    version=1,
    description="Random Forest model with hyperparameter tuning",
)

# Load the current champion (alias-based URI)
model = mlflow.pyfunc.load_model("models:/prod.ml.customer_churn_model@champion")

# Batch inference with Spark
model_udf = mlflow.pyfunc.spark_udf(
    spark,
    model_uri="models:/prod.ml.customer_churn_model@champion",
    result_type="double",
)

predictions = df.withColumn(
    "churn_prediction",
    model_udf(*feature_columns)
)
```

Access to UC-registered models is governed with regular UC grants (`GRANT EXECUTE ON MODEL ...`) — see `unity-catalog.md`.
