**📦 Supply Chain Analytics Dashboard

Tools: Python · SQL · Power BI

Domain: Supply Chain & Logistics

Type: End-to-End Analytics Project**

Why I Built This
Supply chain data is messy by nature — multiple carriers, suppliers across countries, thousands of orders with inconsistent statuses. 
I wanted to build something that didn't just display numbers but actually helped someone make a decision in under 30 seconds.
So I took a raw CSV, cleaned it up in Python, modeled it in SQL, and built a 3-page Power BI dashboard that tracks orders, carrier performance,
and delay patterns — all linked with drill-throughs and bookmark navigation.

**Dashboard Preview **

Overview Page 
<img width="1104" height="561" alt="image" src="https://github.com/user-attachments/assets/728b36e3-eddf-462d-8982-db0701a91f99" />
- High-level KPIs, monthly order trends, carrier performance, and category contribution — all in one view.
  
Order Details Page
<img width="1102" height="562" alt="image" src="https://github.com/user-attachments/assets/b883a8c4-f425-4876-967a-4b8e761bcc4f" />
- Drill-through from any carrier bar to see order-level breakdown, delivery performance, and cost metrics.
  
Delay Analysis Page
<img width="1083" height="558" alt="image" src="https://github.com/user-attachments/assets/9140d8cc-820b-4359-9467-9720f2d93296" />
- Supplier-level delay tracking with country and year filters - point exactly where the delays are coming from.

**How it works**

Raw messy Data - > Python (ETL) -> SQL -> Power BI 

*1.Python ETL *

The raw data from excel has null values, inconsistent order date formats, countries, blank values and no calculated columns. Hers's what I did  to clean up .

```python

import pandas as pd
import numpy as np

# 1. Load the file 
df = pd.read_excel('messy_supply_chain_data.xlsx')
df.columns = df.columns.str.strip().str.lower().str.replace(' ', '_')

# 2. Identify where IDs are missing
is_blank = df['order_id'].isna()

# 3. Create unique IDs for those specific rows
# We use the row index to make sure every "Unknown" is unique
df.loc[is_blank, 'order_id'] = [f"UNKNOWN_{i}" for i in df.index[is_blank]]

# 4. Remove ONLY rows that are 100% identical across all columns
df = df.drop_duplicates()

# 5. Mapping and replacing the Country values with appropriate values
country_map = {'usa': 'USA', 'U.S.A.': 'USA', 'United States': 'USA', 'US': 'USA',
    'de': 'Germany', 'DE': 'Germany', 'aus': 'Australia', 'AUS': 'Australia',
    'india': 'India'}
df['country']= df['country'].replace(country_map)

# 6. Cleaning and standardizing Supplier_name and Product_category column
df['supplier_name'] = df['supplier_name'].str.title()
df['product_category'] = df['product_category'].str.title()

#7. First we drop the supplier_id, then we map the supplier id with the supplier name.
id_lookup = df.dropna(subset=['supplier_id']).set_index('supplier_name')['supplier_id'].to_dict()

# Fill the missing supplier_ids using the map
df['supplier_id'] = df['supplier_id'].fillna(df['supplier_name'].map(id_lookup))

# 8. Convert using format='mixed'
# This tells Pandas: "Some are 20/09/2023, some are Jan 17, 2023. Figure it out."
df['order_date'] = pd.to_datetime(df['order_date'], format='mixed', dayfirst=True, errors='coerce')

#9. Remove leading /trailing spaces in the supplier_name columns
df['supplier_name'] = df['supplier_name'].str.strip()

#10. Removes symbols and converts to string
df['unit_cost'] = df['unit_cost'].astype(str).str.replace('$', '').str.replace(',', '')
# Force to numeric (floats)
df['unit_cost'] = pd.to_numeric(df['unit_cost'], errors='coerce')
# Filling missing values with grou by Median of product Category
df['unit_cost'] = df['unit_cost'].fillna(df.groupby('product_category')['unit_cost'].transform('median'))
# If any are STILL null (because a whole category was missing costs), fill with 0
df['unit_cost'] = df['unit_cost'].fillna(0)

#11. Convert Quantity to numeric
df['quantity'] = pd.to_numeric(df['quantity'], errors='coerce')
#Fix negative quantities
df['quantity'] = df['quantity'].abs()
#Fill missing quantities with 1
df['quantity'] = df['quantity'].fillna(1).astype(int)

#12. Create 2 new columns
df['total_cost'] = df['unit_cost'] * df['quantity']
df['order_year'] = df['order_date'].dt.year
```

**2. SQL Views **

After clean data was loaded into the database, used a CTE with LAG() window function to calculate Month over Month spend growth % — connected directly to Power BI for the MoM trend visual.

