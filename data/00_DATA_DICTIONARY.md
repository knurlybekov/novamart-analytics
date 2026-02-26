# NovaMart E-Commerce Operations Dataset
## Full-Stack Analytics Project Data (Jan 2023 – Dec 2024)

---

## Overview
This dataset simulates 2 years of operations for **NovaMart**, a mid-size US e-commerce company selling across 8 product categories via 45 third-party sellers. It contains **166,000+ rows across 12 interconnected files** covering the complete order-to-delivery lifecycle, customer behavior, inventory, marketing, returns, and customer support.

The data contains **intentionally embedded operational bottlenecks, data quality issues, and hidden patterns** that mirror real-world analytics challenges.

---

## File Inventory

| # | File | Rows | Description |
|---|------|------|-------------|
| 01 | orders.xlsx | 15,000 | Master order table with full lifecycle timestamps |
| 02 | order_items.xlsx | 29,668 | Line-item detail for every order |
| 03 | products.xlsx | 480 | Product catalog with costs, dimensions, categories |
| 04 | customers.xlsx | 6,200 | Customer demographics, acquisition, segments |
| 05 | sellers.xlsx | 45 | Seller/vendor profiles and warehouse assignments |
| 06 | shipping_logistics.xlsx | 91,850 | Granular shipping tracking events |
| 07 | returns_cancellations.xlsx | 3,187 | Return records with reasons and refund details |
| 08 | customer_reviews.xlsx | 8,686 | Ratings and review metadata |
| 09 | inventory_snapshots.xlsx | 8,400 | Weekly stock levels for 80 tracked products |
| 10 | marketing_campaigns.xlsx | 240 | Monthly marketing spend and performance by channel |
| 11 | support_tickets.xlsx | 2,250 | Customer service tickets with resolution tracking |
| 12 | cancellations.xlsx | 375 | Order cancellation records with reasons |

---

## Entity Relationship Map

```
customers ──┐
             ├──→ orders ──┬──→ order_items ──→ products
sellers ─────┘             │
                           ├──→ shipping_logistics
                           ├──→ returns_cancellations ──→ products
                           ├──→ customer_reviews ──→ products
                           ├──→ support_tickets
                           └──→ cancellations

inventory_snapshots ──→ products
marketing_campaigns ──→ (links to customers.acquisition_channel)
```

### Join Keys
- `order_id` — links orders to items, shipping, returns, reviews, tickets, cancellations
- `customer_id` — links orders to customers
- `seller_id` — links orders/items to sellers
- `product_id` — links items, returns, reviews, inventory to products

---

## Column Dictionaries

### 01_orders.xlsx
| Column | Type | Description |
|--------|------|-------------|
| order_id | str | Unique order identifier (ORD-XXXXXX) |
| customer_id | str | FK to customers (~2% intentionally NULL) |
| order_date | datetime | When the order was placed |
| approval_date | datetime | When payment was verified/approved |
| shipped_date | datetime | When the order left the warehouse |
| delivered_date | datetime | When customer received order (~1% have impossible dates) |
| estimated_delivery_date | datetime | Carrier's estimated delivery |
| order_status | str | Delivered / Shipped / Processing / Cancelled |
| customer_region | str | Northeast / Southeast / Midwest / Southwest / West |
| customer_city | str | Customer's city |
| seller_id | str | FK to sellers |
| carrier | str | PrimeLogistics / FastShip / BudgetPost / NationWide / ExpressCarriers |
| warehouse_origin | str | WH-Northeast / WH-Southeast / WH-Midwest / WH-Southwest / WH-West / WH-Central |
| num_items | int | Number of distinct items in order |
| order_total | float | Total order value ($) |
| shipping_cost | float | Shipping cost charged ($) |
| total_weight_kg | float | Total order weight |
| payment_method | str | Credit Card / Debit Card / PayPal / Apple Pay / Google Pay / BNPL |
| is_weekend_order | int | 1 if placed Saturday/Sunday |
| order_channel | str | Web / Mobile App / Tablet |

### 02_order_items.xlsx
| Column | Type | Description |
|--------|------|-------------|
| order_item_id | str | Unique line item ID |
| order_id | str | FK to orders |
| product_id | str | FK to products |
| seller_id | str | FK to sellers |
| quantity | int | Units ordered |
| unit_price | float | Price per unit after discount |
| discount_pct | int | Discount percentage applied (0-30%) |
| item_total | float | quantity × unit_price |
| item_weight_kg | float | Total weight of this line item |

