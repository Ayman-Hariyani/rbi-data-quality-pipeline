# RBI Data Quality Pipeline

**A set of automated checks that catch hidden problems in public economic data, such as blank stretches, sudden jumps and exchange-rate effects, before anyone builds on it.**

Built in Python (pandas) on 25 years of weekly data from the Reserve Bank of India.

---

## Why this project

Public datasets arrive in regular batches, and they can quietly go wrong. If a pipeline trusts every batch blindly, every analysis built on it inherits the errors. This project builds the checks that sit at the entry point and flag problems before the data is used.

The idea comes from my internship at BARC India, where I validated the output of a migrated data pipeline against the original.

## The data

Weekly **Foreign Exchange Reserves** (Weekly Statistical Supplement, Table 2) from the RBI's Database on Indian Economy ([data.rbi.org.in](https://data.rbi.org.in)).

- 1,334 weekly observations, from April 2001 to October 2026
- Total reserves, foreign currency assets, gold, SDRs and the IMF reserve position, each in rupees (crores) and US dollars (millions)
- The raw export is **not** included in this repository. See "How to reproduce" below.

## What it does

| Stage | Notebook | What it does |
|---|---|---|
| 1. Cleaning | `01_cleaning.ipynb` | Turns RBI's raw report export into a clean table and documents the problems found along the way |
| 2. Missing values | `02_quality_checks.ipynb` | Reports, for every column, the missing count and the longest run of missing weeks with its dates. Flags runs longer than 4 weeks |
| 3. Sudden jumps | `03_jump_checks.ipynb` | Scores each week's percent change with the median and MAD (a robust measure of spread), and flags weeks scoring above 3.5 |
| 4. Rupee-dollar consistency | `04_rate_check.ipynb` | Computes the implied exchange rate and runs the jump check on it. Finds no sign of a unit mix-up; a cutoff of 3.5 is too sensitive for this calm series |

## What I found

**Data problems (stage 1 and 2)**

- **52 weeks in a row with no IMF values**, from 2001-04-06 to 2002-03-29, exactly one fiscal year. The check reports it automatically. It looks like a reporting gap.
- **16 weeks with a blank dollar SDR value** although the rupee value exists, in four separate runs (longest: 6 weeks, mid-2006). The rupee amounts are tiny in those weeks, so I think the dollar figure rounds away. That is a hypothesis, not a confirmed explanation.
- My own cleaning step let fake rows through at first: pandas read fiscal-year labels like "2001-02" as February 2001. I caught it because the oldest date was not a week-ending Friday.

**Sudden jumps (stage 3)**

- On dollar reserves, **19 of 1,333 weekly changes are flagged** (1.4%). The two largest are the week of 2026-09-04 (+6.1%) and the week of 2008-10-24 (-5.6%), each scoring about 9.
- **The rupee column gives misleading results.** It flags 16 weeks, but only 7 overlap with the dollar flags. In the week of 2010-05-21, rupee reserves rose 3.85% while dollar reserves moved 0.02%: the implied exchange rate went from 45.06 to 46.78 rupees per dollar. The rupee column also hides some real drops in autumn 2008, because the falling rupee cushioned them. So the jump check runs on the dollar columns.
- A stricter cutoff of 5 flags 6 weeks instead of 19, so the sensitivity can be tuned.

## Design choices

- **Median and MAD instead of mean and standard deviation.** An extreme value inflates the standard deviation and can hide itself. In a small test, adding one outlier raised the standard deviation from 0.67 to 28.5 while the MAD moved from 0.0 to 0.5.
- **Flags, not verdicts.** The checks say "look at this", not "this is wrong". The data has no ground truth, so I cannot separate real events from errors yet.
- **Settings are inputs.** The 4-week gap limit and the 3.5 cutoff are parameters I plan to test, not fixed truths.
- **Safe on edge cases.** The functions were tested on small series with known answers (a gap at the end, an all-missing column, a column with no gaps, two gaps of different lengths), and the jump check returns nothing when the MAD is zero.

## Limitations

- The missing-values check measures only gap length, so it treats the SDR rounding blanks and the IMF gap the same way.
- There is no ground truth, so I cannot yet say how many flagged weeks are real errors.
- Data is downloaded manually from the RBI portal. There is no live feed yet.

## Roadmap (planned, not built yet)

- [x] Exchange-rate consistency check: flag weeks where the rupee and dollar columns disagree
- [ ] Benchmark: inject known faults into clean data and measure how well each check catches them, and how often it raises false alarms
- [ ] A second data source that updates through a live API
- [ ] Heavy checks in PySpark, deployment and scheduling on Azure

## How to reproduce

1. Install the tools: `pip install pandas openpyxl jupyter`
2. Stages 2 and 3 run directly, since `table2_clean.csv` is included.
3. To run stage 1 from the raw data, export it from the RBI portal: Publications > Time-Series Publications > Weekly Statistical Supplement > Foreign Exchange Reserves - Weekly, then export to Excel and save it as `weekly_forex_reserves.xlsx` next to the notebook. (Menus may change over time.)

## Repository contents

```
01_cleaning.ipynb        cleaning the raw export
02_quality_checks.ipynb  missing-values check
03_jump_checks.ipynb     sudden-jump check
table2_clean.csv         cleaned data (output of stage 1)
```

## About me

Final-year B.Sc. Applied Statistics and Data Analytics student at Mithibai College, Mumbai.
GitHub: [Ayman-Hariyani](https://github.com/Ayman-Hariyani) | LinkedIn: [My LinkedIn](https://www.linkedin.com/in/ayaman-hariyani-5b52743a4/)

*Data source: Reserve Bank of India. All data belongs to its publisher.*
