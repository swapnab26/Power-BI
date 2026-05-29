** AR Credit Risk Summary Dashboard **

Domain: Credit & AR Risk Summary

Why I Built This
Accounts Receivable and Credit Risk management is one of the most data-intensive functions in any enterprise — hundreds of customers, thousands of invoices, multiple collection touchpoints, and credit reviews happening simultaneously. I wanted to build an end-to-end pipeline that didn't just visualize data but actually replicated how a real AR team operates — from raw Excel sheets to a production-ready dashboard with automated refresh.
So I took a multi-sheet Excel dataset, cleaned it in Python, modeled it in PostgreSQL with proper fact/dimension schema, and built a 6-page Power BI dashboard covering everything from executive summaries to customer-level drill throughs.

**Technical Stack**
1) Python (pandas, SQLAlchemy) — data cleaning and pipeline automation
2) PostgreSQL — data modeling, views, stored procedures
3) Power BI — dashboard, DAX measures, RLS, field parameters
4) Windows Task Scheduler — automated pipeline scheduling

**Data Challenges & How I Handled Them**
1) 51% missing Tax IDs — investigated distribution across regions and countries, confirmed random missingness, dropped column as not analytically useful.
2) Risk Category inconsistency — data had both High and High Risk as separate values. Identified in Power BI model view, fixed in PostgreSQL using UPDATE CASE WHEN, and added str.replace() in Python pipeline to prevent recurrence
3) Paid invoices with missing payment dates — instead of imputing dates (which would fabricate financial data), created a payment_date_missing_flag column to track these 176 records as a data quality issue
4) Invoice ID duplicates — discovered same invoice assigned to multiple customers under same parent company. Implemented composite primary key (invoice_id + customer_id) instead of dropping records
5) Region/Country mismatch — synthetic data had incorrect region assignments (e.g. China mapped to North America). Documented as data quality observation rather than overriding source data
6) Collections to Invoices foreign key failure — 71% of collection records couldn't be matched to invoices via composite key due to parent/subsidiary billing structure. Documented as source system issue, maintained relationship through customer_id only

**Pipeline Architecture**
Excel (5 sheets) → Python Cleaning → PostgreSQL -> Power BI -> Service 

**Dashboard Preview **

Executive Summary:

<img width="1057" height="547" alt="image" src="https://github.com/user-attachments/assets/99998d3f-312c-4071-a889-a36075c5b092" />

4 KPI cards with YoY comparison and sparklines
Treemap — outstanding balance by region
Waterfall chart — invoice breakdown by payment status
Line chart — invoice vs collection trend by month
Dynamic slicers — Year, Region, Risk Category, Customer Segment

Invoice and Aging Analysis:
<img width="958" height="538" alt="image" src="https://github.com/user-attachments/assets/5498419d-339d-469a-a005-0de3ae12bb34" />

100% stacked bar — aging bucket distribution by region (90+ days dominates at 85-90% — critical finding)
Bar chart — overdue invoices by industry
Scatter plot — days past due vs outstanding balance by risk category

Collections Performance:
<img width="976" height="555" alt="image" src="https://github.com/user-attachments/assets/2d8ac7fe-bfc8-4edf-9a9f-3caefd321bd4" />

Gauge chart — average recovery at 55.11% vs 75% target (below benchmark — highlighted in red)
Donut chart — collection status breakdown
Line chart — monthly collection trend
Bar chart — performance by collector

Credit Risk Overview:
<img width="956" height="542" alt="image" src="https://github.com/user-attachments/assets/e929227f-9ca8-4991-a219-b5a203764146" />
Bar chart — average credit score by risk level
Clustered bar — customer distribution by industry and risk category
Scatter plot — credit score vs credit utilization (utilization exceeding 100% detected in high risk customers)

Customer Drill Through:
<img width="957" height="537" alt="image" src="https://github.com/user-attachments/assets/b831e12c-df5d-49fc-837e-067ac9371a07" />

Triggered by right clicking any customer
Full invoice history with conditional formatting on days past due
Collection transaction history
Credit score trend over time line chart

Dynamic Analysis:
<img width="952" height="535" alt="image" src="https://github.com/user-attachments/assets/9ed02408-bb4b-4fa0-aa57-a93fe148eb94" />

Field parameters for metric switching (Invoice Amount, Outstanding, Collection, Overdue %)
Field parameters for dimension switching (Region, Industry, Segment, Risk Category)
One dynamic chart replacing multiple static ones

How it works

Raw messy Data - > Python (ETL) -> SQL -> Power BI

*1.Python ETL *

Automated schedule every hour using schedule library
Error handling with try/except logging
if_exists='replace' with CASCADE for clean reloads.


import pandas as pd
import importlib
import schedule
print(schedule.__file__)
from sqlalchemy import create_engine
import time

