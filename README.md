# When Delivery Promises Break: Olist Data Warehouse

**Michael Skinner and Sukhman Brar, Project 0.** 
## The business case

Our client is the Head of Marketplace Operations at a Brazilian online marketplace. The company is fictional, and the data is Olist's public 2016–2018 orders. Late deliveries are the client's biggest source of poor reviews. Before funding a logistics fix, the client wants to know where lateness is concentrated, how much it hurts customer satisfaction, and which sellers should be reviewed first. We loaded the marketplace's orders into a cloud data warehouse and answered five questions from it:

1. Which seller-to-customer state corridors and product categories have the highest late-delivery rates?
2. How much lower are review scores when an order arrives late, and does it get worse the later it is?
3. Are delivery promises (estimated dates) too optimistic in some regions?
4. Does a high freight charge relative to price go with lower satisfaction?
5. Which sellers went onto the review watchlist, and did they improve afterwards?

**What we found:** <!-- one or two sentences once the queries run, e.g. "Orders 8+ days late average _ stars vs _ on time; _ corridors account for _% of late items." -->

**What we recommend:** <!-- the three actions from the final slide -->

---

## Dataset

- **Source:** [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) on Kaggle, licensed CC BY-NC-SA 4.0. It's used here for non-commercial education, with attribution to Olist.
- **Files used:** 8 of the 9 CSVs: orders, order items, products, sellers, customers, reviews, payments, and category translation. The geolocation file is skipped because it doesn't answer any of the five questions.
- **Size:** 112,650 order items across 99,441 orders.
- **The data is not committed to this repo.** Download it yourself (see below).

## How it works

```
Kaggle CSVs ─► config.yaml ─► Python ingest job (pandas: validate, clean, dedupe)
                                   ├─► p0_staging.<source>   (clean rows)
                                   ├─► p0_staging.rejects    (bad rows + reason)
                                   └─► p0_staging.load_log   (one row per source per run)
p0_staging ─► sql/warehouse/*.sql ─► p0_warehouse star schema ─► sql/analysis/*.sql ─► charts + deck
```

Everything runs in the free **BigQuery sandbox**. The sandbox doesn't allow `INSERT`, `UPDATE` or `MERGE`, so every table is written by a load job or by `CREATE OR REPLACE TABLE … AS SELECT`. Sandbox tables expire after 60 days, so re-run the job if yours have expired.

## Repository layout

```
projects/P0/
  README.md
  config/config.example.yaml     # copy to config/config.yaml (gitignored)
  src/ingest/                    # config.py, validate.py, clean.py, load.py, log.py, main.py
  src/features.py                # is_late, days_late bucket, freight_ratio, promise_days
  src/build_warehouse.py         # runs sql/warehouse/*.sql in order
  sql/warehouse/                 # dimensions, SCD2 seller dimension, fact table
  sql/analysis/                  # one file per client question
  charts/make_charts.py          # queries BigQuery, saves PNGs to charts/
  tests/                         # pytest + small fixture CSVs
  docs/schema.png                # star schema diagram
  docs/sample_load_log.csv
  docs/sample_rejects.csv
  deck/P0_deck.pdf
  requirements.txt
  .env.example
```

---

## Run it from a fresh clone

### 1. Install

```bash
git clone https://github.com/2494-Gen-AI-CN/MichaelSkinner.git
cd MichaelSkinner/projects/P0
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

You also need the [Google Cloud CLI](https://cloud.google.com/sdk/docs/install) (`gcloud`, `bq`) and, to download the data, the [Kaggle CLI](https://github.com/Kaggle/kaggle-api) (`pip install kaggle`, with a `~/.kaggle/kaggle.json` token).

### 2. Get the data

```bash
kaggle datasets download -d olistbr/brazilian-ecommerce -p data/raw --unzip
```

`data/` is gitignored.

### 3. BigQuery credentials

1. Create a project in the [BigQuery sandbox](https://cloud.google.com/bigquery/docs/sandbox). A Google account is enough, with no credit card.
2. Sign in with Application Default Credentials:

   ```bash
   gcloud auth application-default login
   ```

   Or, using a service-account key, keep the JSON **outside this repo** and point to it:

   ```bash
   export GOOGLE_APPLICATION_CREDENTIALS="$HOME/keys/p0-bigquery.json"
   ```
3. Set your project ID:

   ```bash
   cp .env.example .env          # then edit: GCP_PROJECT=your-project-id
   ```

No credentials or `.env` file are ever committed. `.gitignore` excludes `.env`, `*.json` keys, and `data/`.

### 4. Configure

```bash
cp config/config.example.yaml config/config.yaml
```

The config names each raw file, its format, its key and its schema. The BigQuery project is read from `GCP_PROJECT`. **Adding a source is a config change, not a code change:** add a block under `sources:`.

```yaml
bigquery:
  project: ${GCP_PROJECT}
  staging_dataset: p0_staging
  warehouse_dataset: p0_warehouse
sources:
  - name: order_items
    path: data/raw/olist_order_items_dataset.csv
    format: csv
    key: [order_id, order_item_id]
    schema: {order_id: string, order_item_id: int, product_id: string, seller_id: string,
             shipping_limit_date: timestamp, price: float, freight_value: float}
  # … orders, products, sellers, customers, reviews, payments, category_translation
