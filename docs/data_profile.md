Status: **hypotheses and method only. No result figures yet.** The Olist CSVs were not available when this document was written, so every "expected pattern" below is a hypothesis to verify, not a measured result. Fill the `Result:` lines after running the analysis.

Dataset: https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce

---

## 1. Data Model (what each table gives us)

|Table|Key columns used here|
|---|---|
|`olist_orders_dataset`|`order_id`, `customer_id`, `order_status`, `order_purchase_timestamp`, `order_approved_at`, `order_delivered_carrier_date`, `order_delivered_customer_date`, `order_estimated_delivery_date`|
|`olist_order_items_dataset`|`order_id`, `product_id`, `seller_id`, `shipping_limit_date`, `price`, `freight_value`|
|`olist_order_payments_dataset`|`order_id`, `payment_type`, `payment_installments`, `payment_value`|
|`olist_order_reviews_dataset`|`order_id`, `review_score`, `review_comment_title`, `review_comment_message`, `review_creation_date`, `review_answer_timestamp`|
|`olist_customers_dataset`|`customer_id`, `customer_unique_id`, `customer_zip_code_prefix`, `customer_city`, `customer_state`|
|`olist_sellers_dataset`|`seller_id`, `seller_zip_code_prefix`, `seller_city`, `seller_state`|
|`olist_products_dataset`|`product_id`, `product_category_name` (Portuguese), dimensions, weight|
|`product_category_name_translation`|Portuguese to English category names|
|`olist_geolocation_dataset`|zip prefix to latitude/longitude|

### Assumptions to confirm before any analysis

1. **"Item" is ambiguous.** The data has `product_id` (anonymous, no product name) and `product_category_name`. Questions 3, 5, 6, 7 will be answered at **category level** by default, with `product_id` as a secondary view. Confirm this is what you want.
2. **`customer_id` is per order, `customer_unique_id` is per person.** All "customer" counts must use `customer_unique_id`, or repeat buyers are counted as new customers.
3. **An order can have several payment rows** (split payments, vouchers plus card). "Payment type per transaction" must be defined as either (a) every payment row, or (b) the payment type with the largest `payment_value` per order. State which one you use.
4. **Only delivered orders** should be used for delivery-time and review analyses. Canceled and unavailable orders have missing delivery dates.
5. **Time range is limited** (roughly 2016-2018, with sparse early and late months). Check month coverage before comparing months or reading seasonality, and never compare partial months.

---

## 2. Findings Plan, Per Question

### Q1. Most and least common payment type, and the benefit of the most common vs the least common

- **Method:** count orders per `payment_type` (rule from assumption 3). Then, per type, compare average order value, average installments, review score, cancellation rate and delivery time.
- **Hypothesis:** credit card dominates, and `debit_card` or `not_defined` is rarest. Credit card likely carries higher order values because of installments.
- **Benefit lens:** frequency alone is not a benefit. Judge by order value, completion rate (approved and delivered vs canceled), and review score.
- **Caution:** boleto is a bank slip paid offline, so it may show a delay between purchase and approval. That is a payment mechanic, not a platform fault.
- **Result:** _TBD_

### Q2. Highest and lowest review score, and whether comments explain the score

- **Method:** review scores range 1 to 5. Count by score. For comment analysis, keep only rows with a non-empty `review_comment_message`.
- **Hypothesis:** many reviews have no comment, so comment analysis covers a subset only. That subset is biased toward strongly unhappy or strongly happy customers.
- **Text analysis:** comments are in Portuguese. Use keyword and n-gram frequency per score, grouped by theme (late delivery, wrong or missing item, damaged product, product not as described, praise). Do not translate blindly. Validate themes against a manual sample of 50 to 100 comments.
- **Cross-check:** compare themes with measurable facts. Low scores should coincide with late delivery (`delivered_customer_date` > `estimated_delivery_date`), if the comments are truthful.
- **Result:** _TBD_

### Q3. Time patterns, items bought at certain times, and whether it is time or events

