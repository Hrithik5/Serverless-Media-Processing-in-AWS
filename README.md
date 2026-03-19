
# Serverless Media Processing in AWS

A secure, scalable, and event-driven backend system for ingesting, processing, and serving images (or files) using AWS managed services.

The project demonstrates a hybrid architecture combining containerized APIs with serverless processing, designed for high scalability, strong isolation, and cost efficiency.

# Overview
This project demonstrates AWS serverless best practices by combining:

- Event-Driven Architecture: S3 bucket uploads automatically trigger processing.
- Automated Image Processing: Uses Python and the Pillow library to resize, compress, and convert formats.
- Infrastructure as Code (IaC): Entirely provisioned via modular Terraform.
- Advanced Monitoring: A custom CloudWatch Dashboard with 12 alarms and SNS notifications.
- Security-First Design: Implements IAM least privilege, private S3 buckets, and server-side encryption

# Architecture Diagram 

![image]()


# 🔄 End-to-End System Workflow
This project implements a hybrid serverless architecture that decouples high-speed API ingestion from heavy-duty image processing.
1. 🟢 Phase 1: Request Entry (User → API)

- A user initiates a request via HTTPS to upload or process an image.
-  The request is received by an Application Load Balancer (ALB), which serves as the only public entry point.
-  The ALB routes the traffic to the ECS Fargate service running a Flask API within private subnets, ensuring the compute layer is never directly exposed to the internet.


2. 🟡 Phase 2: API Handling (Synchronous Layer)
-  The Flask API validates the request and generates a unique Job ID for tracking.
-  Initial metadata (e.g., status = PENDING, timestamps) is written to DynamoDB.
-  The file is uploaded to the S3 Source Bucket using IAM roles (no hardcoded credentials) via a VPC Endpoint, keeping all traffic within the AWS internal network.
-  The metadata status in DynamoDB is updated to UPLOADED.
3. 🟠 Phase 3: Event Trigger (The Decoupling Point)
- S3 detects the new object and immediately generates a PutObject event notification.
- This event asynchronously triggers the AWS Lambda function, acting as the core architectural boundary between the API and the processing logic.
4. 🔵 Phase 4: Serverless Processing (Lambda Execution)
- AWS Lambda is invoked with the S3 event payload and retrieves the raw file from the Source Bucket.
- Using the Pillow (PIL) library, the function performs image transformations, generating five distinct variants: Compressed JPEG (85%), Low-Quality JPEG (60%), WebP, PNG, and a Thumbnail.
- Processed variants are saved to the S3 Destination Bucket.
Lambda updates DynamoDB metadata: status = COMPLETED and records the output S3 paths.
5. 🟣 Phase 5: Observability & Monitoring
- Comprehensive Logging: Both ECS and Lambda stream logs to CloudWatch Logs for real-time auditing.
- Metrics & Alarms: The system tracks custom metrics like ProcessingTime and SuccessRates.
- Proactive Alerting: If errors or performance bottlenecks (like timeouts or memory limits) occur, 12 CloudWatch Alarms trigger notifications via SNS (Email/SMS).
6. 🔴 Phase 6: Result Retrieval (Client Access)
- The client polls the API using their Job ID.
- The Flask API queries DynamoDB to check the processing status.
- Once the status is COMPLETED, the API returns a reference or Signed URL for the processed variants stored in S3.

--------------------------------------------------------------------------------
## ⚙️ Supporting Infrastructure Design
- 🔐 Security Posture
Network Isolation: All compute resources (ECS and Lambda) reside in private subnets.
Private Connectivity: Communication with S3 and DynamoDB is handled through VPC Endpoints, eliminating exposure to the public internet.
Least Privilege: IAM policies are restricted to granular actions (e.g., s3:GetObject on source, s3:PutObject on destination).
- 📈 Scaling Behavior
Dynamic Scaling: The ALB distributes load across ECS tasks that scale based on CPU usage, while Lambda scales instantly on a per-event basis.
Managed Persistence: S3 and DynamoDB provide virtually unlimited, transparent scaling for storage and metadata.
- 🔁 Failure Handling
Automatic Retries: Lambda is configured to retry on execution failures.
Visibility: Dedicated alarms monitor Error Rates and Log-Based Patterns to ensure any failures are immediately reported to administrators
---

## 🗺️ Roadmap

Future enhancements:
- [ ] Add API Gateway for direct uploads
- [ ] Implement Step Functions for complex workflows
- [ ] Implement dead letter queue (DLQ)
- [ ] Add cost allocation tags

## 🤝 Contributing

Contributions welcome! Please:
-  Fork the repository
-  Create a feature branch (`git checkout -b feature/new-feature`)
-  Commit your changes (`git commit -m 'Add new feature'`)
-  Push to the branch (`git push origin feature/new-feature`)
-  Open a Pull Request
