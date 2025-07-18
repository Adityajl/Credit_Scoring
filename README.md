# DeFi Wallet Credit Scoring

This project develops a system to assign a credit score to Aave V2 protocol wallets based on their historical transaction behavior. The goal is to identify reliable and responsible users versus those exhibiting risky or exploitative patterns.

## Table of Contents

* [Project Overview](#project-overview)
* [Data Source](#data-source)
* [Methodology](#methodology)
    * [Complete Architecture](#complete-architecture)
    * [Processing Flow](#processing-flow)
    * [Feature Engineering](#feature-engineering)
    * [Credit Scoring Logic](#credit-scoring-logic)
* [Deliverables](#deliverables)
* [How to Run](#how-to-run)
* [Future Enhancements](#future-enhancements)

## Project Overview

The core challenge is to transform raw, transaction-level data from the Aave V2 protocol into meaningful features, and then use these features to compute a credit score for each unique wallet. Scores range from 0 to 1000, where higher scores indicate more reliable behavior.

## Data Source

The analysis is based on a sample of 100K raw, transaction-level data from the Aave V2 protocol. Each record represents a wallet's interaction with the protocol through actions such as `deposit`, `borrow`, `repay`, `redeemunderlying`, and `liquidationcall`.

The dataset can be downloaded from:
* Raw JSON file (~87MB): [https://drive.google.com/file/d/1ISFbAXxadMrt7Zl96rmzzZmEKZnyW7FS/view?usp=sharing](https://drive.google.com/file/d/1ISFbAXxadMrt7Zl96rmzzZmEKZnyW7FS/view?usp=sharing)
* Compressed ZIP file (~10MB): [https://drive.google.com/file/d/14ceBCLQ-BTcydDrFJauVA_PKAZ7VtDor/view?usp=sharing](https://drive.google.com/file/d/14ceBCLQ-BTcydDrFJauVA_PKAZ7VtDor/view?usp=sharing)

**Note:** Please download the `user-wallet-transactions.json` file and place it in the root directory of this project, or update the `TRANSACTIONS_FILE_PATH` variable in `credit_scoring.py`.

## Methodology

### Complete Architecture

The solution follows a sequential data processing pipeline:

1.  **Data Acquisition:** Raw JSON transaction data is loaded.
2.  **Data Preprocessing:** Raw transaction data is cleaned, relevant fields are extracted, and numerical conversions (especially for `amount` and `assetPriceUSD`) are performed. Timestamps are converted to datetime objects.
3.  **Feature Engineering:** Wallet-specific aggregated features are created from the preprocessed transaction data. This transforms granular transaction details into high-level behavioral metrics for each unique wallet.
4.  **Credit Scoring:** A rule-based system (or a machine learning model, if extended) applies a defined logic to the engineered features to calculate a credit score for each wallet.
5.  **Analysis & Output:** The final scores are saved, and a distribution plot is generated to visualize the score spread.

### Processing Flow

The `credit_scoring.py` script orchestrates the entire process:

1.  **`load_data(file_path)`:** Reads the `user-wallet-transactions.json` file into a Pandas DataFrame.
2.  **`preprocess_data(df)`:**
    * Normalizes the nested `actionData` dictionary into flat columns.
    * Converts `amount` strings to human-readable numeric values using predefined token decimals (e.g., converting 'wei' to 'ETH'). **(Crucial: Verify `TOKEN_DECIMALS` in the script for accuracy)**.
    * Converts `assetPriceUSD` to numeric.
    * Calculates `usd_value` for each transaction.
    * Converts Unix timestamps to readable datetime objects.
3.  **`engineer_features(df)`:**
    * Groups the preprocessed DataFrame by `userWallet`.
    * Aggregates various metrics for each wallet, such as:
        * Total unique transactions.
        * First and last transaction dates, leading to `account_age_days`.
        * Number of unique actions and assets.
        * Total USD value for `deposit`, `borrow`, `repay`, and `redeemunderlying` actions.
        * Count of `liquidationcall` events.
        * Maximum and average borrow rates.
    * Calculates derived ratios like `borrow_to_deposit_ratio` and `repayment_ratio`.
    * Computes `net_deposit_flow_usd`.
4.  **`calculate_credit_score_rule_based(wallet_features)`:**
    * Applies a set of predefined rules and weights to the engineered features.
    * Assigns points for positive behaviors (e.g., high deposits, full repayments, long account age) and subtracts points for negative behaviors (e.g., liquidations, high leverage with poor repayment).
    * Clamps the final score between 0 and 1000.
5.  **`analyze_scores(wallet_features, plot_output_path)`:**
    * Generates a histogram visualizing the distribution of calculated credit scores.
    * Saves the plot as `score_distribution.png`.
    * Prints summary statistics and examples of high/low scoring wallets to the console.

### Feature Engineering

The following wallet-level features are engineered to capture different aspects of user behavior:

* **`total_transactions`**: Total number of unique transactions by the wallet.
* **`first_transaction_date` / `last_transaction_date`**: Timestamps of the first and last interactions.
* **`account_age_days`**: Duration (in days) since the first transaction.
* **`num_unique_actions`**: Diversity of actions performed (deposit, borrow, repay, etc.).
* **`num_unique_assets`**: Diversity of assets transacted.
* **`total_deposit_usd`**: Cumulative USD value of all deposits.
* **`total_borrow_usd`**: Cumulative USD value of all borrows.
* **`total_repay_usd`**: Cumulative USD value of all repayments.
* **`total_redeem_usd`**: Cumulative USD value of all redemptions/withdrawals.
* **`num_liquidations`**: Count of `liquidationcall` events (a strong negative indicator).
* **`max_borrow_rate` / `avg_borrow_rate`**: Indicators of borrowing cost.
* **`borrow_to_deposit_ratio`**: Leverage indicator (total borrowed vs. total deposited).
* **`repayment_ratio`**: Efficiency of repayment (total repaid vs. total borrowed). A value $\ge 1$ indicates full repayment.
* **`net_deposit_flow_usd`**: Total deposits minus total redemptions, indicating whether the wallet is a net provider or extractor of liquidity.

### Credit Scoring Logic

The credit score is calculated using a rule-based system, initialized with a base score of 500. Points are added or subtracted based on the engineered features:

* **Positive Factors (Increase Score):**
    * **Account Age:** Longer engagement history.
    * **Total Deposits:** Higher value of assets deposited.
    * **Repayment Ratio:** High ratio indicates responsible borrowing.
    * **Net Deposit Flow:** Positive net flow (more deposits than withdrawals).
    * **Activity & Diversity:** More transactions and unique actions.
* **Negative Factors (Decrease Score):**
    * **Liquidations:** Significant penalty for any liquidation events.
    * **High Borrow-to-Deposit Ratio:** Penalized, especially if combined with a low repayment ratio.
    * **Low Repayment Ratio:** Strong penalty for failing to repay borrowed funds.

The final score is clamped between 0 and 1000 to ensure it falls within the desired range.

## Deliverables

* `credit_scoring.py`: The main Python script containing all the logic.
* `README.md`: This file, explaining the project.
* `analysis.md`: A detailed analysis of the wallet scores and observed behaviors.
* `wallet_credit_scores.csv`: Output CSV file containing wallet addresses and their calculated credit scores.
* `score_distribution.png`: A plot visualizing the distribution of credit scores.

## How to Run

1.  **Clone the repository:**
    ```bash
    git clone <your-repo-url>
    cd <your-repo-name>
    ```
2.  **Download the data:**
    Download the `user-wallet-transactions.json` file from the [Google Drive link](https://drive.google.com/file/d/1ISFbAXxadMrt7Zl96rmzzZmEKZnyW7FS/view?usp=sharing) and place it in the same directory as `credit_scoring.py`.
3.  **Install dependencies:**
    ```bash
    pip install pandas numpy matplotlib seaborn
    ```
4.  **Run the script:**
    ```bash
    python credit_scoring.py
    ```
    The script will output `wallet_credit_scores.csv` and `score_distribution.png` in the same directory. It will also print summary statistics and examples of high/low scoring wallets to the console.

## Future Enhancements

* **Refine Token Decimals:** More robust mapping of `assetSymbol` to its precise decimal places for `amount` conversion.
* **Advanced Feature Engineering:**
    * Time-decaying features (e.g., recent transactions weighted more heavily).
    * Volatility metrics for assets held.
    * Transaction frequency patterns (e.g., rapid deposits/withdrawals).
* **Machine Learning Model:**
    * Instead of a rule-based system, train a supervised regression model if labeled data (wallets with known good/bad behavior) becomes available.
    * Alternatively, use unsupervised clustering (e.g., K-Means, DBSCAN) to group wallets by behavior and assign scores based on cluster characteristics, as demonstrated conceptually in the Jupyter Notebook example.
    * Anomaly detection models (e.g., Isolation Forest) to identify truly "risky" or "bot-like" behavior.
* **External Data Integration:** Incorporate data from other sources (e.g., on-chain analytics, social sentiment) to enrich features.
* **Dynamic Rule Adjustment:** Implement a system to dynamically adjust rule weights or thresholds based on new data or expert feedback.
