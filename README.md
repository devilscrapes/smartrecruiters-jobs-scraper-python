<p align="center"><a href="https://apify.com/DevilScrapes/smartrecruiters-jobs-scraper"><img src="assets/banner.svg" alt="SmartRecruiters Jobs Scraper — Python example" width="100%"></a></p>

# SmartRecruiters Jobs Scraper in Python: enterprise job postings to pandas, JSON and CSV

<p align="center">
<a href="https://apify.com/DevilScrapes/smartrecruiters-jobs-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/smartrecruiters-jobs-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/smartrecruiters-jobs-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

This repository holds a short, tested Python script that drives the **[SmartRecruiters Jobs Scraper](https://apify.com/DevilScrapes/smartrecruiters-jobs-scraper)** on Apify, published by Devil Scrapes. SmartRecruiters is an applicant tracking system whose customers include international employers such as Visa, and each customer's careers board sits at `jobs.smartrecruiters.com/<CompanyId>`. Put one or more of those identifiers into `input.json` and the script collects every currently open posting as a row: title, city, region, country, remote flag, department, function, employment type, industry, the employer's internal reference number and the release timestamp. Apify runs the collection in the cloud. What reaches your disk is `results.json` and `results.csv`, ready for a notebook, a cron-driven report or a warehouse load.

> **Actor:** [SmartRecruiters Jobs Scraper](https://apify.com/DevilScrapes/smartrecruiters-jobs-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [smartrecruiters-jobs-scraper-javascript](https://github.com/devilscrapes/smartrecruiters-jobs-scraper-javascript)

## 🔥 What you get

- **Identifier or link, either works**: a careers address such as `jobs.smartrecruiters.com/Visa` is reduced to its first path segment; anything without the SmartRecruiters host is sent as written, capitals included.
- **Paging follows the employer's own total**: postings are requested 100 at a time until the count SmartRecruiters reports for that employer is reached or your per-company cap is hit, whichever comes first.
- **Filters applied at the source**: keyword, department, city and country travel with every page request, so postings outside your filter are never downloaded or billed. Country codes are lowercased for you.
- **Descriptions on demand**: with `includeDescription` on, each posting's company blurb, job text, qualifications and additional information are joined into one HTML string under `<h2>` headings, and the apply link is replaced with the one SmartRecruiters returns for that posting.
- **One employer failing leaves the others running**: when an employer's requests keep failing after retries, it is marked failed and the run moves on. The final status message counts how many employers returned data.
- **Empty boards are a normal answer**: an employer with no open postings contributes zero rows without raising an error.

## 🎯 Use cases

### Map a multinational's hiring by country

An employer that recruits internationally lists every country's openings on the same board. Group the rows by `location_country` and `function` and you can see which markets a company is staffing up and in which disciplines, a view analysts usually assemble by hand from regional career sites.

### Measure hiring tempo

`released_date` is a full timestamp. Count postings released in the trailing 7, 30 and 90 days for each `company_id`, store the counts after every scheduled run, and the series shows when an employer accelerates or freezes recruitment.

### Benchmark remote openness across an industry

Feed in the identifiers of ten competitors in one sector and compute the share of rows where `is_remote` is true per employer. HR teams use the comparison when they decide how flexible their own job ads should be.

### Build a reference-number index for recruiters

`ref_number` is the requisition code the employer itself uses. Agency recruiters can keep a sheet keyed by it, so when a hiring manager mentions a requisition on a call, the matching posting, location and link are one lookup away.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/smartrecruiters-jobs-scraper-python.git
cd smartrecruiters-jobs-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [SmartRecruiters Jobs Scraper](https://apify.com/DevilScrapes/smartrecruiters-jobs-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

The remainder of `main.py` handles arguments, files and error reporting. The integration itself is five lines with `apify-client`: start the Actor, let the client wait for completion, then read the dataset item by item.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/smartrecruiters-jobs-scraper").call(run_input={'companyIds': ['Visa'], 'maxResultsPerCompany': 10, 'includeDescription': False, 'proxyConfiguration': {'useApifyProxy': True}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`input.json` is passed through untouched. The shipped sample asks for Visa's board, stops after ten postings and leaves descriptions off, which costs one start event plus at most ten row events. Switch `includeDescription` on when you need the posting text, but expect the run to take longer because every posting then needs a second request.

```json
{
  "companyIds": [
    "Visa"
  ],
  "maxResultsPerCompany": 10,
  "includeDescription": false,
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `companyIds` | array | yes | `["Visa"]` | — |
| `searchQuery` | string | no | — | — |
| `department` | string | no | — | — |
| `city` | string | no | — | — |
| `country` | string | no | — | — |
| `maxResultsPerCompany` | integer | no | `100` | — |
| `includeDescription` | boolean | no | `false` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true}` | — |

## 📤 Output

One row equals one posting. `posting_id` is the identifier SmartRecruiters assigns, so it is the right key for deduplication across runs. `location_country` arrives as a lowercase two-letter code, for example `us`. `company_id` is the identifier SmartRecruiters echoes back, which can differ in capitalisation from what you typed. `description_html` stays `null` unless descriptions were requested, and `apply_url` is built from the employer and posting identifiers until a description request supplies the official link.

| Field | Type | Always present |
|---|---|---|
| `posting_id` | string | yes |
| `title` | string | yes |
| `company_id` | string | yes |
| `company_name` | string | yes |
| `location_city` | string or null | no |
| `location_region` | string or null | no |
| `location_country` | string or null | no |
| `is_remote` | boolean | yes |
| `department` | string or null | no |
| `function` | string or null | no |
| `employment_type` | string or null | no |
| `industry` | string or null | no |
| `ref_number` | string or null | no |
| `released_date` | string | yes |
| `apply_url` | string | yes |
| `description_html` | string or null | no |
| `scraped_at` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.002 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/smartrecruiters-jobs-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

This pandas snippet (`pip install pandas`) turns the rows into a hiring profile: postings per country and function, the remote share per employer, and how many roles each employer released in the last 30 days. Each table is printed and the country matrix is saved as `hiring_by_country.csv`.

```python
import json

import pandas as pd

jobs = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
jobs["released"] = pd.to_datetime(jobs["released_date"], utc=True, format="ISO8601")
jobs["country"] = jobs["location_country"].str.upper().fillna("??")

matrix = pd.crosstab(jobs["country"], jobs["function"].fillna("(none)"), margins=True)
matrix.to_csv("hiring_by_country.csv")
print(matrix.to_string())

cutoff = pd.Timestamp.now(tz="UTC") - pd.Timedelta(days=30)
per_employer = jobs.groupby("company_name").agg(
    open_roles=("posting_id", "nunique"),
    remote_share=("is_remote", "mean"),
    released_last_30d=("released", lambda s: int((s >= cutoff).sum())),
    countries=("country", "nunique"),
)
print(per_employer.sort_values("open_roles", ascending=False).to_string())
```

## ❓ FAQ

### How do I find the right company ID?

Open the employer's careers board and look at the address bar: the segment after `jobs.smartrecruiters.com/` is the ID. It is not always the brand name you would guess, so copy it rather than typing it. Pasting the full board link into `companyIds` works too.

### Why does the department filter return nothing?

SmartRecruiters filters departments by an internal numeric ID, such as `868639`, not by the label you see on the board. Run once without the filter, read the departments that appear, and use keyword, city or country filters if you do not have the numeric ID.

### The run succeeded but some employers are missing. Why?

The Actor finishes successfully even if one or every employer failed, so it never discards rows that were already collected. Read the run's status message: it says how many employers returned data. In a scheduled notebook, compare the `company_id` values in your rows with the list you sent.

### Can I run this from a scheduler or a notebook?

Yes. Call `python main.py` from cron, Airflow or a CI schedule, or import `main` and call `main.main(["input.json", "--out", "data"])` inside a notebook cell. Keep `maxResultsPerCompany` small while you are still shaping the analysis.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/smartrecruiters-jobs-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [SmartRecruiters Jobs Scraper on Apify Store](https://apify.com/DevilScrapes/smartrecruiters-jobs-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/smartrecruiters-jobs-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [Workday Jobs Scraper](https://apify.com/DevilScrapes/workday-jobs-scraper) — openings from large employers whose careers sites run on Workday.
- [Workable Jobs Scraper](https://apify.com/DevilScrapes/workable-jobs-scraper) — boards hosted on Workable, common among smaller tech firms.
- [Lever Jobs Scraper](https://apify.com/DevilScrapes/lever-jobs-scraper) — jobs.lever.co boards, split by team and department.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/smartrecruiters-jobs-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with SmartRecruiters.
