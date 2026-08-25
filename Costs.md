## Sizing & Estimated Operational Cost Model

### Baseline Volume Assumptions
* **Monthly Screening Volume:** 100 patients / month
* **Average Patient Record Size:** 2 PDFs per patient, averaging 100 pages per file (20,000 pages / month total OCR volume)
* **Protocol Onboarding:** 2 new trial protocols / month (200 pages total)

### Estimated Monthly AWS Cost Breakdown

| Component                             | Service                                                                | Monthly Cost (USD)                |
| :------------------------------------ | :--------------------------------------------------------------------- | :-------------------------------- |
| **Document Processing**               | Amazon Textract (Async OCR)                                            | $40.00 – $90.00                   |
| **AI & Vector Engine**                | Amazon Bedrock (Agent, KB, Titan Embeddings, Nova Pro / Claude Sonnet) | $50.00 – $120.00                  |
| **Storage & Vectors**                 | Amazon S3 + Amazon S3 Vectors                                          | $10.00 – $25.00                   |
| **Database & Cache**                  | Amazon DynamoDB (On-Demand Mode)                                       | $5.00 – $15.00                    |
| **Orchestration**                     | AWS Step Functions + AWS Lambda + API Gateway                          | $5.00 – $15.00                    |
| **Security & Logging**                | AWS KMS + AWS CloudTrail + AWS CloudWatch + GuardDuty                  | $50.00 – $80.00                   |
| **Total Estimated Operational Spend** |                                                                        | **~$160.00 – $345.00 / month**    |
| **Total Estimated Annual Spend**      |                                                                        | **~$1,920.00 – $4,140.00 / year** |
