# Quant ETL Pipeline (Smart-Trade Showcase)

<p align="center">
  <img src="https://raw.githubusercontent.com/Mat-rixMJ/Smart-Trade/main/dashboard_preview.png" alt="Algodhan Platform Dashboard Mockup" width="800" style="border-radius: 12px; border: 1px solid rgba(255, 255, 255, 0.1); box-shadow: 0 8px 32px rgba(0,0,0,0.5);"/>
</p>

<p align="center">
  <a href="http://15.134.152.37/viewer"><img src="https://img.shields.io/badge/Live_Demo-smtrade.space-6366F1?style=for-the-badge&logo=streamlit" alt="Live Demo"/></a>
  <a href="https://www.python.org"><img src="https://img.shields.io/badge/Python-3.10%2B-blue?style=for-the-badge&logo=python" alt="Python Version"/></a>
  <a href="https://opensource.org/licenses/MIT"><img src="https://img.shields.io/badge/License-MIT-green?style=for-the-badge" alt="License"/></a>
  <a href="https://github.com/Mat-rixMJ/Smart-Trade/stargazers"><img src="https://img.shields.io/github/stars/Mat-rixMJ/Smart-Trade?style=for-the-badge" alt="GitHub Stars"/></a>
  <a href="https://github.com/Mat-rixMJ/Smart-Trade/commits/main"><img src="https://img.shields.io/github/last-commit/Mat-rixMJ/Smart-Trade?style=for-the-badge" alt="Last Commit"/></a>
</p>

---

> [!IMPORTANT]  
> **Repository Notice (Showcase Version)**  
> This is a sanitized, public-facing structural showcase repository. To protect proprietary trading edges, live API keys, and production databases, the core predictive logic and live broker execution components have been abstracted into design templates. The complete, fully operational trading system resides in a private repository.

### Tagline
Production-grade algorithmic trading platform architecture and modular ETL analytics pipeline executing high-probability breakout strategies on NSE equities.

### Description
Algodhan is an automated quantitative trading and analytics platform architecture built in Python, integrating the Fyers API (WebSocket & Order management), SQLite in Write-Ahead Logging (WAL) mode, and Streamlit dashboard visualizers. The platform architecture showcases ingestion of real-time market data, dynamic market regime classification (trending vs. choppy), and compute-intensive technical indicator pipelines. The project exhibits robust systems engineering, clean ETL separation, multi-process daemon configuration, and automated crash-recovery triggers.

