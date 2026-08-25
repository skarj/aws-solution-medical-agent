## System Overview & Technology Stack
The platform uses an asynchronous, serverless architecture on AWS to handle heavy PDF processing, vector retrieval, and generative AI reasoning without running 24/7 idle database servers.

System Architecture Diagram:
+-----------------------------------------------------------------------------------+
|                                  AWS CLOUD ENVIRONMENT                            |
|                                                                                   |
|  [ S3 Buckets ] -------> [ Step Functions ] -------> [ Textract Async ]           |
|  (Patient/Protocol)             |                                |                |
|                                 v                                v                |
|                         [ Bedrock Agent ] <--- [ Bedrock KB + S3 Vectors ]        |
|                                 |                                                 |
|                                 v                                                 |
|                       [ DynamoDB Databases ] <-------> [ Clinical Web UI ]        |
+-----------------------------------------------------------------------------------+
### Core AWS Services
* **Storage & Hosting:** Amazon S3 (Encrypted Buckets)
* **Orchestration:** AWS Step Functions, AWS EventBridge, AWS Lambda
* **Document Processing:** Amazon Textract (Asynchronous OCR & Layout Analysis)
* **AI & RAG Engine:** Amazon Bedrock (Agent, Knowledge Bases, Titan Embeddings, Nova Pro / Claude Sonnet)
* **Vector Store:** Amazon S3 Vectors
* **Database Layer:** Amazon DynamoDB
* **Security & Auth:** AWS KMS, Amazon Cognito, AWS CloudTrail, AWS CloudWatch