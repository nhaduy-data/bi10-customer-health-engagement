# Customer Financial Health & Engagement – Business Intelligence Season 10, Round 2 (Synthesize)

Round 2 of Business Intelligence Season 10 (ITB Club, University of Economics and Law, 2026). The team went on to win the competition: [final round repo](https://github.com/nhaduy-data/bi10-shop-growth-potential).

A consumer-finance company has data on 1 million customers (profiles, loan contracts, sales leads, scoring events and sales calls) but no single view of each customer. The brief had three parts:

1. **Financial Health Score**: how healthy each customer's finances are, on a 0–100 scale.
2. **Engagement Score**: how actively each customer interacts with the company, checked against real business results.
3. **Segmentation and Next Best Action**: combine the two scores and decide what to do with each group.

Slides: [`slides/BI10_Round2_Slides.pdf`](slides/BI10_Round2_Slides.pdf)

## Data

Five tables joined on the customer key, with 4 June 2025 as the cut-off date: 1,000,528 customers, 4.28M loan contracts, 2.1M leads, 4.4M scoring events and 4M calls.

Data quality issues found and handled before any scoring:
- 1.95M contracts appeared twice, once with complete personal fields and once blank; one row per contract was kept.
- 74% of calls fall after the cut-off date, so call metrics only use the 90 days before it.
- Each table uses its own product codes; they were mapped to five product groups.

## Results

**Engagement Score**
- Five behavioral signals chosen from 16 candidates: recency, number of calls, call response, months with a lead, and number of products offered. Each gives up to 20 points, so the 0–100 score can be traced point by point. It reaches AUC 0.670, only 0.008 below a logistic regression.
- Tested without leakage: the method was rebuilt on data before April and checked against May contracts. Signing rates rise steadily from Dormant to High (1.43%, 3.34%, 5.39%, 6.63%), and the High group holds **12.1% of customers but 30.1% of new contract value**.
- Calls measure the company's effort, not the customer's interest, so the score keeps reach and response apart. Customers engaged both ways sign at 7.15%, against 1.54% for those low on both; 76,451 customers are contacted often but rarely respond.

**Financial Health Score**
- FHS = 0.4 × debt-service ratio score + 0.1 × residual income score + 0.5 × late-payment score. A written-off contract caps the score at 50.
- 762,665 customers have enough data for a score. Customers with no evidence are not given a guessed score.

**Segmentation and Next Best Action**
- The two scores barely move together (rank correlation 0.11), so they were crossed into 10 segments: high engagement drives value, but financial health decides the action.
- Stable, engaged customers bring 38.4% of May's new contract value and get personalized offers. Engaged customers under financial pressure (score below 60, 96.9% holding active loans) still took 19.5% of it, so the action there is debt counseling and a pause on cross-selling.
- Each customer gets an action, a channel (app, call or SMS), a product and a check to run before the offer. Demographics are never used to judge risk.

## Files

The folders are as submitted to the judges. Notebooks are written in Vietnamese and keep their outputs.

| Folder | Content |
|---|---|
| `Source_code-notebook/EDA/` | Data exploration and cleaning |
| `Source_code-notebook/Challenge 1/` | Financial Health Score, step by step to the final version |
| `Source_code-notebook/Challenge 2/` | Engagement Score |
| `Source_code-notebook/Challenge 3/` | Segmentation and Next Best Action |
| `Source_code-notebook/BI_ROUND2_NHI_v3.ipynb` | Additional data checks |
| `README/` | Method write-ups for preprocessing and each challenge |
| `Output- dataset/Challenge 1/C1_FINAL/` | Final scoring weights and configuration |

The competition data and every customer-level output are not included. Customer and call IDs shown in notebook outputs are masked as `***`.

## Team

A five-person team project; I was the team leader and did the EDA, data cleaning and the Engagement Score (Challenge 2). I know my own parts in full detail, and my teammates' parts less deeply.

Other rounds: [Round 1](https://github.com/nhaduy-data/bi10-consumer-financial-health) · [Final round](https://github.com/nhaduy-data/bi10-shop-growth-potential)
