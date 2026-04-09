# MLflow Tracking Skill

## Purpose
This skill teaches Genie Code how to use MLflow for experiment tracking, parameter logging, metric recording, artifact management, model versioning, and run comparison on Databricks.

## When to Use
* Tracking machine learning experiments
* Logging model parameters and hyperparameters
* Recording training and validation metrics
* Managing model artifacts and outputs
* Comparing experiment runs
* Versioning models and datasets
* Reproducing experiment results
* Collaborating on ML projects

## Key Concepts

### 1. Experiments and Runs
* **Experiment**: Collection of related runs
* **Run**: Single execution of ML code
* **Nested runs**: Parent-child run relationships
* **Run context**: Automatic logging scope

### 2. Logging
* **Parameters**: Input configurations, hyperparameters
* **Metrics**: Performance measurements over time
* **Artifacts**: Files, models, plots, data
* **Tags**: Metadata for organization

### 3. Autologging
* **Framework integration**: sklearn, TensorFlow, PyTorch, XGBoost
* **Automatic capture**: Parameters, metrics, models
* **Custom callbacks**: Extend autologging

### 4. Model Registry
* **Version control**: Track model versions
* **Stage management**: Dev, Staging, Production
* **Lineage**: Link models to training runs

### 5. Run Comparison
* **Parallel coordinates**: Compare metrics
* **Scatter plots**: Visualize relationships
* **Table view**: Side-by-side comparison

---

## Examples

### Example 1: Basic Experiment Tracking

```python
import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, precision_score, recall_score, f1_score

# Set experiment
mlflow.set_experiment("/Users/username/churn-prediction")

# Start run
with mlflow.start_run(run_name="rf_baseline_v1") as run:
    
    # Load data
    df = spark.table("catalog.ml.training_data").toPandas()
    X = df.drop("churn", axis=1)
    y = df["churn"]
    
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )
    
    # Log parameters
    mlflow.log_param("model_type", "RandomForest")
    mlflow.log_param("test_size", 0.2)
    mlflow.log_param("random_state", 42)
    mlflow.log_param("n_estimators", 100)
    mlflow.log_param("max_depth", 10)
    
    # Train model
    model = RandomForestClassifier(
        n_estimators=100,
        max_depth=10,
        random_state=42
    )
    model.fit(X_train, y_train)
    
    # Predict
    y_pred = model.predict(X_test)
    y_pred_proba = model.predict_proba(X_test)[:, 1]
    
    # Calculate metrics
    accuracy = accuracy_score(y_test, y_pred)
    precision = precision_score(y_test, y_pred)
    recall = recall_score(y_test, y_pred)
    f1 = f1_score(y_test, y_pred)
    
    # Log metrics
    mlflow.log_metric("accuracy", accuracy)
    mlflow.log_metric("precision", precision)
    mlflow.log_metric("recall", recall)
    mlflow.log_metric("f1_score", f1)
    
    # Log model
    mlflow.sklearn.log_model(
        model,
        artifact_path="model",
        registered_model_name="churn_classifier"
    )
    
    # Log additional artifacts
    import matplotlib.pyplot as plt
    from sklearn.metrics import confusion_matrix, ConfusionMatrixDisplay
    
    # Confusion matrix
    cm = confusion_matrix(y_test, y_pred)
    disp = ConfusionMatrixDisplay(confusion_matrix=cm)
    disp.plot()
    plt.savefig("confusion_matrix.png")
    mlflow.log_artifact("confusion_matrix.png")
    
    # Feature importance
    feature_importance = pd.DataFrame({
        'feature': X.columns,
        'importance': model.feature_importances_
    }).sort_values('importance', ascending=False)
    
    feature_importance.to_csv("feature_importance.csv", index=False)
    mlflow.log_artifact("feature_importance.csv")
    
    # Log tags
    mlflow.set_tag("stage", "development")
    mlflow.set_tag("team", "data-science")
    mlflow.set_tag("project", "customer-churn")
    
    print(f"Run ID: {run.info.run_id}")
    print(f"Accuracy: {accuracy:.4f}")
```

### Example 2: Hyperparameter Tuning with MLflow

