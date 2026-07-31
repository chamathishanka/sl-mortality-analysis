# Build Guide — Sri Lankan Mortality & Longevity Analysis

**An analyst portfolio project, built in one day.**

Target applications: SLIC Life (Data Analyst), Superloop (Data Analyst),
PepperCube (Research Associate), Deloitte (Data Analytics).

---

## The brief in one sentence

*Sri Lankan life expectancy has risen steadily for six decades. Quantify it, find
where the improvement actually came from, and say what a life insurer should do
about it.*

That last clause is the whole project. Everything before it is table stakes.

---

## Do this on Windows

You're dual-booted, but for this one **stay on Windows the whole day**. Power BI
Desktop is Windows-only, pandas runs fine there, and the dataset is small enough that
Linux buys you nothing. Crossing the boundary mid-project will cost you an hour you
don't have.

---

## Technology, and when each one enters

| Phase | Tool | Why here and not elsewhere |
|---|---|---|
| 0. Setup | Python 3.11, VS Code, Git | 30 min |
| 1. Fetch | `requests` + World Bank API | Reliable, no auth, no scraping |
| 2. Clean | `pandas` in a notebook (VS Code) | Where 60% of real analyst time goes |
| 3. Analyse | `pandas`, `matplotlib` | Answer the four questions |
| 4. Dashboard | **Power BI Desktop** | The gap in 5 of your applications — close it here |
| 5. Write up | Word or Google Docs → PDF | The artefact that actually gets read |
| 6. Publish | GitHub | Link, don't attach |

**Deliberately not used:** XGBoost, forecasting models, Docker, pipelines. An analyst
project that shows off engineering reads as someone applying for the wrong job.

---

## Phase 0 — Setup (30 min)

```powershell
mkdir sl-mortality-analysis; cd sl-mortality-analysis
python -m venv .venv
.venv\Scripts\activate
pip install pandas requests matplotlib ipykernel openpyxl
git init
mkdir data notebooks output charts
```

Install **Power BI Desktop** from the Microsoft Store now, so it's ready when you
reach Phase 4. It's free.

### Notebooks in VS Code

You do **not** need the Jupyter web app. `.ipynb` is just a file format and VS Code
opens it natively:

1. Install the **Python** and **Jupyter** extensions (both from Microsoft)
2. `pip install ipykernel` (already in the command above)
3. Create any file ending in `.ipynb` — VS Code opens it as a notebook
4. Top-right, select `.venv` as the kernel

Use notebooks rather than plain `.py` scripts here for one specific reason: **GitHub
renders `.ipynb` with charts and outputs visible.** A reviewer sees your analysis
without cloning anything. A `.py` file shows them nothing but code.

Create `.gitignore`:

```
.venv/
__pycache__/
*.pbix.bak
```

---

## Phase 1 — Fetch the data (1 hour)

### 1.1 The indicators

World Bank Open Data. No API key, no rate limits worth worrying about.

| Indicator code | What it is |
|---|---|
| `SP.DYN.LE00.IN` | Life expectancy at birth, total |
| `SP.DYN.LE00.MA.IN` | Life expectancy at birth, male |
| `SP.DYN.LE00.FE.IN` | Life expectancy at birth, female |
| `SP.DYN.AMRT.MA` | Adult mortality rate, male (per 1,000) |
| `SP.DYN.AMRT.FE` | Adult mortality rate, female (per 1,000) |
| `SP.DYN.IMRT.IN` | Infant mortality rate (per 1,000 live births) |
| `SH.DYN.MORT` | Under-5 mortality rate |
| `SP.DYN.CDRT.IN` | Crude death rate |
| `SP.POP.65UP.TO.ZS` | Population aged 65+ (% of total) |
| `SP.POP.DPND.OL` | Old-age dependency ratio |
| `SP.POP.TOTL` | Total population |

Countries: `LKA` (Sri Lanka), plus `IND`, `BGD`, `MDV`, `NPL`, `PAK` as comparators.

> **Verify before you build on them.** Open one URL in a browser first:
> `https://api.worldbank.org/v2/country/LKA/indicator/SP.DYN.LE00.IN?format=json&per_page=100`
> If the shape isn't what you expect, adjust before writing the loop.

