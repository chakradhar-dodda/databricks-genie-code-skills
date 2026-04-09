# Testing Strategies Skill

## Purpose
This skill teaches Genie Code how to implement comprehensive testing strategies for Databricks pipelines including unit tests, integration tests, data validation tests, pipeline testing, and regression testing for reliable data systems.

## When to Use
* Building production data pipelines
* Implementing CI/CD for data projects
* Validating data transformations
* Testing business logic
* Preventing regression issues
* Ensuring data quality
* Documenting expected behavior

## Key Concepts

### 1. Testing Strategy Framework
* **Situation**: Current test coverage and gaps
* **Motivation**: Business impact of bugs
* **Restrictions**: Test requirements and standards (R)
* **Evaluation**: Test metrics and coverage (E = Evaluation)
* **Decisions**: When to test and when to ship (D)

### 2. Test Pyramid
* **Unit Tests**: Test individual functions (fast, many)
* **Integration Tests**: Test component interactions (medium)
* **End-to-End Tests**: Test complete workflows (slow, few)
* **Data Quality Tests**: Validate data properties

### 3. Testing Patterns
* **Test-Driven Development (TDD)**: Write tests first
* **Behavior-Driven Development (BDD)**: Test business scenarios
* **Property-Based Testing**: Test invariants
* **Snapshot Testing**: Compare outputs

### 4. Test Data Management
* **Fixtures**: Reusable test data
* **Mocking**: Simulate dependencies
* **Sampling**: Representative subsets
* **Synthetic Data**: Generated test data

---

## Examples

### Example 1: Unit Testing Data Transformations

```python
import pytest
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, year
from datetime import datetime

class TestDataTransformations:
    """Unit tests for data transformation functions"""
    
    @pytest.fixture(scope="class")
    def spark(self):
        """Create Spark session for testing"""
        return SparkSession.builder \\
            .appName("unit-tests") \\
            .master("local[2]") \\
            .getOrCreate()
    
    def test_clean_customer_data(self, spark):
        """Test customer data cleaning function"""
        # Arrange: Create test data
        test_data = [
            (1, " John Doe ", "john@example.com", "2023-01-15"),
            (2, "Jane Smith", "INVALID_EMAIL", "2023-02-20"),
            (3, None, "bob@example.com", "2023-03-10")
        ]
        df = spark.createDataFrame(
            test_data,
            ["customer_id", "name", "email", "signup_date"]
        )
        
        # Act: Apply transformation
        result = clean_customer_data(df)
        
        # Assert: Verify results
        assert result.count() == 2  # Row with null name filtered out
        
        # Check name trimming
        john = result.filter(col("customer_id") == 1).collect()[0]
        assert john.name == "John Doe"  # Leading/trailing spaces removed
        
        # Check invalid email filtering
        assert result.filter(col("customer_id") == 2).count() == 0
        
        print("✅ test_clean_customer_data passed")
    
    def test_calculate_customer_lifetime_value(self, spark):
        """Test CLV calculation"""
        # Arrange
        test_orders = [
            (1, "2023-01-01", 100.0),
            (1, "2023-02-01", 150.0),
            (2, "2023-01-15", 200.0)
        ]
        orders_df = spark.createDataFrame(
            test_orders,
            ["customer_id", "order_date", "amount"]
        )
        
        # Act
        result = calculate_customer_lifetime_value(orders_df)
        
        # Assert
        customer_1 = result.filter(col("customer_id") == 1).collect()[0]
        assert customer_1.lifetime_value == 250.0  # 100 + 150
        assert customer_1.order_count == 2
        
        customer_2 = result.filter(col("customer_id") == 2).collect()[0]
        assert customer_2.lifetime_value == 200.0
        assert customer_2.order_count == 1
        
        print("✅ test_calculate_customer_lifetime_value passed")
    
    def test_aggregate_sales_by_month(self, spark):
        """Test monthly sales aggregation"""
        # Arrange
        test_sales = [
            ("2023-01-15", "product_a", 100.0),
            ("2023-01-20", "product_a", 150.0),
            ("2023-02-10", "product_a", 200.0),
            ("2023-01-25", "product_b", 300.0)
        ]
        sales_df = spark.createDataFrame(
            test_sales,
            ["sale_date", "product", "amount"]
        )
        
        # Act
        result = aggregate_sales_by_month(sales_df)
        
        # Assert
        jan_product_a = result.filter(
            (col("month") == "2023-01") & (col("product") == "product_a")
        ).collect()[0]
        assert jan_product_a.total_sales == 250.0  # 100 + 150
        
        jan_product_b = result.filter(
            (col("month") == "2023-01") & (col("product") == "product_b")
        ).collect()[0]
        assert jan_product_b.total_sales == 300.0
        
        print("✅ test_aggregate_sales_by_month passed")

# Run tests
pytest.main([__file__, "-v"])
```

