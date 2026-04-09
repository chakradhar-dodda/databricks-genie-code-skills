# Feature Engineering Skill

## Purpose
This skill teaches Genie Code how to perform feature engineering for machine learning on Databricks using Feature Store, transformations, time-based features, categorical encoding, and feature selection techniques.

## When to Use
* Building ML models requiring feature transformations
* Creating reusable features across models
* Implementing time-series features
* Encoding categorical variables
* Handling missing values
* Feature scaling and normalization
* Creating interaction features
* Managing feature lifecycle and versioning

## Key Concepts

### 1. Feature Store
* **Centralized repository**: Share features across teams
* **Online/offline serving**: Training and inference
* **Point-in-time lookups**: Avoid data leakage
* **Feature lineage**: Track feature provenance
* **Versioning**: Manage feature evolution

### 2. Transformations
* **Scaling**: StandardScaler, MinMaxScaler, RobustScaler
* **Encoding**: One-hot, label, target encoding
* **Binning**: Discretize continuous variables
* **Polynomial**: Create interaction features
* **Custom**: Domain-specific transformations

### 3. Time-Based Features
* **Temporal**: Year, month, day, hour, day of week
* **Lags**: Previous values (LAG features)
* **Rolling**: Moving averages, rolling aggregations
* **Seasonality**: Cyclical patterns
* **Time since**: Days since last event

### 4. Categorical Encoding
* **One-hot**: Binary columns for each category
* **Label**: Integer encoding
* **Target**: Mean of target per category
* **Hashing**: Fixed-size hash representation
* **Embedding**: Learn dense representations

### 5. Feature Selection
* **Filter methods**: Correlation, variance threshold
* **Wrapper methods**: RFE, forward/backward selection
* **Embedded methods**: L1/L2 regularization, tree importance
* **Dimensionality reduction**: PCA, SVD

---

## Examples

### Example 1: Feature Store Setup and Usage

```python
from databricks import feature_store
from databricks.feature_store import FeatureStoreClient, FeatureLookup
from pyspark.sql.functions import col, current_timestamp, datediff, count, avg, sum as spark_sum, max, min, countDistinct, when, date_sub, current_date
from datetime import datetime

# Initialize Feature Store client
fs = FeatureStoreClient()

# Create customer features
def create_customer_features(df):
    """
    Create customer-level features
    """
    customer_features = (
        df
        .groupBy("customer_id")
        .agg(
            count("order_id").alias("lifetime_orders"),
            spark_sum("order_amount").alias("lifetime_value"),
            avg("order_amount").alias("avg_order_value"),
            max("order_date").alias("last_order_date"),
            min("order_date").alias("first_order_date"),
            countDistinct("product_id").alias("unique_products"),
            countDistinct(when(col("order_date") >= date_sub(current_date(), 90), col("order_id"))).alias("orders_90d")
        )
        .withColumn("days_since_first_order", datediff(current_date(), col("first_order_date")))
        .withColumn("days_since_last_order", datediff(current_date(), col("last_order_date")))
        .withColumn("order_frequency", col("lifetime_orders") / (col("days_since_first_order") + 1))
        .withColumn("update_timestamp", current_timestamp())
    )
    
    return customer_features

# Create feature table
orders_df = spark.table("catalog.sales.orders")
customer_features = create_customer_features(orders_df)

# Register feature table
fs.create_table(
    name="catalog.features.customer_features",
    primary_keys=["customer_id"],
    df=customer_features,
    description="Customer behavioral features for ML models"
)

# Update features (batch)
fs.write_table(
    name="catalog.features.customer_features",
    df=customer_features,
    mode="merge"
)

# Use features in training
feature_lookups = [
    FeatureLookup(
        table_name="catalog.features.customer_features",
        lookup_key="customer_id",
        feature_names=[
            "lifetime_orders",
            "lifetime_value",
            "avg_order_value",
            "days_since_last_order",
            "order_frequency"
        ]
    )
]

# Create training dataset
training_df = spark.table("catalog.ml.churn_labels")

training_set = fs.create_training_set(
    df=training_df,
    feature_lookups=feature_lookups,
    label="churned",
    exclude_columns=["customer_id", "update_timestamp"]
)

training_pd = training_set.load_df().toPandas()
```

### Example 2: Categorical Encoding Techniques