```python
import mlflow
from sklearn.model_selection import GridSearchCV
from sklearn.ensemble import RandomForestClassifier
import numpy as np

mlflow.set_experiment("/Users/username/churn-prediction-tuning")

# Define parameter grid
param_grid = {
    'n_estimators': [50, 100, 200],
    'max_depth': [5, 10, 15, 20],
    'min_samples_split': [2, 5, 10],
    'min_samples_leaf': [1, 2, 4]
}

# Parent run for hyperparameter search
with mlflow.start_run(run_name="rf_grid_search") as parent_run:
    
    mlflow.log_param("search_type", "GridSearchCV")
    mlflow.log_param("cv_folds", 5)
    mlflow.log_param("scoring", "f1")
    
    # Track total configurations
    total_configs = np.prod([len(v) for v in param_grid.values()])
    mlflow.log_param("total_configurations", total_configs)
    
    best_score = 0
    best_params = {}
    
    # Manual grid search with nested runs
    for n_est in param_grid['n_estimators']:
        for max_d in param_grid['max_depth']:
            for min_split in param_grid['min_samples_split']:
                for min_leaf in param_grid['min_samples_leaf']:
                    
                    # Nested run for each configuration
                    with mlflow.start_run(
                        run_name=f"rf_n{n_est}_d{max_d}_s{min_split}_l{min_leaf}",
                        nested=True
                    ) as child_run:
                        
                        # Log parameters
                        mlflow.log_param("n_estimators", n_est)
                        mlflow.log_param("max_depth", max_d)
                        mlflow.log_param("min_samples_split", min_split)
                        mlflow.log_param("min_samples_leaf", min_leaf)
                        
                        # Train model
                        model = RandomForestClassifier(
                            n_estimators=n_est,
                            max_depth=max_d,
                            min_samples_split=min_split,
                            min_samples_leaf=min_leaf,
                            random_state=42
                        )
                        
                        # Cross-validation
                        scores = cross_val_score(
                            model, X_train, y_train, 
                            cv=5, scoring='f1'
                        )
                        
                        mean_score = scores.mean()
                        std_score = scores.std()
                        
                        # Log metrics
                        mlflow.log_metric("mean_cv_f1", mean_score)
                        mlflow.log_metric("std_cv_f1", std_score)
                        
                        # Track best
                        if mean_score > best_score:
                            best_score = mean_score
                            best_params = {
                                'n_estimators': n_est,
                                'max_depth': max_d,
                                'min_samples_split': min_split,
                                'min_samples_leaf': min_leaf
                            }
    
    # Log best parameters in parent run
    mlflow.log_params({f"best_{k}": v for k, v in best_params.items()})
    mlflow.log_metric("best_cv_f1", best_score)
    
    print(f"Best F1 Score: {best_score:.4f}")
    print(f"Best Parameters: {best_params}")
```

### Example 3: Autologging for Deep Learning

```python
import mlflow
import mlflow.tensorflow
import tensorflow as tf
from tensorflow import keras

mlflow.set_experiment("/Users/username/deep-learning-churn")

# Enable autologging for TensorFlow
mlflow.tensorflow.autolog(
    log_models=True,
    log_datasets=True,
    disable=False,
    exclusive=False,
    disable_for_unsupported_versions=False
)

with mlflow.start_run(run_name="lstm_churn_v1") as run:
    
    # Build model
    model = keras.Sequential([
        keras.layers.LSTM(64, return_sequences=True, input_shape=(10, 20)),
        keras.layers.Dropout(0.3),
        keras.layers.LSTM(32),
        keras.layers.Dropout(0.3),
        keras.layers.Dense(16, activation='relu'),
        keras.layers.Dense(1, activation='sigmoid')
    ])
    
    model.compile(
        optimizer='adam',
        loss='binary_crossentropy',
        metrics=['accuracy', 'AUC']
    )
    
    # Log additional parameters not captured by autolog
    mlflow.log_param("architecture", "LSTM")
    mlflow.log_param("lstm_units", "64,32")
    mlflow.log_param("dropout_rate", 0.3)
    
    # Train model (autolog captures everything)
    history = model.fit(
        X_train, y_train,
        validation_data=(X_val, y_val),
        epochs=50,
        batch_size=32,
        callbacks=[
            keras.callbacks.EarlyStopping(
                monitor='val_loss',
                patience=5,
                restore_best_weights=True
            )
        ]
    )
    
    # Autolog automatically logs:
    # - Model architecture
    # - Training parameters
    # - Metrics per epoch
    # - Final model
    
    # Log custom artifacts
    import matplotlib.pyplot as plt
    
    plt.figure(figsize=(12, 4))
    
    plt.subplot(1, 2, 1)
    plt.plot(history.history['loss'], label='Train Loss')
    plt.plot(history.history['val_loss'], label='Val Loss')
    plt.title('Model Loss')
    plt.legend()
    
    plt.subplot(1, 2, 2)
    plt.plot(history.history['accuracy'], label='Train Accuracy')
    plt.plot(history.history['val_accuracy'], label='Val Accuracy')
    plt.title('Model Accuracy')
    plt.legend()
    
    plt.tight_layout()
    plt.savefig("training_history.png")
    mlflow.log_artifact("training_history.png")
    
    print(f"Run ID: {run.info.run_id}")
```

