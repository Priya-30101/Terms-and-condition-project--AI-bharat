# Design Document: AI-Powered Terms & Conditions Risk Analyzer

## Overview

The AI-Powered Terms & Conditions Risk Analyzer is a serverless AWS application that analyzes legal documents using AI to identify risk clauses, categorize them, and provide risk scores. Users upload documents, receive automated analysis via Amazon Bedrock, and get actionable insights without reading lengthy legal text.

### Key Design Principles

1. **Serverless-First**: AWS managed services eliminate infrastructure management
2. **Security by Design**: Encryption, least privilege access, data privacy
3. **AI-Powered**: Amazon Bedrock foundation models for intelligent analysis
4. **User-Centric**: Clear disclaimers (informational only, not legal advice)

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           USER / FRONTEND APPLICATION                        │
│                         (React/Vue/Angular Web App)                          │
└────────────┬────────────────────────────────────────────────┬────────────────┘
             │                                                 │
             │ HTTPS                                           │ HTTPS
             │                                                 │
┌────────────▼─────────────────────────────────────────────────▼────────────┐
│                          AMAZON API GATEWAY                                │
│                    (REST API with CORS & Rate Limiting)                    │
│                                                                            │
│  POST /upload          GET /results/{id}         DELETE /documents/{id}   │
└────┬───────────────────────────┬──────────────────────────┬───────────────┘
     │                           │                          │
     │                           │                          │
┌────▼────────────┐    ┌─────────▼──────────┐    ┌─────────▼──────────┐
│  AWS LAMBDA     │    │   AWS LAMBDA       │    │   AWS LAMBDA       │
│ Upload Handler  │    │ Results Handler    │    │ Deletion Handler   │
│                 │    │                    │    │                    │
│ • Validate file │    │ • Query DynamoDB   │    │ • Delete from S3   │
│ • Generate ID   │    │ • Return analysis  │    │ • Delete from DB   │
│ • Store in S3   │    │ • Add disclaimer   │    │ • Return success   │
│ • Trigger async │    │                    │    │                    │
└────┬────────────┘    └─────────┬──────────┘    └─────────┬──────────┘
     │                           │                          │
     │ Store                     │ Query                    │ Delete
     │                           │                          │
┌────▼───────────────────────────▼──────────────────────────▼────────────┐
│                           AMAZON S3 BUCKET                              │
│                    (Document Storage - AES-256 Encrypted)               │
│                                                                         │
│  • Stores uploaded PDF/TXT documents                                   │
│  • 30-day lifecycle policy (auto-deletion)                             │
│  • Bucket policy: Deny non-HTTPS requests                              │
└────┬────────────────────────────────────────────────────────────────────┘
     │
     │ Retrieve
     │
┌────▼─────────────────────────────────────────────────────────────────────┐
│                        AWS LAMBDA - ANALYSIS HANDLER                      │
│                      (Async Processing - 60s timeout)                     │
│                                                                           │
│  1. Retrieve document from S3                                            │
│  2. Extract text content                                                 │
│  3. Call Amazon Bedrock with prompt                                      │
│  4. Parse AI response (risk clauses JSON)                                │
│  5. Calculate risk score (weighted algorithm)                            │
│  6. Store results in DynamoDB                                            │
└────┬──────────────────────────────────────────┬─────────────────────────┘
     │                                           │
     │ Invoke Model                              │ Store Results
     │                                           │
┌────▼───────────────────────────────┐    ┌─────▼──────────────────────────┐
│      AMAZON BEDROCK                │    │      AMAZON DYNAMODB           │
│   (AI Text Analysis)               │    │   (Analysis Results Storage)   │
│                                    │    │                                │
│ • Model: Claude 3 Sonnet           │    │ • Table: AnalysisResults       │
│ • 200K token context               │    │ • Key: documentId (String)     │
│ • Prompt-based classification      │    │ • Attributes:                  │
│ • JSON output format               │    │   - riskScore (Number)         │
│ • No data retention                │    │   - riskLevel (String)         │
│                                    │    │   - riskClauses (List)         │
│ Returns:                           │    │   - categoryBreakdown (Map)    │
│ {                                  │    │   - timestamp (String)         │
│   "riskClauses": [                 │    │   - ttl (Number - 90 days)     │
│     {                              │    │                                │
│       "category": "privacy",       │    │ • On-demand capacity           │
│       "text": "...",               │    │ • Default encryption           │
│       "sentiment": "negative",     │    │                                │
│       "severity": "high"           │    │                                │
│     }                              │    │                                │
│   ]                                │    │                                │
│ }                                  │    │                                │
└────────────────────────────────────┘    └────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                          CROSS-CUTTING CONCERNS                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS IAM (Identity & Access Management)                                     │
│  • Lambda execution roles with least privilege                              │
│  • S3: GetObject, PutObject, DeleteObject                                   │
│  • DynamoDB: PutItem, GetItem, DeleteItem                                   │
│  • Bedrock: InvokeModel                                                     │
│  • CloudWatch: PutLogEvents                                                 │
│                                                                              │
│  AMAZON CLOUDWATCH (Monitoring & Logging)                                   │
│  • Lambda function logs (structured JSON)                                   │
│  • Metrics: Invocations, errors, duration                                   │
│  • Alarms: Error rate > 5%, Processing time > 60s                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘


═══════════════════════════════════════════════════════════════════════════════
                              DATA FLOW SEQUENCE
═══════════════════════════════════════════════════════════════════════════════

UPLOAD FLOW:
───────────
1. User → API Gateway: POST /upload {fileName, fileContent (base64), fileType}
2. API Gateway → Upload Handler Lambda
3. Upload Handler:
   a. Validate file (format: PDF/TXT, size < 10MB)
   b. Generate unique documentId (UUID)
   c. Store document in S3 bucket
   d. Trigger Analysis Handler (async invocation)
   e. Return response: {documentId, status: "processing"}
4. Upload Handler → User: 200 OK with documentId

ANALYSIS FLOW (Async):
──────────────────────
5. Analysis Handler Lambda triggered
6. Analysis Handler → S3: Retrieve document by documentId
7. Analysis Handler: Extract text from document
8. Analysis Handler → Bedrock: Invoke model with prompt + document text
9. Bedrock: Analyze text, identify risk clauses, classify categories
10. Bedrock → Analysis Handler: Return JSON with risk clauses
11. Analysis Handler:
    a. Parse Bedrock response
    b. Calculate risk score using weighted algorithm:
       - Privacy: 30%, Liability: 25%, Payment: 25%, Termination: 20%
       - Score = Σ(Category Weight × Category Score)
    c. Determine risk level: Low (0-33), Medium (34-66), High (67-100)
    d. Generate summary (top 3 risks)
12. Analysis Handler → DynamoDB: Store analysis results with TTL (90 days)

RETRIEVAL FLOW:
───────────────
13. User → API Gateway: GET /results/{documentId}
14. API Gateway → Results Handler Lambda
15. Results Handler → DynamoDB: Query by documentId
16. DynamoDB → Results Handler: Return analysis results
17. Results Handler:
    a. Add legal disclaimer
    b. Format response
18. Results Handler → User: 200 OK with analysis results

DELETION FLOW:
──────────────
19. User → API Gateway: DELETE /documents/{documentId}
20. API Gateway → Deletion Handler Lambda
21. Deletion Handler:
    a. Delete document from S3
    b. Delete results from DynamoDB
22. Deletion Handler → User: 200 OK {message: "deleted successfully"}


═══════════════════════════════════════════════════════════════════════════════
                           SECURITY & COMPLIANCE LAYERS
═══════════════════════════════════════════════════════════════════════════════

ENCRYPTION:
  • In Transit: TLS 1.2+ for all API calls
  • At Rest: S3 (AES-256), DynamoDB (default encryption)

ACCESS CONTROL:
  • API Gateway: Rate limiting (100 req/min per IP)
  • IAM: Least privilege roles for Lambda functions
  • S3: Bucket policy denies non-HTTPS requests

DATA PRIVACY:
  • No PII extraction or storage
  • No user authentication (anonymous usage)
  • Auto-deletion: Documents (30 days), Results (90 days)
  • Bedrock: No data retention or model training

```

**Key Architecture Principles**:
1. **Serverless**: All compute via Lambda (no server management)
2. **Event-Driven**: Async processing for AI analysis
3. **Scalable**: Auto-scaling for all services
4. **Secure**: Encryption, IAM, rate limiting
5. **Cost-Effective**: Pay-per-use pricing model

### AWS Services Mapping

| Service | Purpose | Key Configuration |
|---------|---------|-------------------|
| **API Gateway** | REST API endpoints | POST /upload, GET /results/{id}, DELETE /documents/{id} |
| **Lambda** | Backend processing | 4 functions: Upload, Analysis, Results, Deletion |
| **S3** | Document storage | AES-256 encryption, 30-day lifecycle deletion |
| **Bedrock** | AI text analysis | Claude 3 Sonnet model, no data retention |
| **DynamoDB** | Results storage | documentId as key, 90-day TTL |
| **IAM** | Access control | Least privilege roles for Lambda |
| **CloudWatch** | Monitoring | Logs, metrics, alarms for errors/latency |

## Components and Interfaces

### API Endpoints

**POST /upload**
- Input: `{fileName, fileContent (base64), fileType}`
- Output: `{documentId, status: "processing"}`
- Validation: File size < 10MB, format = PDF or TXT

**GET /results/{documentId}**
- Output: `{documentId, timestamp, riskScore, riskLevel, summary, riskClauses[], disclaimer}`
- Error: 404 if document not found

**DELETE /documents/{documentId}**
- Output: `{message: "deleted successfully"}`
- Removes document from S3 and results from DynamoDB

### Lambda Functions

1. **Upload Handler**: Validates file → Stores in S3 → Triggers Analysis Handler → Returns documentId
2. **Analysis Handler**: Retrieves document → Extracts text → Calls Bedrock → Calculates risk score → Stores in DynamoDB
3. **Results Handler**: Queries DynamoDB → Returns analysis results
4. **Deletion Handler**: Deletes from S3 and DynamoDB

### Amazon Bedrock Integration

**Model**: Claude 3 Sonnet (anthropic.claude-3-sonnet-20240229-v1:0)
- 200K token context window
- Strong legal text reasoning
- JSON output for structured parsing

**Prompt Structure**:
```
System: You are a legal document analyzer. Identify risk clauses in these categories:
- Privacy: data collection, sharing, retention
- Liability: limitation of liability, indemnification
- Payment: fees, renewals, price changes
- Termination: account closure, service discontinuation

For each clause: extract text, classify category, assess sentiment, rate severity.
Respond in JSON format.

User: Analyze this document: <document>{text}</document>
```

**Response Format**:
```json
{
  "riskClauses": [
    {
      "category": "privacy|liability|payment|termination",
      "text": "exact clause text",
      "sentiment": "positive|neutral|negative",
      "severity": "low|medium|high"
    }
  ]
}
```

## Data Models

### DynamoDB Schema

**Table**: `AnalysisResults`
**Key**: `documentId` (String)

```json
{
  "documentId": "uuid",
  "timestamp": "ISO-8601",
  "riskScore": 75,
  "riskLevel": "High Risk | Medium Risk | Low Risk",
  "summary": "Top 3 risks description",
  "riskClauses": [
    {
      "category": "privacy|liability|payment|termination",
      "text": "clause text",
      "sentiment": "positive|neutral|negative",
      "severity": "low|medium|high"
    }
  ],
  "categoryBreakdown": {
    "privacy": 30,
    "liability": 25,
    "payment": 15,
    "termination": 5
  },
  "ttl": 1234567890
}
```

### Risk Score Calculation

```
Risk Score = Σ (Category Weight × Category Score)

Weights:
- Privacy: 30%
- Liability: 25%
- Payment: 25%
- Termination: 20%

Category Score = (High × 100 + Medium × 50 + Low × 25) / 300

Risk Levels:
- 0-33: Low Risk
- 34-66: Medium Risk
- 67-100: High Risk
```

## Data Flow

### Document Upload and Analysis

1. User uploads document → API Gateway POST /upload
2. Upload Handler validates → Stores in S3 → Returns documentId
3. Upload Handler triggers Analysis Handler asynchronously
4. Analysis Handler retrieves document → Extracts text → Calls Bedrock
5. Bedrock analyzes text → Returns risk clauses JSON
6. Analysis Handler calculates risk score → Stores in DynamoDB
7. User polls GET /results/{documentId} → Results Handler returns analysis

### Key Workflows

**Upload**: Validate → S3 → Async trigger → Return ID
**Analysis**: S3 → Extract text → Bedrock → Calculate score → DynamoDB
**Retrieval**: DynamoDB query → Return results with disclaimer
**Deletion**: Remove from S3 and DynamoDB

## AI Design

### Model Selection
**Claude 3 Sonnet** (anthropic.claude-3-sonnet-20240229-v1:0)
- 200K token context (handles lengthy documents)
- Strong legal reasoning capabilities
- Reliable JSON output
- No data retention (privacy-preserving)

### Risk Classification Strategy

**Category Keywords**:
- **Privacy**: data, information, share, third party, collect, retain
- **Liability**: liable, indemnify, warranty, disclaimer, limitation
- **Payment**: fee, charge, billing, renewal, price, refund
- **Termination**: terminate, suspend, cancel, close account

**Sentiment**: Negative (favors provider), Neutral (balanced), Positive (protects user)
**Severity**: High (significant risk), Medium (moderate concern), Low (minor issue)

### Risk Scoring Algorithm

```python
def calculate_risk_score(risk_clauses):
    category_counts = count_by_severity(risk_clauses)
    
    category_scores = {}
    for category in ['privacy', 'liability', 'payment', 'termination']:
        high = category_counts[category]['high']
        medium = category_counts[category]['medium']
        low = category_counts[category]['low']
        
        raw_score = (high * 100) + (medium * 50) + (low * 25)
        category_scores[category] = min(100, raw_score / 300 * 100)
    
    weights = {'privacy': 0.30, 'liability': 0.25, 'payment': 0.25, 'termination': 0.20}
    risk_score = sum(category_scores[cat] * weights[cat] for cat in weights)
    
    return round(risk_score, 2)
```

## Security Considerations

### Encryption
- **At Rest**: S3 (AES-256), DynamoDB (default encryption)
- **In Transit**: TLS 1.2+ for all API calls

### Access Control
**Lambda IAM Role**:
- S3: GetObject, PutObject, DeleteObject
- DynamoDB: PutItem, GetItem, DeleteItem
- Bedrock: InvokeModel
- CloudWatch: Logs

**S3 Bucket Policy**: Deny non-HTTPS requests

### API Security
- Rate limiting: 100 requests/minute per IP
- Input validation: File size, format, base64 encoding
- CORS: Restrict to specific origins

### Data Privacy
- No PII extraction from documents
- No user authentication (anonymous)
- Auto-deletion: Documents (30 days), Results (90 days)
- Bedrock: No data retention or model training

## Scalability and Reliability

### Auto-Scaling
- **Lambda**: 1000 concurrent executions (default), reserved concurrency for Analysis Handler
- **DynamoDB**: On-demand capacity (auto-scales to 40K requests/sec)
- **S3**: Unlimited scalability (3,500 PUT/sec, 5,500 GET/sec per prefix)

### Fault Tolerance
- **Retry Logic**: 3 attempts with exponential backoff for Bedrock and DynamoDB
- **High Availability**: Multi-AZ deployment (default for all services)
- **Error Handling**: Graceful degradation, user-friendly error messages

### Performance
- **Lambda**: 1024 MB memory, 60s timeout for Analysis Handler
- **Bedrock**: Claude 3 Haiku for docs < 2MB (faster), Sonnet for > 2MB (better accuracy)
- **Caching**: API Gateway response cache (5 min) for GET /results

## Error Handling

### Error Categories
- **4xx Client Errors**: Invalid format (400), Not found (404), Rate limit (429)
- **5xx Server Errors**: Lambda error (500), Bedrock unavailable (502), Service unavailable (503), Timeout (504)

### Error Response Format
```json
{
  "error": "Human-readable message",
  "code": "ERROR_CODE",
  "timestamp": "ISO-8601",
  "requestId": "uuid"
}
```

### Logging
- **CloudWatch Logs**: Structured JSON logging (INFO/ERROR levels)
- **Metrics**: Processing time, success/failure counts
- **Alarms**: Error rate > 5% (5 min), Processing time > 60s

## Testing Strategy

### Unit Testing
- Test Lambda function logic with mocked AWS services (pytest with moto)
- Test risk score calculation with known inputs
- Test input validation (file format, size)
- Test error handling for service failures

### Property-Based Testing
- Library: Hypothesis (Python) or fast-check (JavaScript/TypeScript)
- Minimum 100 iterations per property test
- Tag format: **Feature: tc-risk-analyzer, Property {N}: {property title}**
- Generators: Random documents, analysis results, document IDs, edge cases

### Integration Testing
- End-to-end flows with AWS services (or LocalStack)
- S3 upload → Lambda → Bedrock → DynamoDB workflow
- API Gateway → Lambda integration
- Verify encryption, IAM permissions, lifecycle policies

### Infrastructure Testing
- Validate AWS resource configuration (IAM, S3, API Gateway)
- Check CloudWatch alarms and metrics
- Use Infrastructure as Code (CloudFormation/CDK)


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Core Properties

**Property 1: Valid file format acceptance**
*For any* document with valid format (PDF/TXT) and size < 10MB, the system should accept upload and return unique documentId.
**Validates: Requirements 2.1, 2.2, 2.7**

**Property 2: File size validation**
*For any* document, if size < 10MB accept, if ≥ 10MB reject with error.
**Validates: Requirements 2.3, 2.5**

**Property 3: Unsupported format rejection**
*For any* document with unsupported format, reject with error message.
**Validates: Requirements 2.6**

**Property 4: Document persistence**
*For any* uploaded document, it should be retrievable from S3 using its documentId.
**Validates: Requirements 2.4**

**Property 5: Unique document identifiers**
*For any* set of uploaded documents, each receives a unique identifier.
**Validates: Requirements 2.7**

**Property 6: Analysis trigger**
*For any* uploaded document, system should automatically trigger AI analysis.
**Validates: Requirements 3.1**

**Property 7: Risk clause sentiment**
*For any* identified risk clause, it should have sentiment (positive/neutral/negative).
**Validates: Requirements 3.6**

**Property 8: Risk clause text extraction**
*For any* identified risk clause, it should contain non-empty text from document.
**Validates: Requirements 3.7**

**Property 9: Single category assignment**
*For any* risk clause, it should be assigned to exactly one category.
**Validates: Requirements 4.1**

**Property 10: Category grouping**
*For any* analysis result with multiple clauses, clauses should be grouped by category.
**Validates: Requirements 4.6**

**Property 11: Risk score bounds**
*For any* completed analysis, risk score should be between 0-100 (inclusive).
**Validates: Requirements 5.1**

**Property 12: Summary generation**
*For any* analysis result with risks, summary should reference top 3 significant clauses.
**Validates: Requirements 5.6**

**Property 13: Analysis result persistence**
*For any* completed analysis, result should be stored in DynamoDB and retrievable by documentId.
**Validates: Requirements 6.1, 6.2**

**Property 14: Result timestamp**
*For any* stored analysis result, it should include timestamp of analysis.
**Validates: Requirements 6.3**

**Property 15: Non-existent document error**
*For any* request with non-existent documentId, return not found error.
**Validates: Requirements 6.5**

**Property 16: PII exclusion**
*For any* document with PII, stored analysis results should not include that PII.
**Validates: Requirements 7.7**

**Property 17: Complete document deletion**
*For any* deletion request, remove both document (S3) and results (DynamoDB).
**Validates: Requirements 9.3, 9.4**

**Property 18: HTTP status code correctness**
*For any* API request: success → 200, client error → 4xx, server error → 5xx.
**Validates: Requirements 10.5, 10.6, 10.7**

**Property 19: Result disclaimer inclusion**
*For any* analysis result, include legal professional consultation notice.
**Validates: Requirements 11.3**

**Property 20: Risk score explanation**
*For any* response with risk score, include explanation that it's automated assessment.
**Validates: Requirements 11.6**

**Property 21: Error logging**
*For any* unexpected error, log details to CloudWatch.
**Validates: Requirements 12.5**

**Property 22: Error message sanitization**
*For any* error response, don't expose system internals or sensitive info.
**Validates: Requirements 12.6**

### Example-Based Tests

**Risk Identification Examples** (Requirements 3.2-3.5):
- Privacy: "We may share your data with third parties"
- Liability: "We are not liable for any damages"
- Payment: "Subscription automatically renews"
- Termination: "We may terminate your account without notice"

**Scoring Examples** (Requirements 5.2-5.5):
- Privacy weight: 30%, Liability: 25%, Payment: 25%, Termination: 20%

**Classification Examples** (Requirements 5.7-5.9):
- Score 0-33: "Low Risk", 34-66: "Medium Risk", 67-100: "High Risk"

**Infrastructure Examples** (Requirements 7.1-7.5, 9.1-9.2, 10.1-10.8, 13.1-13.6):
- S3 encryption (AES-256), TLS 1.2+, IAM roles
- Lifecycle: S3 (30 days), DynamoDB TTL (90 days)
- API endpoints, rate limiting (100 req/min)
- CloudWatch logs, metrics, alarms

**Error Handling Examples** (Requirements 12.1-12.4):
- Retry logic: 3 attempts with exponential backoff
- Service unavailability error messages

## Limitations and Future Enhancements

### Current Limitations
- English-only, PDF/TXT formats only, 10MB max size
- No user authentication, document comparison, or historical tracking
- Fixed risk categories, no batch processing

### Key Future Enhancements
- User authentication, multi-language support, additional file formats
- Document comparison, historical analysis, custom risk categories
- Batch processing, analytics dashboard, compliance checking (GDPR, CCPA)