```

### 5. Run the ingest job

```bash
python -m src.ingest.main --config config/config.yaml
```

The job validates and cleans each source, loads the clean rows to staging, sends bad rows to `p0_staging.rejects` with a reason, and writes one `p0_staging.load_log` row per source.

**Running it twice adds nothing.** Each file's SHA-256 hash is checked against the load log. An already-loaded file is logged as `skipped` with `rows_loaded = 0`.

### 6. Build the warehouse

```bash
python -m src.build_warehouse --config config/config.yaml
```

### 7. Run the tests

```bash
pytest
```

The tests cover the validation, cleansing, feature and idempotency functions using small fixture files. **No test connects to BigQuery.**

### 8. Run the analysis queries

```bash
for f in sql/analysis/*.sql; do echo "== $f"; bq query --project_id="$GCP_PROJECT" --use_legacy_sql=false < "$f"; done
```

Or open any file in the BigQuery console and run it there. Each file starts with the client question it answers and what the answer means.

### 9. Draw the charts

```bash
python charts/make_charts.py
```

This saves three PNGs to `charts/`.

---

## Cleansing rules

These are applied with pandas before loading. Failing rows go to `p0_staging.rejects` with the reason shown.

| Rule | Reject reason |
| --- | --- |
| All expected columns present | `missing column <name>` (stops that source) |
| Keys not null: `order_id`, `product_id`, `seller_id`, `customer_id` | `null key: <column>` |
| Timestamps parse | `bad timestamp: <column>` |
| `price > 0` | `non-positive price` |
| `freight_value >= 0` | `negative freight` |
| `review_score` between 1 and 5 | `review_score out of range` |
| `order_status` is a known status | `unknown status` |
| A `delivered` order has a delivery date on or after its purchase date | `delivered without valid delivery date` |
| Exact duplicates on the source key | Removed and counted in `duplicates_removed`, not rejected |

Transform rules applied in SQL:

- **One review per order.** The reviews file has 99,224 rows for 98,673 orders, and the latest `review_answer_timestamp` is kept.
- **Payments summed to order level** before joining, so items aren't multiplied.
- **Products without a category** (610) are mapped to an `unknown` category, not dropped.

## Star schema

![Star schema](docs/schema.png)

**Fact grain: one row per order item `(order_id, order_item_id)`, 112,650 rows.** Review score and payment value are order-level facts repeated on each item, so order-level questions count with `COUNT(DISTINCT order_id)`.

| Table | Type | Key columns |
| --- | --- | --- |
| `fact_order_item` | Fact | `order_id`, `order_item_id`, `product_key`, `seller_key`, `customer_key`, `purchase_date_key`, `price`, `freight_value`, `delivery_days`, `days_late`, `is_late`, `review_score` |
| `dim_product` | Dimension | `product_key`, `category_en`, `weight_g`, `photos_qty` |
| `dim_seller` | Dimension, SCD type 2 | `seller_key`, `seller_id`, `seller_state`, `perf_tier`, `effective_from`, `effective_to`, `is_current` |
| `dim_customer` | Dimension | `customer_key`, `customer_unique_id`, `customer_state`, `customer_city` |
| `dim_date` | Dimension | `date_key`, `date`, `month`, `quarter`, `year`, `weekday` |

### SCD type 2: seller performance tier

The client's review policy, which we set and which is not a finding, puts a seller on the **Watchlist** for a quarter when they shipped at least 20 delivered items that quarter with a late rate of 15% or more. Everyone else active that quarter is **Standard**.

The tier is recomputed every quarter from the data. When a seller's tier changes, the old row is closed (`effective_to` = quarter end, `is_current = FALSE`) and a new row opens. `dim_seller` is built in a single `CREATE OR REPLACE` using `LAG` to find tier changes and `LEAD` to set `effective_to`. The current row carries `effective_to = 9999-12-31`.

**What changed:** <!-- e.g. "_ sellers changed tier at least once; _ moved onto the Watchlist in 2017 Q4." -->

`sql/analysis/05_seller_watchlist_history.sql` uses this history. It joins each sale to the tier the seller held **on the purchase date** (`effective_from <= purchase_date < effective_to`).

## Engineered features

| Feature | Type | Definition |
| --- | --- | --- |
| `is_late` | Flag | Delivered after the estimated delivery date |
| `days_late` bucket | Bucket | Delivered minus estimated date: early, on time, 1–3, 4–7, 8+ days late |
| `freight_ratio` | Rate | `freight_value / price` |
| `promise_days` | Measure | Estimated delivery date minus purchase date |

## Correlation

**Days late vs. review score, at order level, Spearman ρ = _.** <!-- fill in -->

For the client: the later an order arrives, the lower its review, so cutting lateness is likely to lift ratings.

**Why it may not be a cause:** late orders also tend to be heavier, travel further, or come from weaker sellers, and any of those could lower reviews on their own.

## Analysis files

| File | Client question |
| --- | --- |
| `01_late_rate_by_corridor.sql` | Which seller-to-customer state corridors are late most often? (minimum 100 orders) |
| `02_review_by_lateness.sql` | How do review scores change as orders get later? |
| `03_promise_accuracy_by_region.sql` | Are promised delivery dates too optimistic in some states? |
| `04_freight_burden_by_category.sql` | Do high freight ratios go with lateness and low reviews? |
| `05_seller_watchlist_history.sql` | Did sellers improve after going onto the Watchlist? **(uses the SCD2 history)** |
| `06_value_on_late_orders.sql` | How much merchandise value sits on late orders, by category? |

## Samples

- `docs/sample_load_log.csv`: two runs. The second shows every source `skipped` with 0 rows loaded.
- `docs/sample_rejects.csv`: rejected rows with their reasons.

## Limits

- The data covers 2016–2018 for one Brazilian marketplace. It isn't current, and it isn't Canadian.
- Value on late orders is **not lost revenue**, and freight charged is not shipping cost.
- Reviews are order-level and self-selected.
- All the findings are associations. None of them measure the effect of an intervention.