```python
from pyspark.ml.feature import StringIndexer, OneHotEncoder, VectorAssembler
from pyspark.ml import Pipeline
from pyspark.sql.functions import avg as avg_func, count as count_func

df = spark.table("catalog.sales.customers")

# Method 1: One-Hot Encoding
string_indexer = StringIndexer(
    inputCol="customer_segment",
    outputCol="segment_index",
    handleInvalid="keep"  # Handle unknown categories
)

onehot_encoder = OneHotEncoder(
    inputCols=["segment_index"],
    outputCols=["segment_onehot"],
    dropLast=False  # Keep all categories
)

# Method 2: Label Encoding (ordinal)
region_indexer = StringIndexer(
    inputCol="region",
    outputCol="region_encoded",
    stringOrderType="frequencyDesc"  # Order by frequency
)

# Method 3: Target Encoding
target_encoding = (
    df
    .groupBy("customer_segment")
    .agg(avg_func("churn_flag").alias("segment_churn_rate"))
)

df_with_target_encoding = df.join(target_encoding, "customer_segment")

# Method 4: Frequency Encoding
frequency_encoding = (
    df
    .groupBy("city")
    .agg(count_func("*").alias("city_frequency"))
)

df_with_freq_encoding = df.join(frequency_encoding, "city")

# Create pipeline
pipeline = Pipeline(stages=[
    string_indexer,
    onehot_encoder,
    region_indexer
])

model = pipeline.fit(df)
transformed_df = model.transform(df)

# Method 5: Feature Hashing (for high-cardinality)
from pyspark.ml.feature import FeatureHasher

hasher = FeatureHasher(
    inputCols=["product_id", "category", "sub_category"],
    outputCol="hashed_features",
    numFeatures=1024  # Hash to fixed size
)

hashed_df = hasher.transform(df)

# Method 6: Binary Encoding
from pyspark.sql.functions import when, col

df_binary = (
    df
    .withColumn("is_premium", when(col("customer_tier") == "Premium", 1).otherwise(0))
    .withColumn("is_active", when(col("status") == "Active", 1).otherwise(0))
    .withColumn("has_email", when(col("email").isNotNull(), 1).otherwise(0))
)
```

### Example 3: Time-Based Feature Engineering

```python
from pyspark.sql.functions import (
    year, month, dayofmonth, dayofweek, hour, minute,
    lag, lead, avg, sum as sum_func,
    datediff, months_between, date_format, quarter,
    sin, cos, lit, pi
)
from pyspark.sql.window import Window

df = spark.table("catalog.sales.orders")

# Extract temporal components
df_temporal = (
    df
    .withColumn("order_year", year("order_date"))
    .withColumn("order_month", month("order_date"))
    .withColumn("order_day", dayofmonth("order_date"))
    .withColumn("order_dow", dayofweek("order_date"))  # 1=Sunday, 7=Saturday
    .withColumn("order_hour", hour("order_timestamp"))
    .withColumn("order_quarter", quarter("order_date"))
    
    # Derived temporal features
    .withColumn("is_weekend", when(col("order_dow").isin([1, 7]), 1).otherwise(0))
    .withColumn("is_month_start", when(dayofmonth("order_date") <= 5, 1).otherwise(0))
    .withColumn("is_month_end", when(dayofmonth("order_date") >= 25, 1).otherwise(0))
    .withColumn("is_business_hours", when(col("order_hour").between(9, 17), 1).otherwise(0))
)

# Create cyclical features (for seasonality)
df_cyclical = (
    df_temporal
    # Month cyclical (12 months)
    .withColumn("month_sin", sin(col("order_month") * 2 * pi() / 12))
    .withColumn("month_cos", cos(col("order_month") * 2 * pi() / 12))
    
    # Day of week cyclical (7 days)
    .withColumn("dow_sin", sin(col("order_dow") * 2 * pi() / 7))
    .withColumn("dow_cos", cos(col("order_dow") * 2 * pi() / 7))
    
    # Hour cyclical (24 hours)
    .withColumn("hour_sin", sin(col("order_hour") * 2 * pi() / 24))
    .withColumn("hour_cos", cos(col("order_hour") * 2 * pi() / 24))
)

# Lag and lead features
window_spec = Window.partitionBy("customer_id").orderBy("order_date")

df_lags = (
    df
    .withColumn("previous_order_amount", lag("order_amount", 1).over(window_spec))
    .withColumn("previous_2_order_amount", lag("order_amount", 2).over(window_spec))
    .withColumn("next_order_amount", lead("order_amount", 1).over(window_spec))
    
    # Time differences
    .withColumn("days_since_last_order", 
                datediff(col("order_date"), lag("order_date", 1).over(window_spec)))
    .withColumn("days_to_next_order",
                datediff(lead("order_date", 1).over(window_spec), col("order_date")))
)

# Rolling aggregations (7-day window)
rolling_window = Window.partitionBy("customer_id").orderBy("order_date").rowsBetween(-6, 0)

df_rolling = (
    df
    .withColumn("rolling_7_orders", count_func("*").over(rolling_window))
    .withColumn("rolling_7_avg_amount", avg("order_amount").over(rolling_window))
    .withColumn("rolling_7_total", sum_func("order_amount").over(rolling_window))
    .withColumn("rolling_7_max", max("order_amount").over(rolling_window))
    .withColumn("rolling_7_min", min("order_amount").over(rolling_window))
)

# Expanding window (cumulative)
expanding_window = Window.partitionBy("customer_id").orderBy("order_date").rowsBetween(Window.unboundedPreceding, 0)

df_cumulative = (
    df
    .withColumn("cumulative_orders", count_func("*").over(expanding_window))
    .withColumn("cumulative_spend", sum_func("order_amount").over(expanding_window))
    .withColumn("avg_order_value_to_date", avg("order_amount").over(expanding_window))
)

# Time since last event
df_time_since = (
    df
    .groupBy("customer_id")
    .agg(
        max("order_date").alias("last_order_date"),
        max("login_date").alias("last_login_date")
    )
    .withColumn("days_since_last_order", datediff(current_date(), "last_order_date"))
    .withColumn("days_since_last_login", datediff(current_date(), "last_login_date"))
    .withColumn("months_since_last_order", months_between(current_date(), "last_order_date"))
)
```