- **Method:** extract hour, weekday, month and week from `order_purchase_timestamp`. Build order-count heatmaps (weekday x hour). Then do category x time-bucket shares (for example morning, afternoon, evening, night).
- **Time vs event:** overlay known Brazilian dates (Black Friday, Christmas, Mother's Day, Valentine's Day on Jun 12, Carnival, school-year start) on the daily order series. A spike that lines up with an event and is concentrated in related categories points to an event effect. A stable weekday and hour rhythm every week points to a pure time preference.
- **Caution:** timestamps are in local Brazilian time. Check this before drawing hour-of-day conclusions for an Indonesian audience.
- **Caution:** correlation with a date is not proof of causation. Prices, promotions and marketing are not in the data.
- **Result:** _TBD_

### Q4. How long an order takes to be approved and delivered

- **Method:** compute, for delivered orders only:
    - approval time = `order_approved_at` − `order_purchase_timestamp`
    - carrier handoff time = `order_delivered_carrier_date` − `order_approved_at`
    - delivery time = `order_delivered_customer_date` − `order_purchase_timestamp`
    - estimate gap = `order_delivered_customer_date` − `order_estimated_delivery_date`
- Report mean, median, P90 and P95. Use medians and percentiles, not just the mean, because delivery times are right-skewed.
- Break down by payment type (boleto vs card), customer state and seller state.
- Flag illogical rows (approval before purchase, delivery before handoff) and report how many are excluded.
- **Result:** _TBD_

### Q5. Most and least common item

- **Method:** count by category (translated to English) using order-item rows, and also count distinct orders per category, because one order can hold several items of the same category.
- Report the top 10 and bottom 10. Note that the bottom list may be dominated by categories with only a handful of sales, so also report how many categories fall below a minimum threshold (for example fewer than 10 orders).
- **Result:** _TBD_

### Q6. Where customers live, and whether location changes item preference