### 1.2 `notebooks/01_fetch.ipynb`

```python
import requests, pandas as pd, time
from pathlib import Path

INDICATORS = {
    "SP.DYN.LE00.IN":     "life_exp_total",
    "SP.DYN.LE00.MA.IN":  "life_exp_male",
    "SP.DYN.LE00.FE.IN":  "life_exp_female",
    "SP.DYN.AMRT.MA":     "adult_mortality_male",
    "SP.DYN.AMRT.FE":     "adult_mortality_female",
    "SP.DYN.IMRT.IN":     "infant_mortality",
    "SH.DYN.MORT":        "under5_mortality",
    "SP.DYN.CDRT.IN":     "crude_death_rate",
    "SP.POP.65UP.TO.ZS":  "pop_65plus_pct",
    "SP.POP.DPND.OL":     "old_age_dependency",
    "SP.POP.TOTL":        "population_total",
}
COUNTRIES = ["LKA", "IND", "BGD", "MDV", "NPL", "PAK"]

def fetch(country, code):
    url = (f"https://api.worldbank.org/v2/country/{country}"
           f"/indicator/{code}?format=json&per_page=500")
    r = requests.get(url, timeout=30)
    r.raise_for_status()
    payload = r.json()
    if len(payload) < 2 or payload[1] is None:
        return pd.DataFrame()
    return pd.DataFrame([
        {"country_code": row["countryiso3code"],
         "country": row["country"]["value"],
         "year": int(row["date"]),
         "indicator": INDICATORS[code],
         "value": row["value"]}
        for row in payload[1]
    ])

frames = []
for c in COUNTRIES:
    for code in INDICATORS:
        frames.append(fetch(c, code))
        time.sleep(0.2)          # be polite

raw = pd.concat(frames, ignore_index=True)
Path("../data").mkdir(exist_ok=True)
raw.to_csv("../data/raw_worldbank.csv", index=False)
print(raw.shape, raw["indicator"].nunique(), "indicators")
```

**Checkpoint:** you should have several thousand rows and 11 indicators.

---

## Phase 2 — Clean and shape (1.5 hours)

This phase is the one analyst interviewers actually probe. **Document every decision
in markdown cells** — the reasoning is the evidence, not the code.

### `notebooks/02_clean.ipynb`

```python
import pandas as pd

raw = pd.read_csv("../data/raw_worldbank.csv")

# 1. How bad is the missingness, and where?
missing = (raw.assign(is_null=raw["value"].isna())
              .groupby(["indicator", "country_code"])["is_null"]
              .agg(["sum", "count"]))
missing["pct_missing"] = (missing["sum"] / missing["count"] * 100).round(1)
display(missing.sort_values("pct_missing", ascending=False).head(15))
```

Decisions to make **and write down**:

- **Year range.** World Bank coverage thins before 1960 and the most recent 1–2 years
  are often blank. Pick a defensible window (e.g. 1960–2023) and say why.
- **Missing values.** Do *not* blanket-`fillna`. For a smooth demographic series,
  linear interpolation between known points is defensible; extrapolating past the
  last observation is not. State which you did.
- **Comparators.** If a peer country has patchy coverage, drop it from the comparison
  chart rather than showing a broken line.

```python
clean = (raw.dropna(subset=["value"])
            .query("1960 <= year <= 2023")
            .drop_duplicates(["country_code", "year", "indicator"]))

wide = (clean.pivot_table(index=["country_code", "country", "year"],
                          columns="indicator", values="value")
             .reset_index())

# derived fields — these are what make it analysis rather than reporting
wide["life_exp_gap"] = wide["life_exp_female"] - wide["life_exp_male"]
wide["decade"] = (wide["year"] // 10) * 10

wide.to_csv("../data/clean_panel.csv", index=False)
lka = wide[wide["country_code"] == "LKA"].sort_values("year")
lka.to_csv("../data/sri_lanka.csv", index=False)
```

**Checkpoint:** two tidy CSVs. `clean_panel.csv` feeds Power BI; `sri_lanka.csv` is
your focus dataset.

---

## Phase 3 — Answer the four questions (2 hours)