### Example 4: Feature Scaling and Normalization

```python
from pyspark.ml.feature import StandardScaler, MinMaxScaler, RobustScaler, Normalizer, VectorAssembler, MaxAbsScaler

df = spark.table("catalog.ml.raw_features")

# Assemble features into vector
feature_cols = ["age", "income", "credit_score", "account_balance", "transaction_count"]

assembler = VectorAssembler(
    inputCols=feature_cols,
    outputCol="features",
    handleInvalid="skip"  # Skip rows with invalid values
)

df_vector = assembler.transform(df)

# Method 1: Standard Scaling (z-score normalization)
standard_scaler = StandardScaler(
    inputCol="features",
    outputCol="scaled_features",
    withMean=True,  # Center to mean=0
    withStd=True     # Scale to std=1
)

standard_model = standard_scaler.fit(df_vector)
df_standard = standard_model.transform(df_vector)

print(f"Mean: {standard_model.mean}")
print(f"Std: {standard_model.std}")

# Method 2: Min-Max Scaling (0-1 range)
minmax_scaler = MinMaxScaler(
    inputCol="features",
    outputCol="minmax_features",
    min=0.0,
    max=1.0
)

minmax_model = minmax_scaler.fit(df_vector)
df_minmax = minmax_model.transform(df_vector)

# Method 3: Robust Scaling (resistant to outliers)
robust_scaler = RobustScaler(
    inputCol="features",
    outputCol="robust_features",
    withCentering=True,  # Center by median
    withScaling=True,    # Scale by IQR
    lower=0.25,
    upper=0.75
)

robust_model = robust_scaler.fit(df_vector)
df_robust = robust_model.transform(df_vector)

# Method 4: Max-Abs Scaling (-1 to 1 range)
maxabs_scaler = MaxAbsScaler(
    inputCol="features",
    outputCol="maxabs_features"
)

maxabs_model = maxabs_scaler.fit(df_vector)
df_maxabs = maxabs_model.transform(df_vector)

# Method 5: L2 Normalization (unit vectors)
normalizer = Normalizer(
    inputCol="features",
    outputCol="normalized_features",
    p=2.0  # L2 norm (Euclidean)
)

df_normalized = normalizer.transform(df_vector)

# Method 6: Log Transformation (for skewed data)
from pyspark.sql.functions import log, log1p

df_log = (
    df
    .withColumn("income_log", log1p(col("income")))  # log(1 + x) to handle zeros
    .withColumn("transaction_count_log", log1p(col("transaction_count")))
)
```