### Example 4: Logging Custom Metrics Over Time

```python
import mlflow
import time
from sklearn.metrics import roc_auc_score

mlflow.set_experiment("/Users/username/online-learning")

with mlflow.start_run(run_name="online_learning_v1") as run:
    
    # Simulate online learning with metric logging over time
    model = initialize_model()
    
    for batch_num in range(100):
        # Get batch data
        X_batch, y_batch = get_next_batch()
        
        # Train on batch
        model.partial_fit(X_batch, y_batch)
        
        # Evaluate
        y_pred = model.predict(X_val)
        y_pred_proba = model.predict_proba(X_val)[:, 1]
        
        accuracy = accuracy_score(y_val, y_pred)
        auc = roc_auc_score(y_val, y_pred_proba)
        
        # Log metrics with step number
        mlflow.log_metric("accuracy", accuracy, step=batch_num)
        mlflow.log_metric("auc", auc, step=batch_num)
        mlflow.log_metric("samples_seen", (batch_num + 1) * len(X_batch), step=batch_num)
        
        # Log every 10 batches
        if batch_num % 10 == 0:
            print(f"Batch {batch_num}: Accuracy={accuracy:.4f}, AUC={auc:.4f}")
    
    # Log final model
    mlflow.sklearn.log_model(model, "final_model")
    
    # Log training summary
    mlflow.log_param("total_batches", 100)
    mlflow.log_param("batch_size", len(X_batch))
    mlflow.log_param("total_samples", 100 * len(X_batch))
```

### Example 5: Managing Artifacts and Model Files

```python
import mlflow
import joblib
import json
import pandas as pd

mlflow.set_experiment("/Users/username/model-artifacts")

with mlflow.start_run(run_name="artifact_management_demo") as run:
    
    # Train model
    model = train_model()
    
    # === Log different types of artifacts ===
    
    # 1. Model as pickle
    joblib.dump(model, "model.pkl")
    mlflow.log_artifact("model.pkl", artifact_path="models")
    
    # 2. Configuration as JSON
    config = {
        "model_type": "RandomForest",
        "n_estimators": 100,
        "max_depth": 10,
        "feature_columns": list(X.columns)
    }
    with open("config.json", "w") as f:
        json.dump(config, f, indent=2)
    mlflow.log_artifact("config.json", artifact_path="configs")
    
    # 3. Training data sample
    X_train.head(1000).to_csv("training_sample.csv", index=False)
    mlflow.log_artifact("training_sample.csv", artifact_path="data")
    
    # 4. Model performance report
    report = classification_report(y_test, y_pred, output_dict=True)
    report_df = pd.DataFrame(report).transpose()
    report_df.to_csv("classification_report.csv")
    mlflow.log_artifact("classification_report.csv", artifact_path="reports")
    
    # 5. Multiple plots
    import matplotlib.pyplot as plt
    
    # ROC Curve
    from sklearn.metrics import roc_curve, auc
    fpr, tpr, _ = roc_curve(y_test, y_pred_proba)
    roc_auc = auc(fpr, tpr)
    
    plt.figure()
    plt.plot(fpr, tpr, label=f'ROC (AUC = {roc_auc:.2f})')
    plt.plot([0, 1], [0, 1], 'k--')
    plt.xlabel('False Positive Rate')
    plt.ylabel('True Positive Rate')
    plt.title('ROC Curve')
    plt.legend()
    plt.savefig("roc_curve.png")
    mlflow.log_artifact("roc_curve.png", artifact_path="plots")
    
    # 6. Log entire directory
    import os
    os.makedirs("output", exist_ok=True)
    # Save multiple files to directory
    # ...
    mlflow.log_artifacts("output", artifact_path="all_outputs")
    
    # 7. Log dictionary as params
    mlflow.log_params(config)
    
    # 8. Log text
    mlflow.log_text("Model trained successfully", "status.txt")
    
    # 9. Log dictionary as JSON artifact
    mlflow.log_dict(config, "config_dict.json")
    
    print(f"All artifacts logged to run: {run.info.run_id}")
```

