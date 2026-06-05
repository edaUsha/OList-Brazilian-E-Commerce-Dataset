# Olist E-Commerce: Customer Revenue & Delivery Intelligence Analysis

> *"An e-commerce marketplace is growing in orders but has a near-zero repeat purchase rate. This analysis identifies the operational, product, and experience factors that drive — and kill — customer satisfaction, and quantifies the revenue impact of fixing them."*

---

## Table of Contents
1. [Background and Overview](#background-and-overview)
2. [Data Structure Overview](#data-structure-overview)
3. [Executive Summary](#executive-summary)
4. [Insights Deep Dive](#insights-deep-dive)
   - [Revenue & Growth Diagnosis](#1-revenue--growth-diagnosis)
   - [Customer Segmentation — RFM & K-Means](#2-customer-segmentation--rfm--k-means)
   - [Product Category Revenue Analysis](#3-product-category-revenue-analysis)
   - [Delivery Performance vs Customer Satisfaction](#4-delivery-performance-vs-customer-satisfaction)
5. [Business Recommendations](#business-recommendations)
6. [Technical Stack](#technical-stack)

---

## Background and Overview

### The Business
Olist is a Brazilian e-commerce marketplace that connects small businesses to major retail channels. Between 2016 and 2018, Olist processed over 100,000 orders across multiple product categories and seller regions, primarily serving customers across Brazil.

### The Business Problem
Despite consistent month-on-month order growth — peaking in November 2017 — the business faces a structural retention problem: **97% of customers purchase exactly once and never return.** The marketing team continues acquiring new customers, but the revenue base is fundamentally unstable. There is no retention engine.

This analysis was commissioned to answer three questions:
- Is the revenue decline real, and what is driving it?
- Who are the customers, and which ones matter most?
- Why are customers not returning, and what can the business do about it?

### Analytical Approach
The analysis was conducted in four layers — each building on the previous to move from diagnosis to root cause to recommendation:

```
Revenue Diagnosis → Customer Segmentation → Category Analysis → Delivery Root Cause
```

All analysis was performed in Python using the publicly available Olist Brazilian E-Commerce dataset on Kaggle.

---

## Data Structure Overview

### Source
**Brazilian E-Commerce Public Dataset by Olist**
[https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)

Real, anonymised commercial transaction data. 100,000 orders from 2016–2018 across 9 CSV files.

### Tables Used

| Table | Rows | Purpose |
|---|---|---|
| `olist_orders_dataset.csv` | ~99,000 | Spine of the dataset — every order, status, and delivery timestamp |
| `olist_customers_dataset.csv` | ~99,000 | Customer identity; provides `customer_unique_id` for repeat buyer tracking |
| `olist_order_payments_dataset.csv` | ~103,000 | Actual payment value per order; used for all revenue calculations |
| `olist_order_items_dataset.csv` | ~112,000 | Order-to-product mapping; used for category analysis |
| `olist_products_dataset.csv` | ~32,000 | Product metadata including category name in Portuguese |
| `olist_order_reviews_dataset.csv` | ~100,000 | Customer review scores (1–5) per order |
| `product_category_name_translation.csv` | 71 | Portuguese-to-English category name translation |

### Key Design Decision — `customer_id` vs `customer_unique_id`

> ⚠️ **Critical trap:** Olist assigns a new `customer_id` for every order. A customer who purchased 5 times has 5 different `customer_id` values. All repeat-buyer analysis must use `customer_unique_id`. Using `customer_id` would make every customer appear to have purchased exactly once — silently corrupting all RFM and retention analysis.



---

## Executive Summary

Delivery performance analysis across 95,824 orders reveals a significant paradox: Olist's logistics infrastructure delivers **92% of orders ahead of schedule** — averaging 12 days early — yet this operational excellence is invisible to customers due to conservative delivery estimates. Meanwhile, the **2.9% of orders that arrive late** (2,780 orders, BRL 508K revenue) receive an average review score of **1.70 out of 5**, with 70% resulting in 1-star reviews — customers who are almost certain never to return.

Three zero-to-low-cost interventions — recalibrating delivery estimates, implementing proactive late-order alerts, and marketing the early delivery rate — could meaningfully improve satisfaction scores and repeat purchase intent without any change to logistics operations.

### Key Metrics at a Glance

| Metric | Value |
|---|---|
| Total orders analysed | 95,832 |
| Total revenue | BRL 15,422,462 |
| Date range | September 2016 – October 2018 |
| Unique customers | ~93,000 |
| Repeat purchase rate | **97% of customers bought once** |
| Revenue from top 20% of customers | 53.5% of total revenue |
| Orders delivered early | **92%** |
| Average days delivered early | 11.9 days |
| Late order avg review score | **1.70 / 5** |
| 1-star rate on late orders | **70%** |
| Revenue at confirmed risk | BRL 4,085,234 (26.5%) |

---

## Insights Deep Dive

---

### 1. Revenue & Growth Diagnosis

#### Monthly Revenue Trend
Revenue grew steadily from late 2016 through 2017, peaking sharply in **November 2017** — consistent with Black Friday and seasonal demand. Revenue dipped in December 2017 before recovering to a stable but bumpy trajectory through 2018. The business is not in absolute decline — it is **acquisition-dependent**, meaning revenue tracks new customer volume rather than any retained base.

![Monthly Revenue Trend](segment_customer_count.png)
<img width="782" height="277" alt="image" src="https://github.com/user-attachments/assets/a5d16a00-75e1-4125-ae09-396d013d5240" />


#### The Repeat Purchase Problem
```
Returning customers:   3%   (~2,800 unique customers)
One-time buyers:      97%  (~90,000 unique customers)
```

Every month's revenue depends almost entirely on that month's new customer acquisition.

#### Revenue Concentration
The top 20% of customers by spend drive **53.5% of total revenue**. 

---

### 2. Customer Segmentation — RFM & K-Means


#### RFM Segmentation

Customers were scored on three dimensions: **Recency** (days since last purchase), **Frequency** (number of orders), and **Monetary** (total spend). Each dimension was scored 1–5 using quintile distribution and customers were assigned to eight segments.


1. Needs Attention
2. At Risk
3. Loyal Customers
4. Promising
5. Champions
6. Lost
7. Dormat
8. Potential
   


#### Why RFM Has Limitations Here

In this dataset, **97% of customers have Frequency = 1**. The Frequency dimension is near-constant.

**Since almost all customers place only one order, Frequency provides little variation, making standard RFM segmentation less informative for decision-making.**


#### K-Means Clustering (k=4)

To validate segmentation without human-defined thresholds, K-Means clustering was applied to scaled RFM features. The optimal k=4 was selected based on silhouette score (0.489) balanced with business interpretability — k=2 scored higher (0.739) but produced **business-useless segments**.

| Cluster | Label | Customers | Avg Recency | Avg Frequency | Avg Spend | Total Revenue |
|---|---|---|---|---|---|---|
| 0 | Recent One-Time Buyers | 50,643 | 128 days | 1.0 | BRL 134 | BRL 6.8M |
| 1 | Lapsed One-Time Buyers | 37,526 | 387 days | 1.0 | BRL 133 | BRL 5.0M |
| 2 | Repeat Buyers Going Cold | 2,772 | 220 days | 2.1 | BRL 290 | BRL 803K |
| 3 | High-Spend Lapsed | 2,416 | 240 days | 1.0 | BRL 1,161 | BRL 2.8M |




### 3. Product Category Revenue Analysis

**Question answered:** Which categories drive revenue, and where are the hidden opportunities?

#### Top 10 Categories by Revenue

![Top 10 Product Categories by Revenue](top10_category_revenue.png)
<img width="737" height="297" alt="image" src="https://github.com/user-attachments/assets/24d3abfa-0ed8-4ff9-94b0-226b383e5c8f" />

The top 3 categories account for a significant share of total revenue — indicating category concentration risk. If demand in these categories softens, total revenue is materially impacted.

---

### 4. Delivery Performance vs Customer Satisfaction


#### Delivery Performance Distribution

| Delivery Bucket | Orders | % of Total | Avg Review Score | 1-Star Rate |
|---|---|---|---|---|
| Very Early (≤ −15 days) | 34,771 | 36.3% | **4.32** | 6.5% |
| Early (−14 to −1 days) | 53,392 | 55.7% | **4.28** | 6.6% |
| On Time (0 to +7 days) | 4,880 | 5.1% | 3.06 | 32.7% |
| Late (+8 to +15 days) | 1,601 | 1.7% | 1.67 | 70.1% |
| Very Late (> +15 days) | 1,179 | 1.2% | 1.73 | 69.1% |

![Average Review Score by Delivery Performance](delivery_vs_review.png)
<img width="625" height="297" alt="image" src="https://github.com/user-attachments/assets/b36c2cf9-dc45-43f9-93f2-f05f108b9fe7" />


#### Finding 1 — Operational Strength Hidden in Plain Sight
**92% of orders are delivered early**, with a mean of 11.9 days ahead of the estimated date. 

#### Finding 2 — The Estimation Problem
Delivery estimates are set ~12 days later than actual performance. 

#### Finding 3 — The Late Delivery Crisis
While only 2.9% of orders arrive late, the damage is catastrophic:
- Average review score: **1.70 / 5**
- 1-star review rate: **70%**
- Revenue at risk: **BRL 508,558**
- These customers, in a business already struggling with 97% one-time buyers, are guaranteed not to return

#### Finding 4 — On-Time Scores Worse Than Early
On-Time deliveries (meeting the estimated date) average only **3.06**, with a 32.7% one-star rate. C

![Review Score Distribution by Delivery Performance](review_distribution_by_delivery.png)
<img width="632" height="296" alt="image" src="https://github.com/user-attachments/assets/611a7879-91fc-4448-99b6-899e9ce81c94" />

---

## Business Recommendations

### Recommendation 1 — Recalibrate Delivery Estimates *(Zero cost)*
Set estimated delivery dates 10–12 days earlier to reflect actual median delivery performance. This converts 92% of standard deliveries into genuine early surprises. No logistics change required — pure expectation management. Projected effect: meaningfully higher satisfaction scores across 88,000+ annual orders.

### Recommendation 2 — Proactive Late-Order Alerts *(Low cost)*
Trigger an automated customer notification when a shipment crosses day +3 past the estimate — before the customer notices. Include a discount voucher on the next order. Even retaining 20% of the 2,780 affected customers annually converts guaranteed churners into a second-purchase opportunity, recovering a portion of BRL 508K in at-risk revenue.


---

## Technical Stack

| Tool | Purpose |
|---|---|
| Python 3 | Core analysis language |
| Pandas | Data loading, cleaning, merging, aggregation |
| Matplotlib | All visualisations |
| Seaborn | Heatmap for RFM vs K-Means crosstab |
| Scikit-learn | StandardScaler, KMeans, silhouette_score |
| Jupyter Notebook | Analysis environment |

---

*Analysis by — Data Science Portfolio Project, June 2026*
*Dataset: Brazilian E-Commerce Public Dataset by Olist, Kaggle*