### Example 5: Feature Interaction and Polynomial Features

```python
from pyspark.ml.feature import PolynomialExpansion, Interaction, SQLTransformer
from pyspark.sql.functions import col, sqrt, pow as power

df = spark.table("catalog.ml.features")

# Manual interaction features
df_interactions = (
    df
    # Multiplicative interactions
    .withColumn("age_income_interaction", col("age") * col("income"))
    .withColumn("age_balance_interaction", col("age") * col("account_balance"))
    
    # Polynomial features
    .withColumn("age_squared", col("age") ** 2)
    .withColumn("income_squared", col("income") ** 2)
    .withColumn("age_cubed", col("age") ** 3)
    
    # Ratio features
    .withColumn("income_per_age", col("income") / (col("age") + 1))
    .withColumn("balance_income_ratio", col("account_balance") / (col("income") + 1))
    
    # Root transformations
    .withColumn("income_sqrt", sqrt(col("income")))
    
    # Logarithmic transformations
    .withColumn("income_log", log(col("income") + 1))
    .withColumn("balance_log", log(col("account_balance") + 1))
    
    # Domain-specific features
    .withColumn("high_value_young", 
                when((col("age") < 30) & (col("income") > 100000), 1).otherwise(0))
    .withColumn("credit_utilization", 
                col("account_balance") / (col("credit_limit") + 1))
)

# Polynomial features using ML library
assembler = VectorAssembler(
    inputCols=["age", "income", "credit_score"],
    outputCol="input_features"
)

df_vector = assembler.transform(df)

polynomial = PolynomialExpansion(
    degree=2,  # Degree 2: x, x^2, x*y for all pairs
    inputCol="input_features",
    outputCol="poly_features"
)

df_poly = polynomial.transform(df_vector)

# Feature interactions using Interaction transformer
# First, vectorize each feature separately
age_assembler = VectorAssembler(inputCols=["age"], outputCol="age_vector")
income_assembler = VectorAssembler(inputCols=["income"], outputCol="income_vector")
region_assembler = VectorAssembler(inputCols=["region_encoded"], outputCol="region_vector")

df_vectors = (
    df
    .transform(age_assembler.transform)
    .transform(income_assembler.transform)
    .transform(region_assembler.transform)
)

interaction = Interaction(
    inputCols=["age_vector", "income_vector", "region_vector"],
    outputCol="interaction_features"
)

df_interaction = interaction.transform(df_vectors)

# SQL-based feature engineering
sql_transformer = SQLTransformer(
    statement="""
    SELECT *,
        age * income as age_income,
        CASE 
            WHEN age < 30 THEN 'young'
            WHEN age < 50 THEN 'middle'
            ELSE 'senior'
        END as age_group,
        income / (age + 1) as income_per_year
    FROM __THIS__
    """
)

df_sql_features = sql_transformer.transform(df)
```

### Example 6: Feature Selection Methods

