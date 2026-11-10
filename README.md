# Tunda Box: Junior Data Analyst Assessment

**Candidate:** Chala Wakshuma  |  **Date:** 9 November 2026

Responses to the Tunda Box Junior Data Analyst Assessment (fictional fruit and vegetable delivery company in Nairobi; all data is invented). Every number below is produced by `solutions.ipynb` and cross-checked with `assert` statements.

## Repository contents

| File | What it is |
| --- | --- |
| `README.md` | This file: my written responses to every question |
| `THOUGHT_PROCESS.md` | README explaining my approach, assumptions and the checks I ran |
| `solutions.ipynb` | Notebook with the SQL (run in SQLite) and Python (pandas) solutions, all outputs visible |
| `Junior-Data-Analyst-Candidate-Assessment.docx` | The assessment document with all responses filled in (same answers as this README) |
| Google Drive copy | The same document saved to Drive: **[[PASTE YOUR GOOGLE DRIVE LINK HERE](https://docs.google.com/document/d/1tDZH4eTX54Pws5csI1w0JC281_gz8eYx/edit?usp=sharing&ouid=117859859304265881748&rtpof=true&sd=true)]** |

**How to run:** [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Chala-prog/Junior-Data-Analyst-Assessment/blob/main/solutions.ipynb) then choose Runtime > Restart session and run all. Only `pandas` and the Python standard library are used.

## Headline results

| Question | Answer |
| --- | --- |
| Orders by status (S1) | 16 completed, 2 cancelled, 2 refunded (20 unique orders) |
| Top city by completed revenue (S2) | Nairobi, KES 20,650 (Kisumu only leads if the KES 185,000 outlier is real) |
| Customers who never ordered (S3) | 109 (Eldoret) and 110 (Kisumu) |
| Repeat purchase rate (S4) | 40% of all customers (4 of 10); 50% of buyers (4 of 8) |
| Month-on-month revenue (S5) | Jun 9,000; Jul 6,800 (-24.4%); Aug 7,000 (+2.9%); Sep 10,300 (+47.1%), outlier excluded |
| Completed M-Pesa revenue (D2) | KES 208,050 after removing the duplicate (209,550 on the raw sheet) |
| Typical order (D3) | Median KES 2,400 vs mean KES 14,540 |
| A/B test (T2) | +1.0 point (5.0% to 6.0%), p = 0.33: not significant, do not ship yet |
| Falling conversion (Part 5) | Mix shift, not a worse channel: paid social 20% to 44% of traffic; total orders +13.4% |

**Assumptions used throughout:** revenue means completed orders only; the duplicate order 1006 is counted once; city names are trimmed and standardised; the missing amount on order 1009 is not imputed; order 1013 (KES 185,000) is treated as a probable typo and results are shown both ways; order 1020 (customer 999) is left out of city and customer metrics.

---

# My responses

## Part 1: Your experience

### Your analysis

**Work example (my own experience).** I took a messy Excel data set (duplicate rows, errors and missing values), cleaned and organised it, and built pivot tables and charts so managers could track performance at a glance. 
**Question:** can the managers trust this data and see how things are going without digging through raw rows? 
**What changed:** the data was quality-checked once and turned into a clear, reusable view that managers used for their tracking.


### Conflicting numbers

I would not assume either number is right. My steps:

- **Check definitions:** what each of us counts (order statuses, refunds, date range, timezone, currency).
- **Check sources and filters:** same table, same extract date, same row counts.
- **Look for data issues:** duplicates, nulls, orphan records. In this very sample one duplicated order changes completed revenue by KES 1,500.
- **Reconcile row by row** on a small slice until the first difference appears and the gap is explained.
- **Agree one written definition** with my colleague, share the reconciled number with stakeholders and document the rule so it doesn't happen again.

## Part 2: SQL and Python

### S1. Orders by status

Setup used by all queries below (views on top of the raw tables):

```sql
-- orders_clean: removes the duplicated order 1006
CREATE VIEW orders_clean AS
SELECT DISTINCT * FROM orders;

-- customers_clean: trimmed, title-case city (Postgres: INITCAP(TRIM(city)))
CREATE VIEW customers_clean AS
SELECT customer_id,
       UPPER(SUBSTR(TRIM(city),1,1)) || LOWER(SUBSTR(TRIM(city),2)) AS city,
       signup_date, channel
FROM customers;
```

```sql
SELECT status, COUNT(*) AS orders
FROM orders_clean
GROUP BY status
ORDER BY orders DESC;
```

| status | orders |
| --- | --- |
| completed | 16 |
| cancelled | 2 |
| refunded | 2 |

20 unique orders (21 rows in the extract). Order 1006 appears twice, so counting the raw table would wrongly show 17 completed orders.

### S2(a). Revenue by city

```sql
SELECT c.city,
       SUM(o.amount_kes) AS revenue_kes,
       SUM(CASE WHEN o.amount_kes < 100000 THEN o.amount_kes END) AS revenue_ex_outlier
FROM orders_clean o
JOIN customers_clean c ON c.customer_id = o.customer_id
WHERE o.status = 'completed'
GROUP BY c.city
ORDER BY revenue_ex_outlier DESC;
```

| city | revenue as recorded (KES) | excluding order 1013 (KES) |
| --- | --- | --- |
| Kisumu | 187,100 | 2,100 |
| Nairobi | 20,650 | 20,650 |
| Mombasa | 5,150 | 5,150 |
| Nakuru | 3,300 | 3,300 |

**Answer: Nairobi (KES 20,650).** As recorded, Kisumu looks first only because of one KES 185,000 order (1013), which is about 75 times the typical order and probably a typo (e.g. 1,850). If that order were real, Kisumu would lead. The city labels "Nairobi", "nairobi" and "Nairobi " are merged into one city, and the duplicate order 1006 is counted once. One order (1020) has no matching customer and drops out of the city totals.

### S2(b). What do you send in ten minutes?

I send a short, honest message with the cleaned number and the caveats, not the raw figure:

-	Nairobi is our top city: KES 20,650 in completed-order revenue in this sample, well ahead of Mombasa (KES 5,150). Caveats: I removed one duplicate order and standardised city names. One Kisumu order of KES 185,000 looks like a data-entry error or outlier. So, I excluded it until it is confirmed. If it is real, Kisumu would be first. One order has no matching customer and is left out. This is a small sample, so treat it as directional.

### S3. Customers who never ordered

```sql
SELECT c.customer_id, c.city, c.signup_date, c.channel
FROM customers_clean c
LEFT JOIN orders_clean o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL;
```

| customer_id | city | signup_date | channel |
| --- | --- | --- | --- |
| 109 | Eldoret | 2026-06-21 | Organic |
| 110 | Kisumu | 2026-07-05 | Paid Social |

### S4. Repeat purchase rate

```sql
WITH n AS (
    SELECT customer_id, COUNT(*) AS completed_orders
    FROM orders_clean
    WHERE status = 'completed'
    GROUP BY customer_id
), t AS (
    SELECT (SELECT COUNT(*) FROM customers_clean) AS customers,
           COUNT(*) AS buyers,
           SUM(CASE WHEN n.completed_orders >= 2 THEN 1 ELSE 0 END) AS repeat_buyers
    FROM n
    JOIN customers_clean c ON c.customer_id = n.customer_id   -- drops orphan 999
)
SELECT repeat_buyers, buyers, customers,
       ROUND(100.0 * repeat_buyers / customers, 1) AS pct_of_all_customers,
       ROUND(100.0 * repeat_buyers / buyers, 1)    AS pct_of_buyers
FROM t;
```

| metric | value |
| --- | --- |
| Repeat buyers (2+ completed orders) | 4 (customers 101, 102, 104, 107) |
| Customers who bought at least once | 8 |
| All customers | 10 |
| Repeat rate, all customers | 40.0% |
| Repeat rate, buyers only | 50.0% |

**How I worked it out:** 
-	Keep completed orders only, 
-	remove the duplicate row, 
-	count orders per customer, and call a customer a repeat buyer if they have two or more.
The question asks about "our customers", so the headline is 40% (4 of 10); among the 8 customers who actually bought, it is 50%. Customer 105 only looks like a repeat buyer if the duplicate order 1006 is left in, and customer 999 is not in the customers table, so neither is counted.


### S5. Month-on-month revenue

```sql
-- Postgres: use DATE_TRUNC('month', order_date) instead of strftime
WITH m AS (
    SELECT strftime('%Y-%m', order_date) AS month,
           SUM(amount_kes) AS revenue_kes,
           SUM(CASE WHEN amount_kes < 100000 THEN amount_kes END) AS revenue_ex_outlier
    FROM orders_clean
    WHERE status = 'completed'
    GROUP BY 1
), p AS (
    SELECT *,
           LAG(revenue_kes) OVER (ORDER BY month)         AS prev,
           LAG(revenue_ex_outlier) OVER (ORDER BY month)  AS prev_ex
    FROM m
)
SELECT month, revenue_kes,
       ROUND(100.0 * (revenue_kes - prev) / prev, 1)                 AS mom_pct,
       revenue_ex_outlier,
       ROUND(100.0 * (revenue_ex_outlier - prev_ex) / prev_ex, 1)    AS mom_pct_ex_outlier
FROM p
ORDER BY month;
```

| month | revenue as recorded | MoM | revenue excl. order 1013 | MoM |
| --- | --- | --- | --- | --- |
| 2026-06 | 9,000 | n/a | 9,000 | n/a |
| 2026-07 | 6,800 | -24.4% | 6,800 | -24.4% |
| 2026-08 | 192,000 | +2,723.5% | 7,000 | +2.9% |
| 2026-09 | 10,300 | -94.6% | 10,300 | +47.1% |

Read the right-hand columns: the as-recorded;
-	August spike and September crash are produced by the single KES 185,000 order. 
-	July is lower because the duplicate order 1006 is removed, the refunded order 1007 is excluded and order 1009 has no amount. 
-	September includes order 1020 (customer not found); without it September is KES 8,400 (+20.0%).


## Part 3: Data quality and spreadsheets

### D1. Audit the data

| Issue found | What I would do |
| --- | --- |
| Duplicate order (1006 appears twice) | De-duplicate on order_id; find out why the extract duplicates rows |
| Inconsistent cities ("nairobi", "Nairobi ") | TRIM and standardise case; use a controlled city list |
| Missing amount (order 1009 is completed but NULL) | Do not guess or impute revenue; check the source system; exclude from sums and flag |
| Extreme outlier (order 1013 = KES 185,000 vs ~2,400 typical) | Verify with the source or ops (likely extra digits); exclude until confirmed |
| Orphan order (1020 has customer_id 999, not in customers) | Check the customer pipeline; keep in totals but out of city or customer analysis; add a foreign key |
| Order before signup (1003 dated 11 Jun, customer 103 signed up 20 Jun) | Check dates and timezones; it is cancelled so low impact, but signals a timestamp issue |
| Refunded and cancelled orders carry amounts (4 orders) | Define revenue as completed-only or net of refunds, and say so in every report |
| Very small sample (10 customers, 20 unique orders) | Label results directional; do not extrapolate |

### D2. Spreadsheet formulas

**Total completed M-Pesa revenue** (orders in columns A to F):

```excel
=SUMIFS(E:E, D:D, "completed", F:F, "M-Pesa")
```

On the raw sheet this returns **KES 209,550**. Remove the duplicate row first (Data > Remove Duplicates on column A) to get **KES 208,050**, and note that KES 185,000 of that is the suspicious order 1013 (KES 23,050 without it). SUMIFS ignores the blank amount in order 1009.

**City for each order** (new column G on the orders sheet, copied down):

```excel
=IFERROR(PROPER(TRIM(XLOOKUP(B2, customers!A:A, customers!B:B))), "Not found")
```

Without XLOOKUP: =IFERROR(PROPER(TRIM(VLOOKUP(B2, customers!A:B, 2, FALSE))), "Not found"). TRIM and PROPER fix the messy city labels, and order 1020 shows "Not found" instead of an error.

### D3. Mean or median

**Median.** 
-	Across the 15 completed orders that have an amount, the mean is KES 14,540 but the median is KES 2,400. One KES 185,000 order pulls the mean far above what a normal order looks like, while the median stays close to the typical basket. Without that outlier the mean (KES 2,364) and median (KES 2,375) agree. I would report the median and mention the outlier separately.

## Part 4: Statistics

### T1(a). Explaining a p-value

A p-value is the probability that the result we observed (like a difference in sales between two groups) happened purely by random chance rather than a real effect. A low p-value (typically $< 0.05$) gives us high confidence that our change actually worked.
-	Imagine our change did nothing at all. The p-value tells us how often we would still see a difference at least as big as the one we saw, just by luck. If that is rare (usually under 5%), luck is an unlikely explanation, so we take the result seriously. It does not tell us the probability that the change worked, or how big or valuable the effect is.

### T1(b). Do SMS offers cause the extra spending?

**Causation vs. Correlation:** No, SMS offers do not automatically cause extra spending. Customers who opt-in to SMS might already be more loyal or higher-spending brand advocates (selection bias). To prove causation, a randomized controlled A/B test is required.
**Not necessarily.** This is a correlation. Customers who receive SMS offers may already be the more engaged or higher-spending ones, for example if we targeted them or they opted in, so the difference could exist without any SMS (selection bias, a confounder). To show the SMS causes extra spending I would randomly hold out a group that gets no SMS and compare spending between the two groups, or at least compare each customer's spend before and after against a similar control group.

### T2. Read an A/B test

| variant | users | conversions | conversion rate |
| --- | --- | --- | --- |
| Current | 1,000 | 50 | 5.0% |
| New | 1,000 | 60 | 6.0% |

**Analysis :**
•	Current conversion: $50 / 1000 = 5\%$
•	New checkout conversion: $60 / 1000 = 6\%$ (a 20% relative lift).

**Recommendation:** Do not ship immediately. Run a test for statistical significance (chi-square or z-test). With a sample size of only 1,000 users per variant, a 1% absolute difference is likely not statistically significant and could easily be noise.


## Part 5: Business case

### Part 5(a). What is happening?

**Working Through the Numbers**
   **Channel-Level Breakdown:**
       **Organic:**
              - September: 2,400 orders / 40,000 sessions = 6.0% CR
              - October: 2,318 orders / 38,000 sessions = 6.1% CR (Conversion slightly improved)
       **Paid Social:**
              - September: 200 orders / 10,000 sessions = 2.0% CR
              - October: 630 orders / 30,000 sessions = 2.1% CR (Conversion slightly improved)
**Overall Conversion Rate:**
       o	September: $2,600 / 50,000 = \mathbf{5.2\%}$
       o	October: $2,948 / 68,000 = \mathbf{4.34\%}$ (rounded to 4.3%)
       
**What is happening? (Simpson's Paradox):** The drop in overall conversion rate from 5.2% to 4.3% is not caused by falling performance within individual channels. In fact, conversion rates for both Organic and Paid Social improved slightly between September and October. Instead, the decline is driven entirely by a traffic mix shift (composition effect): low-converting Paid Social traffic surged from 20% of total traffic (10k out of 50k) in September to 44.1% of total traffic (30k out of 68k) in October. Because Paid Social naturally converts at a lower rate (~2%) than Organic (~6%), scaling up Paid Social pulled down the blended company-wide average.

**What else would you want to know?**
	**Customer Acquisition Cost (CAC) vs. Lifetime Value (LTV):** Is Paid Social profitable despite its lower conversion rate, and what are the customer quality/retention metrics across channels?
	**Marketing Spend & Attribution:** Why did Paid Social traffic triple? Did budget increase, or did acquisition costs change?
	**Funnel Drop-off:** At which stage of the checkout funnel do Paid Social users drop off compared to Organic users?


### Part 5(b). Recommendation (two sentences)

**Recommendation to the Head of Growth**
-	Paid social is not hurting conversion rates; rather, our overall conversion drop is a mix shift caused by scaling low-converting traffic from 20% to 44% of our total volume. 
-	Instead of pausing paid social, we should analyze its CAC and LTV profitability and optimize the channel's landing page funnel to improve its 2.1% conversion rate."


## Part 6: Your questions

### Questions for the team

- How are data quality problems such as duplicate records or payment anomalies monitored in your pipelines today?
- Which tools and data sources does the team use, and how clean are they?
- Is there one agreed definition of revenue and conversion, and who owns it?
- How does the team handle it when two numbers don't match?