### Example 2: Integration Testing Pipelines

```python
class TestCustomerPipeline:
    """Integration tests for complete customer data pipeline"""
    
    @pytest.fixture(scope="class")
    def setup_test_environment(self, spark):
        """Set up test tables and data"""
        # Create test bronze table
        bronze_data = [
            (1, "John Doe", "john@example.com", "2023-01-15", "active"),
            (2, "Jane Smith", "jane@example.com", "2023-02-20", "active"),
            (3, "Bob Jones", "bob@example.com", "2023-03-10", "inactive")
        ]
        bronze_df = spark.createDataFrame(
            bronze_data,
            ["id", "name", "email", "signup_date", "status"]
        )
        bronze_df.write.mode("overwrite").saveAsTable("test_bronze.customers")
        
        # Create test orders table
        orders_data = [
            (1, 1, "2023-01-20", 100.0),
            (2, 1, "2023-02-15", 150.0),
            (3, 2, "2023-03-01", 200.0)
        ]
        orders_df = spark.createDataFrame(
            orders_data,
            ["order_id", "customer_id", "order_date", "amount"]
        )
        orders_df.write.mode("overwrite").saveAsTable("test_bronze.orders")
        
        yield
        
        # Cleanup
        spark.sql("DROP TABLE IF EXISTS test_bronze.customers")
        spark.sql("DROP TABLE IF EXISTS test_bronze.orders")
        spark.sql("DROP TABLE IF EXISTS test_silver.customers")
        spark.sql("DROP TABLE IF EXISTS test_gold.customer_metrics")
    
    def test_bronze_to_silver_pipeline(self, spark, setup_test_environment):
        """Test bronze to silver transformation"""
        # Act: Run bronze to silver pipeline
        run_bronze_to_silver_pipeline(
            source_table="test_bronze.customers",
            target_table="test_silver.customers"
        )
        
        # Assert: Verify silver table
        silver_df = spark.table("test_silver.customers")
        
        # Check record count
        assert silver_df.count() == 2  # Inactive customers filtered out
        
        # Check data quality
        assert silver_df.filter(col("email").isNull()).count() == 0
        assert silver_df.filter(col("name").contains("  ")).count() == 0
        
        # Check schema
        expected_cols = {"customer_id", "name", "email", "signup_date", "processed_at"}
        actual_cols = set(silver_df.columns)
        assert expected_cols.issubset(actual_cols)
        
        print("✅ test_bronze_to_silver_pipeline passed")
    
    def test_end_to_end_customer_metrics(self, spark, setup_test_environment):
        """Test complete pipeline: bronze -> silver -> gold"""
        # Act: Run complete pipeline
        run_bronze_to_silver_pipeline(
            source_table="test_bronze.customers",
            target_table="test_silver.customers"
        )
        
        run_silver_to_gold_metrics(
            customers_table="test_silver.customers",
            orders_table="test_bronze.orders",
            target_table="test_gold.customer_metrics"
        )
        
        # Assert: Verify gold table
        gold_df = spark.table("test_gold.customer_metrics")
        
        # Check record count
        assert gold_df.count() == 2
        
        # Check customer 1 metrics
        customer_1 = gold_df.filter(col("customer_id") == 1).collect()[0]
        assert customer_1.total_orders == 2
        assert customer_1.lifetime_value == 250.0  # 100 + 150
        assert customer_1.avg_order_value == 125.0
        
        # Check customer 2 metrics
        customer_2 = gold_df.filter(col("customer_id") == 2).collect()[0]
        assert customer_2.total_orders == 1
        assert customer_2.lifetime_value == 200.0
        
        print("✅ test_end_to_end_customer_metrics passed")
```