```python
from pyspark.ml.feature import VectorAssembler, ChiSqSelector, UnivariateFeatureSelector, VarianceThresholdSelector
from pyspark.ml.stat import Correlation
from pyspark.ml.classification import RandomForestClassifier
import pandas as pd

df = spark.table("catalog.ml.training_data")

feature_cols = ["feature1", "feature2", "feature3", "feature4", "feature5", 
                "feature6", "feature7", "feature8", "feature9", "feature10"]

assembler = VectorAssembler(
    inputCols=feature_cols,
    outputCol="features"
)

df_vector = assembler.transform(df)

# Method 1: Correlation-based selection
correlation_matrix = Correlation.corr(df_vector, "features").head()[0]

# Convert to pandas for easier analysis
corr_pd = pd.DataFrame(
    correlation_matrix.toArray(),
    columns=feature_cols,
    index=feature_cols
)

# Remove highly correlated features (>0.9)
def remove_correlated_features(corr_matrix, threshold=0.9):
    """Remove features with correlation above threshold"""
    to_drop = set()
    for i in range(len(corr_matrix.columns)):
        for j in range(i+1, len(corr_matrix.columns)):
            if abs(corr_matrix.iloc[i, j]) > threshold:
                to_drop.add(corr_matrix.columns[j])
    return list(to_drop)

high_corr_features = remove_correlated_features(corr_pd)
print(f"Dropping highly correlated features: {high_corr_features}")

# Method 2: Chi-Square selector (for categorical features)
chi_selector = ChiSqSelector(
    numTopFeatures=5,  # Select top 5 features
    featuresCol="features",
    outputCol="selected_features",
    labelCol="label"
)

chi_model = chi_selector.fit(df_vector)
df_chi_selected = chi_model.transform(df_vector)

print(f"Selected feature indices: {chi_model.selectedFeatures}")

# Method 3: Univariate Feature Selection
univariate_selector = UnivariateFeatureSelector(
    featuresCol="features",
    outputCol="selected_features",
    labelCol="label",
    selectionMode="fpr",  # False positive rate
    selectionThreshold=0.05
)

univariate_model = univariate_selector.fit(df_vector)
df_univariate_selected = univariate_model.transform(df_vector)

# Method 4: Variance Threshold
variance_selector = VarianceThresholdSelector(
    featuresCol="features",
    outputCol="selected_features",
    varianceThreshold=0.1  # Remove low-variance features
)

variance_model = variance_selector.fit(df_vector)
df_variance_selected = variance_model.transform(df_vector)

# Method 5: Feature Importance from Random Forest
rf = RandomForestClassifier(
    featuresCol="features",
    labelCol="label",
    numTrees=100,
    maxDepth=10
)

rf_model = rf.fit(df_vector)

# Get feature importances
feature_importance = rf_model.featureImportances
importance_dict = dict(zip(feature_cols, feature_importance.toArray()))
sorted_importance = sorted(importance_dict.items(), key=lambda x: x[1], reverse=True)

print("\nFeature Importance:")
for feature, importance in sorted_importance:
    print(f"  {feature}: {importance:.4f}")

# Select top N important features
top_n = 5
top_features = [f[0] for f in sorted_importance[:top_n]]
print(f"\nTop {top_n} features: {top_features}")

# Method 6: Recursive Feature Elimination (manual implementation)
def recursive_feature_elimination(df, feature_cols, label_col, n_features_to_select):
    """
    Simple RFE implementation
    """
    remaining_features = feature_cols.copy()
    
    while len(remaining_features) > n_features_to_select:
        # Train model with current features
        assembler = VectorAssembler(inputCols=remaining_features, outputCol="features")
        df_temp = assembler.transform(df)
        
        rf = RandomForestClassifier(featuresCol="features", labelCol=label_col, numTrees=50)
        rf_model = rf.fit(df_temp)
        
        # Get feature importances
        importances = dict(zip(remaining_features, rf_model.featureImportances.toArray()))
        
        # Remove least important feature
        least_important = min(importances, key=importances.get)
        remaining_features.remove(least_important)
        
        print(f"Removed: {least_important}, Remaining: {len(remaining_features)}")
    
    return remaining_features

selected_features_rfe = recursive_feature_elimination(df, feature_cols, "label", 5)
print(f"\nRFE Selected Features: {selected_features_rfe}")
```

### Example 7: Complete Feature Engineering Pipeline

