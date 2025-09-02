# AWS Document Knowledge Converter

An AWS-based document conversion system that automatically converts PDF and DOCX files to markdown format for LLM RAG (Retrieval-Augmented Generation) applications. Built with CloudFormation for infrastructure as code.

[![AWS](https://img.shields.io/badge/AWS-Serverless-orange)](https://aws.amazon.com/)
[![Python](https://img.shields.io/badge/Python-3.11-blue)](https://python.org)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

## 📋 Table of Contents

- [Features](#-features)
- [Architecture](#-architecture)
- [Prerequisites](#-prerequisites)
- [Quick Start](#-quick-start)
- [Deployment](#-deployment)
- [Configuration](#-configuration)
- [Monitoring](#-monitoring)
- [API Usage](#-api-usage)
- [Troubleshooting](#-troubleshooting)
- [Contributing](#-contributing)
- [License](#-license)

## 🚀 Features

- **Automatic Conversion**: Converts PDF and DOCX files to markdown upon upload
- **Serverless Architecture**: Fully managed AWS services with auto-scaling
- **Reliable Processing**: SQS integration prevents data loss during failures
- **Comprehensive Monitoring**: CloudWatch dashboards and alerts
- **Multi-format Support**: PDF and DOCX file processing
- **Batch Processing**: Efficiently handles multiple files simultaneously
- **Dead Letter Queue**: Failed conversions are captured for investigation
- **Infrastructure as Code**: CloudFormation templates for consistent deployments

## 🏗️ Architecture

The system consists of two CloudFormation stacks:

### Infrastructure Stack (`kcInfrastructure.yaml`)
- **S3 Buckets**: Source documents and markdown output storage
- **IAM Roles**: Execution and task roles for Lambda functions
- **SQS Queues**: Document conversion events and dead letter queue
- **CloudWatch**: Log groups, alarms, and monitoring dashboard
- **SSM Parameters**: Configuration management

### Lambda Stack (`kslambda.yaml`)
- **Lambda Function**: Document processing with Python 3.11
- **Lambda Layer**: Document processing libraries (pdfplumber, python-docx, pandas)
- **Event Source Mapping**: SQS polling for document processing
- **S3 Notifications**: Triggers for new document uploads

### Data Flow
```
1. Upload PDF/DOCX → S3 Source Bucket
2. S3 Notification → SQS Queue
3. Lambda Polls → SQS Queue
4. Document Processing → Markdown Output
5. Results Stored → S3 Output Bucket
```

## 📋 Prerequisites

- **AWS Account** with appropriate permissions
- **AWS CLI** configured with your credentials
- **Git** for cloning the repository
- **Python 3.11** (for local development/testing)

### Required AWS Permissions

Your AWS account needs these permissions:
- CloudFormation stack creation
- S3 bucket operations
- Lambda function deployment
- SQS queue management
- IAM role creation
- CloudWatch monitoring

## 🚀 Quick Start

1. **Clone the repository**
   ```bash
   git clone https://github.com/gitdavehorsley/llmKnowledgeConvert.git
   cd llmKnowledgeConvert
   ```

2. **Deploy the infrastructure**
   ```bash
   aws cloudformation create-stack \
     --stack-name knowledge-converter-infrastructure \
     --template-body file://kcInfrastructure.yaml \
     --capabilities CAPABILITY_NAMED_IAM
   ```

3. **Deploy the Lambda function**
   ```bash
   aws cloudformation create-stack \
     --stack-name knowledge-converter-lambda \
     --template-body file://kslambda.yaml \
     --capabilities CAPABILITY_NAMED_IAM
   ```

4. **Upload a document to test**
   ```bash
   aws s3 cp sample-document.pdf s3://your-source-bucket/
   ```

## 📦 Deployment

### Infrastructure Stack Deployment

```bash
aws cloudformation create-stack \
  --stack-name knowledge-converter-infrastructure \
  --template-body file://kcInfrastructure.yaml \
  --parameters \
    ParameterKey=ProjectName,ParameterValue=knowledge-converter \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=EnableDetailedMonitoring,ParameterValue=true \
  --capabilities CAPABILITY_NAMED_IAM
```

### Lambda Stack Deployment

```bash
aws cloudformation create-stack \
  --stack-name knowledge-converter-lambda \
  --template-body file://kslambda.yaml \
  --parameters \
    ParameterKey=ProjectName,ParameterValue=knowledge-converter \
    ParameterKey=Environment,ParameterValue=dev \
    ParameterKey=LambdaMemorySize,ParameterValue=1536 \
    ParameterKey=LambdaTimeout,ParameterValue=600 \
  --capabilities CAPABILITY_NAMED_IAM
```

### Production Deployment

For production deployments, use S3-based Lambda code and layers:

```bash
# Upload Lambda layer
aws s3 cp document-processing-layer.zip s3://your-deployment-bucket/layers/

# Upload Lambda function code
aws s3 cp document-converter.zip s3://your-deployment-bucket/lambda/

# Deploy with S3 references
aws cloudformation create-stack \
  --stack-name knowledge-converter-prod \
  --template-body file://kslambda.yaml \
  --parameters \
    ParameterKey=DeploymentPackageMethod,ParameterValue=s3 \
    ParameterKey=CodeS3Bucket,ParameterValue=your-deployment-bucket \
    ParameterKey=LayerDeploymentMethod,ParameterValue=s3 \
    ParameterKey=LayerS3Bucket,ParameterValue=your-deployment-bucket \
  --capabilities CAPABILITY_NAMED_IAM
```

## ⚙️ Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `MARKDOWN_BUCKET_NAME` | Output bucket for converted documents | Auto-generated |
| `LOG_LEVEL` | Logging level (DEBUG, INFO, WARNING, ERROR) | INFO |
| `ENABLE_DETAILED_MONITORING` | Enable custom CloudWatch metrics | false |
| `MAX_FILE_SIZE_MB` | Maximum file size to process | 50 |
| `ENABLE_IMAGE_PROCESSING` | Process images in PDFs | false |

### Lambda Configuration

| Parameter | Description | Default | Range |
|-----------|-------------|---------|-------|
| `LambdaMemorySize` | Memory allocation in MB | 1536 | 512-10240 |
| `LambdaTimeout` | Function timeout in seconds | 600 | 60-900 |
| `LambdaArchitecture` | Processor architecture | x86_64 | x86_64, arm64 |
| `BatchSize` | SQS messages per Lambda invocation | 10 | 1-10 |

### SQS Configuration

| Setting | Value | Description |
|---------|-------|-------------|
| Visibility Timeout | 300s | Message processing timeout |
| Message Retention | 4 days | How long to keep messages |
| Max Message Size | 256KB | Maximum message size |
| Receive Wait Time | 20s | Long polling enabled |

## 📊 Monitoring

### CloudWatch Dashboard

The system includes a comprehensive CloudWatch dashboard with:

- **Queue Status**: SQS queue depth and message throughput
- **Lambda Performance**: Duration, errors, and throttling
- **Document Processing**: Conversion success rates and processing times
- **Resource Utilization**: Memory and CPU usage trends

Access the dashboard:
```bash
# Get dashboard name from CloudFormation outputs
aws cloudformation describe-stacks \
  --stack-name knowledge-converter-infrastructure \
  --query 'Stacks[0].Outputs[?OutputKey==`DashboardName`].OutputValue'
```

### Key Alarms

| Alarm | Threshold | Description |
|-------|-----------|-------------|
| Queue Depth | >100 messages | Too many pending conversions |
| Lambda Errors | >1 error | Function execution failures |
| DLQ Messages | >1 message | Failed conversions requiring attention |
| Lambda Duration | >500s | Approaching timeout limit |

### Logs and Metrics

```bash
# View Lambda logs
aws logs tail /aws/lambda/knowledge-converter-document-converter \
  --follow

# Get queue metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/SQS \
  --metric-name ApproximateNumberOfMessagesVisible \
  --dimensions Name=QueueName,Value=knowledge-converter-document-conversion-events \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 300 \
  --statistics Average
```

## 🔧 API Usage

### Direct Lambda Invocation

You can invoke the Lambda function directly for testing:

```bash
aws lambda invoke \
  --function-name knowledge-converter-document-converter \
  --payload '{"Records": [{"s3": {"bucket": {"name": "test-bucket"}, "object": {"key": "test.pdf"}}}]' \
  response.json
```

### Function URL (Optional)

If enabled, you can call the Lambda function via HTTP:

```bash
curl -X POST \
  https://your-function-url.amazonaws.com \
  -H "Content-Type: application/json" \
  -d '{"test": "data"}'
```

## 🐛 Troubleshooting

### Common Issues

**1. SQS Messages Not Processing**
```bash
# Check Lambda event source mapping
aws lambda list-event-source-mappings \
  --function-name knowledge-converter-document-converter

# Check SQS queue depth
aws sqs get-queue-attributes \
  --queue-url your-queue-url \
  --attribute-names ApproximateNumberOfMessages
```

**2. Lambda Function Errors**
```bash
# View recent errors
aws logs filter-log-events \
  --log-group-name /aws/lambda/knowledge-converter-document-converter \
  --filter-pattern ERROR \
  --start-time $(date -d '1 hour ago' +%s)000
```

**3. S3 Upload Not Triggering**
```bash
# Verify bucket notifications
aws s3api get-bucket-notification-configuration \
  --bucket your-source-bucket-name
```

### Debug Mode

Enable detailed logging by setting the environment variable:
```bash
LOG_LEVEL=DEBUG
```

### Dead Letter Queue Investigation

Check failed messages in the DLQ:
```bash
aws sqs receive-message \
  --queue-url your-dlq-url \
  --max-number-of-messages 10
```

## 🤝 Contributing

We welcome contributions! Please follow these steps:

1. **Fork the repository**
2. **Create a feature branch**
   ```bash
   git checkout -b feature/your-feature-name
   ```
3. **Make your changes**
4. **Test thoroughly**
5. **Submit a pull request**

### Development Setup

1. **Set up virtual environment**
   ```bash
   python3 -m venv venv
   source venv/bin/activate
   ```

2. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

3. **Run tests**
   ```bash
   python -m pytest tests/
   ```

4. **Validate CloudFormation**
   ```bash
   aws cloudformation validate-template --template-body file://kslambda.yaml
   ```

### Code Standards

- Use Python type hints
- Follow PEP 8 style guidelines
- Add comprehensive docstrings
- Write unit tests for new features
- Update documentation for changes

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- AWS for providing the serverless platform
- The open-source community for document processing libraries
- Contributors and maintainers of this project

## 📞 Support

- **Issues**: [GitHub Issues](https://github.com/gitdavehorsley/llmKnowledgeConvert/issues)
- **Discussions**: [GitHub Discussions](https://github.com/gitdavehorsley/llmKnowledgeConvert/discussions)
- **Documentation**: This README and inline code comments

---

**Built with ❤️ using AWS CloudFormation and Python**
