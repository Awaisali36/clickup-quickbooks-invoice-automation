# Customer Deduplication Logic

## Overview

Before any invoice is created, the system checks QuickBooks for an
existing customer record. This prevents duplicate entries, billing
confusion, and broken payment histories.

---

## Deduplication Flow

```
On deal close:

  Extract from ClickUp:
    customer_name
    customer_email (primary match key)
    company_name   (secondary match key)

  Query QuickBooks:
    GET /v3/company/{companyId}/query
    SELECT * FROM Customer
    WHERE PrimaryEmailAddr = '{customer_email}'

  Result: customer found?
    YES → Use existing QuickBooks customer ID
          Skip creation entirely
          Proceed directly to invoice generation

    NO  → Check by company name as fallback:
          SELECT * FROM Customer
          WHERE CompanyName = '{company_name}'

          Found by name?
            YES → Flag for review (possible email mismatch)
                  Use existing ID with warning in Slack alert
            NO  → Create new QuickBooks customer
                  Store new customer ID
                  Proceed to invoice generation
```

---

## Invoice Generation Payload

```json
{
  "Line": [
    {
      "Amount": "{{deal_amount}}",
      "DetailType": "SalesItemLineDetail",
      "SalesItemLineDetail": {
        "ItemRef": { "value": "1", "name": "Services" },
        "Qty": 1,
        "UnitPrice": "{{deal_amount}}"
      },
      "Description": "{{deal_name}} — {{deal_close_date}}"
    }
  ],
  "CustomerRef": {
    "value": "{{quickbooks_customer_id}}"
  },
  "DueDate": "{{due_date}}",
  "PaymentMethodRef": "{{payment_terms}}",
  "DocNumber": "{{auto_reference_number}}"
}
```

---

## Error Handling & Retries

| Scenario | Behaviour |
|---|---|
| QuickBooks API timeout | Retry 3× with 15s backoff |
| Duplicate invoice detected | Block creation, Slack alert fired |
| Missing deal amount in ClickUp | Hold invoice, Slack alert for manual review |
| Customer create fails | Retry once, then alert ops team |
| Google Sheets log fails | Invoice still created, log failure separately |

---

## Google Sheets Audit Schema

| Column | Value |
|---|---|
| `timestamp` | Trigger datetime |
| `deal_name` | ClickUp deal title |
| `customer_name` | QuickBooks customer name |
| `customer_status` | existing / new / flagged |
| `invoice_id` | QuickBooks invoice ID |
| `amount` | Invoice total |
| `due_date` | Payment due date |
| `slack_notified` | TRUE / FALSE |
| `status` | success / failed / pending |
| `error_message` | If applicable |
