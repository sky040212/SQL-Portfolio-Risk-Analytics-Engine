# Portfolio Risk & Analytics Engine (Multi-Asset Hedge Fund Case Study)

## 1. Project Overview & Real-World Use Case
This system implements a production-grade portfolio risk and quantitative analytics engine tailored for a **Multi-Asset Hedge Fund** (referred to here as *Aletheia Global Capital*).

### The Use Case
*Aletheia Global Capital* manages a multi-asset portfolio with exposures across diverse market sectors (Technology, Financials, Healthcare, Consumer Goods, and Energy) using liquid US equities. The fund needs to:
- Ingest automated end-of-day market data (prices, market cap, sector info).
- Maintain a robust, audit-ready relational transaction store.
- Monitor portfolio-level metrics: daily returns, annualized volatility, Sharpe ratio, and Value at Risk (VaR).
- Conduct risk-factor slice-and-dice analyses across sectors and asset classes to detect concentration risks.

---

## 2. System Architecture
```
[ yfinance API ]
       │  (Daily End-Of-Day & Metadata Ingestion)
       ▼
[ Ingestion & ETL Pipeline ]
       │  (Validation, Cleaning, & Type-Casting)
       ▼
[ SQLite Relational DB ] <──> [ Transaction Ledger & Position State Engine ]
       │
       ├─► [ Quantitative Analytics Engine ]
       │         ├─ Daily Returns & Log Volatilities
       │         ├─ Portfolio Sharpe Ratio & Monte Carlo/Parametric VaR
       │         └─ Benchmark Beta & Covariance Matrices
       │
       ▼
[ Visualization & Reporting Layer ]
       ├─ interactive Plotly Dashboards (Sector Allocations, Drawdowns)
       └─ SQL-based Performance & Audit Reports
```

---

## 3. SQLite Database Schema
The relational database schema ensures full referential integrity and strict typing using SQLite:

```sql
-- 1. COMPANIES
CREATE TABLE companies (
    ticker TEXT PRIMARY KEY,
    company_name TEXT NOT NULL,
    sector TEXT NOT NULL,
    market_cap REAL
);

-- 2. PORTFOLIO
CREATE TABLE portfolio (
    portfolio_id INTEGER PRIMARY KEY AUTOINCREMENT,
    portfolio_name TEXT NOT NULL UNIQUE,
    inception_date TEXT NOT NULL,
    base_currency TEXT DEFAULT 'USD'
);

-- 3. TRANSACTIONS
CREATE TABLE transactions (
    transaction_id INTEGER PRIMARY KEY AUTOINCREMENT,
    portfolio_id INTEGER NOT NULL,
    ticker TEXT NOT NULL,
    transaction_type TEXT CHECK(transaction_type IN ('BUY', 'SELL')) NOT NULL,
    quantity REAL NOT NULL CHECK(quantity > 0),
    price REAL NOT NULL CHECK(price >= 0),
    transaction_date TEXT NOT NULL,
    FOREIGN KEY (portfolio_id) REFERENCES portfolio(portfolio_id),
    FOREIGN KEY (ticker) REFERENCES companies(ticker)
);

-- 4. HOLDINGS (Current State)
CREATE TABLE holdings (
    holding_id INTEGER PRIMARY KEY AUTOINCREMENT,
    portfolio_id INTEGER NOT NULL,
    ticker TEXT NOT NULL,
    current_quantity REAL NOT NULL CHECK(current_quantity >= 0),
    average_cost REAL NOT NULL CHECK(average_cost >= 0),
    last_updated TEXT NOT NULL,
    FOREIGN KEY (portfolio_id) REFERENCES portfolio(portfolio_id),
    FOREIGN KEY (ticker) REFERENCES companies(ticker)
);

-- 5. PRICES (Historical Market Data)
CREATE TABLE prices (
    price_id INTEGER PRIMARY KEY AUTOINCREMENT,
    ticker TEXT NOT NULL,
    price_date TEXT NOT NULL,
    close_price REAL NOT NULL CHECK(close_price >= 0),
    volume INTEGER,
    FOREIGN KEY (ticker) REFERENCES companies(ticker),
    UNIQUE(ticker, price_date)
);
```

---

## 4. Implementation & Operational Methodology

1. **Data Pipeline Mechanics**: Automated ingestion of historical price series and metadata from `yfinance`. This is parsed, cast to correct SQLite types, and loaded into tables with constraints checking.
2. **State & Transaction Ledger**: Positions in `holdings` are updated dynamically through database transactions matching actual fund ledger events.
3. **Quantitative Risk Calculations**:
   - **Returns**: $R_t = \ln(P_t / P_{t-1})$
   - **Portfolio Volatility**: $\sigma_p = \sqrt{w^T \Sigma w}$
   - **Value at Risk (VaR)**: 95% and 99% Parametric and historical Monte Carlo simulation models.
4. **Reporting**: Multi-table SQL JOINs executing aggregations to calculate real-time portfolio weights, realized/unrealized P&L, and sector exposure.
