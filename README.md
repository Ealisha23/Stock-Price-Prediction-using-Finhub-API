# Stock-Price-Prediction-using-Finhub-API

A data-collection pipeline for stock price prediction. It pulls **hourly price data** (Yahoo Finance) and **company news** (Finnhub API) for dozens of large US companies, and stores the results in CSV, AWS S3 and PostgreSQL so they can be used to train a prediction model.

> **Project status:** early stage. The data-gathering part is working. The model-training part has not been built yet (see [Roadmap](#roadmap)).

---

## Table of contents

1. [The idea behind the project](#the-idea-behind-the-project)
2. [What's in this repository](#whats-in-this-repository)
3. [How it works](#how-it-works)
4. [Tech stack](#tech-stack)
5. [Getting started](#getting-started)
6. [Using the notebooks](#using-the-notebooks)
7. [Companies covered](#companies-covered)
8. [Data you get](#data-you-get)
9. [Known issues and limitations](#known-issues-and-limitations)
10. [Roadmap](#roadmap)
11. [Disclaimer](#disclaimer)

---

## The idea behind the project

Stock prices don't move in a vacuum. A product launch, an earnings surprise or a lawsuit headline can shift a price within hours. Most beginner stock predictors only look at past prices. This project takes a broader view: it collects **both** the numbers (price history) and the **words** (news headlines and summaries), so a model can eventually learn from the two together.

Right now the repo covers step one: getting clean, organised data from reliable sources and storing it somewhere convenient.

---

## What's in this repository

| File | What it does |
|---|---|
| `main_py.ipynb` | The exploration notebook. Sets up the libraries, downloads hourly Apple (AAPL) prices, calculates hourly percentage change, and fetches live quotes, market caps and the latest news for a handful of companies. |
| `news_data_csv.ipynb` | The bulk data pipeline. Pulls news for 38 companies since January 2024, combines it into one table, saves it as `news_data.csv`, uploads it to AWS S3, and loads it into a PostgreSQL database on AWS RDS. |
| `README.md` | This file. |

---

## How it works

```
 Yahoo Finance (yfinance)          Finnhub API
 hourly OHLC price data            quotes, company profiles, news
          \                              /
           \                            /
            v                          v
            Python + pandas (Google Colab)
                       |
        +--------------+---------------+
        |              |               |
        v              v               v
    CSV file      AWS S3 bucket   PostgreSQL (AWS RDS)
```

### 1. Price data (`yfinance`)
The notebooks download **hourly** candles (Open, High, Low, Close) for a stock, for example Apple from 2025-01-01 to today. A `Percent Change` column is then added using `pct_change()` on the closing price, which tells you how much the price moved from one hour to the next. This is a simple but useful feature for any prediction model.

### 2. Live snapshot (`Finnhub`)
For a list of companies, the notebook fetches:
- the **current share price** (`quote`)
- the **market capitalisation** (`company_profile2`)
- the **latest news headlines** (`company_news`, printing the three most recent)

A one-second pause between requests keeps the script inside Finnhub's free-tier rate limit.

### 3. News collection at scale (`Finnhub`)
`news_data_csv.ipynb` loops over 38 tickers, collects all news from **1 January 2024 to today**, and merges everything into a single pandas DataFrame. Each article includes its category, timestamp, headline, summary, source, related ticker and a link.

### 4. Storage
The combined news data is saved in three ways:
- **Locally** as `news_data.csv`
- **AWS S3**, uploaded with `boto3`
- **PostgreSQL on AWS RDS**, using `psycopg2`

---

## Tech stack

- **Language:** Python 3
- **Environment:** Google Colab / Jupyter Notebook
- **Data sources:** [Finnhub API](https://finnhub.io/), [yfinance](https://github.com/ranaroussi/yfinance)
- **Data handling:** pandas, NumPy
- **Visualisation (installed):** Matplotlib
- **Machine learning (installed, not yet used):** scikit-learn, TensorFlow
- **Cloud and storage:** AWS S3 (`boto3`), AWS RDS PostgreSQL (`psycopg2`)

---

## Getting started

### Prerequisites

- Python 3.9 or newer
- A free Finnhub API key: sign up at [finnhub.io](https://finnhub.io/register)
- Optional: an AWS account if you want the S3 and PostgreSQL steps

### Installation

```bash
git clone https://github.com/Ealisha23/Stock-Price-Prediction-using-Finhub-API.git
cd Stock-Price-Prediction-using-Finhub-API

pip install yfinance finnhub-python pandas numpy matplotlib scikit-learn tensorflow boto3 psycopg2-binary jupyter
```

### Set up your credentials

**Never paste keys or passwords directly into a notebook.** Keep them in environment variables instead.

```bash
export FINNHUB_API_KEY="your_finnhub_key"

# only needed for S3 / RDS
export AWS_ACCESS_KEY_ID="your_aws_access_key"
export AWS_SECRET_ACCESS_KEY="your_aws_secret_key"
export AWS_DEFAULT_REGION="us-east-1"
export DB_HOST="your-rds-endpoint"
export DB_NAME="your_database"
export DB_USER="your_user"
export DB_PASSWORD="your_password"
```

Then read them in Python like this:

```python
import os
import finnhub

finnhub_client = finnhub.Client(api_key=os.environ["FINNHUB_API_KEY"])
```

On Google Colab, use the **Secrets** panel (the key icon in the left sidebar) and read values with `google.colab.userdata.get("FINNHUB_API_KEY")`.

---

## Using the notebooks

1. Start Jupyter (`jupyter notebook`) or open the files in Google Colab.
2. Open `main_py.ipynb` first. It's the quickest way to confirm your setup works. Run the install and import cells, then the Yahoo Finance and Finnhub cells.
3. Open `news_data_csv.ipynb` to build the full news dataset. Expect it to take a few minutes, because it makes 38 API calls with a one-second delay between each.
4. Skip the S3 and PostgreSQL cells if you only need the CSV.

**Changing the stocks or dates:** edit the `symbols` list and the `start_date` / `end_date` variables near the top of the relevant cell.

---

## Companies covered

The news pipeline covers 38 large, well-known US companies across tech, finance, healthcare, energy and consumer sectors:

`MSFT` `AAPL` `GOOG` `AMZN` `TSLA` `NVDA` `WMT` `ORCL` `NFLX` `CRM` `IBM` `CSCO` `UBER` `MCD` `ADBE` `V` `LLY` `JPM` `MA` `UNH` `XOM` `META` `AVGO` `BRK-B` `COST` `BAC` `PG` `HD` `JNJ` `ABBV` `KO` `TMUS` `CVX` `WFC` `AXP` `NOW` `PLTR` `PFE`

---

## Data you get

**News data** (`news_data.csv`), one row per article:

| Column | Meaning |
|---|---|
| `category` | News category (usually "company") |
| `datetime` | Publish time as a Unix timestamp |
| `headline` | Article headline |
| `id` | Finnhub's unique article ID |
| `image` | Thumbnail link |
| `related` | Ticker the article relates to |
| `source` | Publisher, e.g. Yahoo, SeekingAlpha |
| `summary` | Short article summary |
| `url` | Link to the article |

**Price data** (from yfinance), one row per hour: `Close`, `High`, `Low`, `Open`, plus the calculated `Percent Change`.

---

## Known issues and limitations

Being upfront about these will save you time:

- **Security: credentials are exposed in the notebooks.** A Finnhub API key, AWS keys and a database password are written directly in the code and are visible in the public repo. These should be **revoked or rotated immediately** and replaced with environment variables as shown above. Removing them from the latest commit isn't enough, because they stay in the Git history.
- **The PostgreSQL step won't run as written.** The table is created with columns `title`, `published_at`, `content` and `source`, but the CSV uses `headline`, `datetime`, `summary` and `source`. The column names need to be mapped. The CSV path in that cell is also not valid.
- **No prediction model yet.** TensorFlow and scikit-learn are installed and `train_test_split` is imported, but no model is trained or evaluated.
- **News and prices aren't merged.** They are collected separately and not yet aligned by timestamp.
- **Cell clutter.** Several cells are duplicated, and a few Colab housekeeping cells (zipping `sample_data`, `!pwd`, `!ls`) aren't part of the project.
- **Ticker issue.** `TCS` in the live-snapshot list is Tata Consultancy Services' Indian ticker and isn't a standard US symbol on Finnhub's free tier, so it may return empty data.
- **Market cap units.** Finnhub returns market capitalisation in millions of dollars, but the notebook prints it with a `$` sign and no unit, which makes the numbers look far smaller than they are.
- **API limits.** The Finnhub free tier is rate-limited, and the `company_news` endpoint returns a limited number of articles per request, so very long date ranges may give incomplete coverage.
- **Hourly data limit.** Yahoo Finance only provides roughly the last two years of hourly data.

---

## Roadmap

- [ ] Move all secrets to environment variables and rotate the exposed ones
- [ ] Fix the CSV-to-PostgreSQL column mapping
- [ ] Merge price data and news by timestamp
- [ ] Run sentiment analysis on headlines and summaries (for example VADER or FinBERT)
- [ ] Add technical indicators (moving averages, RSI, volatility)
- [ ] Build and compare models, such as a baseline regression and an LSTM
- [ ] Evaluate with proper time-based splits (no shuffling) and metrics like RMSE and directional accuracy
- [ ] Add charts of predicted vs actual prices
- [ ] Convert notebook code into reusable `.py` modules and add a `requirements.txt`

---

## Disclaimer

This project is for **learning and research only**. Nothing here is financial advice. Stock prediction is hard, and past patterns don't guarantee future results. Please don't make real investment decisions based on this code.

---

## Acknowledgements

- [Finnhub](https://finnhub.io/) for the market and news API
- [yfinance](https://github.com/ranaroussi/yfinance) for historical price data