### Example 3: Data Quality Testing with Great Expectations

```python
import great_expectations as ge
from great_expectations.dataset import SparkDFDataset

def test_customer_data_quality():
    """
    Data quality tests using Great Expectations
    
    Restrictions: Define data quality requirements
    """
    
    # Load data as GE dataset
    df = spark.table("gold.customers")
    ge_df = SparkDFDataset(df)
    
    # Test 1: Column existence
    assert ge_df.expect_column_to_exist("customer_id").success
    assert ge_df.expect_column_to_exist("name").success
    assert ge_df.expect_column_to_exist("email").success
    
    # Test 2: No null values in key columns
    assert ge_df.expect_column_values_to_not_be_null("customer_id").success
    assert ge_df.expect_column_values_to_not_be_null("email").success
    
    # Test 3: Unique customer IDs
    assert ge_df.expect_column_values_to_be_unique("customer_id").success
    
    # Test 4: Email format
    assert ge_df.expect_column_values_to_match_regex(
        "email",
        r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
    ).success
    
    # Test 5: Status values
    assert ge_df.expect_column_values_to_be_in_set(
        "status",
        ["active", "inactive", "pending"]
    ).success
    
    # Test 6: Lifetime value range
    result = ge_df.expect_column_values_to_be_between(
        "lifetime_value",
        min_value=0,
        max_value=1000000
    )
    assert result.success
    
    # Test 7: Recent data freshness
    result = ge_df.expect_column_max_to_be_between(
        "last_order_date",
        min_value=datetime.now() - timedelta(days=90)
    )
    assert result.success
    
    print("✅ All data quality tests passed")

# Run data quality tests
test_customer_data_quality()
```

### Example 4: Regression Testing

```python
class TestPipelineRegression:
    """Regression tests to prevent breaking changes"""
    
    def test_output_schema_stability(self, spark):
        """Ensure output schema hasn't changed"""
        # Arrange: Load expected schema
        expected_schema = {
            "customer_id": "bigint",
            "name": "string",
            "email": "string",
            "lifetime_value": "decimal(10,2)",
            "order_count": "bigint",
            "last_order_date": "date"
        }
        
        # Act: Get actual schema
        df = spark.table("gold.customer_metrics")
        actual_schema = {field.name: field.dataType.simpleString() 
                        for field in df.schema.fields}
        
        # Assert: Schema hasn't changed
        for col_name, expected_type in expected_schema.items():
            assert col_name in actual_schema, f"Missing column: {col_name}"
            assert actual_schema[col_name] == expected_type, \\
                f"Type mismatch for {col_name}: expected {expected_type}, got {actual_schema[col_name]}"
        
        print("✅ test_output_schema_stability passed")
    
    def test_historical_data_consistency(self, spark):
        """Verify pipeline produces consistent results on historical data"""
        # Arrange: Load historical snapshot
        historical_snapshot = spark.table("test_snapshots.customer_metrics_2024_01_01")
        
        # Act: Rerun pipeline on same date range
        current_output = run_pipeline_for_date_range(
            start_date="2024-01-01",
            end_date="2024-01-01"
        )
        
        # Assert: Results match historical snapshot
        diff = historical_snapshot.exceptAll(current_output)
        assert diff.count() == 0, f"Found {diff.count()} rows that differ from historical snapshot"
        
        print("✅ test_historical_data_consistency passed")
    
    def test_performance_regression(self, spark):
        """Ensure pipeline performance hasn't degraded"""
        import time
        
        # Arrange: Load test dataset
        test_df = spark.table("test_data.large_customer_dataset")  # 1M rows
        
        # Act: Measure execution time
        start_time = time.time()
        result = apply_transformations(test_df)
        result.count()  # Trigger computation
        execution_time = time.time() - start_time
        
        # Assert: Execution within acceptable time
        max_execution_time = 30  # seconds
        assert execution_time < max_execution_time, \\
            f"Pipeline took {execution_time:.1f}s, exceeds threshold of {max_execution_time}s"
        
        print(f"✅ test_performance_regression passed ({execution_time:.1f}s)")
```

