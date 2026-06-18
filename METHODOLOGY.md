# 📋 Methodology — Decoding the Green Squares
### A Complete Walkthrough of How This Project Was Built

---

## Project at a Glance

```
Raw Dirty Data (105,000 rows)
        │
        ▼
┌───────────────────┐
│  Phase 1          │  Explore — understand what's broken
│  Data Exploration │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Phase 2          │  Clean — fix every issue found
│  Data Cleaning    │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Feature          │  Engineer — create 9 new columns
│  Engineering      │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Phase 3          │  Analyze — answer 20 business questions
│  Deep Analysis    │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Phase 4          │  Visualize — 18 charts, 8 chart types
│  Visualization    │
└────────┬──────────┘
         │
         ▼
Clean Analysis Dataset (101,807 rows, 30 columns)
```

---

## Dataset Overview

| Property | Value |
|---|---|
| Raw file | `github_illusion_dirty.csv` |
| Raw rows | 105,000 |
| Raw columns | 21 |
| Total missing values | 72,000+ |
| Duplicate usernames | 3,193 |
| Final clean rows | 101,807 |
| Final columns | 30 |

---

## Phase 1 — Data Exploration

**Goal:** Read the data carefully before touching anything. Every cleaning
decision in Phase 2 was based on something discovered here.

### What Was Checked

```
df.shape          → dimensions of the dataset
df.dtypes         → data type of every column
df.head()         → first 5 rows — visual sanity check
df.describe()     → min, max, mean, percentiles for numeric columns
df.columns        → full list of column names
df.info()         → column types + non-null counts in one view
df.isnull().sum() → missing value count per column
df.duplicated()   → exact duplicate rows
df.duplicated(subset=['username']) → duplicate usernames specifically
```

### Numeric Column Min/Max Check

A dedicated loop was written to check `min` and `max` for every numeric
column — this is what caught the outliers:

```python
numeric_cols = ['total_repos','total_commits','avg_commit_hour','stars',
                'forks','star_fork_ratio','followers','following',
                'readme_score','has_bio','has_profile_pic',
                'suspicion_score','profile_age_days']

for col in numeric_cols:
    print(col, 'min:', df[col].min(), 'max:', df[col].max())
```

| Column | Min Found | Max Found | Problem |
|---|---|---|---|
| `avg_commit_hour` | **-3** | **100** | Valid range is 0-23 only |
| `profile_age_days` | **-99** | **99999** | Negative age is impossible |
| `star_fork_ratio` | 0 | **inf** | Division by zero where forks = 0 |

### Column-by-Column Dirty Issues Found

```
commit_streak    → stored as "145 days" text, not a number
bot_like_score   → 12 different representations: 1, "yes", "YES",
                   True, 0, "no", "false", "maybe", etc.
language         → python / Python / PYTHON all treated as different
country          → IN / india / IND / India all treated as different
account_created  → 5 mixed date formats in same column
last_commit_date → same 5 formats + "unknown" and "never" as strings
star_fork_ratio  → 10,167 rows with inf value
suspicion_score  → 31,558 rows (30%) missing — needs to be engineered
```

### Additional Specific Checks

```python
# Check inf values in star_fork_ratio
(df['star_fork_ratio'] == np.inf).sum()     # → 10,167

# Check commit_streak format
df['commit_streak'].astype(str).str.contains('days').sum()  # → 31,107

# Check bot_like_score unique values
df['bot_like_score'].unique()

# Check language casing chaos
df['language'].nunique()                    # → 34 unique values
df['language'].value_counts(dropna=False).head(15)

# Check country inconsistencies
df['country'].value_counts(dropna=False)

# Check date column formats
df['account_created'].head(10).tolist()

# Check invalid strings in last_commit_date
df['last_commit_date'][
    df['last_commit_date'].isin(['N/A','never','unknown','',' '])
].value_counts()
```

---

## Phase 2 — Data Cleaning

**Goal:** Fix every issue found in Phase 1, one column at a time.

### Cleaning Steps in Order

