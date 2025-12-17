# Comprehensive Examples of Prompt Improvement

This reference provides real-world examples showing the complete flow from vague data analysis prompts through research, question formulation, and execution.

## Example Format

Each example follows this structure:

1. **Original Prompt**: What the user said
2. **Evaluation**: Clarity assessment
3. **Research Plan** (if vague): What to investigate (Inspect data, check schema, etc.)
4. **Research Findings** (if vague): What was discovered (Missing values, outliers, column names)
5. **Questions Asked** (if vague): AskUserQuestion tool call
6. **User Answers** (if vague): What user selected
7. **Execution**: How the request was fulfilled

## Vague Prompts (Research Required)

### Example 1: Generic Data Cleaning

**Original Prompt:**

```
clean the data

```

**Evaluation:**

* Target: ❌ (which dataset? which columns?)
* Action: ❌ (cleaning is broad: handle nulls? duplicates? type conversion?)
* Criteria: ~ (clean for what purpose?)
* Context: Check active files

**Conversation History Check:**
User recently uploaded `customer_churn.csv`.

**Decision:** RESEARCH REQUIRED

**Research Plan:**

```
1. Load the dataframe and check .info() for data types
2. Check for missing values (.isnull().sum())
3. Check for duplicates (.duplicated().sum())
4. Inspect unique values in categorical columns for inconsistencies

```

**Research Execution:**

```python
# Step 1 & 2
df = pd.read_csv('customer_churn.csv')
print(df.info())
print(df.isnull().sum())
# Result: 'TotalCharges' is object (should be float), 150 missing values in 'Age'

# Step 3
print(df.duplicated().sum())
# Result: 23 duplicate rows found

# Step 4
print(df['PaymentMethod'].unique())
# Result: ['Electronic check', 'electronic check', 'Mailed check'] (Inconsistent casing)

```

**Research Findings:**

* `TotalCharges` column is wrong type (Object instead of Float) due to empty strings.
* 150 missing values in `Age` column.
* 23 duplicate rows.
* Inconsistent casing in `PaymentMethod` ('Electronic check' vs 'electronic check').

**Questions Asked:**

```json
{
  "questions": [
    {
      "question": "How should missing values in 'Age' be handled?",
      "header": "Missing Values",
      "multiSelect": false,
      "options": [
        {
          "label": "Fill with Median",
          "description": "Replace NaNs with the median age (robust to outliers). Best for preserving data size."
        },
        {
          "label": "Drop Rows",
          "description": "Remove all rows with missing Age values. Ensures data quality but reduces dataset size."
        },
        {
          "label": "Fill with Mean",
          "description": "Replace NaNs with the average age."
        }
      ]
    },
    {
      "question": "What other cleaning steps should be applied?",
      "header": "Cleaning Steps",
      "multiSelect": true,
      "options": [
        {
          "label": "Remove Duplicates",
          "description": "Drop the 23 duplicate rows found in the dataset."
        },
        {
          "label": "Fix 'TotalCharges' data type",
          "description": "Convert object to numeric, coercing errors to NaN."
        },
        {
          "label": "Standardize 'PaymentMethod'",
          "description": "Fix inconsistent casing (e.g., 'electronic check' -> 'Electronic check')."
        }
      ]
    }
  ]
}

```

**User Answer:**

1. "Fill with Median"
2. ["Remove Duplicates", "Fix 'TotalCharges' data type", "Standardize 'PaymentMethod'"]

**Execution:**
Execute Pandas script to fill NaNs with median, drop duplicates, `pd.to_numeric` on charges, and apply string title casing.

---

### Example 2: Ambiguous Visualization Request

**Original Prompt:**

```
plot the sales trends

```

**Evaluation:**

* Target: ❌ (which dataset? which time column?)
* Action: ✅ (plot)
* Criteria: ❌ (frequency? breakdown by category?)
* Context: `sales_data_2024.csv` is in the workspace.

**Research Plan:**

```
1. Inspect columns in sales_data_2024.csv
2. Identify datetime columns and numerical columns
3. Check distinct categories (Region, Product)

```

**Research Execution:**

```python
# Analysis
Columns: ['order_date', 'product_category', 'region', 'sales_amount', 'quantity']
Date range: 2024-01-01 to 2024-12-31
Categories: 5 main product categories, 4 regions

```

**Research Findings:**

* Dataset covers full year 2024.
* `order_date` is available for time series.
* Can aggregate by Sum or Mean.
* Can group by Category or Region.

**Questions Asked:**