### Example 6: Run Comparison and Analysis

```python
import mlflow
from mlflow.tracking import MlflowClient
import pandas as pd

client = MlflowClient()

# Get experiment
experiment = client.get_experiment_by_name("/Users/username/churn-prediction")
experiment_id = experiment.experiment_id

# Search runs with filters
runs = client.search_runs(
    experiment_ids=[experiment_id],
    filter_string="metrics.accuracy > 0.85",
    order_by=["metrics.accuracy DESC"],
    max_results=10
)

# Create comparison dataframe
comparison_data = []

for run in runs:
    comparison_data.append({
        'run_id': run.info.run_id,
        'run_name': run.data.tags.get('mlflow.runName', 'N/A'),
        'accuracy': run.data.metrics.get('accuracy', 0),
        'precision': run.data.metrics.get('precision', 0),
        'recall': run.data.metrics.get('recall', 0),
        'f1_score': run.data.metrics.get('f1_score', 0),
        'n_estimators': run.data.params.get('n_estimators', 'N/A'),
        'max_depth': run.data.params.get('max_depth', 'N/A'),
        'start_time': pd.to_datetime(run.info.start_time, unit='ms')
    })

comparison_df = pd.DataFrame(comparison_data)
print(comparison_df)

# Find best run
best_run = comparison_df.loc[comparison_df['accuracy'].idxmax()]
print(f"\nBest Run:")
print(f"  Run ID: {best_run['run_id']}")
print(f"  Accuracy: {best_run['accuracy']:.4f}")
print(f"  Parameters: n_estimators={best_run['n_estimators']}, max_depth={best_run['max_depth']}")

# Load best model
best_model = mlflow.sklearn.load_model(f"runs:/{best_run['run_id']}/model")

# Compare multiple metrics
import matplotlib.pyplot as plt

fig, axes = plt.subplots(2, 2, figsize=(12, 10))

metrics = ['accuracy', 'precision', 'recall', 'f1_score']
for idx, metric in enumerate(metrics):
    ax = axes[idx // 2, idx % 2]
    comparison_df.plot(
        x='run_name',
        y=metric,
        kind='bar',
        ax=ax,
        legend=False
    )
    ax.set_title(f'{metric.capitalize()} Comparison')
    ax.set_xlabel('Run')
    ax.set_ylabel(metric.capitalize())
    plt.setp(ax.xaxis.get_majorticklabels(), rotation=45, ha='right')

plt.tight_layout()
plt.savefig("run_comparison.png")
plt.show()
```

### Example 7: Production ML Pipeline with MLflow