- **Method:** count distinct `customer_unique_id` per state and per city. Expect concentration in the southeast (São Paulo state is the likely leader). Verify.
- **Preference test:** build a state x category share table and test for differences (chi-square on top states vs top categories, or compare each state's category share to the national share). Report effect size, not just p-values, because with a large sample almost everything is "significant".
- **Caution:** small states have few orders, so their shares are noisy. Set a minimum sample size.
- **Caution:** `customer_zip_code_prefix` is a prefix, not an exact address. Geolocation has multiple coordinates per prefix, so aggregate (for example median) before mapping.
- **Result:** _TBD_

### Q7. Do different sellers sell the same item, and does seller location change what is sold

- **Method (same item):** count distinct `seller_id` per `product_id` and per category. Note that if each `product_id` is a unique listing, most products will have exactly one seller, and category level is the more meaningful view.
- **Method (location):** seller state x category share table, same approach as Q6.
- **Result:** _TBD_

### Q8. Orders overdue on shipping date

"Shipping date" is ambiguous. Two defensible definitions, report both:

1. **Seller overdue:** `order_delivered_carrier_date` > `shipping_limit_date` (the seller handed the parcel to the carrier late). This is the literal "shipping" deadline.
2. **Delivery overdue:** `order_delivered_customer_date` > `order_estimated_delivery_date` (the customer received it late).

- Report counts and percentages of delivered orders for each, and the delay in days (median and P90).
- Also check orders that are not delivered yet but already past the estimated date.
- Link to Q2: compare review scores for on-time vs late orders.
- **Result:** _TBD_

### Q9. Did the seller answer the customer review?

- **Important data limitation:** the reviews table has `review_creation_date` and `review_answer_timestamp`. As documented, `review_answer_timestamp` is when the customer **answered the satisfaction survey**, not a seller reply. **There is no column in this dataset that records a seller's response.**
- **Consequence:** this question cannot be answered from the data as it exists. Do not present `review_answer_timestamp` as a seller response rate.
- **What can be measured instead:** the lag between survey sent (`review_creation_date`) and customer answer, which is a customer engagement metric.
- **Decision:** treat this as a data-collection gap (see Section 4).
- **Result:** not answerable from this dataset.

---

## 3. Other Reference Findings (beyond your list)

1. **Repeat purchase rate.** Share of `customer_unique_id` with more than one order. Likely low for a marketplace, which makes retention a key lever.
2. **Freight vs price.** Ratio of `freight_value` to `price` by category and state. High freight on low-priced items may suppress sales or hurt reviews.
3. **Installments behavior.** Link `payment_installments` to order value and category. Big-ticket categories likely rely on installments.
4. **Seller concentration.** Share of revenue from the top 10 percent of sellers (Pareto check). Heavy concentration is a platform risk.
5. **Cross-state shipping.** Share of orders where customer state differs from seller state, and its effect on delivery time and review score.
6. **Order status funnel.** Share of orders canceled, unavailable or never delivered, and by which payment type or seller state.
7. **Product listing quality.** Relationship between photo count, description length or name length (in `olist_products_dataset`) and sales or review score.
8. **Multi-seller orders.** Orders containing items from several sellers, and whether they get worse delivery or reviews.
9. **Review score vs delivery delay.** Dose-response curve: score by number of days late. This quantifies how costly lateness is.
10. **Data quality audit.** Missing timestamps, impossible date orders, duplicate reviews per order, missing category names. This should be run first, because it affects everything else.

---

## 4. Decisions to Take, Per Finding

Decisions are conditional. Pick the branch that the verified result supports.

|#|If the result shows...|Then decide to...|
|---|---|---|
|Q1|One payment type has clearly higher order value or completion|Promote it at checkout (default selection, small incentive). Do not remove the rarest type unless it adds cost without revenue.|
|Q1|Boleto has long approval delay and higher cancellation|Add payment reminders and a payment-deadline notice. Consider instant methods as the default.|
|Q2|Low scores are mostly about delivery delay|Prioritize logistics fixes (Q4, Q8) over product-quality work.|
|Q2|Low scores are mostly about wrong or damaged items|Introduce seller quality checks, packaging standards and return-rate monitoring.|
|Q3|Strong weekday and hour peaks|Schedule campaigns, customer support staffing and seller alerts to match the peaks.|
|Q3|Spikes tied to events and specific categories|Build an annual event calendar, pre-stock and pre-promote those categories.|
|Q4|Long approval time|Investigate payment and fraud-check steps for avoidable delay.|
|Q4|Long carrier handoff time|Set and enforce seller handling-time targets.|
|Q4|Delivery regularly beats or misses the estimate|Recalibrate estimated delivery dates by region so promises match reality.|
|Q5|A few categories dominate and many are tiny|Focus acquisition and seller onboarding on top categories. Review whether tiny categories deserve to stay.|
|Q6|Customers concentrated in a few states|Prioritize logistics and marketing there. Evaluate regional fulfillment for distant states.|
|Q6|Clear regional category differences|Localize recommendations and homepage by region.|
|Q7|Many sellers per category with price competition|Add seller ranking and quality criteria. Avoid a race to the bottom.|
|Q7|Sellers cluster by state and category|Recruit sellers in underrepresented states to cut shipping distance.|
|Q8|High seller-late rate|Introduce late-shipment penalties or seller performance scores, and auto-notify customers early.|
|Q8|Delivery late despite on-time handoff|Review carrier contracts and routes.|
|Q9|Not answerable (data gap)|Start logging seller replies (reply text, timestamp, seller_id) and track reply rate and reply time as a seller KPI.|

---

## 5. Execution Plan (success criteria)

1. **Data audit** → verify: row counts per table, null rates, duplicate keys, date-order sanity checks documented.
2. **Build one clean order-level table** (orders + items + payments + reviews + customer and seller state) → verify: row count reconciles with the orders table, and the payment-per-order rule is applied exactly once.
3. **Answer Q1 to Q8 and extra findings** → verify: every `Result:` line has a number, a sample size and an exclusion count.
4. **Validate** → verify: spot-check 5 orders by hand against raw CSVs, and recompute two headline numbers independently.
5. **Fill the decision table** with only the branches the data supports.

## 6. Known Limitations

- Anonymized data (names replaced, products unnamed), so product-level insight is limited to category and attributes.
- Single marketplace, roughly two years, so conclusions about seasonality rest on very few repetitions of each month.
- Observational only: no prices over time, promotions or marketing spend, so causal claims are out of reach.
- Review comments are sparse and in Portuguese, so text findings are directional.

---

## Sources

- Olist Brazilian E-Commerce dataset, Kaggle: https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce