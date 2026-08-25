## Security, Governance & Compliance Architecture

* **Data Encryption:** All S3 buckets, DynamoDB records, and S3 Vectors are encrypted at rest using AWS KMS Customer-Managed Keys (CMK). Data in transit is secured using TLS 1.3.
* **Access Control:** User authentication is managed by Amazon Cognito with MFA. Service-to-service access uses fine-grained IAM roles following least-privilege principles.
* **Audit & Logging:** AWS CloudTrail captures all system API requests. AWS CloudWatch monitors Lambda logs and Step Functions state transitions for compliance auditing under the AWS Business Associate Addendum (BAA).
