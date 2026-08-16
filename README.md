# Netflix Data Preprocessing

End-to-end data preprocessing on the Netflix Movies and TV Shows dataset — covering inspection, missing values, duplicates/text consistency, date-time handling, mixed-format numeric cleaning, and outlier detection/handling.

## Dataset
[Netflix Movies and TV Shows (Kaggle)](https://www.kaggle.com/datasets/shivamb/netflix-shows) — 8,807 rows, 12 columns.

## What was done

### 1. Data Inspection
Checked `.shape`, `.info()`, `.describe()`, `.dtypes`, and `.isnull().sum()` first to understand the dataset before touching anything.

### 2. Missing Values
- `director`, `cast`, `country` → filled with `"Unknown"`. These columns had a large share missing (30%+ for `director`), so dropping those rows would have wasted a lot of otherwise-usable data.
- `rating`, `duration`, `date_added` → rows dropped (`dropna`). Only a handful of rows were affected here (single digits out of 8,807), so dropping had no meaningful impact on the dataset or downstream analysis.

### 3. Duplicates & Text Consistency
- Checked `.duplicated().sum()` — no duplicate rows found.
- Checked `.unique()` / `.nunique()` on categorical columns (`type`, `rating`, `country`) for inconsistent spacing/casing.
- Applied `.str.strip()` where a mismatch in `nunique()` before/after suggested hidden whitespace.

### 4. Date-Time Conversion
`date_added` contained mixed formats, so it was converted using:
```python
df['date_added'] = pd.to_datetime(df['date_added'], format='mixed', dayfirst=True)
```
`year_added` and `month_added` were then extracted for time-based analysis.

### 5. Splitting by Type
`duration` mixes two different units depending on content type — minutes for movies, seasons for TV shows — so the dataframe was split before cleaning:
```python
movies = df[df['type'] == 'Movie'].copy()
tv_shows = df[df['type'] == 'TV Show'].copy()
```

### 6. Cleaning `duration`
- Movies: stripped `" min"` and converted to float.
- TV Shows: stripped both `" Seasons"` (plural) and `" Season"` (singular) — in that order, since "Season" is a substring of "Seasons" and removing it first would leave stray characters.

### 7. Outlier Detection — IQR
Used the IQR method (not Z-score) on `movies['duration']` because the distribution is right-skewed, not normal — Z-score assumes symmetry and is distorted by skewed data, while IQR relies on quartiles and is more robust to skew.
```
Q1 = 25th percentile, Q3 = 75th percentile
IQR = Q3 - Q1
bounds = Q1 - 1.5*IQR, Q3 + 1.5*IQR
```
449 outliers were flagged.

### 8. Outlier Handling — Log Transform
Rather than removing or capping the outliers, a log transform (`np.log1p`) was applied. Reasoning: very short or very long movies (short films, epics) are legitimate values, not data-entry errors. Removing them would waste real data, and capping would flatten real differences between long films. Log transform compresses the scale without deleting or distorting any values — confirmed by the histogram shifting from a heavily right-skewed shape to a roughly symmetric one after the transform.

## Key takeaway
Each step was chosen based on what the specific column represented (identifier vs. category vs. measurement) and what the data actually looked like — not applied as a default/blind formula. Decisions like fillna vs. dropna, and IQR vs. Z-score, were made by first checking the data's shape and missing-value proportions, not templated in advance.
