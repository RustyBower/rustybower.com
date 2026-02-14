---
title: "Finding Fraud in $1 Trillion of Medicaid Data with DuckDB"
date: 2026-02-14
draft: false
tags: ["data", "duckdb", "fraud", "medicaid", "python", "jupyter", "parquet"]
---

CMS recently published provider-level Medicaid spending data from T-MSIS — every fee-for-service, managed care, and CHIP claim from 2018 through 2024, aggregated by billing provider, procedure code, and month. 227 million rows. $1.09 trillion in payments. I wanted to see what falls out when you run some basic fraud heuristics against it.

The dataset is available at [opendata.hhs.gov/datasets/medicaid-provider-spending/](https://opendata.hhs.gov/datasets/medicaid-provider-spending/).

## The dataset

The download is a single 2.94 GB parquet file with seven columns:

| Column | Type | Description |
|--------|------|-------------|
| `BILLING_PROVIDER_NPI_NUM` | string | NPI of the billing provider |
| `SERVICING_PROVIDER_NPI_NUM` | string | NPI of the servicing provider |
| `HCPCS_CODE` | string | Procedure code |
| `CLAIM_FROM_MONTH` | string | Month (YYYY-MM) |
| `TOTAL_UNIQUE_BENEFICIARIES` | int64 | Unique patients |
| `TOTAL_CLAIMS` | int64 | Claim count |
| `TOTAL_PAID` | double | Dollars paid by Medicaid |

Here's what the first few rows look like — the data is sorted by `TOTAL_PAID` descending, so the biggest line items are at the top:

```text
┌──────────────────────────┬────────────────────────────┬────────────┬──────────────────┬────────────────────────────┬──────────────┬──────────────┐
│ BILLING_PROVIDER_NPI_NUM │ SERVICING_PROVIDER_NPI_NUM │ HCPCS_CODE │ CLAIM_FROM_MONTH │ TOTAL_UNIQUE_BENEFICIARIES │ TOTAL_CLAIMS │  TOTAL_PAID  │
├──────────────────────────┼────────────────────────────┼────────────┼──────────────────┼────────────────────────────┼──────────────┼──────────────┤
│ 1376609297               │ 1376609297                 │ T1019      │ 2024-07          │                      39765 │      1205701 │ 118887675.31 │
│ 1376609297               │ 1376609297                 │ T1019      │ 2024-08          │                      39677 │      1152534 │ 115561066.11 │
│ 1376609297               │ 1376609297                 │ T1019      │ 2024-05          │                      39678 │      1157235 │  112823255.3 │
│ 1376609297               │ 1376609297                 │ T1019      │ 2024-06          │                      39834 │      1164582 │ 111449173.13 │
│ 1376609297               │ 1376609297                 │ T1019      │ 2024-09          │                      39527 │      1099808 │ 111199832.57 │
└──────────────────────────┴────────────────────────────┴────────────┴──────────────────┴────────────────────────────┴──────────────┴──────────────┘
```

Right away you notice NPI `1376609297` billing $118M in a single month for T1019 (personal care services) with 1.2 million claims. That's going to come up again.

A quick summary of the full dataset:

```python
con.sql(f"""
    SELECT
        COUNT(*) as total_rows,
        COUNT(DISTINCT BILLING_PROVIDER_NPI_NUM) as unique_billing_npis,
        COUNT(DISTINCT SERVICING_PROVIDER_NPI_NUM) as unique_servicing_npis,
        COUNT(DISTINCT HCPCS_CODE) as unique_procedures,
        MIN(CLAIM_FROM_MONTH) as earliest_month,
        MAX(CLAIM_FROM_MONTH) as latest_month,
        SUM(TOTAL_PAID) as total_paid
    FROM '{PATH}'
""")
```

**227M rows**, **617,503 billing providers**, **1.6M servicing providers**, **10,881 procedure codes**, spanning **84 months** (Jan 2018 – Dec 2024), totaling **$1.09 trillion**.

## Why parquet, not CSV

HHS offers this dataset as a ZIP download. Inside is parquet, not CSV. This matters a lot for a file this size.

**Compression.** The parquet file is 2.94 GB. The same data in CSV would be roughly 15-20 GB. Parquet uses column-oriented compression (dictionary encoding for the repeated NPI strings, run-length encoding for sorted columns, and Snappy/ZSTD compression on top). The three NPI/code string columns have high cardinality but lots of repetition within row groups — exactly the pattern parquet compresses best.

**Column pruning.** If your query only touches `BILLING_PROVIDER_NPI_NUM` and `TOTAL_PAID`, parquet readers skip the other five columns entirely. With CSV, you always read every byte of every row. For a 7-column file, that's up to 5/7ths of I/O you never have to do.

**Predicate pushdown.** DuckDB can push `WHERE` clauses down into the parquet scan, skipping entire row groups whose min/max statistics prove no rows match. When I filter `WHERE TOTAL_CLAIMS > 1000`, DuckDB doesn't even read row groups where the max claims value is under 1,000.

**Type preservation.** CSV makes you guess types and parse strings. Parquet knows `TOTAL_PAID` is a `double` and `TOTAL_CLAIMS` is an `int64` — no parsing overhead, no type ambiguity.

For a dataset with 227M rows, these differences compound. The parquet file isn't just smaller — it's fundamentally faster to query.

## Don't load 227M rows into pandas

My first attempt was naive:

```python
import pandas as pd
df = pd.read_parquet('/data/medicaid-provider-spending.parquet')
```

The Jupyter kernel OOM-killed immediately on a 16 GB pod.

The math makes it obvious in hindsight. Parquet decompresses to a much larger in-memory representation, and pandas stores each string as a Python object with ~56 bytes of overhead regardless of string length. Three string columns across 227M rows:

```text
227M rows × 3 string columns × ~56 bytes/object ≈ 38 GB
+ 2 int64 columns × 8 bytes × 227M              ≈  3.6 GB
+ 1 float64 column × 8 bytes × 227M              ≈  1.8 GB
                                                  ≈ 43 GB total
```

Even a 64 GB pod would be tight once you start doing groupbys that create intermediate DataFrames.

**The fix: DuckDB.** It queries parquet files directly on disk using memory-mapped I/O, streaming through row groups without materializing the full dataset. Peak memory usage stays in the low single-digit GBs. Every query in this analysis ran in under 3 minutes on that same 16 GB pod.

```python
import duckdb

PATH = '/data/medicaid-provider-spending.parquet'
con = duckdb.connect()

con.sql(f"SELECT COUNT(*) FROM '{PATH}'")
# → (227083361,)
```

No `read_parquet()`, no DataFrames, no OOM. DuckDB treats the parquet file as a table and pushes the full query plan — filters, aggregations, window functions — down to the scan.

The trick is to do all the heavy lifting in DuckDB SQL, then call `.df()` only on the final result sets (which are thousands of rows, not millions):

```python
# Heavy aggregation stays in DuckDB — result is ~68K rows, perfectly fine for pandas
outliers = con.sql(f"""
    SELECT ... FROM '{PATH}'
    GROUP BY ... HAVING z_score > 3
""").df()
```

## Five fraud signals

Healthcare fraud detection comes down to finding providers whose billing patterns deviate sharply from their peers. I built five independent signals and combined them into a composite risk score. Here's every query.

### 1. Cost-per-beneficiary outliers

For each provider and procedure code, I sum the total dollars paid and divide by unique beneficiaries across all months. Then z-score within each procedure code (restricting to procedures with 20+ providers for stable statistics). Anything above z=3 is flagged.

```python
outliers_cost = con.sql(f"""
    WITH prov_proc AS (
        SELECT
            BILLING_PROVIDER_NPI_NUM,
            HCPCS_CODE,
            SUM(TOTAL_PAID) as total_paid,
            SUM(TOTAL_UNIQUE_BENEFICIARIES) as total_benes,
            SUM(TOTAL_CLAIMS) as total_claims,
            COUNT(DISTINCT CLAIM_FROM_MONTH) as months_active,
            SUM(TOTAL_PAID) / GREATEST(SUM(TOTAL_UNIQUE_BENEFICIARIES), 1)
                as paid_per_bene
        FROM '{PATH}'
        GROUP BY BILLING_PROVIDER_NPI_NUM, HCPCS_CODE
    ),
    proc_stats AS (
        SELECT
            HCPCS_CODE,
            COUNT(*) as num_providers,
            AVG(paid_per_bene) as mean_ppb,
            STDDEV_POP(paid_per_bene) as std_ppb
        FROM prov_proc
        GROUP BY HCPCS_CODE
        HAVING COUNT(*) >= 20
    ),
    scored AS (
        SELECT
            p.*,
            (p.paid_per_bene - s.mean_ppb) / NULLIF(s.std_ppb, 0)
                as z_paid_per_bene
        FROM prov_proc p
        JOIN proc_stats s USING (HCPCS_CODE)
    )
    SELECT * FROM scored
    WHERE z_paid_per_bene > 3
    ORDER BY z_paid_per_bene DESC
""").df()
```

This scans the full 2.94 GB parquet file, builds the two-level aggregation entirely in DuckDB, and returns only the outliers. It ran in **192 seconds**.

**Result:** 68,037 provider/procedure combinations flagged across **27,348 unique billing NPIs**.

The top hit: NPI `1467653303` billing **$10,416 per beneficiary** for CPT 99213 — a standard 15-minute office visit that typically pays around $57. A z-score of **183**. They billed $541K for 52 beneficiaries in a single month. Either those were the most expensive office visits in the history of medicine, or something's wrong.

Other standouts:
- NPI `1437412525`: **$22,064/beneficiary** for 92507 (speech therapy), z=94
- NPI `1073608998`: **$65,778/beneficiary** for J3490 (unclassified drugs), z=68
- NPI `1427138726`: **$8,817/beneficiary** for 99232 (subsequent hospital care), z=62 — and they did it across 16 months, billing $6M total

### 2. Billing mill detection

In Medicaid, the billing provider is the entity that submits the claim, and the servicing provider is the one who actually performed the service. Most providers bill for their own services — `BILLING_PROVIDER_NPI_NUM = SERVICING_PROVIDER_NPI_NUM`. A billing mill funnels claims through many servicing providers under one billing entity, taking a cut.

```python
billing_mill = con.sql(f"""
    SELECT
        BILLING_PROVIDER_NPI_NUM,
        COUNT(DISTINCT SERVICING_PROVIDER_NPI_NUM) as num_servicing_npis,
        COUNT(DISTINCT HCPCS_CODE) as num_procedures,
        SUM(TOTAL_PAID) as total_paid,
        SUM(TOTAL_CLAIMS) as total_claims,
        SUM(TOTAL_UNIQUE_BENEFICIARIES) as total_benes,
        COUNT(DISTINCT CLAIM_FROM_MONTH) as months_active
    FROM '{PATH}'
    GROUP BY BILLING_PROVIDER_NPI_NUM
    ORDER BY num_servicing_npis DESC
""").df()
```

The distribution tells the story:

```text
count    617,503
mean           4.6
std           33.0
25%            1.0
50%            1.0    ← median is 1
75%            1.0    ← 75th percentile is still 1
max        5,746
```

The median and 75th percentile are both **1**. Most providers bill for themselves. Then there's NPI `1679525919` with **5,746 servicing NPIs**, billing **$863 million** across 84 months with 1,579 distinct procedure codes. That's either a massive health system or something worth investigating.

I flag the top 1% by servicing NPI count as potential billing mills.

### 3. Volume impossibilities

Each row in this dataset is already aggregated to one billing-provider/servicing-provider/procedure/month combination. So when `TOTAL_CLAIMS` exceeds 1,000 for a single row, that means one provider billed over 1,000 claims for one procedure code in one month — averaging 50+ per working day with zero days off.

```python
high_volume = con.sql(f"""
    SELECT
        BILLING_PROVIDER_NPI_NUM,
        SERVICING_PROVIDER_NPI_NUM,
        HCPCS_CODE,
        CLAIM_FROM_MONTH,
        TOTAL_CLAIMS,
        TOTAL_UNIQUE_BENEFICIARIES,
        TOTAL_PAID
    FROM '{PATH}'
    WHERE TOTAL_CLAIMS > 1000
    ORDER BY TOTAL_CLAIMS DESC
""").df()
```

**Result:** 1,783,926 rows exceeded 1,000 claims/month across **30,143 billing NPIs**, totaling **$391 billion**.

The single most extreme: NPI `1225163876` billed **1,607,071 claims** in February 2022 for code `1286Z` — but only to 1,906 beneficiaries, and was paid just $230K. That's 842 claims per beneficiary in a single month. The low payment amount suggests these may be bulk capitation or encounter records rather than fee-for-service fraud, but the ratio is still bizarre.

The real volume monster is NPI `1376609297` (the same provider from the first row of the dataset) — they consistently bill 1.1-1.2 million claims per month for T1019, each month paying out $100M+. They occupy 19 of the top 20 highest-volume rows.

### 4. Spending spike detection

Fraudulent providers often show a "ramp and run" pattern — billing spikes dramatically before the entity disappears or gets caught. I used `LAG()` to compute month-over-month spending growth per provider and flagged anything with >5x growth where the spike month exceeded $50K:

```python
spikes = con.sql(f"""
    WITH monthly AS (
        SELECT
            BILLING_PROVIDER_NPI_NUM,
            CLAIM_FROM_MONTH,
            SUM(TOTAL_PAID) as monthly_paid,
            SUM(TOTAL_CLAIMS) as monthly_claims,
            SUM(TOTAL_UNIQUE_BENEFICIARIES) as monthly_benes
        FROM '{PATH}'
        GROUP BY BILLING_PROVIDER_NPI_NUM, CLAIM_FROM_MONTH
    ),
    with_prev AS (
        SELECT *,
            LAG(monthly_paid) OVER (
                PARTITION BY BILLING_PROVIDER_NPI_NUM
                ORDER BY CLAIM_FROM_MONTH
            ) as prev_paid
        FROM monthly
    )
    SELECT
        BILLING_PROVIDER_NPI_NUM,
        CLAIM_FROM_MONTH,
        prev_paid,
        monthly_paid,
        monthly_paid / GREATEST(prev_paid, 1) as growth_ratio,
        monthly_claims,
        monthly_benes
    FROM with_prev
    WHERE monthly_paid / GREATEST(prev_paid, 1) > 5
      AND monthly_paid > 50000
      AND prev_paid IS NOT NULL
    ORDER BY growth_ratio DESC
""").df()
```

This query aggregates 227M rows to provider-month level, computes lag-based growth ratios via a window function, and filters — all pushed down into DuckDB's execution engine. It ran in **91 seconds**.

**Result:** 18,916 spike events across **13,527 providers**.

The top spikes all share a pattern: `prev_paid` of $0 (or near-zero) followed by millions in the next month. NPI `1770700221` went from $0 to **$4.76M** in July 2023. NPI `1336117670` went from $0 to **$4.74M** in February 2018. These are providers that appear out of nowhere with massive billing.

NPI `1144347824` is interesting — they appear four times in the top 20 with spikes in Dec 2020, Feb 2021, Apr 2021, and Jun 2021. Repeated $0-to-$550K+ cycles suggest an on-again-off-again billing pattern.

### 5. Procedure concentration

Most providers bill across multiple procedure codes, even within a narrow specialty. A provider deriving >95% of all Medicaid revenue from a single HCPCS code at high dollar volumes can indicate upcoding (always picking the most expensive code) or phantom billing (billing for services never rendered using a single code).

```python
concentrated = con.sql(f"""
    WITH prov_proc AS (
        SELECT
            BILLING_PROVIDER_NPI_NUM,
            HCPCS_CODE,
            SUM(TOTAL_PAID) as total_paid,
            SUM(TOTAL_CLAIMS) as total_claims,
            SUM(TOTAL_UNIQUE_BENEFICIARIES) as total_benes
        FROM '{PATH}'
        GROUP BY BILLING_PROVIDER_NPI_NUM, HCPCS_CODE
    ),
    prov_total AS (
        SELECT BILLING_PROVIDER_NPI_NUM,
               SUM(total_paid) as provider_total
        FROM prov_proc
        GROUP BY BILLING_PROVIDER_NPI_NUM
    ),
    ranked AS (
        SELECT
            p.*,
            t.provider_total,
            p.total_paid / t.provider_total as pct_of_total,
            ROW_NUMBER() OVER (
                PARTITION BY p.BILLING_PROVIDER_NPI_NUM
                ORDER BY p.total_paid DESC
            ) as rn
        FROM prov_proc p
        JOIN prov_total t USING (BILLING_PROVIDER_NPI_NUM)
        WHERE t.provider_total > 500000
    )
    SELECT * FROM ranked
    WHERE rn = 1 AND pct_of_total > 0.95
    ORDER BY provider_total DESC
""").df()
```

**Result:** **25,397 providers** with >$500K total and >95% from one code, representing **$193 billion**.

The dominant code across the top 20 is **T1019** (personal care services). NPI `1376609297` tops the list at **$5.5 billion**, 98% from T1019. The next five are also T1019 providers at $1-3B each. These are likely large home and community-based services (HCBS) agencies — the concentration on one code may be legitimate program design, but the scale demands scrutiny.

NPI `1932341898` stands out: **$997M, 99.99% from H0044** (supported employment), with only two procedure codes total. That's extreme even for this list.

## Composite scoring

Each provider gets a binary flag for each of the five signals. The composite fraud score is just the sum.

```python
import pandas as pd

all_providers = con.sql(f"""
    SELECT BILLING_PROVIDER_NPI_NUM,
           SUM(TOTAL_PAID) as total_spending
    FROM '{PATH}'
    GROUP BY BILLING_PROVIDER_NPI_NUM
""").df()

risk = all_providers.copy()

flag1_npis = set(outliers_cost['BILLING_PROVIDER_NPI_NUM'])
risk['flag_cost_outlier'] = risk['BILLING_PROVIDER_NPI_NUM'] \
    .isin(flag1_npis).astype(int)

threshold_mill = billing_mill['num_servicing_npis'].quantile(0.99)
flag2_npis = set(billing_mill[
    billing_mill['num_servicing_npis'] >= threshold_mill
]['BILLING_PROVIDER_NPI_NUM'])
risk['flag_billing_mill'] = risk['BILLING_PROVIDER_NPI_NUM'] \
    .isin(flag2_npis).astype(int)

flag3_npis = set(high_volume['BILLING_PROVIDER_NPI_NUM'])
risk['flag_high_volume'] = risk['BILLING_PROVIDER_NPI_NUM'] \
    .isin(flag3_npis).astype(int)

flag4_npis = set(spikes['BILLING_PROVIDER_NPI_NUM'])
risk['flag_spike'] = risk['BILLING_PROVIDER_NPI_NUM'] \
    .isin(flag4_npis).astype(int)

flag5_npis = set(concentrated['BILLING_PROVIDER_NPI_NUM'])
risk['flag_concentrated'] = risk['BILLING_PROVIDER_NPI_NUM'] \
    .isin(flag5_npis).astype(int)

flag_cols = [
    'flag_cost_outlier', 'flag_billing_mill',
    'flag_high_volume', 'flag_spike', 'flag_concentrated'
]
risk['fraud_score'] = risk[flag_cols].sum(axis=1)
```

This is the one step where I use pandas — the DuckDB queries already reduced 227M rows down to manageable result sets (tens of thousands of NPIs), so the set-membership lookups and joins are trivial.

**Distribution:**

| Score | Providers |
|-------|-----------|
| 0 | 539,161 |
| 1 | 57,424 |
| 2 | 17,690 |
| 3 | 3,034 |
| 4 | 194 |
| **2+** | **20,918** |

The 20,918 providers with 2+ flags account for **$535 billion** — nearly half of all Medicaid spending in the dataset.

## The top 5

For the highest-scoring providers, I pulled their full procedure breakdowns and monthly spending ranges:

```python
for npi in top5:
    proc_breakdown = con.sql(f"""
        SELECT HCPCS_CODE,
               SUM(TOTAL_PAID) as paid,
               SUM(TOTAL_CLAIMS) as claims,
               SUM(TOTAL_UNIQUE_BENEFICIARIES) as benes
        FROM '{PATH}'
        WHERE BILLING_PROVIDER_NPI_NUM = '{npi}'
        GROUP BY HCPCS_CODE ORDER BY paid DESC LIMIT 5
    """).df()
```

### NPI 1700090834 — Score 4/5 ($1.13B)
**Flags:** cost outlier, billing mill, high volume, spike

```text
HCPCS_CODE         paid        claims    benes     pct
     H0019  553,957,700   2,884,496   155,197    53.2%
     H0004  188,211,147   1,572,012   505,021    18.1%
     H0005  108,477,327   1,623,733   203,179    10.4%
     H0020  103,053,473   7,020,805   276,149     9.9%
     H0006   88,021,893   1,166,081   255,743     8.4%

Monthly spending range: $37,576 — $25,877,770
```

All H-codes (behavioral health). H0019 alone is $554M for day treatment services across 155K beneficiaries. The monthly range swinging from $38K to $25.9M means there were massive ramp-up periods. This has the profile of a large behavioral health managed care entity — but one that also trips cost outlier and billing mill flags.

### NPI 1932341898 — Score 4/5 ($997M)
**Flags:** cost outlier, high volume, spike, concentrated

```text
HCPCS_CODE         paid        claims    benes     pct
     H0044  996,688,991     304,663   283,937   100.0%
     T2028      112,131       1,468        36     0.0%

Monthly spending range: $512,298 — $21,818,712
```

**$997M from a single code.** H0044 is supported employment — helping people with disabilities find and maintain jobs. 283,937 beneficiaries at an average of $3,511 each. Only two procedure codes ever billed. The 43x spread between minimum and maximum monthly spending ($512K to $21.8M) is hard to explain with normal program growth.

### NPI 1114931391 — Score 4/5 ($382M)
**Flags:** cost outlier, high volume, spike, concentrated

```text
HCPCS_CODE         paid        claims    benes     pct
     S3620  370,523,694   1,324,711   974,114    97.1%
     83655    8,882,287     685,614   667,070     2.3%
     85018    1,601,721     623,389   607,346     0.4%
     80061      691,330      45,838    44,922     0.2%
     82947       49,396      11,154    10,951     0.0%

Monthly spending range: $47 — $15,497,422
```

97% from S3620 (newborn screening panel). The monthly range from **$47 to $15.5M** is the most extreme spread in the top 5. This likely represents a state-contracted newborn screening lab — the code and beneficiary counts support that. But a $47 month to $15.5M month means either the contract changed drastically or there are data quality issues. The secondary codes (83655, 85018) are basic lab tests, consistent with a lab operation.

### NPI 1629283197 — Score 4/5 ($306M)
**Flags:** billing mill, high volume, spike, concentrated

```text
HCPCS_CODE         paid        claims    benes     pct
     T1015  298,897,046     718,309   655,601    99.1%
     99393      722,772      52,037    39,635     0.2%
     99392      688,366      47,287    38,251     0.2%
     99394      608,918      31,656    24,001     0.2%
     92014      591,683      27,987    16,169     0.2%

Monthly spending range: $594,623 — $5,028,749
```

99% from T1015 (clinic-based behavioral health). The secondary codes (99392-99394) are preventive care E&M visits, and 92014 is an eye exam — suggesting this billing entity has a clinic component too. But the billing mill flag means claims are flowing through many servicing NPIs under this one billing number.

### NPI 1619341716 — Score 4/5 ($303M)
**Flags:** cost outlier, billing mill, high volume, spike

```text
HCPCS_CODE         paid       claims    benes     pct
     99211   45,853,871    488,964   412,175    34.6%
     90832   37,594,923    253,130   125,580    28.4%
     99213   29,286,349    255,935   204,037    22.1%
     99214    9,967,573     88,118    73,463     7.5%
     99212    9,740,312     66,604    55,192     7.4%

Monthly spending range: $66,177 — $7,160,676
```

The most diversified billing pattern of the top 5. A mix of E&M visits (99211-99214) and psychotherapy (90832). The cost outlier flag means they're charging significantly more per beneficiary than peers for these common codes. The billing mill flag means many servicing NPIs are operating under this one billing entity. The 108x spread between min and max monthly billing ($66K to $7.2M) indicates major scaling events.

## Caveats

These are flags, not findings. Several patterns have legitimate explanations:

- **Large health systems and MCOs** naturally have many servicing NPIs — that's not fraud, it's organizational structure. The billing mill signal needs to be interpreted in context.
- **State-contracted labs** (like the newborn screening provider at NPI `1114931391`) can have legitimate volume and spending spikes tied to contract awards and state program changes.
- **T1019/T1015/H-codes** are commonly used in home and community-based services (HCBS) and behavioral health, where high volumes per billing entity may reflect program design rather than fraud. States often contract with large managed care entities for these services.
- **This data is aggregated** — we can't see individual claim details, diagnosis codes, patient demographics, or service locations. Many fraud patterns only become visible at the claim level.
- **The "active months" field** showing 1 for several providers is a bug in the deep-dive query (it's pulling `COUNT(DISTINCT CLAIM_FROM_MONTH)` within a window function that only sees one row). These providers were active across many months — the composite data shows it.

The real value is in combining signals. A provider that's both a cost outlier *and* has billing mill patterns *and* shows spending spikes is far more suspicious than one that only trips a single wire. The 194 providers at score 4/5 are the ones I'd start investigating.

## Technical notes

The full analysis runs as a Jupyter notebook backed by DuckDB on a 16 GB Kubernetes pod. Some implementation details worth noting:

**DuckDB's parquet performance is remarkable.** The z-score query (Signal 1) does a full scan → two-level aggregation → window function → filter, all on 227M rows of parquet. It completes in 192 seconds. The spending spike query uses `LAG()` window functions over the aggregated monthly data — 91 seconds. These are the kinds of queries that would require Spark or a data warehouse for most teams.

**The pandas handoff is intentional.** DuckDB returns result sets via `.df()` only after reducing 227M rows to thousands. The composite scoring step — five set-membership lookups and a sum — is trivial in pandas. There's no reason to push that into SQL.

**`GREATEST(x, 1)` prevents division by zero.** Several providers have zero beneficiaries or zero prior-month spending. Rather than filtering them out (and potentially missing interesting cases), I clamp the denominator to 1. This means zero-bene providers get a per-bene cost equal to their total paid — which is correct behavior for flagging purposes.