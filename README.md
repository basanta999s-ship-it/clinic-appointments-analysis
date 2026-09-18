# Clinic Appointments — Data Cleaning & Exploratory Analysis

A pandas project that takes a deliberately messy clinic appointments dataset, cleans it into
analysis-ready form, and explores how patient age relates to department choice, billing, and
follow-up rates.

All work lives in a single notebook: [`Clinic_data.ipynb`](Clinic_data.ipynb).

---

## The dataset

`messy_clinic_appointments.csv` — **1,000 appointment records**, 10 columns.

| Column | Type | Notes |
|---|---|---|
| `patient_id` | int | **Not unique** — only 101 distinct IDs, so rows are *appointments*, not patients |
| `patient_name` | str | Includes titles and suffixes (`Dr.`, `Mrs.`, `MD`, `DDS`, `Jr.`) |
| `age` | int | 18 – 90 |
| `gender` | str | 8 different spellings for 2 values (see below); 50 missing |
| `appointment_date` | str | Two competing formats: `M/D/YYYY` and `D-MMM-YY` |
| `booking_date` | str | Same two competing formats |
| `doctor` | str | 990 distinct names across 1,000 rows |
| `department` | str | 4 values: Cardiology, Orthopedics, Neurology, General |
| `billing_amount` | str | Four currency symbols (`$`, `€`, `£`, `Rs`) + stray whitespace; 50 missing |
| `follow_up_required` | str | 6 spellings for a boolean: `Yes`/`Y`/`1`/`No`/`N`/`0` |

### What makes it messy

- **Gender** arrives as `M`, `Male`, `male`, `1`, `F`, `Female`, `female`, `0` — plus 50 blanks.
- **Dates** mix `2/26/2026` and `30-Nov-25` in the same column.
- **Billing** is stored as text: `£425.8`, `€344.26`, `Rs85.76`, `$84.44 ` (note the trailing space).
- **Follow-up** is a boolean wearing six different costumes.
- **Missing values** in `gender` (50) and `billing_amount` (50).

---

## Cleaning pipeline

The notebook works through six steps, in order:

1. **Missing values & duplicates** — audit nulls per column, drop exact duplicate rows, coerce
   `'N/A'` / `'null'` / `'None'` / `''` string sentinels to real `NaN`, then drop rows missing a
   patient ID or name.
2. **Gender standardization** — map all 8 variants onto `Male` / `Female`, with unmapped and
   missing values collapsed into `Other`.
   Result: 477 Female / 473 Male / 50 Other.
3. **Date standardization** — parse `appointment_date` and `booking_date` into real
   `datetime64` columns. *(See [Known limitations](#known-limitations) — this step currently
   drops about half the dates.)*
4. **Billing cleanup** — strip every non-numeric character with a regex and cast to `float`.
   Result: mean billing amount **£276.12** (currency symbols were discarded, so amounts are
   treated as a single unit).
5. **Follow-up normalization** — map the six spellings onto integer `0` / `1`.
6. **Whitespace & types** — `.str.strip()` across all object columns, confirm `age` is an integer.

---

## Exploratory analysis

### Age bands

Ages are binned into four groups with `pd.cut`:

```python
bins   = [18, 35, 50, 65, 91]
labels = ["18-34", "35-49", "50-64", "65+"]
```

### Does age predict which department a patient visits?

Crosstab of age group against department:

| age_group | Cardiology | General | Neurology | Orthopedics |
|---|---|---|---|---|
| 18-34 | 56 | 60 | 63 | 72 |
| 35-49 | 44 | 34 | 43 | 57 |
| 50-64 | 50 | 49 | 72 | 46 |
| 65+ | 84 | 88 | 95 | 87 |

A chi-square test of independence gives **χ² = 11.28, p = 0.2571**.

Since p > 0.05, **age group and department are not significantly related** in this dataset — the
visual differences in the grouped bar chart are consistent with random variation. The 65+ band
simply has more appointments overall, spread fairly evenly across all four departments.

### Follow-up rate by age group

| age_group | patients | follow-ups | rate |
|---|---|---|---|
| 65+ | 354 | 192 | 54.2% |
| 18-34 | 251 | 132 | 52.6% |
| 50-64 | 217 | 111 | 51.2% |
| 35-49 | 178 | 79 | 44.4% |

Follow-up rates sit in a narrow 44–54% band. The 35-49 group is the least likely to need a
follow-up; 65+ the most — but the spread is modest.

### Other checks

- **Outliers** — IQR fence (`Q1 − 1.5·IQR`, `Q3 + 1.5·IQR`) on `billing_amount`, visualized with
  a boxplot. Billing is broadly uniform across the ~£50–500 range, so the fence catches nothing
  dramatic.
- **Gender split** — 47.7% Female, 47.3% Male, 5.0% Other (the unrecoverable missing values).
- **Doctor load** — 990 distinct doctors over 1,000 appointments, so caseload analysis isn't
  meaningful here; the doctor column reads as synthetic.

---

## Known limitations

**Date parsing loses roughly half the rows.** Step 3 calls:

```python
pd.to_datetime(df[col], errors='coerce', dayfirst=False)
```

on a column holding two formats. pandas infers a single format from the first value and coerces
everything that doesn't match to `NaT`. The result:

- `appointment_date`: **494 / 1000** parsed
- `booking_date`: **514 / 1000** parsed

Any downstream analysis of wait times or seasonality would be running on half the data. A fix is
to parse each format separately and combine:

```python
a = pd.to_datetime(df[col], format='%m/%d/%Y', errors='coerce')
b = pd.to_datetime(df[col], format='%d-%b-%y', errors='coerce')
df[col] = a.fillna(b)
```

**Currency symbols are discarded, not converted.** `£425.8`, `€344.26`, and `Rs85.76` are all
reduced to bare numbers and averaged together. The reported mean of 276.12 mixes four currencies
and is therefore not a real monetary figure. Treat it as a unitless magnitude unless the symbols
are mapped to a common currency first.

**`patient_id` is not a primary key.** 101 IDs across 1,000 rows means repeat visits (or
synthetic ID reuse). `drop_duplicates()` finds nothing because full rows differ, but any
per-patient aggregation needs a `groupby('patient_id')`, not a row count.

---

## Running it

```bash
git clone https://github.com/<your-username>/clinic-appointments-analysis.git
cd clinic-appointments-analysis
pip install -r requirements.txt
jupyter notebook Clinic_data.ipynb
```

The notebook reads `messy_clinic_appointments.csv` from the working directory, so run Jupyter
from the repository root.

## Requirements

pandas, numpy, matplotlib, seaborn, scipy, jupyter — see [`requirements.txt`](requirements.txt).