```python
from pyspark.ml import Pipeline
from pyspark.ml.feature import *
from databricks.feature_store import FeatureStoreClient

class FeatureEngineeringPipeline:
    """
    Complete feature engineering pipeline
    """
    
    def __init__(self, categorical_cols, numerical_cols, target_col):
        self.categorical_cols = categorical_cols
        self.numerical_cols = numerical_cols
        self.target_col = target_col
        self.pipeline = None
    
    def build_pipeline(self):
        """Build comprehensive feature engineering pipeline"""
        
        stages = []
        
        # Handle missing values
        imputer = Imputer(
            inputCols=self.numerical_cols,
            outputCols=[f"{c}_imputed" for c in self.numerical_cols],
            strategy="mean"
        )
        stages.append(imputer)
        
        # Encode categorical variables
        indexers = [
            StringIndexer(
                inputCol=col,
                outputCol=f"{col}_indexed",
                handleInvalid="keep"
            )
            for col in self.categorical_cols
        ]
        stages.extend(indexers)
        
        # One-hot encode
        encoder = OneHotEncoder(
            inputCols=[f"{c}_indexed" for c in self.categorical_cols],
            outputCols=[f"{c}_encoded" for c in self.categorical_cols]
        )
        stages.append(encoder)
        
        # Scale numerical features
        num_assembler = VectorAssembler(
            inputCols=[f"{c}_imputed" for c in self.numerical_cols],
            outputCol="numerical_features"
        )
        stages.append(num_assembler)
        
        scaler = StandardScaler(
            inputCol="numerical_features",
            outputCol="scaled_numerical",
            withMean=True,
            withStd=True
        )
        stages.append(scaler)
        
        # Combine all features
        final_assembler = VectorAssembler(
            inputCols=["scaled_numerical"] + [f"{c}_encoded" for c in self.categorical_cols],
            outputCol="features"
        )
        stages.append(final_assembler)
        
        # Create pipeline
        self.pipeline = Pipeline(stages=stages)
        
        return self.pipeline
    
    def fit_transform(self, df):
        """Fit pipeline and transform data"""
        model = self.pipeline.fit(df)
        transformed_df = model.transform(df)
        return model, transformed_df
    
    def save_to_feature_store(self, df, table_name, primary_keys):
        """Save features to Feature Store"""
        fs = FeatureStoreClient()
        
        fs.create_table(
            name=table_name,
            primary_keys=primary_keys,
            df=df,
            description="Engineered features from pipeline"
        )
        
        print(f"Features saved to {table_name}")


# Usage
categorical_cols = ["customer_segment", "region", "product_category"]
numerical_cols = ["age", "income", "account_balance", "credit_score"]
target_col = "churned"

# Create and run pipeline
fe_pipeline = FeatureEngineeringPipeline(categorical_cols, numerical_cols, target_col)
pipeline = fe_pipeline.build_pipeline()

# Load data
df = spark.table("catalog.ml.raw_customer_data")

# Fit and transform
model, transformed_df = fe_pipeline.fit_transform(df)

# Save to Feature Store
fe_pipeline.save_to_feature_store(
    transformed_df,
    "catalog.features.customer_features_v2",
    ["customer_id"]
)

# Save pipeline model
model.write().overwrite().save("/mnt/models/feature_pipeline")
```

---

## Best Practices

### 1. Feature Store Usage
* Centralize feature definitions in Feature Store
* Version features properly
* Document feature semantics and business logic
* Use point-in-time lookups to avoid data leakage
* Monitor feature drift over time

### 2. Categorical Encoding
* Use one-hot for low cardinality (<10 categories)
* Use target encoding for high cardinality with caution
* Always handle unknown categories (handleInvalid="keep")
* Consider ordinal encoding when natural order exists

### 3. Scaling
* Always scale before distance-based algorithms (KNN, SVM)
* Use StandardScaler for normally distributed features
* Use RobustScaler when outliers are present
* MinMaxScaler when you need bounded ranges (0-1)

### 4. Time-Based Features
* Create cyclical features (sin/cos) for seasonal patterns
* Use lag features cautiously to avoid data leakage
* Calculate rolling statistics for trend capture
* Handle missing historical data gracefully

### 5. Feature Selection
* Remove highly correlated features (>0.9 correlation)
* Use domain knowledge to guide selection
* Validate on holdout set, not training set
* Monitor feature importance over time

### 6. Pipeline Management
* Use ML Pipelines for reproducibility
* Save fitted pipelines for inference
* Version control pipeline definitions
* Test pipelines on sample data first

---

## Common Pitfalls

### ❌ Data Leakage
* **Problem**: Using future information in features
* **Solution**: Use point-in-time joins, careful with lag features

### ❌ Inconsistent Transformations
* **Problem**: Different transformations in training vs inference
* **Solution**: Use ML Pipelines, save fitted transformers

### ❌ Not Handling Missing Values
* **Problem**: Model failures on new data
* **Solution**: Impute or handle explicitly in pipeline

### ❌ Ignoring Feature Drift
* **Problem**: Model performance degrades over time
* **Solution**: Monitor feature distributions, retrain periodically

### ❌ Over-Engineering Features
* **Problem**: Too many features, overfitting
* **Solution**: Use feature selection, focus on signal

---

## Related Skills

* [Model Deployment](../model-deployment/SKILL.md) - For using features in production
* [MLflow Tracking](../mlflow-tracking/SKILL.md) - For tracking experiments
* [Data Modeling](../data-modeling/SKILL.md) - For upstream data preparation
* [Data Quality Checks](../data-quality-checks/SKILL.md) - For feature validation

---

**Last Updated**: 2024-04-09
