# Thought process

This file explains how I approached the Tunda Box assessment: the order I worked in, the decisions I made about the messy data, how I checked my answers, and what I would do with more time. The answers themselves are in `README.md` and the working is in `solutions.ipynb`.

## 1. How I approached it

1. **Read the data before answering anything.** The sample tables are small but deliberately messy, so I audited them first (notebook section D1). Every later answer depends on how those problems are handled.
2. **Decide the rules once, then apply them everywhere.** I wrote down a small set of rules (below) and encoded them as two SQL views, `orders_clean` and `customers_clean`. Every query reads from those views, so S1 to S5, D2 and D3 all agree with each other.
3. **Answer in two layers.** First the number, then the caveat a manager needs to trust it. Where a single bad record changes the answer (the KES 185,000 order), I show the result both ways instead of silently choosing.
4. **Check every result a second way.** Each SQL result is reproduced with pandas and compared with `assert`. If the two disagree, the notebook stops.

## 2. The data problems and the decision I made for each

| Problem | Decision | Why |
| --- | --- | --- |
| Order 1006 appears twice | Keep one copy (`SELECT DISTINCT`) | An order id should be unique; counting it twice inflates orders by one and revenue by KES 1,500 |
| City labels `"Nairobi"`, `"nairobi"`, `"Nairobi "` | `TRIM` and standardise case | Otherwise Nairobi splits into three cities |
| Order 1009 is completed with a NULL amount | Do not impute; it is not added to sums | Inventing a revenue figure is worse than reporting a gap, and the source system can answer it |
| Order 1013 is KES 185,000 (about 75x the typical order) | Flag as a probable typo; report results as recorded and without it | It decides the top city and the August spike, so the reader must see both versions |
| Order 1020 belongs to customer 999, who does not exist | Keep in total revenue, leave out of city and customer metrics | It is a real recorded sale, but it cannot be tied to a city or a customer |
| Order 1003 is dated before customer 103 signed up | Note it; no change to answers | It is cancelled, so the impact is nil, but it signals a timestamp or data-entry issue |
| Refunded and cancelled orders have amounts | Revenue = completed orders only | Stated in every answer so the definition is never a surprise |

## 3. Choices that change a number

- **Repeat purchase rate (S4)** is 40% of *all* customers or 50% of *buyers*. The question says "our customers", so 40% is the headline, and the 50% is shown next to it. A repeat buyer is a customer with two or more completed orders.
- **Month-on-month revenue (S5)** keeps order 1020, because monthly revenue does not need a customer match. Without it, September is KES 8,400 (+20.0%) rather than KES 10,300 (+47.1%). Both are stated.
- **Mean or median (D3):** the median, with the numbers: mean KES 14,540 vs median KES 2,400 with the outlier; KES 2,364 vs KES 2,375 without it.
- **A/B test (T2):** I report the absolute lift (1 point) next to the relative lift (20%), a z-test, the confidence interval (-1.0 to +3.0 points) and the sample size that would be needed (about 8,100 per variant). A shuffle test in the notebook gives a similar answer (about 0.37 vs 0.33 for the z-test); both say "not significant".
- **Business case (Part 5):** I compared the channels, then the mix, then asked what the blended rate would have been with September's mix (5.3%), instead of stopping at "paid social converts worse". The recommendation is about unit economics (cost per order, customer value) because the table has no cost data, and I say so rather than guessing.

## 4. How I checked my work

- Every SQL result is recomputed in pandas and compared with `assert`, so wrong numbers fail the notebook.
- The notebook was restarted and run from top to bottom, so every cell shows its output and the run counts start at 1.
- Totals reconcile: city revenue (KES 216,200) plus the unmatched order 1020 (KES 1,900) equals total monthly completed revenue (KES 218,100), and the status counts add to the 20 unique orders. Both are asserted in the notebook.
- Each cleaning step is a visible line of code or a named SQL view, so no decision is hidden.

## 5. What I would do with more time or real access

- Ask the data owner about orders 1009, 1013 and 1020 instead of assuming.
- Add the paid-social spend to Part 5 and compute cost per order and return on ad spend by channel.
- Run the A/B test to its required sample size and add guardrail metrics (average order value, failed payments).
- Turn the cleaning rules into automated data-quality checks (duplicate ids, orphan keys, outlier flags) so the problems are caught before anyone builds a report.