```json
{
  "questions": [
    {
      "question": "What time granularity do you prefer?",
      "header": "Time Frequency",
      "multiSelect": false,
      "options": [
        {
          "label": "Monthly",
          "description": "Aggregate sales by month. Good for identifying seasonality."
        },
        {
          "label": "Weekly",
          "description": "Aggregate sales by week. Better for detailed operational analysis."
        },
        {
          "label": "Daily",
          "description": "Raw daily data. Can be noisy but shows exact peaks."
        }
      ]
    },
    {
      "question": "Do you want to break down the trend?",
      "header": "Grouping",
      "multiSelect": false,
      "options": [
        {
          "label": "No, Total Sales only",
          "description": "Single line chart showing global sales trend."
        },
        {
          "label": "By Product Category",
          "description": "Multi-line chart comparing different product categories over time."
        },
        {
          "label": "By Region",
          "description": "Multi-line chart comparing regional performance."
        }
      ]
    }
  ]
}

```

**User Answers:**

1. "Monthly"
2. "By Product Category"

**Execution:**
Resample data to Monthly frequency, group by Category, sum `sales_amount`, and generate a multi-line Matplotlib/Seaborn plot.

---

### Example 3: Vague SQL Optimization

**Original Prompt:**

```
optimize this query

```

**Evaluation:**

* Target: ✅ (The user likely pasted a SQL query or highlighted one)
* Action: ✅ (optimize)
* Criteria: ❌ (optimize for read speed? write speed? cost?)
* Context: User provided a complex SQL block.

**Context Check:**
User provided:

```sql
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.signup_date > '2023-01-01'
ORDER BY o.amount DESC

```

**Research Plan:**

```
1. Analyze query structure (SELECT *, Joins, Where clauses)
2. Check schema/indexes if available (simulated or look at schema.sql)
3. Identify potential bottlenecks (Full Table Scans, large result sets)

```

**Research Findings:**

* `SELECT *` fetches unnecessary columns.
* Filtering on `u.signup_date`.
* Sorting by `o.amount`.
* Join key `user_id` usually indexed, but `signup_date` might not be.

**Questions Asked:**

```json
{
  "questions": [
    {
      "question": "What is the primary goal of the optimization?",
      "header": "Optimization Goal",
      "multiSelect": false,
      "options": [
        {
          "label": "Reduce Execution Time",
          "description": "Focus on indexing and reducing scan scope."
        },
        {
          "label": "Reduce Network/Data Load",
          "description": "Focus on selecting only necessary columns instead of SELECT *."
        }
      ]
    },
    {
      "question": "Can we assume indexes exist on these columns?",
      "header": "Index Availability",
      "multiSelect": true,
      "options": [
        {
          "label": "Index on users(signup_date)",
          "description": "Crucial for the WHERE clause filtering."
        },
        {
          "label": "Index on orders(user_id, amount)",
          "description": "Composite index to help with JOIN and ORDER BY."
        }
      ]
    }
  ]
}

```

**User Answers:**

1. "Reduce Network/Data Load"
2. ["Index on users(signup_date)"]

**Execution:**
Rewrite query to specify columns (e.g., `o.id, o.amount, u.name`) instead of `*`, and provide the `CREATE INDEX` statement for `signup_date` as a recommendation.

---

## Clear Prompts (Proceed Immediately)

### Example 4: Specific Pandas Transformation

**Original Prompt:**

```
Convert the 'date_created' column to datetime objects and filter for rows where 'status' is 'active' in df_main

```

**Evaluation:**

* Target: ✅ (`df_main`)
* Action: ✅ (convert type, filter)
* Criteria: ✅ (specific column names and values provided)
* Context: Not needed

**Decision:** PROCEED IMMEDIATELY (no research or questions)

**Execution:**

```python
df_main['date_created'] = pd.to_datetime(df_main['date_created'])
filtered_df = df_main[df_main['status'] == 'active']

```

---

### Example 5: Specific Metric Calculation

**Original Prompt:**

```
Calculate the daily average Request Count and Token Usage per model from the usage_logs.csv

```

**Evaluation:**

* Target: ✅ (`usage_logs.csv`)
* Action: ✅ (groupby and mean aggregation)
* Criteria: ✅ (Daily frequency, specific metrics: Request Count, Token Usage, grouping by Model)
* Context: Clear data analysis task

**Decision:** PROCEED IMMEDIATELY

**Execution:**

```python
df = pd.read_csv('usage_logs.csv')
df['date'] = pd.to_datetime(df['timestamp']).dt.date
daily_stats = df.groupby(['date', 'model'])[['request_count', 'token_usage']].mean().reset_index()

```

---

### Example 6: Clear Export Request

**Original Prompt:**

```
Save the resulting dataframe to 'analysis_results.xlsx', sheet name 'Summary', without the index

```

**Evaluation:**

* Target: ✅ (current dataframe)
* Action: ✅ (export to Excel)
* Criteria: ✅ (Filename, Sheet Name, Index=False specified)
* Context: Clear output requirement

**Decision:** PROCEED IMMEDIATELY

**Execution:**

```python
df.to_excel('analysis_results.xlsx', sheet_name='Summary', index=False)

```

