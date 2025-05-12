# 📖 Bill's Adventure: Building a dbt Model to Find His Porto Paradise

## 🌍 Objective

Meet **Bill**, an avid traveler with a love for coastal cities, great food, and comfy lodgings. This summer, Bill has one mission: **find the perfect place to stay in Porto**. But instead of browsing reviews manually, Bill decides to harness the power of **data modeling with dbt**. This playbook documents Bill's journey step by step using:

- ✨ VS Code (with Power User extension)
- 💾 dbt-core (on Microsoft Windows)
- ❄️ Snowflake
- 🏃 Snippets and custom macros
- 🧠 AI-assisted schema generation

By the end of this guide, Bill not only finds the best stay, but you’ll have learned the full data modeling pipeline in dbt.

---

## 👣 Step 1: Preparation - Installing Tools for the Adventure

#### 📂 Step 1: Update the Existing dbt Project

Assume a project is already set up. For HW we can navigate to the root of our dbt project `dbt-analytics`.

> [!TIP] 💡
> After running `git reset --hard origin/HEAD`, Git resets to the latest commit on the default branch (main). You can verify this by checking the default branch on GitHub.

![alt text](image.png)

```bash
git checkout main                        # Switch to the main branch
git pull                                 # Get the latest changes from remote
git fetch                                # Update your references to remote branches
git reset --hard origin/HEAD             # Force reset local main to match remote HEAD
git checkout -b AI-testing-<your_alias>  # Create a new branch for your work

```

