# Payment Anomaly Detector

A Python tool that runs four audit data analytics tests on a full population of real government payments, not a sample, and flags the transactions an auditor should look at first.

I ran it on the **Texas Education Agency's FY2025 Check Register**: 61,775 payments totaling **$40.2 billion** to 3,300+ school districts and vendors, from September 2024 to August 2025. The whole dataset is tested in a few seconds.

| Headline result | |
|---|---|
| Payments tested | 61,775 ($40.2B) |
| Potential duplicate payment pairs | 62 |
| High-risk pairs after triage | **7, totaling $1.8M** |
| Largest flagged item | $1,249,104.08 to Texas Tech University, repeated within 6 days |
| Weekend postings | 0 |
| Benford's Law | Significant deviation, explained by recurring fixed payments |

---

## Why I made this

Duplicate and irregular payments are one of the most common risks in accounts payable, and they're easy to miss when an auditor can only sample a small share of transactions. I wanted to see what it looks like to test **100% of a population** the way modern audit data analytics does, on a real dataset instead of a textbook example.

## What it tests

| Test | What it flags | Why it matters |
|---|---|---|
| **Duplicate payment testing** | Same vendor, same amount, paid within 7 days | The classic test for double-paid invoices |
| **Round-dollar analysis** | Amounts that land exactly on $100 or $1,000 multiples | Real invoices are rarely perfectly round; round amounts can point to estimates or manual overrides |
| **Weekend posting check** | Payments dated on a Saturday or Sunday | Unusual for routine processing; can indicate entries made outside normal controls |
| **Benford's Law** | Deviation in the distribution of leading digits, tested with a chi-square goodness-of-fit test | Naturally occurring financial data follows a predictable digit pattern; large deviations are worth explaining |

## Results

| Metric | Result |
|---|---|
| Total payments analyzed | 61,775 |
| Total dollar value | $40,178,684,871.89 |
| Potential duplicate pairs | 62 |
| Round-dollar payments | 1,780 (2.9%): 551 round to $1,000, 1,229 more round to $100 |
| Weekend postings | 0 (0.00%) |
| Benford's Law chi-square | 99.60 (p < 0.0001) |

### Risk-based triage of the 62 duplicate pairs

A list of 62 flags isn't useful until it's prioritized, so I reviewed every pair in Excel and sorted them into red / yellow / green tiers using a rule based on dollar size and timing:

- **High priority (red):** any repeat of **$50,000 or more**, or a **next-day repeat of at least $1,000**.
- **Medium (yellow):** meaningful amounts repeated two or more days apart.
- **Low (green):** small reimbursements and recurring charges that repeat by design.

The $1,000 floor on next-day repeats matters. A next-day repeat of $12.84 is almost certainly a routine split reimbursement, not a control failure. For example, a $485 next-day repeat to the Texas Association of School Administrators stayed in the low tier, while a $1,170 next-day repeat to the same vendor cleared the bar.

**Seven pairs, totaling $1.8M, met the high-priority bar:**

| Vendor | Amount | Gap between payments |
|---|---|---|
| Texas Tech University | $1,249,104.08 | 6 days |
| State Office of Administrative Hearings | $420,685.65 | 1 day |
| Trademark Media Corporation | $69,988.55 | 7 days |
| Nederland ISD | $55,000.00 | 5 days |
| C & T Consulting Services LLP | $3,344.00 | 1 day |
| The Bruman Group PLLC | $3,040.00 | 1 day |
| Texas Association of School Administrators | $1,170.00 | 1 day |

Most of the remaining flags were low risk. For example, the Texas Comptroller's office shows up repeatedly with identical $50 and $435 charges, which is clearly a standing arrangement rather than an error.

**Important caveat:** the dataset only includes vendor, date, and amount, with no invoice numbers or department codes. This triage identifies which payments are worth pulling documentation on, not which ones are confirmed errors. A real audit team would request the supporting invoices for the high-priority items next.

### The other tests

- **Round dollars (2.9%)** is unremarkable on its own.
- **Zero weekend postings** is a good sign. It points to a tightly controlled, business-day-only payment process.
- **Benford's Law** needed a closer look. A chi-square of 99.60 with 8 degrees of freedom is about six times the 0.05 significance threshold (≈15.5), so this is not a borderline result. But a significant deviation isn't automatically a red flag. This dataset is full of **recurring, fixed-amount payments** (the same vendor paid the same amount over and over), which naturally skews leading digits away from Benford's expected pattern. That's a structural feature of government disbursement data, not evidence of manipulation.

![Benford's Law: observed vs. expected leading digit distribution](benford_chart.png)

## Limitations

- The 7-day duplicate window and the triage thresholds are judgment calls. Both can be adjusted in the code.
- Benford's Law works best on naturally occurring, unconstrained amounts. Recurring fixed payments limit how much it can tell you here.
- Without invoice numbers or GL codes, the analysis is narrower than what an audit team with client access would run.

## How I made it

I used AI-assisted development (Claude) to write the Python code. My part was the audit side: choosing which tests to run, setting the risk thresholds, reviewing and tiering all 62 flagged pairs by hand in Excel, and working out why the Benford's Law result deviated.

## Run it yourself

1. Install the dependencies:
   ```bash
   pip install -r requirements.txt
   ```
2. Download the dataset from TEA (it's too large to include in this repo) and save it in the project folder as `25-cr-report.csv`:
   ```
   https://tea.texas.gov/about-tea/agency-finances/check-register/25-cr-report.csv
   ```
3. Run:
   ```bash
   python anomaly_detector.py
   ```

It outputs:
- `anomaly_summary.csv`: headline numbers
- `flagged_duplicates.csv`: every flagged duplicate pair
- `benford_chart.png`: observed vs. expected leading-digit distribution

## What's next

- **Follow-up testing:** reviewing each high-priority vendor's full payment history to separate recurring scheduled payments from true one-off duplicates.
- **Web app:** a Streamlit version that lets anyone upload their own spreadsheet, map their column names, and run the same four tests.

## Tools

Python · pandas · NumPy · SciPy · Matplotlib · Excel
