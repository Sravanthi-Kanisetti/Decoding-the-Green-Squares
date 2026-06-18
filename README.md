# 🔍 Decoding the Green Squares
### Detecting Fake & Bot-Like Developer Profiles Using Python & Behavioral Data Analysis

> **1 in 3 GitHub profiles is fake** — and the single strongest way to
> detect it isn't stars, followers, or commit streaks.
> **It's README quality** (correlation: **-0.74**).

---

## 🧠 The Question This Project Asks

GitHub's "green squares" — the activity heatmap on every developer's profile
— are supposed to signal how active and productive a developer is. Stars,
followers, and commit streaks are treated as credibility signals by
recruiters and the community.

**But what if those signals are manufactured?**

This project analyzes **101,807 GitHub profiles** to find out — and the
findings are uncomfortable.

---

## ⚡ Key Findings at a Glance

| Finding | Number |
|---|---|
| Profiles flagged as suspicious | **34,699 out of 101,807 (34.08%)** |
| README score — genuine vs suspicious | **6.29 / 10 vs 2.28 / 10** |
| README ↔ suspicion correlation | **-0.74** (strongest single predictor) |
| Profiles with 1000+ stars but fewer than 10 forks | **2,668 — 99.5% flagged suspicious** |
| Suspicious profiles committing at 12am–4am | **66.09% vs 0.012% genuine** |
| Mass-follow profiles (follow ratio < 0.05) | **28,080 — 98.4% flagged suspicious** |
| Follow ratio — genuine vs suspicious | **13.53 vs 2.52** |
| High-frequency vs low-frequency commit rate | **71.3% suspicious vs 12.0%** |
| Does account age predict authenticity? | **No — flat ~34% across all ages** |
| Profiles triggering all 4 red flags | **9,124 — 100% caught by the score** |

---

## 💡 The Most Counterintuitive Finding

> Account age does NOT predict whether a profile is genuine.
> A 8-year-old account is just as likely to be suspicious (33.9%)
> as a brand-new one (34.0%).
>
> But README quality does — with a correlation of -0.74.
> **Bots can automate commits, stars, and follows.
> They cannot write good documentation.**

---

## 🛠️ What Was Built

A complete end-to-end analysis pipeline — from raw dirty data to
18 visualizations — using only Python, Pandas, Matplotlib, and Seaborn.

```
Phase 1 → Explore    105,000 rows · 21 columns · 72,000+ dirty values
Phase 2 → Clean      3,193 duplicates removed · 12 cleaning steps
Phase 2b→ Engineer   9 new behavioral features · custom suspicion score
Phase 3 → Analyze    20 business questions answered with real numbers
Phase 4 → Visualize  18 charts · 8 different chart types
```

---

## ⚙️ Feature Engineering Highlights

The most important part of this project — 9 new columns were created
to capture behavioral signals not visible in the raw data:

| Feature | What It Measures |
|---|---|
| `follow_ratio` | followers ÷ following — bots follow thousands, nobody follows back |
| `commit_frequency` | commits per year — abnormally high = automated |
| `suspicious_hrs` | commits between 12am–4am — 66% of bots, 0.012% of genuine |
| `repo_commit_ratio` | repos ÷ commits — many empty repos = spam signal |
| `suspicion_score` | weighted score (0–1) combining 7 red-flag signals |
| `is_suspicious` | binary flag — `suspicion_score >= 0.5` |
| `suspicion_tier` | Clean / Borderline / Likely Bot / Confirmed Bot |

### How the Suspicion Score Works

```
+0.25  star_fork_ratio > 50     → high stars, zero real engagement
+0.20  follow_ratio < 0.05      → mass-following bot pattern
+0.20  readme_score < 3         → poor documentation quality
+0.15  commits at 12am–4am      → inhuman commit timing
+0.10  streak = 100/200/300/365 → suspiciously round numbers
+0.05  no bio                   → incomplete profile
+0.05  no profile picture       → incomplete profile
────────────────────────────────
 1.00  max possible score
```

---

## 🧹 Data Cleaning Highlights

The raw dataset had **72,000+ dirty values** across 21 columns. Key issues
fixed before any analysis:

- **3,193 duplicate usernames** removed
- **5 different date formats** in the same column → standardized to datetime
- `commit_streak` stored as `"145 days"` text → converted to integer
- `bot_like_score` had **12 different representations** (`1`, `"yes"`,
  `"YES"`, `True`, `"no"`, `"false"`, `"maybe"`) → standardized to `0`/`1`
- `avg_commit_hour` had values like `-3` and `100` → fixed to valid 0–23 range
- `star_fork_ratio` had **10,167 `inf` values** → capped at 99th percentile
  (not dropped — zero forks is the most suspicious signal)
- `language` had 34 casing variants → 13 clean categories
- `country` had 35+ code/name variants → 16 standard names

---

## 📊 Visualizations — 18 Charts, 8 Chart Types

| Chart Type | Count | Used For |
|---|---|---|
| Bar | 7 | Comparisons, rankings, group averages |
| Scatter | 2 | Stars vs Forks, Followers vs Following |
| Histogram | 2 | README score distribution, Commit streak distribution |
| Heatmap | 1 | Feature correlation matrix |
| Box plot | 1 | README score by suspicion tier |
| Violin | 1 | README score distribution shape |
| Strip | 1 | Follow ratio — individual data points (n=500) |
| Swarm | 1 | README score — individual data points (n=200) |
| Reg plot | 1 | README vs suspicion score with trend line |

---

## 📁 Repository Structure

```
decoding-the-green-squares/
│
├── Notebooks/
│   └── Decoding_the_green_squares.ipynb   ← all 4 phases in one notebook
│
├── Data/
│   ├── github_illusion_dirty.csv          ← raw: 105,000 rows, 21 columns
│   ├── github_illusion_clean.csv          ← clean: 101,807 rows, 30 columns
│   └── github_illusion_analysis.csv       ← analysis-ready: 33 columns
│
├── Charts/
│   └── chart_01 to chart_20.png           ← all exported chart images
│
├── METHODOLOGY.md                         ← full step-by-step process
└── README.md
```

---

## 🚀 How to Run

```bash
git clone https://github.com/<your-username>/decoding-the-green-squares.git
cd decoding-the-green-squares
pip install pandas numpy matplotlib seaborn jupyter
jupyter notebook Notebooks/Decoding_the_green_squares.ipynb
```
Run all cells from top to bottom. The notebook generates all 3 CSV files
and all charts automatically.

---

## 🛠️ Tech Stack

`Python` · `Pandas` · `NumPy` · `Matplotlib` · `Seaborn` · `Jupyter Notebook`

---

## 🎯 Conclusion

After analyzing 101,807 GitHub profiles using 30 behavioral signals:

- **34.08%** are flagged as suspicious — roughly **1 in 3**
- Stars, followers, and commit streaks are all gameable and widely gamed
- **README quality is the one signal that cannot be faked** — correlation
  of -0.74 with suspicion score
- 9,124 profiles triggered all 4 major red flags simultaneously —
  **100% of them were caught by the suspicion score**

The green squares on a GitHub profile tell you very little about whether
someone is a genuine developer. The README tells you almost everything.

