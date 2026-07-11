# 🧾 Paper Trail

A serverless, mobile-friendly receipt tracker built on AWS. Take a photo of a receipt, and it turns itself into a categorized expense, ready for a CSV report at month-end or financial year-end.

Built as part of the AWS Weekend Productivity Challenge.

![Paper Trail banner](./docs/banner.png)

---

## What it does

- **Sign in** through Cognito Hosted UI (Authorization Code + PKCE, no client secret).
- **Upload a receipt** by camera capture on mobile or file picker on desktop.
- Behind the scenes: **Textract** extracts the raw fields from the receipt, and **Bedrock (Claude)** normalizes them into a clean merchant, amount, currency, date, and category.
- **Browse expenses** in a running list, filtered by month or shown in total.
- **Download a CSV report** for the current month or current financial year with one tap, or a custom date range for anything else.

No manual data entry, no spreadsheet wrangling.

## Architecture

![Architecture diagram](./docs/architecture.png)

```
User → CloudFront → S3 (static frontend)
User → Cognito Hosted UI (login)
User → API Gateway (JWT-authorized)
         ├─ POST /receipts  → Lambda (ReceiptUpload)  → S3 (receipts bucket)
         │                                                  │
         │                                     S3 ObjectCreated event
         │                                                  ▼
         │                                   Lambda (ProcessReceipt)
         │                                       ├─ Textract AnalyzeExpense
         │                                       ├─ Bedrock InvokeModel (Claude)
         │                                       └─ DynamoDB PutItem
         ├─ GET  /expenses  → Lambda (ListExpenses)  → DynamoDB Query
         └─ GET  /report    → Lambda (GenerateReport) → DynamoDB Query → CSV
```

### AWS services used

| Service | Purpose |
|---|---|
| Amazon Cognito | User authentication (Hosted UI, JWT issuance) |
| Amazon S3 | Static frontend hosting + receipt storage |
| Amazon CloudFront | HTTPS delivery of the frontend |
| Amazon API Gateway (HTTP API) | JWT-authorized REST routes |
| AWS Lambda | Upload handling, receipt processing, expense queries, CSV generation |
| Amazon Textract | `AnalyzeExpense` OCR extraction from the receipt image |
| Amazon Bedrock | Claude normalizes Textract's output into structured fields |
| Amazon DynamoDB | Stores each expense, keyed by user and date |

All infrastructure is defined in a single CloudFormation template, no servers to patch or scale.

## Repo structure

```
.
├── receipt-tracker-infra.yaml   # CloudFormation template (all AWS resources)
├── index.html                   # Static frontend (no build step required)
└── docs/
    ├── banner.png
    └── architecture.png
```

## Prerequisites

- An AWS account with the AWS CLI configured
- [Model access](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html) enabled in the Bedrock console for the model you'll use (default: `anthropic.claude-3-5-sonnet-20241022-v2:0`)

## Deploy

```bash
aws cloudformation deploy \
  --template-file receipt-tracker-infra.yaml \
  --stack-name receipt-tracker \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
      CognitoDomainPrefix=your-unique-prefix \
      FinancialYearStartMonth=4
```

`CognitoDomainPrefix` must be globally unique across all AWS accounts (it becomes `https://<prefix>.auth.<region>.amazoncognito.com`). `FinancialYearStartMonth` defaults to `4` (April); set it to `1` for a calendar-year financial year, or whatever fits your locale.

Grab the outputs you'll need for the frontend:

```bash
aws cloudformation describe-stacks \
  --stack-name receipt-tracker \
  --query 'Stacks[0].Outputs' \
  --output table
```

## Configure and deploy the frontend

Open `index.html` and fill in the `CONFIG` object near the top of the `<script>` tag with the stack outputs:

```js
const CONFIG = {
  userPoolClientId: '...',       // Output: UserPoolClientId
  cognitoHostedUiDomain: '...',  // Output: CognitoHostedUiDomain
  apiEndpoint: '...',            // Output: ApiEndpoint
  financialYearStartMonth: 4     // must match the FinancialYearStartMonth parameter above
};
```

Then upload it to the frontend bucket and invalidate CloudFront:

```bash
aws s3 cp index.html s3://<FrontendBucketName>/index.html

aws cloudfront create-invalidation \
  --distribution-id <your-distribution-id> \
  --paths "/*"
```

`FrontendBucketName` is also in the stack outputs. The CloudFront distribution ID can be found with:

```bash
aws cloudfront list-distributions \
  --query "DistributionList.Items[?Origins.Items[0].DomainName=='<FrontendBucketName>.s3.<region>.amazonaws.com'].Id" \
  --output text
```

## API reference

All routes require `Authorization: Bearer <Cognito ID token>`.

| Method | Path | Description |
|---|---|---|
| `POST` | `/receipts` | Upload a receipt. Body: `{ "filename": "...", "contentType": "image/jpeg", "fileBase64": "..." }`. Max 5MB decoded. |
| `GET` | `/expenses?start=YYYY-MM-DD&end=YYYY-MM-DD` | List expenses in a date range. Omit both params for all-time. |
| `GET` | `/report?period=month&value=YYYY-MM` | Download a CSV for a given month. |
| `GET` | `/report?period=financial_year&value=YYYY` | Download a CSV for the financial year starting in `YYYY`. |
| `GET` | `/report?period=custom&start=YYYY-MM-DD&end=YYYY-MM-DD` | Download a CSV for a custom range. |

## Known issues / TODO

- The bundled `index.html` currently targets the `POST /receipts` upload API described above. If you're working from an earlier copy of this frontend that calls `GET /upload-url` and PUTs directly to S3, update it to send a JSON body to `POST /receipts` instead, that older direct-to-S3 flow is no longer part of this architecture.
- No automated tests yet. Contributions welcome.
- No token refresh: the Cognito ID token expires in 1 hour, at which point the app requires signing in again rather than silently refreshing.

## License

MIT, or your license of choice, update this section before publishing.