```python
import mlflow
from mlflow.tracking import MlflowClient
from databricks.feature_store import FeatureStoreClient

class MLProductionPipeline:
    """
    Production ML pipeline with comprehensive MLflow tracking
    """
    
    def __init__(self, experiment_name):
        self.experiment_name = experiment_name
        mlflow.set_experiment(experiment_name)
        self.client = MlflowClient()
        self.fs = FeatureStoreClient()
    
    def train_and_log(self, model, X_train, y_train, X_val, y_val, config):
        """Train model with comprehensive logging"""
        
        with mlflow.start_run(run_name=config['run_name']) as run:
            
            # Log system info
            mlflow.set_tag("mlflow.source.type", "NOTEBOOK")
            mlflow.set_tag("environment", config.get('environment', 'development'))
            mlflow.set_tag("dataset_version", config.get('dataset_version', 'v1'))
            
            # Log all config parameters
            mlflow.log_params(config)
            
            # Log dataset info
            mlflow.log_param("train_samples", len(X_train))
            mlflow.log_param("val_samples", len(X_val))
            mlflow.log_param("n_features", X_train.shape[1])
            mlflow.log_param("class_balance", y_train.value_counts().to_dict())
            
            # Train model
            model.fit(X_train, y_train)
            
            # Evaluate
            train_score = model.score(X_train, y_train)
            val_score = model.score(X_val, y_val)
            
            y_pred_val = model.predict(X_val)
            y_pred_proba_val = model.predict_proba(X_val)[:, 1]
            
            # Log comprehensive metrics
            from sklearn.metrics import (
                accuracy_score, precision_score, recall_score, f1_score,
                roc_auc_score, log_loss
            )
            
            metrics = {
                "train_score": train_score,
                "val_score": val_score,
                "val_accuracy": accuracy_score(y_val, y_pred_val),
                "val_precision": precision_score(y_val, y_pred_val),
                "val_recall": recall_score(y_val, y_pred_val),
                "val_f1": f1_score(y_val, y_pred_val),
                "val_auc": roc_auc_score(y_val, y_pred_proba_val),
                "val_logloss": log_loss(y_val, y_pred_proba_val)
            }
            
            mlflow.log_metrics(metrics)
            
            # Log model with signature
            from mlflow.models.signature import infer_signature
            signature = infer_signature(X_train, model.predict(X_train))
            
            mlflow.sklearn.log_model(
                model,
                "model",
                signature=signature,
                input_example=X_train.iloc[:5],
                registered_model_name=config['model_name']
            )
            
            # Log artifacts
            self._log_evaluation_artifacts(y_val, y_pred_val, y_pred_proba_val)
            
            return run.info.run_id, metrics
    
    def _log_evaluation_artifacts(self, y_true, y_pred, y_pred_proba):
        """Log evaluation plots and reports"""
        import matplotlib.pyplot as plt
        from sklearn.metrics import confusion_matrix, ConfusionMatrixDisplay
        from sklearn.metrics import roc_curve, auc, precision_recall_curve
        
        # Confusion Matrix
        cm = confusion_matrix(y_true, y_pred)
        disp = ConfusionMatrixDisplay(confusion_matrix=cm)
        disp.plot()
        plt.savefig("confusion_matrix.png")
        mlflow.log_artifact("confusion_matrix.png")
        plt.close()
        
        # ROC Curve
        fpr, tpr, _ = roc_curve(y_true, y_pred_proba)
        roc_auc = auc(fpr, tpr)
        
        plt.figure()
        plt.plot(fpr, tpr, label=f'ROC (AUC = {roc_auc:.2f})')
        plt.plot([0, 1], [0, 1], 'k--')
        plt.xlabel('FPR')
        plt.ylabel('TPR')
        plt.title('ROC Curve')
        plt.legend()
        plt.savefig("roc_curve.png")
        mlflow.log_artifact("roc_curve.png")
        plt.close()
        
        # Precision-Recall Curve
        precision, recall, _ = precision_recall_curve(y_true, y_pred_proba)
        
        plt.figure()
        plt.plot(recall, precision)
        plt.xlabel('Recall')
        plt.ylabel('Precision')
        plt.title('Precision-Recall Curve')
        plt.savefig("pr_curve.png")
        mlflow.log_artifact("pr_curve.png")
        plt.close()

# Usage
pipeline = MLProductionPipeline("/Users/username/production-churn")

config = {
    'run_name': 'rf_production_v1',
    'model_name': 'churn_classifier_prod',
    'environment': 'production',
    'dataset_version': 'v2024_04',
    'n_estimators': 100,
    'max_depth': 10
}

run_id, metrics = pipeline.train_and_log(model, X_train, y_train, X_val, y_val, config)
print(f"Training complete. Run ID: {run_id}")
print(f"Metrics: {metrics}")
```

---

## Best Practices

### 1. Experiment Organization
* Use descriptive experiment names
* Group related runs in same experiment
* Use tags for filtering (team, project, stage)
* Archive old experiments

### 2. Parameter Logging
* Log all hyperparameters
* Log data preprocessing steps
* Log feature engineering choices
* Log random seeds for reproducibility

### 3. Metric Logging
* Log both training and validation metrics
* Use step parameter for time-series metrics
* Log business metrics alongside technical metrics
* Record inference time/latency

### 4. Artifact Management
* Save model artifacts with clear naming
* Log data samples for debugging
* Save evaluation plots
* Version configuration files

### 5. Model Registry
* Register production-ready models
* Use stage transitions (Staging → Production)
* Add model descriptions and tags
* Link to training run for lineage

---

## Related Skills

* [Model Deployment](../model-deployment/SKILL.md) - For deploying tracked models
* [Feature Engineering](../feature-engineering/SKILL.md) - For feature tracking
* [Monitoring Observability](../monitoring-observability/SKILL.md) - For production monitoring

---

**Last Updated**: 2024-04-09