```
Raw Data (105,000 rows)
    │
    ├── Step 1: Remove duplicate usernames
    │          105,000 → 101,807 rows (-3,193)
    │
    ├── Step 2: Fix account_created
    │          Parse 5 mixed date formats → datetime
    │
    ├── Step 3: Fix last_commit_date
    │          Replace "N/A"/"never"/"unknown" → NaN first
    │          Then parse mixed formats → datetime
    │
    ├── Step 4: Fix commit_streak
    │          Strip " days" text → convert to float → fill median → convert to int
    │
    ├── Step 5: Fix language
    │          fillna → lowercase → strip → replace nulls → title case → category
    │
    ├── Step 6: Fix country
    │          fillna → lowercase → map to standard names → title case → category
    │
    ├── Step 7: Fix star_fork_ratio
    │          Compute 99th percentile cap → replace inf with cap → fillna 0
    │
    ├── Step 8: Fix avg_commit_hour
    │          Values outside 0-23 → NaN → fill with median
    │
    ├── Step 9: Fix bot_like_score
    │          lowercase → map 12 variants → 0/1/NaN → Int64
    │
    ├── Step 10: Fix profile_age_days
    │           Negative or >6000 → NaN → fill with median
    │
    ├── Step 11: Fill stars, forks, has_bio, has_profile_pic
    │           All → fillna(0) → int
    │
    └── Step 12: Fill followers, following, readme_score,
                 total_repos, total_commits, commit_streak
                 All → fillna(median)
                 commit_streak → final convert to int
```

### Cleaning Decisions Explained

| Column | Why This Approach |
|---|---|
| Duplicate usernames | `keep='first'` — assume first record is original |
| `avg_commit_hour` outliers | Set to NaN, fill with median — median is resistant to outliers unlike mean |
| `profile_age_days` outliers | Negative age and 99999 days (~273 years) are physically impossible |
| `star_fork_ratio` inf | Capped at 99th percentile instead of NaN — preserves the suspicious signal (zero forks is the most suspicious case) |
| `bot_like_score` | 12 variants explicitly mapped — `errors='coerce'` alone would silently mishandle edge cases |
| `stars`, `forks` | Filled with 0 — no record of stars/forks means none, not average |
| `readme_score` | Filled with median — median is safer when distribution is skewed |
| `language`, `country` | `fillna` BEFORE `astype('str')` — prevents pandas StringDtype from keeping pd.NA instead of converting to 'nan' |

### Before vs After

| Metric | Before | After |
|---|---|---|
| Total rows | 105,000 | 101,807 |
| Duplicate usernames | 3,193 | 0 |
| Missing values (core columns) | 72,000+ | 0 |
| `avg_commit_hour` outliers | present | fixed |
| `profile_age_days` outliers | present | fixed |
| `star_fork_ratio` inf values | 10,167 | 0 |
| `commit_streak` dtype | object (text) | int |
| `bot_like_score` variants | 12 | 2 (0 or 1) |
| `language` unique values | 34 (casing chaos) | ~13 (clean) |
| `country` unique values | 35+ (codes + names) | 16 (standard names) |

---

## Feature Engineering

**Goal:** Create new columns that capture behavioral patterns not visible
in the raw data. These columns are what make the suspicion scoring possible.

### New Columns Created

| Column | Formula | What It Captures |
|---|---|---|
| `account_age_years` | `(today - account_created).days / 365.25` | How old the account is |
| `commit_frequency` | `total_commits / account_age_years` | Commits per year — abnormally high = bot signal |
| `follow_ratio` | `followers / following` | Low ratio = mass-follow bot behavior |
| `repo_commit_ratio` | `total_repos / total_commits` | Many repos with few commits = spam repos |
| `suspicious_hrs` | `1 if hour <= 4 or hour >= 23 else 0` | Committing at 12am-4am = likely automated |
| `suspicion_score` | Weighted formula (see below) | Overall fakeness probability (0-1) |
| `is_suspicious` | `1 if suspicion_score >= 0.5 else 0` | Binary flag |
| `suspicion_tier` | pd.cut into 4 bins | Clean / Borderline / Likely Bot / Confirmed Bot |
| `age_bucket` | pd.cut into 6 bins | Account age groups for analysis |
| `freq_level` | Custom function + apply | Low / Medium / High commit frequency |