### 03_products.xlsx
| Column | Type | Description |
|--------|------|-------------|
| product_id | str | Unique product identifier |
| product_name | str | Product display name |
| category | str | 8 categories (Electronics, Clothing, Home & Kitchen, etc.) |
| subcategory | str | 48 subcategories |
| price | float | List price ($) |
| cost_price | float | Wholesale/COGS ($) |
| weight_kg | float | Unit weight |
| length_cm / width_cm / height_cm | float | Package dimensions |
| supplier_lead_time_days | int | Days to restock from supplier |
| is_fragile | int | Requires careful handling |
| is_hazardous | int | Hazmat shipping restrictions |
| date_added | date | When product was listed |
| is_active | int | Currently available for sale |

### 04_customers.xlsx
| Column | Type | Description |
|--------|------|-------------|
| customer_id | str | Unique customer identifier |
| first_name / last_name | str | Customer name |
| email_domain | str | Email provider domain |
| city / region / zip_code | str | Location data |
| acquisition_channel | str | How they found NovaMart (links to marketing) |
| signup_date | date | Account creation date |
| customer_segment | str | New / Active / VIP / At-Risk / Churned |
| lifetime_value_estimate | float | Estimated CLV ($) |
| preferred_payment | str | Default payment method |
| email_opt_in | int | Marketing email consent |
| has_app | int | Has mobile app installed |

### 05_sellers.xlsx
| Column | Type | Description |
|--------|------|-------------|
| seller_id | str | Unique seller identifier |
| seller_name | str | Business name |
| warehouse_location | str | Assigned warehouse hub |
| city / state | str | Seller HQ location |
| avg_processing_hours | float | Historical avg time to process orders |
| quality_score | float | Seller quality rating (1-5) |
| total_products_listed | int | Number of active listings |
| join_date | date | When seller joined platform |
| is_verified | int | Identity/quality verified |
| fulfillment_type | str | Seller Fulfilled / Warehouse Fulfilled / Dropship |
| return_policy_days | int | Return window length |

### 06_shipping_logistics.xlsx
| Column | Type | Description |
|--------|------|-------------|
| event_id | str | Unique event identifier |
| order_id | str | FK to orders |
| carrier | str | Carrier name |
| warehouse_origin | str | Origin warehouse |
| event_type | str | Order Received / Payment Verified / Picked & Packed / Shipped / At Regional Hub / Out for Delivery / Delivery Exception / Exception Resolved / Delivered |
| event_timestamp | datetime | When the event occurred |
| event_sequence | int | Sequence number within order |

### 07_returns_cancellations.xlsx
| Column | Type | Description |
|--------|------|-------------|
| return_id | str | Unique return identifier |
| order_id / order_item_id / product_id / customer_id | str | Foreign keys |
| return_initiated_date | date | When return was requested |
| return_received_date | date | When item arrived back (~12% NULL = not yet received) |
| return_reason | str | Defective / Wrong Item / Not as Described / Changed Mind / Size Issue / Late Delivery / Better Price / Quality |
| return_condition | str | New/Unopened / Like New / Used / Damaged |
| refund_amount | float | Refund issued ($) |
| refund_method | str | Original Payment / Store Credit / Exchange |
| return_status | str | Completed / In Transit / Pending Inspection / Rejected |
| restocking_fee | float | Fee charged for buyer's-remorse returns |

### 08_customer_reviews.xlsx
| Column | Type | Description |
|--------|------|-------------|
| review_id | str | Unique review identifier |
| order_id / customer_id / product_id | str | Foreign keys |
| rating | int | Overall rating 1-5 |
| review_date | date | When review was posted |
| delivery_rating | int | Delivery experience rating 1-5 |
| product_rating | int | Product quality rating 1-5 |
| was_delivery_late | int | 1 if delivered after estimate |
| days_late | int | Number of days past estimate |
| review_topics | str | Pipe-delimited topic tags |
| has_photo | int | Review includes photo |
| helpful_votes | int | Upvotes from other customers |
| verified_purchase | int | Confirmed buyer |

