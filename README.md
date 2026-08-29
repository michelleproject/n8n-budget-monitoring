# Budget Monitoring Workflow

An n8n workflow that reads personal transactions from Google Sheets, aggregates spending by category, compares each category against an NTD budget limit, and sends a Gmail summary when spending exceeds a limit.

The workflow supports both manual testing and scheduled execution.

## What it does

1. Reads transaction rows from the Google Sheets transaction tab
2. Groups transactions by category
3. Calculates total spending and transaction count for each category
4. Writes the results to a summary tab using append-or-update, so existing category rows are updated instead of duplicated
5. Compares each category total against its NTD budget limit
6. Sends a Gmail summary showing every category and clearly flags categories that exceed their limits
7. Reports `(NO LIMIT SET)` when a transaction category does not have a matching budget

## Example email output

```text
Total spending: NT$4,170

[OVER] Health: NT$702 of NT$500 - over by NT$202 (2 transactions)
[OVER] Shopping: NT$695 of NT$1,500
Subscriptions: NT$281 of NT$300 (4 transactions)
Groceries: NT$736 of NT$1,200 (3 transactions)
Transport: NT$158 of NT$150 - over by NT$8 (5 transactions)
Utilities: NT$568 of NT$1,200 (2 transactions)
Food & Drink: NT$390 of NT$400 (5 transactions)
Entertainment: NT$950 of NT$1,000 (3 transactions)

Over budget: Health, Transport
```

The email subject also changes depending on whether one or more categories exceed their limits.

## Google Sheets structure

The workflow uses two tabs.

### Transactions

```text
Date | Description | Amount | Category
```

Example:

```text
2026-07-01 | Grocery shopping | 650 | Groceries
2026-07-02 | Bus fare | 50 | Transport
```

Amounts should be stored as plain numbers in NTD. Do not include `NT$`, commas, or other formatting in the cell value.

### Summary

```text
Category | Total | Transaction Count
```

This tab is updated by the workflow. It matches rows by category and updates existing rows instead of creating duplicates.

## Budget limits

Budget limits are defined in the `Check NTD budget limits` Code node.

```javascript
const budgets = {
  "Groceries": 1200,
  "Health": 500,
  "Shopping": 1500,
  "Transport": 150,
  "Utilities": 1200,
  "Subscriptions": 300,
  "Food & Drink": 400,
  "Entertainment": 1000
};
```

The values represent NTD budget limits.

Category names must match the `Category` column exactly. A mismatch is reported as `(NO LIMIT SET)` so the category does not remain silently unmonitored.

## Workflow triggers

The workflow includes two triggers:

- Manual Trigger for testing
- Schedule Trigger for unattended execution

The schedule can be adjusted directly in n8n to fit your preferred review cycle.

## Setup

Requires:

- n8n
- Google Sheets OAuth credential
- Gmail OAuth credential
- A Google Spreadsheet with the required tabs and columns

Steps:

1. Import `budgeting_workflow.json` into n8n
2. Reconnect your Google Sheets and Gmail credentials
3. Create the `Transactions` and `Summary` tabs
4. Point the Google Sheets nodes to your spreadsheet
5. Set the Gmail recipient to your own email address
6. Update the NTD budget limits in the `Check NTD budget limits` node
7. Adjust the schedule if needed
8. Run the workflow manually first to verify the configuration

## Design notes

The workflow separates aggregation from budget checking.

`Aggregate by category` handles transaction totals and counts.

`Check NTD budget limits` handles budget rules, formatting, and the email content.

This separation makes it easier to change budget logic without modifying the aggregation step.

A category without a matching budget is deliberately surfaced as `(NO LIMIT SET)` instead of being treated as within budget.

## Repository safety

Do not commit real Google account credentials, OAuth client secrets, access tokens, or private spreadsheet data.

The n8n workflow export may still contain credential metadata, spreadsheet IDs, cached Google Sheets URLs, and other environment-specific identifiers. For a public portfolio repository, replace these with placeholders before publishing.

The included spreadsheet data should remain dummy or non-sensitive data.