### Suspicion Score Formula

The `suspicion_score` was engineered for the 31,558 rows where it was
missing. It uses 7 independent red-flag signals, each weighted by how
strongly it predicts fake behavior:

```
suspicion_score = 0

if star_fork_ratio > 50    → +0.25  (high stars, zero engagement)
if follow_ratio < 0.05     → +0.20  (mass-following, nobody follows back)
if readme_score < 3        → +0.20  (poor documentation quality)
if suspicious_hrs == 1     → +0.15  (commits at 12am-4am)
if commit_streak in
   [100,200,300,365,500]   → +0.10  (suspiciously round numbers)
if has_bio == 0            → +0.05  (no profile bio)
if has_profile_pic == 0    → +0.05  (no profile picture)

Final score = min(total, 1.0)
Profiles with score >= 0.5 → is_suspicious = 1
```

### Why These Weights?

```
Star/fork ratio (0.25) — Strongest signal. Real engagement
                          requires intent. Stars can be botted,
                          forks cannot.

Follow ratio (0.20)    — Bots mass-follow to gain followers back.
                          The ratio reveals this instantly.

README score (0.20)    — Proven strongest predictor (correlation
                          -0.74). Bots cannot fake good documentation.

Suspicious hours (0.15)— 66% of suspicious profiles commit at
                          12am-4am vs 0.004% of genuine ones.

Round streaks (0.10)   — Automation scripts hit targets and stop.
                          Humans have organic, messy numbers.

No bio / no pic (0.05) — Minor signals but consistent across
  (each)                  confirmed bot profiles.
```

---

## Phase 3 — Deep Analysis (20 Business Questions)

**Goal:** Answer specific, quantified questions using Pandas before
building any charts. This section proves the findings numerically.

### Questions and Techniques Used

| # | Question | Pandas Technique |
|---|---|---|
| Q1 | Genuine vs suspicious count | `value_counts()` |
| Q2 | % of suspicious profiles | `.mean() * 100` |
| Q3 | Average README by group | `groupby + mean` |
| Q4 | Top 10 profiles by stars | `sort_values + head` |
| Q5 | Stars > 1000 but forks < 10 | Boolean filtering with `&` |
| Q6 | % of Q5 flagged suspicious | `.mean() * 100` on filtered df |
| Q7 | % committing at suspicious hours | `groupby + mean` |
| Q8 | Average follow_ratio by group | `groupby + mean` |
| Q9 | Mass-follow profiles (ratio < 0.05) | Boolean filter + mean |
| Q10 | Top countries by profile count | `value_counts` |
| Q11 | Country with highest suspicion | `groupby + mean + sort` |
| Q12 | Language with highest avg stars | `groupby + mean + sort` |
| Q13 | Most suspicious language (500+) | `isin + groupby + mean` |
| Q14 | Account age vs suspicious crosstab | `pd.cut + pd.crosstab` |
| Q15 | Avg stats per suspicion tier | `pd.cut + groupby + mean` |
| Q16 | Top 5 most suspicious profiles | `sort_values + head` |
| Q17 | Language summary table | `groupby + agg` |
| Q18 | Country x tier pivot table | `pd.pivot_table` |
| Q19 | Suspicious % by commit frequency | Custom function + `.apply()` |
| Q20 | Profiles triggering all 4 red flags | Multiple `&` conditions |

### Key Findings from Analysis

| Finding | Number |
|---|---|
| Profiles flagged suspicious | **34,733 / 101,807 (34.12%)** |
| README score — genuine vs suspicious | **6.3/10 vs 2.3/10** |
| Commits at 12am-4am — genuine vs suspicious | **0.004% vs 66.04%** |
| Fake popularity (1000+ stars, <10 forks) | **2,668 profiles — 99.6% suspicious** |
| Mass-follow profiles (ratio < 0.05) | **28,240 — 97% suspicious** |
| Follow ratio — genuine vs suspicious | **13.43 vs 2.36** |
| High vs low frequency committers | **~6x more likely to be suspicious** |
| Does account age predict suspicion? | **No — flat at ~34% across all ages** |
| Profiles triggering all 4 red flags | **9,124 — 100% caught by score** |
| Strongest predictor (correlation) | **README score: -0.74** |