### Example 5: Test Data Management

```python
class TestDataFixtures:
    """Reusable test data fixtures"""
    
    @staticmethod
    def create_sample_customers(spark, count=100):
        """Generate synthetic customer data"""
        from faker import Faker
        fake = Faker()
        
        customers = [
            (
                i,
                fake.name(),
                fake.email(),
                fake.date_between(start_date='-2y', end_date='today'),
                fake.random_element(elements=('active', 'inactive'))
            )
            for i in range(1, count + 1)
        ]
        
        return spark.createDataFrame(
            customers,
            ["customer_id", "name", "email", "signup_date", "status"]
        )
    
    @staticmethod
    def create_sample_orders(spark, customer_ids, orders_per_customer=5):
        """Generate synthetic order data"""
        from faker import Faker
        fake = Faker()
        
        orders = []
        order_id = 1
        
        for customer_id in customer_ids:
            for _ in range(orders_per_customer):
                orders.append((
                    order_id,
                    customer_id,
                    fake.date_between(start_date='-1y', end_date='today'),
                    round(fake.random.uniform(10.0, 500.0), 2)
                ))
                order_id += 1
        
        return spark.createDataFrame(
            orders,
            ["order_id", "customer_id", "order_date", "amount"]
        )
    
    @staticmethod
    def create_test_environment(spark):
        """Set up complete test environment"""
        # Create customers
        customers_df = TestDataFixtures.create_sample_customers(spark, count=100)
        customers_df.write.mode("overwrite").saveAsTable("test.customers")
        
        # Create orders
        customer_ids = [row.customer_id for row in customers_df.select("customer_id").collect()]
        orders_df = TestDataFixtures.create_sample_orders(spark, customer_ids)
        orders_df.write.mode("overwrite").saveAsTable("test.orders")
        
        print("✅ Test environment created")
        return customers_df, orders_df

# Usage
customers_df, orders_df = TestDataFixtures.create_test_environment(spark)
```

### Example 6: CI/CD Integration

```python
# conftest.py - Pytest configuration
import pytest
from pyspark.sql import SparkSession

def pytest_configure(config):
    """Configure pytest"""
    config.addinivalue_line(
        "markers", "integration: mark test as integration test"
    )
    config.addinivalue_line(
        "markers", "slow: mark test as slow running"
    )

@pytest.fixture(scope="session")
def spark():
    """Create Spark session for all tests"""
    return SparkSession.builder \\
        .appName("databricks-tests") \\
        .master("local[4]") \\
        .config("spark.sql.shuffle.partitions", "4") \\
        .getOrCreate()

# Run tests in CI/CD
# pytest tests/ -v --tb=short --junit-xml=test-results.xml
```

---

## Best Practices

### 1. Test Strategy 
* **Situation**: Assess current test coverage
* **Motivation**: Understand cost of bugs
* **Restrictions**: Define test requirements (R)
* **Evaluation**: Measure test metrics (E)
* **Decisions**: Balance speed vs. coverage (D)

### 2. Test Organization
* Separate unit, integration, and E2E tests
* Use descriptive test names
* Keep tests independent
* Use fixtures for setup/teardown

### 3. Test Data
* Use small, focused datasets
* Generate synthetic data
* Version control test fixtures
* Clean up after tests

### 4. Continuous Testing
* Run unit tests on every commit
* Run integration tests nightly
* Run E2E tests before release
* Track test metrics over time

### 5. Test Maintenance
* Keep tests fast
* Remove flaky tests
* Update tests with code changes
* Review test failures promptly

---

## Related Skills

* [Data Contracts](../data-contracts/SKILL.md) - For quality requirements
* [Monitoring Observability](../monitoring-observability/SKILL.md) - For test monitoring
* [Documentation Practices](../documentation-practices/SKILL.md) - For test documentation

---

**Last Updated**: 2026-04-09
