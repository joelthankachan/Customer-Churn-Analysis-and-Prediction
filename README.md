<h1 align="center">Customer Churn Analysis and Prediction</h1>

<p align="center">
  <b>From raw data to decisions: SQL Server &middot; Machine Learning &middot; Power BI</b>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/SQL%20Server-111111?style=for-the-badge&logo=microsoftsqlserver&logoColor=F5C200" alt="SQL Server">
  <img src="https://img.shields.io/badge/Python-111111?style=for-the-badge&logo=python&logoColor=F5C200" alt="Python">
  <img src="https://img.shields.io/badge/scikit--learn-111111?style=for-the-badge&logo=scikitlearn&logoColor=F5C200" alt="scikit-learn">
  <img src="https://img.shields.io/badge/Power%20BI-F5C200?style=for-the-badge&logo=powerbi&logoColor=111111" alt="Power BI">
</p>

<p align="center">
  <img src="PowerBI%20Dashboard/1.%20Summary_Page.png" alt="Customer Retention Dashboard: Summary page" width="100%">
</p>

---

## Overview

Losing customers is expensive, and most of the time the warning signs are already sitting in the data.

In this project I took a telecom customer dataset and followed it all the way through: cleaned and prepared it in **SQL Server**, trained a **Random Forest** model in **Python** to predict who is likely to leave, and presented everything in a two-page **Power BI** dashboard that a non-technical person can read in under a minute.

The project answers three simple questions:

1. **How bad is the churn problem?** (How many customers leave, and how much?)
2. **Why do customers leave?** (Which groups, contracts, services and reasons stand out?)
3. **Who is likely to leave next?** (Which new customers should the business pay attention to first?)

## Contents