Refer to [CONTRIBUTING.md](http://contributing.md/) for branch naming standards.

#### 👁 Step 2: Install dbt Power User Extension

1. Open **VS Code**
2. Go to Extensions tab
3. Search for `dbt Power User` by Altimate Inc. (Over 300K downloads, ✨ 5-star rating!)
4. Click **Install** 

More info of `dbt Power User` [here](https://marketplace.visualstudio.com/items?itemName=innoverio.vscode-dbt-power-user)


> [!NOTE]
> Your sql files in the repo should have the dbt orange logo next to it. In the case it not you can enforce sql file to use jinja-sql by going to VSCode settings or press `ctrl + ,` and navigate to Text Editor > Files to change/add: Item=*.sql and Value=jinja-sql    
<details>
<summary>Manual VS Code override screenshot</summary>

![alt text](image-1.png)

</details>

#### 🌟 Step 3: Add Custom Snippets

In your keyboard: 
- press `ctrl + shift + p ` type snippets
- select `Configure Snippets` from dropdown
- type `jinja-sql.json` to select it from the list
- add the following json text inside { } in `jinja-sql.json` file


<details>
<summary>HW Basic Snippets</summary>

```json
"HW Macro: Limit Data Processing": {
  "prefix": "hw-macro-limit-data",
  "body": [
    "{{ limit_data_processing(",
    "    date_column='${1:event_time}',",
    "    in_filters={'${2:country}': ['${3:US}', '${4:CA}'], '${5:brand}': ['${6:Nike}', '${7:Adidas}']}",
    ") }}"
  ],
  "description": "HW macro for limiting dev data processing (filters + date)"
},
"HW Config: Table Materialization": {
  "prefix": "hw-config-table",
  "body": [
    "{{",
    "  config(",
    "    materialized='table',",
    "    tags=['${1:DBT}']",
    "  )",
    "}}"
  ],
  "description": "HW config block for standard table model"
},
"HW Template: Model Template": {
  "prefix": "hw-template-model",
  "body": [
    "{{",
    "  config(",
    "    materialized='table',",
    "    tags=['DBT']",
    "  )",
    "}}",
    "",
    "WITH source AS (",
    "  SELECT *",
    "    FROM {{ ref('model') }}",
    ")",
    "",
    "SELECT * FROM source"
    "-- model: {{ this.name }}, generated on {{ run_started_at }}"
    ],
  "description": "HW template block for standard model"
}

```

> [!NOTE]
> This is not going to work if your sql file is not set to `jinja-sql` based on step 2.
> If for some reason you do not want to use `jinja-sql` make sure you add the snippet based on your desired association e.g.`sql` or `snowflake-sql`

</details>

---

## 👣 Step 2: Loading the Data - Seeding Bill's Options

In the dbt-analytics repo, we have: **test_accommodations_seed_v2.csv** inside the `/seeds` folder. Execute the following command to load the seed data in our warehouse.

```bash
dbt seed -s test_accommodations_seed_v2 --full-refresh

```

---

## 👣 Step 3: Model-Building - Bill Filters the Noise

Bill creates models to refine his options. Here’s how they unfold:

#### 🧰 Staging Model: `stg_accommodations.sql`

- Applies filters using `limit_data_processing`, `regex_match`, and `convert_date_string` macros. If you have not seen them yet, check them out in `/macros/utils` folder.

- Create a new model under `/models/staging/test` lets name the file `stg_accommodations.sql`
- Before copying the sql, lets try to use the config block snippet. At the top of the sql file type `hw` and select `hw-config-table` to generate the config boilerplate. Then copy the following sql:

```sql
WITH source AS (
    SELECT * FROM {{ ref('test_accommodations_seed_v2') }}
),
converted AS (
    SELECT *, {{ convert_date_string('available_from', '%Y-%m-%d') }} AS available_date
    FROM source
),
dev_limited AS (
    {{ limit_data_processing(
        date_column='available_date',
        in_filters={'country': ['PT'], 'brand': ['Local', 'BlueChain']}
    ) }}
)

SELECT * FROM dev_limited
WHERE {{ regex_match('name', '^Porto') }}
-- model: {{ this.name }}, generated on {{ run_started_at }}
```

#### 🔄 Intermediate Model: `int_best_accommodations.sql`

- Uses `bin_value` to categorize prices
- Create a new model under `/models/intermediate/test` lets name the file `int_best_accommodations.sql`

```sql
{% set rating = 4.5 %}

WITH source AS (
SELECT
    accommodation_id, name, type,
    {{ bin_value('price') }} AS price_bucket,
    rating, location, brand
FROM {{ ref('stg_accommodations') }}
WHERE rating >= {{ ratings }}
)

SELECT * FROM source
-- model: {{ this.name }}, generated on {{ run_started_at }}
```

- In the terminal execute `trunk lint models/intermediate/test/int_best_accommodations.sql`
- Then execute `trunk fix models/intermediate/test/int_best_accommodations.sql`

#### 🏨 Final Mart: `final_best_porto_stay.sql`

- Focuses on affordable, top-rated Porto stays
- Save the `final_best_porto_stay.sql` in `/models/marts/test`

```sql
WITH source AS (
SELECT name, type, price_bucket, rating, brand
FROM {{ ref('int_best_accommodations') }}
WHERE location = 'Porto'
ORDER BY price_bucket
LIMIT 10
)
SELECT * FROM source
-- model: {{ this.name }}, generated on {{ run_started_at }}
```

---

## 👣 Step 4: Testing & Inspecting - Bill Validates the Truth

#### 🔨 Run & Compile

```bash
dbt compile -s stg_accommodations
# Compiles the model

dbt run -s stg_accommodations
# Executes the model and builds in Snowflake

```

Now let's try the **dbt Power User**, on the top right of an sql file you will find the following buttons:
- by selecting one of the icons you are executing a dbt command without using the terminal
<details>
<summary>dbt Power User Pannel</summary>

![alt text](image-2.png)
</details>

#### 🧰 dbt Power User Buttons (Top Right in VS Code)

| Button | Button Name                | Description                                                                  |
|--------|----------------------------|-------------------------------------------------------------------------------|
| 🏃‍♂️     | Run dbt model              | Runs `dbt run -s <current model>` to compile and execute the model.          |
| ✅     | Test dbt model             | Runs `dbt test -s <current model>` to apply all associated tests.            |
| 📊     | Compiled dbt Preview model | Runs `dbt compile -s <current model>` to compile model.                      |
| ▶️     | Execute dbt SQL            | Runs `dbt show -s <current model> limit N` to execute model with limit.      |
| 🔨     | Build dbt model            | Runs `dbt build -s <current model>` to run, test, and snapshot if applicable.|


#### 🔍 Inspect Query Plan in Snowflake

1. Open Snowflake Console
2. Go to **Monitoring > Query History**
3. Select **Filters** query → **SQL Text** and type `model: <current model>`
4. Select the query filtered from the list and Click **Query Profile**
5. Click **Create Table** box and **Step 2** botton

- Here you will find how your query plan execution of the model.

#### 📊 Retrieve Table Schema
In Snowflake execute the sql command:

```sql
SELECT 
    cols.table_schema,
    cols.table_name,
    tbls.table_type,        -- 'BASE TABLE' or 'VIEW'
    cols.column_name,
    cols.data_type
FROM dbt_analytics.information_schema.columns cols
JOIN dbt_analytics.information_schema.tables tbls
  ON cols.table_catalog = tbls.table_catalog
 AND cols.table_schema  = tbls.table_schema
 AND cols.table_name    IN ('stg_accommodations', 'int_best_accommodations', 'final_best_porto_stay', 'accommodations_seed_v2')
 WHERE cols.table_schema = 'DEV' 
ORDER BY cols.table_schema, cols.table_name, cols.ordinal_position;

```

Download the output and feed it to AI with a prompt:
<details>
<summary>YAML Prompt:</summary>

Add the Schema after `Columns Metadata:` and add some context of the model at `Contex:`
```
I have a Snowflake tables or views I want to document in dbt.

Please generate a valid `<model>.yml` snippet for the model based on the metadata below.
Model Name: `fct_customer_engagement`
**Context:**
This model is used as an input to calculate downstream conversion rates by campaign channel.” This is not part of the description but is provided to help you understand how the model is used.

Columns Metadata:
TABLE_SCHEMA | TABLE_NAME	| TABLE_TYPE	| COLUMN_NAME	| DATA_TYPE


**Instructions:**

- Output a valid `<TABLE_NAME>.yml` snippet following dbt format.
- Use `description: |` blocks for both the model and each column.
- For each column:
    - Add a `data_type:` field.
    - Use the `COMMENT` if available for the `description`.
    - If no `COMMENT`, write a smart default based on the column name and model context.
- Do not include tests or meta tags at this time.

```

</details>

> Expect a yaml file like.

#### 🔜 Create Schema YAML

```yaml
version: 2
models:
  - name: stg_accommodations
    columns:
      - name: accommodation_id
        tests:
          - not_null
      - name: accommodation_key
        tests:
          - unique
      - name: rating
        tests:
          - not_null
      - name: price
        tests:
          - not_null

```

#### ✅  Test

Add combo test:

```yaml
  - name: accommodations_seed_v2
    tests:
      - unique_combination:
          combination: ['accommodation_id', 'location', 'price']
          severity: warn

```

Execute dbt test command to validate test
```bash
dbt test -s accommodations_seed_v2

```

---

## 👣 Step 5: Exploration - Visualize and Sample

#### 🌐 Visualize Lineage

- Open `final_best_porto_stay.sql`
- Click "**Lineage**" in dbt Power User tab

#### ☕ Sample Safely with Limit

- Right-click → Execute dbt SQL with limit 5
- ⚠️ Avoid large costs by always setting a limit

```bash
dbt show -s final_best_porto_stay

```
🏆 **Congratulations!** We have finally got the answer where Bill's accomodation in Porto! 

---

## 👣 Step 6: Clean Up

Log and drop tables:

```sql
{% set first_name = target.user.split('@')[0].split('.')[0] %}
{% do log("DROP TABLE IF EXISTS dbt.dev." ~ first_name ~";", info=True) %}

```

Use log output to remove all test tables in Snowflake.

---

### 🌟 Finale: Bill Books His Dream Stay

Bill’s data-driven journey pays off. His model shows him a top-rated, affordable hostel with ocean views in downtown Porto. His summer is set — and so is your knowledge of building dbt models the right way.

---

Happy travels, and even happier modeling!