---

## Phase 4 — Visualization

**Goal:** Turn every numerical finding into a chart. 18 charts across
8 different chart types.

### Charts Built

| # | Chart Title | Type | Key Finding Shown |
|---|---|---|---|
| 1 | Genuine vs Suspicious Profiles | Bar | 34.12% suspicious headline |
| 2 | Suspicion Tier Breakdown | Pie | 4-tier trust distribution |
| 3 | Average README Score | Bar | 6.3 vs 2.3 gap |
| 4 | Stars vs Forks | Scatter | Fake popularity cluster |
| 5 | README Score Distribution | Histogram | Bimodal distribution |
| 6 | README Score by Suspicion Tier | Box | Score collapses tier by tier |
| 7 | Commit Streak Distribution | Histogram | Round number spikes |
| 8 | Average Follow Ratio | Bar | 13.43 vs 2.36 gap |
| 9 | Followers vs Following | Scatter | Mass-follow pattern |
| 10 | Suspicion Rate by Language | Bar | Rust highest at 34.8% |
| 11 | Top 10 Countries by Count | Bar | US leads, India second |
| 12 | Suspicion Rate by Account Age | Bar | Flat ~34% across all ages |
| 13 | Suspicion Rate by Commit Frequency | Bar | 6x difference Low vs High |
| 14 | Correlation Heatmap | Heatmap | README -0.74 strongest |
| 15 | README Score by Tier (Violin) | Violin | Shape of distribution |
| 16 | Follow Ratio Individual Profiles | Strip | Real data points (n=500) |
| 17 | README Score Individual Profiles | Swarm | Real data points (n=200) |
| 18 | README vs Suspicion Score | Reg | Trend line confirms -0.74 |

### Chart Types Used

```
Bar chart   ████████████████████  7 charts  (comparisons & rankings)
Scatter     ████████              2 charts  (relationship between 2 variables)
Histogram   ████████              2 charts  (distribution shape)
Box plot    ████                  1 chart   (median, spread, outliers)
Violin      ████                  1 chart   (distribution shape + density)
Strip       ████                  1 chart   (individual data points)
Swarm       ████                  1 chart   (individual points, no overlap)
Heatmap     ████                  1 chart   (correlation matrix)
Reg plot    ████                  1 chart   (scatter + trend line)
```

### Technical Decisions in Visualization

| Decision | Reason |
|---|---|
| `random_state=42` on all samples | Reproducible charts — same output every run |
| `sample(5000)` for scatter | 100K+ points make scatter unreadable |
| `sample(500)` for strip | Performance — strip plots slow with large data |
| `sample(200)` for swarm | Swarm arranges every point manually — very slow above 300 |
| `sample(1000)` for reg | Trend line needs enough points but not all 101K |
| `bar_label` on bar charts | Shows exact numbers without needing to read axes |
| `warnings.filterwarnings('ignore')` | Cleaner output — suppresses seaborn palette warnings |

---

## Tools and Libraries

| Library | Version | Used For |
|---|---|---|
| `pandas` | 2.x | Data loading, cleaning, analysis |
| `numpy` | 1.x | Numeric operations, NaN handling |
| `matplotlib` | 3.x | Base chart rendering |
| `seaborn` | 0.13.x | Statistical visualizations |
| `jupyter` | - | Interactive notebook environment |

---



## Headline Finding

After analyzing 101,807 GitHub profiles across 30 behavioral and
activity signals:

> **README quality — not stars, followers, commit streaks, or account
> age — is the single strongest predictor of whether a developer
> profile is genuine or fake. The correlation between README score
> and suspicion score is -0.74. Bots can automate everything except
> writing good documentation.**
