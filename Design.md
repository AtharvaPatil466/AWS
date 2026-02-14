# System Design: Adaptive Learning Platform
## Technical Architecture & Implementation

**Version**: 1.0  
**Date**: February 15, 2026  
**Audience**: AWS Hackathon Technical Judges

---

## Architecture Highlights

🏗️ **Serverless-First**: Lambda + SageMaker + Bedrock for zero-ops ML  
📊 **Hierarchical ML**: GNN → CQL → PEARL → Bedrock pipeline  
⚡ **Real-Time Inference**: <500ms end-to-end recommendation latency  
🔄 **Event-Driven**: Asynchronous processing for resilience  
📈 **Auto-Scaling**: Handles 100+ concurrent users seamlessly  
🛡️ **Production-Ready**: Circuit breakers, retries, graceful degradation

---

## Table of Contents

1. [Design Overview](#design-overview)
2. [System Architecture](#system-architecture)
3. [ML Architecture](#ml-architecture)
4. [API Design](#api-design)
5. [Data Architecture](#data-architecture)
6. [AWS Infrastructure](#aws-infrastructure)
7. [Security Architecture](#security-architecture)
8. [Scalability & Performance](#scalability--performance)
9. [Error Handling & Resilience](#error-handling--resilience)
10. [Monitoring & Observability](#monitoring--observability)
11. [Key Design Decisions](#key-design-decisions)
12. [Implementation Roadmap](#implementation-roadmap)

---

## Design Overview

### Core Design Principles

The platform is built on five foundational design principles that guide all architectural decisions:

#### 1. Hierarchical ML Architecture

- **Layer 1 (Foundation)**: GNN models domain knowledge structure
- **Layer 2 (Safety)**: CQL learns safe recommendation policies
- **Layer 3 (Personalization)**: PEARL enables rapid student adaptation
- **Layer 4 (Generation)**: AWS Bedrock creates dynamic content
- **Rationale**: Each layer builds on the previous, enabling modularity and interpretability

#### 2. Serverless-First Architecture

- Use managed services (Lambda, SageMaker, Bedrock, DynamoDB)
- Zero infrastructure management overhead
- Built-in auto-scaling and high availability
- **Rationale**: Focus engineering effort on ML innovation, not ops

#### 3. Event-Driven & Asynchronous

- Long-running tasks (training) decoupled from API layer
- Non-blocking recommendations for better UX
- Event triggers for retraining and analytics
- **Rationale**: Better user experience and system resilience

#### 4. Data-Centric Design

- Log every interaction for continual learning
- Immutable interaction logs for reproducibility
- Versioned model artifacts in S3
- **Rationale**: Enable offline RL, debugging, and model improvement

#### 5. Fail-Fast with Graceful Degradation

- Validate inputs early at API Gateway
- Fallback hierarchy: PEARL → CQL → Heuristic
- Circuit breakers prevent cascading failures
- **Rationale**: Ensure demo reliability and build user trust

---

## System Architecture

### High-Level Architecture

The system follows a layered architecture with clear separation of concerns:

#### Architecture Layers

1. **Presentation Layer**: React frontend with D3.js visualizations
2. **API Gateway Layer**: REST API with authentication, rate limiting, CORS
3. **Application Layer**: Lambda functions orchestrating business logic
4. **ML Inference Layer**: SageMaker endpoints + AWS Bedrock
5. **Data Layer**: DynamoDB (state), Neptune (graph), S3 (artifacts)

### Key Components

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Frontend | React + TypeScript + MUI | Student dashboard & visualizations |
| API Gateway | AWS API Gateway | REST endpoints, auth, rate limiting |
| Orchestration | AWS Lambda | Serverless business logic |
| ML Models | SageMaker Endpoints | GNN, PEARL, CQL inference |
| Content Gen | AWS Bedrock (Claude) | Personalized explanations |
| Student Data | DynamoDB | User state & interactions |
| Knowledge Graph | Neptune | Concept relationships |
| Storage | S3 | Models, logs, content library |
| Monitoring | CloudWatch | Metrics, logs, alarms |

### Data Flow

Request flow for a content recommendation:

1. Student clicks 'Get Recommendation' in React frontend
2. API Gateway validates request, checks rate limits
3. Lambda function retrieves student state from DynamoDB
4. Lambda calls SageMaker endpoints in sequence: GNN → PEARL → CQL
5. Lambda validates recommendation meets safety constraints
6. Lambda calls Bedrock to generate personalized explanation
7. Lambda logs interaction to S3 for offline learning
8. Response returned to frontend (<500ms total latency)

---

## ML Architecture

### Model Pipeline

The ML architecture implements a hierarchical inference pipeline:

#### Inference Flow

1. **Step 1**: GNN encodes student knowledge state as graph embeddings
2. **Step 2**: PEARL adapts policy based on student context (5-10 interactions)
3. **Step 3**: CQL validates recommendation is within safe policy bounds
4. **Step 4**: Causal inference validates expected learning gain
5. **Step 5**: Bedrock generates personalized content (explanations, hints)

### Model Deployment

| Model | Instance Type | Purpose | Latency Target |
|-------|--------------|---------|----------------|
| GNN | ml.m5.xlarge | Knowledge state encoding | <100ms |
| PEARL | ml.c5.2xlarge | Personalized policy | <150ms |
| CQL | ml.m5.large | Safe policy validation | <100ms |
| Bedrock | Serverless | Content generation | <2000ms |

### SageMaker Configuration

Each model is deployed as a real-time SageMaker endpoint:

- **Container**: Custom PyTorch containers with model artifacts
- **Auto-scaling**: Target 70% CPU utilization (1-5 instances)
- **Health checks**: /ping endpoint every 60 seconds
- **Model data**: Downloaded from S3 on container start
- **Inference format**: JSON input/output for easy integration

### Fallback Strategy

Robust error handling ensures the system never fails silently:

#### Graceful Degradation

- If **PEARL fails** → Fall back to CQL base policy (safe recommendations)
- If **CQL fails** → Fall back to simple heuristic (recommend easier content)
- If **Bedrock fails** → Return recommendation without personalized explanation
- If **all ML fails** → Return last successful recommendation from cache

---

## API Design

### REST Endpoints

The API exposes five core endpoints via API Gateway:

| Method | Endpoint | Purpose | Lambda Handler |
|--------|----------|---------|----------------|
| POST | /students/{id}/recommend | Get content recommendation | recommendation-handler |
| POST | /students/{id}/interact | Log student interaction | state-update-handler |
| GET | /students/{id}/state | Get student knowledge state | state-update-handler |
| POST | /content/generate | Generate personalized content | content-gen-handler |
| GET | /knowledge-graph | Get knowledge graph data | analytics-handler |

### Request/Response Format

Example recommendation request:

```json
POST /students/12345/recommend
{
  "context": {
    "time_of_day": "morning",
    "session_number": 3
  }
}
```

Response:

```json
{
  "content_id": "algebra_101",
  "difficulty": 0.6,
  "predicted_gain": 0.15,
  "explanation": "Based on your progress...",
  "reasoning": "Causal analysis shows..."
}
```

### API Gateway Configuration

- **Authentication**: API Key (X-API-Key header)
- **Rate Limiting**: 100 requests/minute per API key
- **CORS**: Enabled for frontend domain
- **Request Validation**: JSON schema validation on inputs
- **Throttling**: Burst limit 200, steady rate 100/sec

---

## Data Architecture

### DynamoDB Tables

Three core tables for transactional data:

| Table | Partition Key | Sort Key | Purpose |
|-------|--------------|----------|---------|
| StudentStates | student_id | - | Current knowledge state |
| Interactions | student_id | timestamp | Interaction history |
| ContentLibrary | content_id | - | Available content metadata |

#### StudentStates Schema

Stores current knowledge state for each student:

```json
{
  "student_id": "12345",
  "knowledge_vector": [0.3, 0.7, 0.5, ...],
  "learning_velocity": [0.1, 0.05, 0.2, ...],
  "pearl_context": {...},
  "last_updated": "2026-02-15T10:30:00Z"
}
```

### Neptune Graph

Knowledge graph models concept relationships:

- **Nodes**: Concepts (e.g., 'Linear Equations', 'Quadratic Formula')
- **Edges**: Prerequisites (e.g., 'requires', 'builds_on')
- **Properties**: Difficulty, estimated_time, content_ids
- **Queries**: Shortest path, prerequisite chains, related concepts

### S3 Organization

Hierarchical storage for models and logs:

```
s3://adaptive-learning-platform/
  models/
    gnn/v1.0/model.tar.gz
    pearl/v2.1/model.tar.gz
    cql/v1.5/model.tar.gz
  logs/
    interactions/2026/02/15/
    training/2026/02/
  content/
    videos/
    exercises/
```

---

## AWS Infrastructure

### Service Integration

Deep integration with 10+ AWS services:

| Service | Usage | Configuration |
|---------|-------|--------------|
| SageMaker | ML training & hosting | 3 endpoints, auto-scaling |
| Bedrock | Content generation | Claude 3.5 Sonnet, on-demand |
| Lambda | API handlers | Python 3.11, 512MB RAM, 30s timeout |
| API Gateway | REST API | Regional, API key auth |
| DynamoDB | State storage | On-demand capacity |
| Neptune | Knowledge graph | db.t3.medium instance |
| S3 | Artifact storage | Standard tier, versioning enabled |
| CloudWatch | Monitoring | Logs, metrics, dashboards, alarms |
| IAM | Access control | Least-privilege roles |
| CloudFormation | IaC | Stack for entire infrastructure |

### Deployment Architecture

Infrastructure as Code using CloudFormation:

- Single CloudFormation stack deploys entire platform
- Nested stacks for ML, API, and data layers
- Parameter store for configuration management
- Blue-green deployment for zero-downtime updates
- Rollback capability for failed deployments

### Cost Optimization

Strategies to keep costs under $50 for demo:

- **SageMaker**: Use smaller instances (ml.m5.large ~$0.115/hr)
- **Lambda**: Leverage free tier (1M requests/month)
- **DynamoDB**: On-demand pricing for low traffic
- **Neptune**: Use smallest instance (db.t3.medium ~$0.092/hr)
- **S3**: Minimal storage (<10GB), standard tier
- **Bedrock**: Optimize prompts, use caching where possible

---

## Security Architecture

### Security Layers

Multi-layered security approach:

#### Layer 1: Network Security

- API Gateway enforces HTTPS-only connections
- VPC for SageMaker endpoints (private subnets)
- Security groups restrict traffic to necessary ports
- No direct internet access to ML infrastructure

#### Layer 2: Authentication & Authorization

- **API Gateway**: API key authentication for all endpoints
- **IAM roles**: Least-privilege access for Lambda functions
- **SageMaker**: Endpoint access restricted to specific IAM roles
- **DynamoDB**: Fine-grained access control via IAM policies

#### Layer 3: Data Security

- **Encryption at rest**: All DynamoDB tables, S3 buckets encrypted (AES-256)
- **Encryption in transit**: TLS 1.2+ for all service communication
- **S3 bucket policies**: Block public access by default
- **CloudWatch Logs**: Encrypted with KMS keys

### IAM Role Design

Separate roles for each Lambda function:

- `recommendation-handler-role`: Read DynamoDB, invoke SageMaker
- `state-update-handler-role`: Read/write DynamoDB, write S3
- `content-gen-handler-role`: Invoke Bedrock, read DynamoDB
- `sagemaker-execution-role`: Read S3 models, write CloudWatch

---

## Scalability & Performance

### Performance Targets

| Metric | Target | Strategy |
|--------|--------|----------|
| API Latency (p95) | <500ms | Parallel ML calls, caching |
| Concurrent Users | 100+ | Lambda auto-scaling, DynamoDB on-demand |
| Recommendations/sec | 20+ | SageMaker endpoint auto-scaling |
| Data Throughput | 1000 writes/sec | DynamoDB adaptive capacity |
| ML Inference | <300ms | Optimized models, batch inference |

### Scalability Mechanisms

#### Lambda Auto-Scaling

- **Concurrent executions**: Up to 1000 (default regional limit)
- **Cold start mitigation**: Provisioned concurrency for critical functions
- **Timeout**: 30 seconds to allow for ML inference
- **Memory**: 512MB optimized for Python ML libraries

#### SageMaker Auto-Scaling

- **Target metric**: 70% CPU utilization
- **Min instances**: 1 per endpoint (ensure availability)
- **Max instances**: 5 per endpoint (cost control)
- **Scale-up**: Add instance if CPU > 70% for 3 minutes
- **Scale-down**: Remove instance if CPU < 50% for 15 minutes

### Caching Strategy

- **API Gateway**: Cache GET requests for 60 seconds
- **Lambda**: In-memory cache for frequently accessed data
- **Frontend**: Browser cache for static content
- **Recommendations**: Cache last 10 recommendations per student

---

## Error Handling & Resilience

### Resilience Patterns

#### 1. Circuit Breaker Pattern

- Prevents cascading failures from overloading failing endpoints
- If 5 consecutive SageMaker errors, circuit opens for 60 seconds
- Requests fast-fail instead of waiting for timeout
- Automatic recovery when endpoint healthy again

#### 2. Retry with Exponential Backoff

- Transient failures automatically retried (network issues, throttling)
- Wait times: 1s, 2s, 4s, 8s (up to 3 retries)
- Jitter added to prevent thundering herd
- Non-retryable errors (4xx) fail immediately

#### 3. Graceful Degradation

- If PEARL unavailable → Use CQL base policy
- If CQL unavailable → Use simple heuristic
- If Bedrock unavailable → Skip personalized explanation
- System remains functional even with partial failures

### Error Response Codes

| Code | Meaning | Action |
|------|---------|--------|
| 200 | Success | Return result |
| 400 | Bad Request | Validate input, return error |
| 429 | Rate Limit | Throttle client, retry later |
| 500 | Internal Error | Log, retry if transient |
| 503 | Service Unavailable | Use fallback, alert team |

---

## Monitoring & Observability

### CloudWatch Dashboards

Three primary dashboards for system visibility:

#### Dashboard 1: API Health

- API Gateway request count (time series)
- 4xx/5xx error rates (%)
- Latency percentiles (p50, p95, p99)
- Lambda invocations by function
- Lambda errors and durations

#### Dashboard 2: ML Performance

- SageMaker endpoint invocations
- Endpoint latency (p50, p95)
- Model inference errors
- Learning gain (custom metric)
- PEARL adaptation speed (custom metric)

#### Dashboard 3: Resource Utilization

- DynamoDB read/write capacity
- Lambda concurrent executions
- SageMaker instance CPU/memory
- S3 storage used
- Estimated daily cost

### Alarms & Alerts

| Severity | Condition | Action |
|----------|-----------|--------|
| Critical | API error rate > 10% | PagerDuty/SMS |
| Critical | SageMaker health check fail | PagerDuty/SMS |
| Warning | Latency p95 > 500ms | Email |
| Warning | Daily cost > $50 | Email |
| Info | New deployment complete | Slack |

---

## Key Design Decisions

### Decision 1: Why Lambda over EC2?

Rationale for serverless architecture:

✅ Zero server management, focus on code  
✅ Automatic scaling (0 to 1000+ concurrent executions)  
✅ Pay-per-request pricing (cost-efficient for demo)  
✅ Built-in high availability across AZs  
❌ **Trade-off**: Cold starts (~1s for Python ML libraries)  
**Mitigation**: Provisioned concurrency for critical functions

### Decision 2: Why DynamoDB over RDS?

Rationale for NoSQL choice:

✅ Simpler schema for key-value lookups (student state)  
✅ On-demand capacity eliminates capacity planning  
✅ Sub-10ms latency for reads/writes  
✅ Native AWS integration (IAM, CloudWatch)  
❌ **Trade-off**: Limited query flexibility vs SQL  
**Mitigation**: Use Neptune for complex graph queries

### Decision 3: Why Neptune over Neo4j?

Rationale for managed graph database:

✅ Fully managed (backups, patching, monitoring)  
✅ Native AWS integration (VPC, IAM, CloudWatch)  
✅ Gremlin support for graph traversals  
✅ High availability with read replicas  
❌ **Trade-off**: Higher cost than self-managed Neo4j  
**Justification**: Simplifies operations for hackathon demo

### Decision 4: Real-Time vs Batch Inference?

Why SageMaker real-time endpoints:

✅ Required for interactive demo (<500ms latency)  
✅ Supports personalization (PEARL needs student context)  
✅ Simpler architecture vs batch + scheduling  
❌ **Trade-off**: Higher cost than batch inference  
**Future**: Consider batch for non-interactive recommendations

---

## Implementation Roadmap

### Phase 1: Infrastructure Setup (Day 1)

- Create AWS account, configure IAM users/roles
- Deploy CloudFormation stack (VPC, S3, DynamoDB)
- Set up Neptune cluster, load knowledge graph
- Configure CloudWatch dashboards and alarms

### Phase 2: ML Development (Days 2-4)

- Train GNN on synthetic knowledge graph data
- Train CQL on offline student trajectories
- Meta-train PEARL across diverse student personas
- Deploy models to SageMaker endpoints
- Test inference latency and accuracy

### Phase 3: API Development (Days 5-6)

- Implement Lambda functions (Python)
- Set up API Gateway routes and validation
- Integrate with SageMaker and Bedrock
- Add error handling and retry logic
- Test end-to-end API flow

### Phase 4: Frontend Development (Days 7-8)

- Build React components (Dashboard, Knowledge Graph)
- Implement D3.js visualizations
- Connect to API via Axios
- Add loading states and error handling
- Deploy to S3 + CloudFront

### Phase 5: Testing & Demo Prep (Days 9-10)

- End-to-end testing with realistic scenarios
- Load testing (100+ concurrent users)
- Prepare demo script and walkthrough
- Create backup demo data
- Final rehearsal and bug fixes

---

## Conclusion

This design document presents a comprehensive, production-ready architecture for the Adaptive Learning Platform. The system demonstrates:

### Key Strengths

✅ **Modular**: Clear separation of concerns across ML layers  
✅ **Scalable**: Serverless architecture handles 100+ concurrent users  
✅ **Resilient**: Circuit breakers, retries, graceful degradation  
✅ **Observable**: Comprehensive monitoring via CloudWatch  
✅ **Secure**: Multi-layered security (network, IAM, encryption)  
✅ **AWS-Native**: Deep integration with 10+ AWS services  
✅ **Cost-Efficient**: Optimized for <$50 demo budget  
✅ **Production-Ready**: IaC, CI/CD, automated deployment

### Competitive Advantages

What sets this design apart:

- Novel hierarchical ML architecture (GNN → CQL → PEARL → Bedrock)
- Real-time personalization in 5-10 interactions (vs 50+ typical)
- Safety-first approach using conservative offline RL
- Explainable recommendations via causal inference
- Fully serverless, zero-ops deployment
- Enterprise-grade resilience and monitoring

### Next Steps

With this design approved, the team will:

- Execute the 10-day implementation roadmap
- Conduct daily standups to track progress
- Perform continuous testing and iteration
- Prepare comprehensive demo walkthrough
- Document lessons learned for future improvements

---

## Appendix

### A. AWS Service Limits

| Service | Default Limit | Our Usage | Request Increase? |
|---------|--------------|-----------|-------------------|
| Lambda Concurrent Exec | 1000 | ~50 | No |
| SageMaker Endpoints | 20 | 3 | No |
| DynamoDB Tables | 2500 | 3 | No |
| API Gateway Requests | 10000/sec | ~100/sec | No |
| S3 Buckets | 100 | 1 | No |

### B. Estimated Costs (Hackathon Demo)

| Service | Configuration | Duration | Cost |
|---------|--------------|----------|------|
| SageMaker (3 endpoints) | ml.m5.large + ml.c5.2xlarge + ml.m5.xlarge | 48 hours | ~$25 |
| Neptune | db.t3.medium | 48 hours | ~$4.50 |
| Lambda | 100,000 requests | 48 hours | Free tier |
| DynamoDB | On-demand, low traffic | 48 hours | ~$1 |
| S3 | 10GB storage | 48 hours | <$1 |
| Bedrock | ~500 requests | 48 hours | ~$5 |
| API Gateway | 50,000 requests | 48 hours | Free tier |
| CloudWatch | Standard metrics | 48 hours | Free tier |

**Total Estimated Cost**: $36.50 (well under $50 budget)

### C. Technology Stack Summary

| Layer | Technologies |
|-------|-------------|
| Frontend | React, TypeScript, Material-UI, D3.js, Redux |
| API | API Gateway, Lambda (Python 3.11) |
| ML Frameworks | PyTorch, scikit-learn, NetworkX |
| ML Services | SageMaker, Bedrock (Claude 3.5) |
| Data Storage | DynamoDB, Neptune, S3 |
| Monitoring | CloudWatch Logs, Metrics, Dashboards, Alarms |
| Infrastructure | CloudFormation, IAM |
| CI/CD | GitHub Actions (optional) |

---

## Ready for Implementation

This design has been reviewed and approved for implementation.  
All architectural decisions are justified and documented.  
The system is optimized for the AWS Hackathon judging criteria.

**Let's build something amazing!** 🚀

---

*End of Design Document*