`notebooks/03_analysis.ipynb`. One section per question, each ending with a written
sentence stating the finding. If you can't write the sentence, you haven't finished
the question.

### Q1 — How much longevity has been gained, and how is it split by sex?

```python
first, last = lka.iloc[0], lka.iloc[-1]
gain = last["life_exp_total"] - first["life_exp_total"]
years = last["year"] - first["year"]
print(f"{gain:.1f} years gained over {years} years "
      f"= {gain/years*10:.2f} years per decade")
print(f"Female-male gap: {first['life_exp_gap']:.1f} → {last['life_exp_gap']:.1f} yrs")
```

### Q2 — Where did the improvement come from?

Compare percentage decline in infant, under-5 and adult mortality across the window.
The usual story is that early-life mortality collapsed while adult mortality improved
far less. **Check whether that holds here** — don't assume it.

```python
for col in ["infant_mortality", "under5_mortality",
            "adult_mortality_male", "adult_mortality_female"]:
    pct = (last[col] - first[col]) / first[col] * 100
    print(f"{col:28s} {first[col]:8.1f} → {last[col]:7.1f}  ({pct:+.1f}%)")
```

### Q3 — How fast is the population ageing?

```python
p65 = lka[["year", "pop_65plus_pct", "old_age_dependency"]].dropna()
recent = p65[p65["year"] >= 1990]
slope = (recent["pop_65plus_pct"].iloc[-1] - recent["pop_65plus_pct"].iloc[0]) \
        / (recent["year"].iloc[-1] - recent["year"].iloc[0])
print(f"65+ share rising {slope*10:.2f} percentage points per decade")
```

### Q4 — Is Sri Lanka an outlier regionally?

Plot life expectancy for all six countries on one chart. Sri Lanka has historically
led South Asia on health outcomes relative to income — verify whether that's still
true and whether the lead is narrowing.

### Charts to export to `charts/`

1. Life expectancy 1960–2023, male vs female, with the gap shaded
2. Mortality decline by category, indexed to 100 at 1960
3. 65+ population share over time
4. Regional comparison line chart

```python
import matplotlib.pyplot as plt
plt.rcParams.update({"figure.dpi": 150, "font.size": 10})
# ... one cell per chart, each ending: plt.savefig("../charts/01_life_expectancy.png",
#     bbox_inches="tight")
```

---

## Phase 4 — Power BI dashboard (2 hours)

This phase exists as much to close your Power BI gap as to serve the project.

### 4.1 Load

`Get Data → Text/CSV → data/clean_panel.csv` and `data/sri_lanka.csv`.
In **Transform Data** (Power Query), confirm column types — `year` as whole number,
metrics as decimal. Close & Apply.

### 4.2 Measures (DAX)

Create these in the model, not as calculated columns:

```dax
Latest Life Expectancy =
CALCULATE(MAX('sri_lanka'[life_exp_total]),
          FILTER('sri_lanka', 'sri_lanka'[year] = MAX('sri_lanka'[year])))

Years Gained =
VAR First = MINX('sri_lanka', 'sri_lanka'[life_exp_total])
VAR Last  = [Latest Life Expectancy]
RETURN Last - First

Gender Gap =
AVERAGE('sri_lanka'[life_exp_female]) - AVERAGE('sri_lanka'[life_exp_male])

Pop 65+ Share = AVERAGE('sri_lanka'[pop_65plus_pct])
```

### 4.3 Layout — one page, four visuals

- **Top row:** three KPI cards — Latest Life Expectancy, Years Gained, Gender Gap
- **Left:** line chart, life expectancy male vs female over time
- **Right:** line chart, mortality indicators indexed to 1960
- **Bottom left:** area chart, 65+ population share
- **Bottom right:** line chart, regional comparison with a country slicer

Add a **year range slicer** across the top. Title it plainly:
*"Sri Lanka: Six Decades of Longevity Gain, 1960–2023."*

### 4.4 Export

Save `sl_mortality.pbix` into the repo. Screenshot the dashboard to
`charts/dashboard.png` — that screenshot goes in your README and your PDF.

---

## Phase 5 — The two-page PDF (1.5 hours)