```SQL
-- View 1: Created Month Spend (MOM) using LAG function and as CTE
CREATE VIEW `supply_chain`.`v_supply_chain_performance` AS
WITH `monthlyspend` AS (
    SELECT 
        DATE_FORMAT(`order_date`, '%Y-%m') AS `order_month`,
        ROUND(SUM(`total_cost`), 2) AS `current_spend`
    FROM `supply_chain`.`stg_supply_chain`
    GROUP BY DATE_FORMAT(`order_date`, '%Y-%m')
)
SELECT 
    `order_month`,
    `current_spend`,
    LAG(`current_spend`) OVER (ORDER BY `order_month`) AS `previous_spend`,
    ROUND((`current_spend` - LAG(`current_spend`) OVER (ORDER BY `order_month`)), 2) AS `dollar_diff`,
    ROUND((((`current_spend` - LAG(`current_spend`) OVER (ORDER BY `order_month`)) 
        / LAG(`current_spend`) OVER (ORDER BY `order_month`)) * 100), 2) AS `spend_growth_pct`
FROM `monthlyspend`

-- View 2: Fact Order Table 
CREATE VIEW `supply_chain`.`fact_orders` AS
    SELECT 
        `supply_chain`.`stg_supply_chain`.`order_id` AS `order_id`,
        `supply_chain`.`stg_supply_chain`.`supplier_id` AS `supplier_id`,
        `supply_chain`.`stg_supply_chain`.`product_id` AS `product_id`,
        `supply_chain`.`stg_supply_chain`.`order_status` AS `order_status`,
        `supply_chain`.`stg_supply_chain`.`order_date` AS `order_date`,
        `supply_chain`.`stg_supply_chain`.`shipment_carrier` AS `shipment_carrier`,
        CAST(`supply_chain`.`stg_supply_chain`.`unit_cost`
            AS DECIMAL (10 , 2 )) AS `unit_cost`,
        `supply_chain`.`stg_supply_chain`.`quantity` AS `quantity`,
        ROUND((`supply_chain`.`stg_supply_chain`.`unit_cost` * `supply_chain`.`stg_supply_chain`.`quantity`),
                2) AS `total_cost`
    FROM
        `supply_chain`.`stg_supply_chain`

-- View 3: Dimension Product Table 
CREATE VIEW `supply_chain`.`dim_products` AS
    SELECT 
        TRIM(UPPER(`supply_chain`.`stg_supply_chain`.`product_id`)) AS `product_id`,
        ANY_VALUE(`supply_chain`.`stg_supply_chain`.`product_category`) AS `product_category`,
        ANY_VALUE(`supply_chain`.`stg_supply_chain`.`product_name`) AS `product_name`
    FROM
        `supply_chain`.`stg_supply_chain`
    WHERE
        ((`supply_chain`.`stg_supply_chain`.`product_id` IS NOT NULL)
            AND (`supply_chain`.`stg_supply_chain`.`product_id` <> ''))
    GROUP BY TRIM(UPPER(`supply_chain`.`stg_supply_chain`.`product_id`))
    
-- View 4 : Dimension Supplier Table 
CREATE VIEW `supply_chain`.`dim_suppliers` AS
    SELECT 
        TRIM(UPPER(`supply_chain`.`stg_supply_chain`.`supplier_id`)) AS `supplier_id`,
        ANY_VALUE(`supply_chain`.`stg_supply_chain`.`supplier_name`) AS `supplier_name`,
        ANY_VALUE(`supply_chain`.`stg_supply_chain`.`country`) AS `country`
    FROM
        `supply_chain`.`stg_supply_chain`
    WHERE
        ((`supply_chain`.`stg_supply_chain`.`supplier_id` IS NOT NULL)
            AND (`supply_chain`.`stg_supply_chain`.`supplier_id` <> ''))
    GROUP BY TRIM(UPPER(`supply_chain`.`stg_supply_chain`.`supplier_id`))

```

**3. Power BI — DAX Measures**
The dashboard uses several custom DAX measures for dynamic KPIs and conditional formatting

```POWER BI

-- Year over Year Orders Growth
Orders YoY% = 
VAR CY = COUNTROWS(Orders)
VAR PY = CALCULATE(COUNTROWS(Orders), SAMEPERIODLASTYEAR(Date[Date]))
RETURN DIVIDE(CY - PY, PY)

-- Delay Rate
Delay Rate =
DIVIDE(
    COUNTROWS(FILTER(Orders, Orders[Status] = "Delayed")),
    COUNTROWS(Orders)
)

-- Dynamic Order Status Chart Title
Status Chart Title = 
"Order Status Distribution (" & 
FORMAT(COUNTROWS(Orders), "#,##0") & 
" orders)"

-- Most Delayed Supplier
Most Delayed Supplier =
FIRSTNONBLANK(
    TOPN(1,
        VALUES(Supplier[supplier_name]),
        CALCULATE(
            COUNTROWS(FILTER(Orders, Orders[Status] = "Delayed"))
        )
    ),
1)
```

**Dashboard Features**

**Overview Page**

4 KPI cards with conditional formatting (green/red based on performance)
Monthly orders trend — 2023 vs 2024 comparison
Orders by carrier with YoY delta labels
Category contribution clustered bar chart
Country and Carrier slicers

**Order Details Page (Drill-through from Carrier chart)**

Automatically filters to selected carrier
KPIs: Total Orders, Avg Order Value, Delay Rate, On-Time Rate
Orders over time line chart
Order status distribution bar chart
Full order-level table with status pills
Year slicer for flexible filtering

**Delay Analysis Page**
Total Delayed, Delay Rate, Most Delayed Supplier KPIs
Delays by Supplier — sorted bar chart
Delays by Category — sorted bar chart
Delays by Month — trend line 2023 vs 2024
Supplier, Year, and Country slicers

**Key Insights From the Data **
1) Indo Materials Ltd is the most delayed supplier — responsible for 32 out of 156 delayed orders
2) Furniture category has the highest delay count (33) despite not being the highest volume category
3) FedEx volume dropped significantly in 2024 — worth reviewing SLA compliance
4) Overall delay rate sits at 15.6% — above the typical 10% industry benchmark
5) Revenue grew 22.4% YoY despite order volume dropping — suggesting a shift toward higher value orders

