### 09_inventory_snapshots.xlsx
| Column | Type | Description |
|--------|------|-------------|
| snapshot_id | str | Unique snapshot identifier |
| snapshot_date | date | Weekly Monday snapshot |
| product_id | str | FK to products |
| warehouse | str | Warehouse location |
| quantity_on_hand | int | Total units in warehouse |
| quantity_reserved | int | Units allocated to pending orders |
| quantity_available | int | Units available for new orders |
| reorder_point | int | Threshold triggering restock |
| is_stockout | int | 1 if quantity = 0 |
| weekly_demand | int | Units sold that week |
| weekly_received | int | Units restocked that week |
| days_of_supply | float | Estimated days until stockout |

### 10_marketing_campaigns.xlsx
| Column | Type | Description |
|--------|------|-------------|
| campaign_id | str | Unique campaign identifier |
| month | str | YYYY-MM |
| channel | str | Marketing channel (matches customer acquisition_channel) |
| spend | float | Monthly spend ($) |
| impressions | int | Ad views |
| clicks | int | Ad clicks |
| conversions | int | Purchases attributed |
| revenue_attributed | float | Revenue from conversions ($) |
| new_customers_acquired | int | First-time buyers |
| cost_per_click | float | spend / clicks |
| cost_per_acquisition | float | spend / conversions |
| return_on_ad_spend | float | revenue / spend (ROAS) |
| campaign_type | str | Awareness / Conversion / Retargeting / Seasonal |

### 11_support_tickets.xlsx
| Column | Type | Description |
|--------|------|-------------|
| ticket_id | str | Unique ticket identifier |
| order_id / customer_id | str | Foreign keys |
| created_date | datetime | When ticket was opened |
| resolved_date | datetime | When ticket was closed (NULL if open) |
| issue_type | str | WISMO / Damaged / Wrong Item / Return / Refund / Quality / Missing / Delivery / Cancellation / Payment / Modification / Address / ETA |
| priority | str | High / Medium / Low |
| channel | str | Email / Live Chat / Phone / Social Media |
| resolution_hours | float | Hours to resolve (NULL if unresolved) |
| status | str | Resolved / Open / Escalated / Awaiting Customer |
| satisfaction_score | int | CSAT 1-5 (NULL if unresolved) |
| agent_id | str | Support agent identifier |
| first_response_minutes | float | Time to first reply |
| num_interactions | int | Back-and-forth messages |

### 12_cancellations.xlsx
| Column | Type | Description |
|--------|------|-------------|
| cancellation_id | str | Unique cancellation identifier |
| order_id / customer_id | str | Foreign keys |
| cancellation_date | datetime | When order was cancelled |
| cancellation_reason | str | Customer Request / Payment Failed / Out of Stock / Fraud / Address / Duplicate / Price Changed |
| was_order_total | float | Value of cancelled order ($) |

---

## Data Quality Issues (Intentional)
These simulate real-world data problems you'll need to handle:

1. **~2% of orders have NULL customer_id** — broken join, needs investigation
2. **~1% of delivered orders have delivered_date BEFORE order_date** — impossible timestamps
3. **~12% of returns have NULL return_received_date** — item not yet received back
4. **Some review ratings may not align with delivery/product ratings** — survey inconsistency
5. **Inventory snapshots only cover 80 of 480 products** — incomplete coverage

---

## Suggested Analysis Roadmap

### Phase 1: Data Cleaning & Pipeline (SQL + Python)
- Handle NULLs, impossible dates, broken joins
- Build a star schema / dimensional model
- Create calculated fields (processing time, transit time, is_late, profit margin)

### Phase 2: Funnel & Bottleneck Analysis
- Map the order lifecycle funnel with median durations at each stage
- Identify the top bottleneck stages by volume and dollar impact
- Segment by warehouse, carrier, category, time period

### Phase 3: Root Cause Deep Dives
- Why does WH-Southeast underperform?
- What drives the winter shipping crisis?
- Which product-seller combos have the worst return economics?
- Is the Q4 crunch a capacity issue or a process issue?

### Phase 4: Financial Impact Quantification
- Calculate cost of late deliveries (returns + negative reviews + churn)
- Model revenue lost to stockouts
- Build a marketing ROI comparison with channel reallocation recommendations
- Estimate savings from fixing the top 3 bottlenecks

### Phase 5: Dashboard & Storytelling
- Executive KPI dashboard (orders, revenue, fulfillment rate, NPS)
- Operations drill-down (warehouse × carrier performance matrix)
- Marketing efficiency scorecard
- "Top 5 Revenue Leaks" summary with prioritized recommendations