### Deployed Live Viewer
👉 **[http://smtrade.space](http://15.134.152.37/viewer/)** *(Interactive Presentation-Grade Performance Dashboard)*

---

## 📑 Table of Contents
1. [Key Features](#-key-features)
2. [Showcase Structure Explored](#-showcase-structure-explored)
3. [System Architecture](#-system-architecture)
4. [Tech Stack](#-tech-stack)
5. [Strategy Overview](#-strategy-overview)
6. [Backtesting Results](#-backtesting-results)
7. [Running the Sandbox locally](#-running-the-sandbox-locally)
8. [Environment Variables](#-environment-variables)
5. [Strategy and Backtesting Interface](#-strategy-and-backtesting-interface)
6. [Running the Sandbox locally](#-running-the-sandbox-locally)
7. [Environment Variables](#-environment-variables)
8. [Project Structure](#-project-structure)
9. [Roadmap](#-roadmap)
10. [Disclaimer](#-disclaimer)
11. [Author](#-author)
12. [License](#-license)

---

## ⚡ Key Features

- 📡 **High-Throughput Ingestion Engine**: Asynchronous, multi-instrument real-time market data stream handler utilizing WebSockets with automated reconnection logic (see `core/ws_manager.py`).
- 📊 **Decoupled Quantitative ETL Pipeline**: Modular data processing layer that computes technical indicators (EMA, RSI, ADX, ATR, VWAP) using vector operations in Pandas and NumPy.
- ⚙️ **Dynamic Market Regime Classification**: State-machine architecture designed to categorize volatility regimes (e.g., high vs. low volatility, trending vs. mean-reverting) to adjust system operational states.
- 🛡️ **Modular Strategy Interface**: Clean, template-driven layout (`strategy/strategy_template.py`) demonstrating a decoupled execution model for technical signals.
- 💼 **Architectural Risk Control Interface**: Abstract models illustrating capital-at-risk scaling, volatility-based sizing, and customizable stop-loss/take-profit rules.
- 🗄️ **High-Concurrency Persistence**: Optimized SQLite database connection settings utilizing Write-Ahead Logging (WAL) and `PRAGMA synchronous=NORMAL` to handle simultaneous read/write actions from parallel threads.
- 🖥️ **Analytical Dashboard**: Presentation-grade interactive Streamlit UI utilizing Plotly charts, heatmaps, and aggregate risk metric tracking.

---

## 🔍 Showcase Structure Explored

This showcase repository is designed to demonstrate professional systems engineering and clean architectural patterns:
- **Asynchronous I/O**: Inspect [core/ws_manager.py](core/ws_manager.py) to see how streaming WebSocket ticks are processed concurrently.
- **Data Pipeline**: Review [core/regime_detector.py](core/regime_detector.py) to see clean segregation of mathematical transformations.
- **Risk Layer Architecture**: Check out [core/risk_manager.py](core/risk_manager.py) for database logging, connection pooling, and safety validation boundaries.
- **Daemonization & Deployment**: See [docs/](docs/) for systemd services and Nginx configuration templates.

---

## 🏗️ System Architecture

```text
                  +-------------------------------------------------+
                  |                INGESTION LAYER                  |
                  |  [WebSockets Manager]      [Data Provider API]  |
                  |   (Real-time LTP ticks)   (Historical Bars)     |
                  +-----------+-----------------------+-------------+
                              |                       |
                              v                       v
                  +-------------------------------------------------+
                  |                 ANALYTICAL CORE                 |
                  |                [Execution Engine]               |
                  |                       |                         |
                  |     +-----------------+-----------------+       |
                  |     |                                   |       |
                  |     v                                   v       |
                  |  [Regime Detector]             [Signal Generator]|
                  |  (Gap & Volatility analysis)   (Technical Conflu- |
                  |  (Market State Classifier)      ence Pipelines)  |
                  |     |                                   |       |
                  |     +-----------------+-----------------+       |
                  |                       |                         |
                  |                       v                         |
                  |                [Risk Manager]                   |
                  |           (Capital Protection Boundary)         |
                  +-----------------------+-------------------------+
                                          |
                                          v
                  +-------------------------------------------------+
                  |                PERSISTENCE LAYER                |
                  |   [SQLite Database]       [State File Cache]    |
                  |     (WAL Mode Log)          (system_state.json) |
                  +-----------+-----------------------+-------------+
                              |                       |
                              +-----------+-----------+
                                          |
                                          v
                  +-------------------------------------------------+
                  |               PRESENTATION LAYER                |
                  |             [Public Viewer UI]                  |
                  |                 (Port 8502)                     |
                  |                       ^                         |
                  |                       |                         |
                  |             [Nginx Reverse Proxy]               |
                  |                  (Port 80/443)                  |
                  +-------------------------------------------------+
```

---

## 🛠️ Tech Stack

| Layer | Technology | Purpose |
| :--- | :--- | :--- |
| **Language** | Python 3.10+ | Core application logic and execution engine |
| **Data Ingestion** | WebSocket API Client / yfinance | Real-time LTP tick stream ingestion and historical data fetching |
| **Data Processing** | Pandas, NumPy, pandas-ta | Vectorized technical indicators, resampling, and regime analysis |
| **Database** | SQLite (WAL Mode) | High-concurrency trade ledger logging and state persistence |
| **Dashboard** | Streamlit, Plotly | Live admin control panel and read-only interactive analytics dashboard |
| **Deployment** | Cloud VM (Ubuntu 22.04 LTS) | 24/7 cloud hosting with low-latency network connectivity |
| **Process Control**| Systemd | Deploys execution loops and dashboards as background service daemons |
| **Security/Web** | Nginx | Reverse proxy for Streamlit dashboards, SSL, and network isolation |
| **Notifications** | Telegram Bot API, SMTP | Live crash warnings, system heartbeats, and daily reports |

---

## 📈 Strategy and Backtesting Interface

The platform provides a modular framework for plugging in quantitative trading strategies and running localized simulation checks:
- **Signal Confluence**: The system decouples trend filtering, momentum confirmation, and support/resistance levels.
- **Risk Overlay**: Each entry signal is validated against active risk parameters before execution.
- **Performance Evaluation**: Includes a standard backtesting module that processes historical data to compute benchmark performance metrics such as Win Rate, Profit Factor, and Maximum Drawdown.

*(Note: Real-time live execution endpoints and proprietary algorithm models have been removed from this public layout to protect IP.)*

---

## 💻 Running the Sandbox Locally

You can run the public Streamlit dashboard in a sandbox state populated with mock trades:

1. **Clone the Repository**:
   ```bash
   git clone https://github.com/Mat-rixMJ/Smart-Trade.git
   cd Smart-Trade
   ```

2. **Install Dependencies**:
   ```bash
   pip install -r requirements.txt
   ```

3. **Configure Settings**:
   ```bash
   cp .env.example .env
   ```

4. **Populate Sandbox Data**:
   Generate mock trade records to populate the SQLite database:
   ```bash
   python dashboard/populate_data.py
   ```

5. **Launch the Dashboard**:
   ```bash
   streamlit run dashboard/public_dashboard.py
   ```
   Open `http://localhost:8501/` to explore the presentation UI.

---

## ⚙️ Environment Variables

| Variable | Description | Required | Default |
| :--- | :--- | :--- | :--- |
| `DHAN_CLIENT_ID` | Client ID for Dhan API integration | No | - |
| `DHAN_ACCESS_TOKEN` | Personal Access Token for Dhan API | No | - |
| `FYERS_CLIENT_ID` | App ID / Client ID for Fyers API | Yes (if Fyers) | - |
| `FYERS_SECRET_KEY` | App Secret Key for Fyers API | Yes (if Fyers) | - |
| `FYERS_REDIRECT_URI` | Redirect URL registered in Fyers Dashboard | Yes | `http://127.0.0.1:5000/` |
| `TELEGRAM_BOT_TOKEN` | Token for Telegram notification bot | Yes | - |
| `TELEGRAM_CHAT_ID` | Telegram chat ID for logs and alerts receiver | Yes | - |
| `EMAIL_SENDER` | SMTP sender gmail address | Yes | - |
| `EMAIL_RECEIVER` | Destination email for daily trade reports | Yes | - |
| `EMAIL_APP_PASSWORD` | App-specific Google password for SMTP auth | Yes | - |

---

## 📂 Project Structure

```text
Smart-Trade-/
├── .env.example              # Template for API credentials and settings
├── .gitignore                # Git ignore configuration
├── requirements.txt          # Python dependencies list
├── config.py                 # System configuration and settings
├── main.py                   # Simplified execution engine bootstrap
├── core/
│   ├── ws_manager.py         # Asynchronous WebSocket listener for tick streaming
│   ├── regime_detector.py    # Gap & Volatility morning market regime classifier
│   ├── risk_manager.py       # Positions sizing, database logging, and risk controls
│   └── notifier.py           # Alert manager for Telegram updates and daily report emails
├── strategy/
│   ├── strategy_template.py  # Abstracted layout strategy model for user implementation
│   └── exchange.py           # Multi-exchange utility helper
├── dashboard/
│   ├── public_dashboard.py   # Streamlit public visual dashboard viewer
│   └── populate_data.py      # Seed database generator for sandbox simulation
└── docs/
    ├── Nginx_setup.conf      # Sample reverse-proxy gateway routing configuration
    └── systemd_setup.service # Sample systemd service process control configuration
```

---

## 🚀 Roadmap

- [ ] **Sanitized Backtester Sandbox**: Standardize a backtest script utilizing historical csv datasets for local sandbox tests.
- [ ] **Advanced Indicators Extension**: Incorporate standard Bollinger Band and MACD histogram metrics inside the strategy template.
- [ ] **React.js Dashboard Version**: Re-architecture the frontend visualizer into a clean modern dashboard template using Next.js.

---

## ⚠️ Disclaimer

This software is for educational purposes only. Trading financial instruments involves high risk, and you may lose more than your initial capital. The author is not a SEBI-registered financial advisor, and this platform does not constitute investment advice. Perform your own due diligence before deploying real capital.

---

## 👤 Author

**Manoj Kumar**
- **GitHub**: [Mat-rixMJ](https://github.com/Mat-rixMJ)
- **LinkedIn**: [Manoj Kumar](https://www.linkedin.com/in/manoj-kumar-algotrader)

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