def run_pipeline():
    try:
        print('pipeline started:')
        FILE_PATH = 'C:/Users/SWAPNA RAVULA/Downloads/enterprise_ar_credit_risk_dataset.xlsx'
        DB_connection = 'postgresql+psycopg2://postgres:root@localhost:5432/ar_credit_risk'
        engine = create_engine(DB_connection)
        customers = pd.read_excel(FILE_PATH, sheet_name = 'Customers_Master')
        invoices = pd.read_excel(FILE_PATH, sheet_name = 'Invoices_Fact')
        credit_reviews = pd.read_excel(FILE_PATH, sheet_name = 'Credit_Reviews')
        collections = pd.read_excel(FILE_PATH, sheet_name = 'Collections_Transactions')
        rls = pd.read_excel(FILE_PATH, sheet_name = 'Security_RLS_Table')
        print('sheets loaded successfully')
        #clean columns
        for df in [customers, invoices, credit_reviews, collections , rls]:
            df.columns = df.columns.str.lower()
        #clean customers
        customers['state'] = customers['state'].fillna('N/A')
        customers['risk_category'] = customers['risk_category'].str.replace(' Risk', '')
        #drop the tax_id
        customers = customers.drop(columns=['tax_id'])
        #fill it with unknown for now , later merge with credit reviews table
        customers['risk_category'] = customers['risk_category'].fillna('Unknown')
        #clean invoices
        invoices['payment_date_missing_flag'] = (
            (invoices['payment_status'] == 'Paid') &
            (invoices['payment_date'].isnull())
        ).astype(int)
     #clean credit_reviews
        credit_reviews['credit_score'] = credit_reviews.groupby('risk_level')['credit_score'].transform(
            lambda x: x.fillna(x.median())
        )
        print('cleaning complete')
        customers.to_sql('customers',engine, schema='dim',if_exists='replace', index=False)
        invoices.to_sql('invoices',engine,schema ='fact',if_exists='replace', index=False)
        credit_reviews.to_sql('credit_reviews',engine,schema='fact',if_exists='replace', index=False)
        collections.to_sql('collections',engine,schema='fact',if_exists='replace', index=False)
        print('All tables loaded successfully')
        print('pipeline completed')
    except Exception as e:
        print(f'Pipeline failed: {e}')
  #schedule it
schedule.every(1).hours.do(run_pipeline)
# keep the script running
while True:
    schedule.run_pending()
    time.sleep(60)


**PostgreSQL Data Model**

Fact tables — invoices, collections, credit_reviews
Dimension table — customers
Views — customer_invoice_summary, customer_collection_summary, customer_credit_summary, customer_master_summary
Stored Procedure — update_risk_category() dynamically updates risk categories based on latest credit score using ROW_NUMBER() window function

-- Views -- 

CREATE VIEW dim.customer_credit_summary AS
WITH latest_reviews AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY review_date DESC) AS rn
    FROM fact.credit_reviews
)
SELECT 
    c.customer_id,
    c.customer_name,
    lr.internal_risk_rating,
    lr.credit_score,
    lr.external_risk_score,
    lr.approval_status,
    lr.risk_level,
    lr.requested_credit_limit,
    lr.approved_credit_limit
FROM dim.customers c
JOIN latest_reviews lr ON c.customer_id = lr.customer_id
WHERE lr.rn = 1

2) CREATE VIEW  dim.customer_master_summary AS
SELECT 
    cs.customer_id,
    cs.customer_name,
    -- invoice summary
    cs.total_amount,
    cs.outstanding_balance,
    cs.paid_invoices,
    cs.overdue,
    cs.open_invoices,
    cs.partial_invoices,
    -- collection summary
    cc.total_collection_amount,
    cc.avg_recovery_percentage,
    cc.collected_payments,
    cc.partial_payments,
    cc.promise_to_pay,
    cc.escalated_payments,
    -- credit summary
    customer_credit.credit_score,
    customer_credit.risk_level,
    customer_credit.internal_risk_rating,
    customer_credit.approval_status,
    customer_credit.requested_credit_limit,
    customer_credit.approved_credit_limit
FROM dim.customers_invoice_summary cs
LEFT JOIN dim.customers_collection_summary cc ON cs.customer_id = cc.customer_id
LEFT JOIN dim.customer_credit_summary customer_credit ON cs.customer_id = customer_credit.customer_id

-- Stored Procedure -- 
CREATE OR REPLACE PROCEDURE update_risk_category()
language plpgsql
as $$
BEGIN 
	update dim.customers c
	SET risk_category = 
	CASE WHEN cr.credit_score < 500 THEN 'High Risk'
	WHEN cr.credit_score between 500 AND 700 THEN 'Medium Risk'
	ELSE 'Low Risk'
	END
	FROM (
		SELECT DISTINCT ON(customer_id) customer_id , credit_score
		FROM fact.credit_reviews 
		ORDER BY customer_id, review_date DESC
		) cr
	WHERE cr.customer_id = c.customer_id ;
END;
$$;
   
3. Power BI — DAX Measures The dashboard uses several custom DAX measures for dynamic KPIs and conditional formatting

YoY Invoice Growth = 
    var CurrentYear = [Total Invoice Amount]
    var LastYear = [Last Year Invoice Amount]
    Return 
    divide(CurrentYear - LastYear, LastYear)

Paid Invoices Count = COUNTROWS
    (FILTER('fact invoices', 
    'fact invoices'[payment_status] = "Paid"))

**Key Insights From the Data **

90+ day overdue accounts for 85-90% of outstanding balance across all regions — critical collections problem affecting every region equally
Average DSO of 97 days — significantly above the 30-day industry benchmark, indicating serious collection inefficiency
Recovery rate at 55.11% — below the 75% target, meaning nearly half of overdue amounts are not being recovered
335 escalated customers — high escalation volume suggesting frontline collections is struggling
Credit utilization exceeding 100% detected in high risk customers — customers borrowing beyond their approved limits
Automotive industry has highest overdue invoice count (153) — industry specific credit risk worth monitoring
Credit scores barely differentiated by risk category (Low: 584, High: 582, Medium: 575) — suggesting risk categorization may need recalibration

**RLS Implementation**

Dynamic Row Level Security using USERPRINCIPALNAME() and LOOKUPVALUE()
Regional Manager role filters data by assigned region from Security_RLS_Table
Tested in Power BI Desktop and published to Service
