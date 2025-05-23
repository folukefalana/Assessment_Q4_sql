# Assessment_Q4_sql

## Introduction

- This project calculates the estimated CLV of users using data from two tables:

- users_customuser: Contains customer profiles including signup dates.

- savings_savingsaccount: Contains transaction records.

By joining and analyzing these datasets, the company can identify high-value customers and tailor marketing strategies accordingly.
Customer Lifetime Value (CLV) Estimation

## Objective
I want to calculate the Estimated Customer Lifetime Value (CLV) for each customer using:

- Account tenure in months

- Number of transactions

- Profit per transaction (fixed at 0.1% of the transaction value) 


## Complete Query

WITH txn_summary AS (
    SELECT 
        owner_id,
        COUNT(*) AS total_transactions,
        SUM(confirmed_amount) AS total_transaction_value
    FROM savings_savingsaccount
    GROUP BY owner_id
),
tenure_calc AS (
    SELECT 
        owner_id,
        CONCAT(first_name, ' ', last_name) AS name,
        TIMESTAMPDIFF(MONTH, enabled_at, CURRENT_DATE()) AS tenure_months
    FROM users_customuser
),
clv_calc AS (
    SELECT 
        t.owner_id AS customer_id,
        t.name,
        t.tenure_months,
        COALESCE(s.total_transactions, 0) AS total_transactions,
        COALESCE(s.total_transaction_value, 0) AS total_transaction_value,
        CASE 
            WHEN COALESCE(s.total_transactions, 0) > 0 
                 AND t.tenure_months > 0 THEN 
                (COALESCE(s.total_transaction_value, 0) * 0.001 / s.total_transactions) 
            ELSE 0 
        END AS avg_profit_per_transaction
    FROM tenure_calc t
    LEFT JOIN txn_summary s ON t.owner_id = s.owner_id
)
SELECT 
    customer_id,
    name,
    tenure_months,
    total_transactions,
    ROUND(
        (total_transactions / NULLIF(tenure_months, 0)) * 12 * avg_profit_per_transaction,
        2
    ) AS estimated_clv
FROM clv_calc
ORDER BY estimated_clv DESC;

## My Approach 

I started the first Common Table Expressions named txn_summary. 

```
WITH txn_summary AS (
```

I Select each unique customer (owner_id), counts how many transactions they made (COUNT(*)), sums the total transaction values using confirmed_amount

and group results by each customer.

```
    SELECT 
        owner_id,
        COUNT(*) AS total_transactions,
        SUM(confirmed_amount) AS total_transaction_value
    FROM savings_savingsaccount
    GROUP BY owner_id
),
```

I write a second Common Table Expressions named tenure_calc, which calculates how long each customer has had their account.

```
tenure_calc AS (
```

I select each customer’s unique ID, combines first and last names into one name field, 
use TIMESTAMPDIFF to find how many full months have passed between the account's enabled_at date and today and Label the difference as tenure_months

```
    SELECT 
        owner_id,
        CONCAT(first_name, ' ', last_name) AS name,
        TIMESTAMPDIFF(MONTH, enabled_at, CURRENT_DATE()) AS tenure_months
    FROM users_customuser
),
```
I write the third CTE, named clv_calc, which brings together the results from the previous two CTEs and calculates the average profit per transaction.

```
clv_calc AS (
```

Retrieves customer ID, name, and tenure from the tenure_calc (aliased as t).

```
    SELECT 
        t.owner_id AS customer_id,
        t.name,
        t.tenure_months,
```

Retrieves transaction info from txn_summary (aliased as s). COALESCE(..., 0) ensures we get 0 instead of NULL for customers with no transactions.

```
  COALESCE(s.total_transactions, 0) AS total_transactions,
  COALESCE(s.total_transaction_value, 0) AS total_transaction_value,
```

Calculates average profit per transaction:

- 0.1% of total transaction value divided by number of transactions

- Ensures no division by zero by using conditions

```
        CASE 
            WHEN COALESCE(s.total_transactions, 0) > 0 
                 AND t.tenure_months > 0 THEN 
                (COALESCE(s.total_transaction_value, 0) * 0.001 / s.total_transactions) 
            ELSE 0 
        END AS avg_profit_per_transaction
```

Joins the tenure_calc with txn_summary on customer ID.

LEFT JOIN ensures we keep customers who have no transactions (NULLs handled by COALESCE)

```
    FROM tenure_calc t
    LEFT JOIN txn_summary s ON t.owner_id = s.owner_id
)
```

Writes the final output query, selecting key customer metrics.

```
SELECT 
    customer_id,
    name,
    tenure_months,
    total_transactions,
```

Computes the estimated CLV using the formula:

- (Transactions÷Months)×12×Profit Per Transaction

- NULLIF(..., 0) avoids division by zero

- ROUND(..., 2) ensures we show only two decimal places

```
    ROUND(
        (total_transactions / NULLIF(tenure_months, 0)) * 12 * avg_profit_per_transaction,
        2
    ) AS estimated_clv
```

Uses the final CTE clv_calc as the data source and sort results so customers with the highest CLV appear first

```
FROM clv_calc
ORDER BY estimated_clv DESC;
```

## Challenges

1. Handling Missing or Null Signup Dates

Challenge: Some users had missing enabled_at values.

Solution: Tenure was set to 0 in such cases and NULLIF() was used to avoid division errors.

2. Customers With No Transactions

Challenge: Not all customers had transactions recorded, leading to NULL values.

Solution: Used LEFT JOIN and COALESCE() to handle NULL values by defaulting to zero.

3. Preventing Division by Zero

Challenge: Zero tenure or transaction count could lead to divide-by-zero errors.

Solution: Used NULLIF() and CASE WHEN logic to handle and avoid such errors.

4. Profit Per Transaction Calculation

Challenge: Required cautious division when transactions or total value were zero.

Solution: Applied CASE logic to ensure calculations only occurred when both values were valid.