---

## Bypass Prompts (Pass Through)

### Example 7: Explanation Request

**Original Prompt:**

```
* explain what the .groupby code block above is doing

```

**Detection:**
Bypass prefix `*` detected

**Action:**
Strip `*`, pass through as general conversation/explanation request.

**Execution:**
Provide natural language explanation of the Pandas logic.

---

## Context-Dependent Prompts

### Example 8: Notebook Cell Context

**Context:**

```
[System: User is viewing a Jupyter Notebook. Cell 5 threw a ValueError: "x and y must be the same size"]

```

**Prompt:**

```
fix this error

```

**Evaluation:**

* Target: ✅ (Cell 5 code)
* Action: ✅ (Fix array shape mismatch)
* Criteria: ✅ (Code runs without error)
* Context: ✅ (Error message explains the dimension mismatch)

**Decision:** PROCEED IMMEDIATELY

**Execution:**
Check the lengths of arrays passed to the plotting function (e.g., `len(x)` vs `len(y)`), truncate or pad them to match, or fix the data generation logic.

---

### Example 9: Data Preview Context

**Previous Message:**

```
Assistant: "I have loaded the data. columns are: ['id', 'revenue', 'cost', 'profit_margin']"

```

**Current Prompt:**

```
show me the distribution of the last one

```

**Evaluation:**

* Target: ✅ (`profit_margin` column)
* Action: ✅ (Visualize distribution)
* Criteria: ~ (Histogram is the standard default)
* Context: ✅ ("the last one" refers to `profit_margin` from previous turn)

**Decision:** PROCEED (context makes this clear)

**Execution:**
Plot a histogram of the `profit_margin` column using `plt.hist()` or `sns.histplot()`.

---

## Complex Multi-Question Scenarios

### Example 10: Building a Prediction Model

**Original Prompt:**

```
build a model to predict house prices

```

**Evaluation:**

* Target: ❌ (which features?)
* Action: ❌ (Model building is complex)
* Criteria: ❌ (Accuracy metric? Algorithm preference?)
* Context: `housing_data.csv` is available.

**Research Plan:**

```
1. Check correlation matrix for feature selection
2. Check target variable distribution ('price')
3. Identify categorical variables needing encoding

```

**Research Findings:**

* Target: `price`.
* Features: `sqft`, `bedrooms`, `zipcode` (categorical), `year_built`.
* Strong correlation with `sqft`.

**Questions Asked:**

```json
{
  "questions": [
    {
      "question": "Which algorithm should we use?",
      "header": "Model Selection",
      "multiSelect": false,
      "options": [
        {
          "label": "Linear Regression",
          "description": "Simple, interpretable baseline. Good for understanding relationships."
        },
        {
          "label": "Random Forest Regressor",
          "description": "Non-linear, robust to outliers, usually higher accuracy but less interpretable."
        },
        {
          "label": "XGBoost",
          "description": "High performance, gradient boosting. Industry standard for tabular data competitions."
        }
      ]
    },
    {
      "question": "How should we handle the categorical 'zipcode' column?",
      "header": "Feature Engineering",
      "multiSelect": false,
      "options": [
        {
          "label": "One-Hot Encoding",
          "description": "Create binary columns for each zip code. Can create high dimensionality."
        },
        {
          "label": "Target Encoding",
          "description": "Replace zip code with the average house price of that area. Efficient but risks data leakage."
        },
        {
          "label": "Drop Column",
          "description": "Exclude location data if cardinality is too high."
        }
      ]
    },
    {
      "question": "What is the evaluation metric?",
      "header": "Metric",
      "multiSelect": false,
      "options": [
        {
          "label": "RMSE (Root Mean Squared Error)",
          "description": "Penalizes large errors heavily. Standard for price prediction."
        },
        {
          "label": "MAE (Mean Absolute Error)",
          "description": "Average dollar amount error. Easier to explain to stakeholders."
        }
      ]
    }
  ]
}

```

**User Answers:**

1. "Random Forest Regressor"
2. "One-Hot Encoding"
3. "MAE"

**Execution:**
Preprocess data (One-Hot encode zipcodes), split train/test sets, train `RandomForestRegressor`, and calculate MAE on the test set.

---

### Summary: Decision Patterns (Data Analysis)

### Proceed Immediately If:

* Dataset, columns, and specific operation (groupby, filter, plot type) are named.
* Logic is mathematically clear (e.g., "calculate mean").
* Context provides the specific dataframe or SQL query.

### Research and Ask If:

* Verbs are subjective ("clean", "analyze", "optimize", "fix").
* Aggregation level is missing (Daily vs Monthly).
* Missing value handling strategy is not defined.
* Visualization type is not specified for generic data.

### Pass Through If:

* Request implies explanation rather than code execution.
* User explicitly asks to bypass checks (`*`).