- [The data](#the-data)
- [How it all fits together](#how-it-all-fits-together)
- [Part 1: SQL Server](#part-1-sql-server)
- [Part 2: Machine Learning](#part-2-machine-learning)
- [Part 3: Power BI dashboard](#part-3-power-bi-dashboard)
- [What the data says](#what-the-data-says)
- [Recommendations](#recommendations)
- [Limitations and next steps](#limitations-and-next-steps)
- [Repository structure](#repository-structure)
- [How to run it](#how-to-run-it)
- [What this project taught me](#what-this-project-taught-me)
- [Acknowledgements](#acknowledgements)

---

## The data

| | |
|---|---|
| **Customers** | 6,418 |
| **Columns** | 32 (demographics, location, services used, contract and billing details, revenue, and customer status) |
| **Customer status** | `Stayed`, `Churned` or `Joined` |
| **Training set** | 6,007 customers with a known outcome (`Stayed` or `Churned`) |
| **Prediction set** | 411 new customers (`Joined`), whose outcome is not yet known |

The key idea behind the split is simple: a model can only learn from customers whose outcome we already know. The 411 new customers have not had time to leave or stay, so they are the ones we predict on.

## How it all fits together

```
 Raw data
    |
    v
 SQL Server  ->  stg_Churn (staging)  ->  prod_Churn (cleaned)
                                               |
                          +--------------------+--------------------+
                          |                                         |
                 vw_ChurnData (known outcome)              vw_JoinData (new customers)
                          |                                         |
                          v                                         v
                 Python: train Random Forest  -------->  Python: predict churn
                                                                    |
                                                                    v
                                       Power BI: Summary page + Prediction page
```

Python reads straight from the SQL views, so there are no manual Excel exports in between. Re-run the notebook and it always uses the latest data.

---

## Part 1: SQL Server

Everything starts in SQL. The full script is in [`Churn Analysis SQL.sql`](Churn%20Analysis%20SQL.sql).

**1. Explore the raw data (`stg_Churn`)**
I loaded the raw file into a staging table and profiled it before touching anything: gender split, contract types, revenue by customer status, and customers per state. This gave me a feel for the data and a baseline to compare against later.

**2. Check for missing values**
I wrote a query that counts the NULLs in all 32 columns, so I could see exactly where the gaps were instead of guessing.

**3. Clean it into a production table (`prod_Churn`)**
Missing values were replaced with meaningful defaults using `ISNULL`:

| Column(s) | NULL replaced with | Why |
|---|---|---|
| `Value_Deal`, `Internet_Type` | `None` | No deal or no internet service |
| `Multiple_Lines`, `Online_Security`, `Online_Backup`, `Device_Protection_Plan`, `Premium_Support`, `Streaming_TV`, `Streaming_Movies`, `Streaming_Music`, `Unlimited_Data` | `No` | The customer does not have the service |
| `Churn_Category`, `Churn_Reason` | `Others` | Customers who stayed have no churn reason |

I then re-ran the NULL check on `prod_Churn` to confirm the clean-up worked.

**4. Create two views, one for each job**

```sql
-- Customers whose outcome we know: used to train the model
CREATE VIEW vw_ChurnData AS
    SELECT * FROM prod_Churn WHERE Customer_Status IN ('Churned', 'Stayed');

-- New customers: used to predict who might leave
CREATE VIEW vw_JoinData AS
    SELECT * FROM prod_Churn WHERE Customer_Status = 'Joined';
```

A view is a saved query, so it always reflects the latest data and keeps the filtering logic in one place. Both Power BI and Python use these views.

---

## Part 2: Machine Learning

The notebook is here: [`Churn_Prediction_ML.ipynb`](Customer%20Churn%20Prediction%20ML%20CODE/Churn_Prediction_ML.ipynb)

### The approach

1. **Load the data straight from SQL Server** with SQLAlchemy and pyodbc (`vw_ChurnData` for training, `vw_JoinData` for predictions).
2. **Remove columns that would leak the answer.** I dropped `Churn_Category` and `Churn_Reason`, because they are only filled in *after* a customer has already left. Keeping them would make the model look brilliant in testing and useless in real life. I also dropped `Customer_ID`, which is just a label.
3. **Encode the text columns.** 19 categorical columns (such as contract type, payment method and state) were converted into numbers with `LabelEncoder`. The target was set to `Stayed = 0`, `Churned = 1`.
4. **Split the data 80/20** into training and test sets (`random_state=42` so the results can be repeated).
5. **Train a Random Forest classifier** with 100 trees.
6. **Check feature importance** to see which factors the model relies on most.
7. **Predict on the 411 new customers** and keep the ones flagged as likely to churn.

### How well does it work?

Evaluated on 1,202 customers the model had never seen:

| | Precision | Recall | F1-score |
|---|---|---|---|
| **Stayed** | 0.86 | 0.94 | 0.90 |
| **Churned** | 0.80 | 0.62 | 0.70 |

**Overall accuracy: 85%**

The confusion matrix shows where the model is right and wrong:

| | Predicted: Stayed | Predicted: Churned |
|---|---|---|
| **Actually stayed** | 810 | 54 |
| **Actually churned** | 128 | 210 |

In plain English: when the model says a customer will leave, it is right **80% of the time**. It catches **62%** of the customers who really do leave. That is a solid starting point, and the gap in recall is the first thing I would improve (see [Next steps](#limitations-and-next-steps)).

### The result

Applied to the 411 new customers, the model flagged **378** as likely to churn. These are exported to `Prediction.csv` and shown on the dashboard's Prediction page, with a full profile of who they are.

---

## Part 3: Power BI dashboard

The dashboard has two pages and a custom black-and-gold theme. I wanted it to look clean and professional while keeping the important numbers easy to spot.

### Page 1: Summary

The overall picture of churn: key numbers at the top, then the breakdown by demographics, geography, account details, services and churn reasons.

<p align="center">
  <img src="PowerBI%20Dashboard/1.%20Summary_Page.png" alt="Summary page" width="100%">
</p>

### Page 2: Prediction

Who the model thinks is at risk: a profile of the predicted churners on the left, and the full customer list on the right so a team could act on it straight away.

<p align="center">
  <img src="PowerBI%20Dashboard/2.%20Prediction_Page.png" alt="Prediction page" width="100%">
</p>

The report also includes filters (monthly change range and marital status) and a button to move between the two pages. Measures such as Total Customers, New Joiners, Total Churn and Churn Rate are written in DAX.

---

## What the data says

**The headline:** out of **6,418** customers, **1,732** have churned, a churn rate of **26.99%**.

**Contract type is the biggest signal.**
Month-to-month customers churn at **46.5%**. One-year contracts churn at **11.0%** and two-year contracts at just **2.7%**, so month-to-month customers are about 17 times as likely to leave as two-year customers.

**Competitors are the main reason people leave.**
761 of the 1,732 churned customers (about 44%) left for a competitor, mostly because the competitor had better devices (289), a better offer (274), more data (106) or faster download speeds (92). Attitude of staff comes second, with 301 customers (service provider 93, support person 208).

**Fiber Optic customers are the most likely to leave.**
Churn is **41.1%** for Fiber Optic, compared with 25.7% for Cable and 19.4% for DSL.

**Payment method matters.**
Customers paying by **mailed check** churn at **37.8%** and bank withdrawal at **34.4%**, while credit card customers churn at only **14.8%**.

**Older customers churn more.**
Customers over 50 churn at **31.6%**, compared with roughly 23 to 24% for every younger age group.

**Location stands out.**
Jammu & Kashmir has the highest churn rate at **57.2%**, followed by Assam (38.1%) and Jharkhand (34.5%).

**Who the model flags as at risk (378 customers):**
- **94%** are on month-to-month contracts (355 of 378).
- They skew older: 130 are over 50 and 127 are aged 36 to 50.
- 246 are female and 132 are male.
- The largest groups are in Uttar Pradesh (43), Maharashtra (38) and Tamil Nadu (37).

---

## Recommendations

If I were advising the business, these are the actions the data points to:

1. **Move customers off month-to-month plans.** Offer a small discount or perk for switching to a one- or two-year contract. This is the single biggest lever.
2. **Look into the Fiber Optic experience.** A 41% churn rate suggests a problem with price, quality or support for this product.
3. **Answer the competition.** Most churn is about better devices and better offers, so review device deals and data plans against rivals.
4. **Encourage automatic payments.** Customers paying by mailed check or bank withdrawal churn far more than credit card customers.
5. **Train and monitor support teams.** Staff attitude is the second biggest churn reason and it is something the company controls directly.
6. **Contact the 378 at-risk customers first.** Start with the month-to-month group, since they make up almost all of the list.

---

## Limitations and next steps

I would rather be upfront about where this can be improved:

- **Recall for churners is 62%,** so the model misses some customers who really do leave. Next I would try class balancing, tuning the decision threshold, and comparing with other models such as XGBoost and Logistic Regression.
- **The model flags a high share of new customers** (378 of 411). New joiners all have short tenure and low revenue so far, and the model associates those with churn. The output is best read as a **risk ranking** rather than a precise forecast. Using `predict_proba` to rank customers by churn probability would be a more useful next step.
- **Some customers have negative monthly charges** in the source data (for example credits or data-entry errors). I kept them as they were and they are worth investigating with the data owner.
- **No hyperparameter tuning yet.** The model uses default Random Forest settings with 100 trees.

---

## Repository structure

```
Customer-Churn-Analysis-and-Prediction/
├── README.md
├── Churn Analysis SQL.sql                  # SQL: exploration, cleaning, views
├── Power BI dashboard (.pbix)              # Two-page interactive dashboard
├── PowerBI Dashboard/                      # Dashboard screenshots
│   ├── 1. Summary_Page.png
│   ├── 2. Prediction_Page.png
│   └── Dashboard Screenshots.pdf
├── Customer Churn Prediction ML CODE/
│   ├── Churn_Prediction_ML.ipynb           # Python: model training and prediction
│   └── Prediction.csv                      # Customers predicted to churn
└── Project Resources/                      # Supporting project files
```

## How to run it

**You will need:** SQL Server with SSMS, Python 3.9 or later, Power BI Desktop, and the [Microsoft ODBC Driver 18 for SQL Server](https://learn.microsoft.com/en-us/sql/connect/odbc/download-odbc-driver-for-sql-server).

1. **Set up the database.** Create a database called `db_Churn`, import the raw data into a table named `stg_Churn`, then run `Churn Analysis SQL.sql` to build `prod_Churn` and the two views.
2. **Install the Python packages:**
   ```bash
   pip install pandas numpy scikit-learn sqlalchemy pyodbc matplotlib seaborn
   ```
3. **Check the connection.** The notebook connects with Windows authentication. If your server name or driver is different, update the connection string in the notebook:
   ```python
   engine = create_engine(
       "mssql+pyodbc://@localhost/db_Churn"
       "?driver=ODBC+Driver+18+for+SQL+Server"
       "&trusted_connection=yes"
       "&TrustServerCertificate=yes"
   )
   ```
4. **Run the notebook** top to bottom. It trains the model, prints the results and saves `Prediction.csv`.
5. **Open the `.pbix` file** in Power BI Desktop and refresh the data.

## What this project taught me

- **Data leakage is easy to miss and costly.** Dropping the churn reason and category felt obvious only after I asked, "would I know this *before* the customer leaves?"
- **Accuracy alone is not enough.** The 85% looks good, but recall tells the real story: the model still misses 38% of the people who leave.
- **Connecting Python straight to SQL** made the workflow cleaner and repeatable than passing Excel files around.
- **A good dashboard is design as well as data.** Choosing one clear colour for churn and keeping the layout simple made the story far easier to follow.

## Acknowledgements

The overall project structure was inspired by a [YouTube tutorial](https://www.youtube.com/watch?v=QFDslca5AX8). I extended it by connecting Python directly to SQL Server, writing my own analysis and recommendations, and designing a custom black-and-gold dashboard theme with my own backgrounds.