**This is the deliverable that gets read.** Budget properly; the writing takes longer
than you expect.

### Structure

```
Page 1
  Title: Longevity in Sri Lanka — What Six Decades of Mortality Data
         Mean for Life Insurers
  Author, date, one-line data source note

  1. Question         (2 sentences — why this matters to an insurer)
  2. Data & method    (4 sentences — source, window, cleaning decisions)
  3. Findings         (4 bullets, each a number and a claim)
     Charts 1 and 2

Page 2
     Charts 3 and 4
  4. What this means  (the section that matters — see below)
  5. Limitations      (3 bullets — shows judgement, not weakness)
```

### Section 4 is the whole project

Not *"life expectancy increased."* Something closer to:

> Life expectancy has risen roughly X years per decade since 1990, while the 65+
> share of the population is growing Y percentage points per decade. For a life
> insurer this cuts two ways: mortality-based products become progressively
> **more** profitable as policyholders live longer than priced, while annuity and
> retirement products face **rising** longevity risk. If the trend holds, annuity
> pricing assumptions set on historical tables will understate liabilities. The
> practical implication is that reserving should use projected rather than static
> mortality, and the product mix argues for expanding protection lines faster than
> annuity lines.

Fill in the real numbers. **Do not invent any.** If the data doesn't support a claim,
weaken the claim.

### Limitations to state

- World Bank figures are modelled estimates, not raw civil registration
- No cause-of-death breakdown, so drivers are inferred
- COVID-era years distort 2020–2022 and are noted but not adjusted

Export to PDF. Name it `Sri_Lanka_Longevity_Analysis_Chamath_Ishanka.pdf`.

---

## Phase 6 — Publish (30 min)

### README.md — conclusion first

```markdown
# Sri Lanka: Six Decades of Longevity Gain

**Finding:** Life expectancy rose X years since 1960, driven overwhelmingly by
a Y% fall in infant mortality rather than improvements in adult survival. The
65+ population share is now growing Z pp per decade — a direct longevity-risk
exposure for annuity portfolios.

📄 [Full analysis (PDF)](output/Sri_Lanka_Longevity_Analysis.pdf)
📊 [Power BI dashboard](dashboard/sl_mortality.pbix) · screenshot below

![dashboard](charts/dashboard.png)

## Data
World Bank Open Data, 11 indicators, Sri Lanka + 5 regional comparators, 1960–2023.

## Method
Fetched via World Bank API → cleaned in pandas (missingness documented in
`notebooks/02_clean.ipynb`) → analysed → visualised in Power BI.

## Repo
notebooks/ · data/ · charts/ · dashboard/ · output/
```

```powershell
git add .
git commit -m "Sri Lanka longevity analysis"
gh repo create sl-mortality-analysis --public --source=. --push
```

---

## What you send with an application

**Attach:** the tailored CV, and the two-page PDF. Nothing else.
**Link inline:** the GitHub repo.

Do not attach the `.pbix`, the notebooks, or a zip. One CV, one PDF, one link.

Email line that does the work:

> I recently analysed six decades of Sri Lankan mortality data and what it implies
> for life insurance pricing — a two-page summary is attached, with the full working
> at github.com/chamathishanka/sl-mortality-analysis.

---

## Timing

| Phase | Time | Cumulative |
|---|---|---|
| 0. Setup | 0:30 | 0:30 |
| 1. Fetch | 1:00 | 1:30 |
| 2. Clean | 1:30 | 3:00 |
| 3. Analyse | 2:00 | 5:00 |
| 4. Power BI | 2:00 | 7:00 |
| 5. Write-up | 1:30 | 8:30 |
| 6. Publish | 0:30 | 9:00 |

---

## Rules for today

1. **Ship v1 today.** SLIC closes within days. A finished modest analysis beats an
   unfinished ambitious one, every time.
2. **No modelling.** No XGBoost, no forecasting, no pipeline. Resist this.
3. **Every chart needs a sentence** stating what it shows. If you can't write it,
   cut the chart.
4. **Write down cleaning decisions as you make them.** Reconstructing your reasoning
   at 9pm is how projects lose their most interview-relevant material.
5. **The recommendation is the project.** Everything else is supporting evidence.
