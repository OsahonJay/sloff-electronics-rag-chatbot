
# Sloff Electronics RAG Chatbot — AWS Bedrock

> A Retrieval-Augmented Generation (RAG) chatbot built on Amazon Bedrock, using a Bedrock Knowledge Base, OpenSearch Serverless vector store, Titan Embeddings G1, and Claude Sonnet 4.6 as the foundation model.

**CloudBoosta Academy — Group B, Option 1**

*Technical lead: Osahon Igbogbo. Built in collaboration with Emmanuel Onuoha, Bolarinwa Akinde, Chinedu Izuelu, and Israel Olayinka.*

---

## Overview

This project deploys a RAG chatbot for Sloff Electronics — a fictional UK electronics retailer. The chatbot answers natural language customer queries by retrieving answers from proprietary company documents stored in Amazon S3, rather than relying on the LLM's training data.

**Result:** 5/5 test queries answered accurately with source citations and zero hallucinations.

> ⚠️ **Read the [Production Gaps](#production-gaps--improvements) section before deploying this in any real environment.** Three gaps were identified in a review after presentation that affect security and reliability.

---

## Architecture

```
INGESTION PHASE — runs once when documents are uploaded
──────────────────────────────────────────────────────────
Amazon S3  ──Read Docs──►  Titan Embeddings G1  ──Store Vectors──►  OpenSearch Serverless
(5 PDFs)                   (text → 1536-dim vectors)                (vector store)

IAM Service Role ──S3 Read──► S3
IAM Service Role ──OpenSearch Write──► OpenSearch Serverless
⚠️  IAM Service Role does NOT control Claude — see notes below

QUERY PHASE — runs every time a user asks a question
──────────────────────────────────────────────────────────
User ──Query──► Bedrock KB ──Vector Search──► OpenSearch ──Retrieve Chunks──► Claude Sonnet 4.6
                                                                                      │
                         Response + citation [1] ◄────────────────────────────────────┘
```

### IAM Role vs Claude Access

| Access mechanism | Controls |
|---|---|
| IAM service role (auto-generated) | Bedrock's permission to read S3 and write to OpenSearch only |
| IAM user credentials | Calling the Bedrock InvokeModel API to invoke Claude |
| Anthropic use case approval | One-time account-level gate to enable Claude — separate from IAM |

### AWS Services

| Service | Purpose | Config |
|---|---|---|
| Amazon S3 | Source document storage | `sloff-electronics-kb-docs` · `us-east-1` |
| Amazon Bedrock KB | RAG orchestration | `sloff-electronics-kb` · vector store type |
| Titan Embeddings G1 | Text → vector conversion | 1536 dimensions |
| OpenSearch Serverless | Vector database | Auto-provisioned by Bedrock |
| Claude Sonnet 4.6 | Response generation | Anthropic via Bedrock InvokeModel API |
| AWS IAM Service Role | S3 read + OpenSearch write | Auto-generated · scoped only |

---

## Knowledge Base Documents

| File | Contents |
|---|---|
| `sloff_product_catalogue.pdf` | Laptops, smartphones, accessories — specs and UK pricing |
| `sloff_returns_policy.pdf` | Return windows by category, eligibility, step-by-step process |
| `sloff_warranty_policy.pdf` | Coverage, exclusions, SloffCare+ extended warranty, claims |
| `sloff_customer_faq.pdf` | Troubleshooting guides, order tracking, account support |
| `sloff_shipping_policy.pdf` | UK, EU, international delivery options, costs, cut-off times |

---

## How to Replicate

### Prerequisites

- AWS account — **use an IAM user, not root** (Bedrock KB blocks root user operations)
- Region: `us-east-1` (required for full Bedrock + OpenSearch Serverless support)
- Anthropic model access submitted via Bedrock Model Catalog **before** build day

### Step 1 — IAM Setup

```
IAM → Users → Create user
Username: <your-admin-username>
Policy: AdministratorAccess
Enable console access: yes
```

> ⚠️ Sign out of root and sign in as this IAM user before proceeding.

### Step 2 — S3 Bucket

```
S3 → Create bucket
Name: sloff-electronics-kb-docs
Region: us-east-1
Block all public access: ON
Versioning: Enable (do this before uploading any documents)
```

Upload all 5 PDF documents to the bucket.

### Step 3 — Anthropic Model Access

```
Bedrock → Model Catalog → Submit use case details (yellow banner)
```

> ⚠️ Claude is an Anthropic model. AWS does not control access to it. Submit this form before building anything else.

### Step 4 — Create Bedrock Knowledge Base

```
Bedrock → Build → Knowledge Bases → Create
→ Knowledge Base with vector store

Name: sloff-electronics-kb
IAM: Create and use a new service role (auto-generated)
Data source: Amazon S3 → s3://<your-bucket-name>
Embedding model: Amazon Titan Embeddings G1 - Text
Vector store: Quick create new vector store
```

Wait 3–5 minutes for OpenSearch Serverless provisioning.

### Step 5 — Sync Documents

```
Knowledge Base → Data Sources → select source → Sync
```

> ⚠️ Creation does not equal ingestion. Sync must be explicitly triggered.

### Step 6 — Test

```
Knowledge Base → Test Knowledge Base
Model: Claude Sonnet 4.6
```

---

## Test Results

| Query | Source Document | Answer | Status |
|---|---|---|---|
| How long do I have to return a laptop? | Returns Policy | 30 days from delivery | ✅ PASS |
| What is the price of the SloffBook Pro 15? | Product Catalogue | £1,199 inc. VAT | ✅ PASS |
| Is liquid damage covered under warranty? | Warranty Policy | No — physical damage excluded | ✅ PASS |
| My SloffPhone X12 Pro is overheating. What should I do? | Customer FAQ | 3-step troubleshooting guide returned | ✅ PASS |
| How much does next day delivery cost? | Shipping Policy | £12.99, cut-off 1:00 PM | ✅ PASS |

---

## Production Gaps & Improvements

Three gaps were identified in a peer review after our cohort presentation. Documented here so anyone using this as a reference understands what is missing before deploying in a real environment.

---

### Gap 1: AWS Well-Architected Framework

| Pillar | Demo Status | Production Fix |
|---|---|---|
| Operational Excellence | ⚠️ No CloudWatch logging | Enable KB query logging, CloudWatch dashboards |
| Security | ⚠️ No bucket policy, no encryption | See Gap 2 below |
| Reliability | ⚠️ No S3 versioning, no backup | See Gap 3 below |
| Performance Efficiency | ⚠️ No latency benchmarking | Tune chunking, benchmark retrieval |
| Cost Optimisation | ✅ Serverless deleted after demo | Add AWS Budgets alert, S3 lifecycle policy |

---

### Gap 2: Unauthorised S3 Access

**Fix 1 — Explicit bucket policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnauthorisedPutObject",
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::sloff-electronics-kb-docs/*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalArn": [
            "arn:aws:iam::ACCOUNT_ID:user/Sloff-Admin"
          ]
        }
      }
    }
  ]
}
```

**Fix 2 — Server-side encryption:**
```
S3 → Bucket → Properties → Default Encryption → SSE-KMS
```

**Fix 3 — S3 access logging:**
```
S3 → Bucket → Properties → Server access logging → Enable
```

**Fix 4 — VPC Gateway Endpoint:**
```
VPC → Endpoints → Create endpoint
→ Service: com.amazonaws.us-east-1.s3
→ Type: Gateway
```

---

### Gap 3: Document Deletion

**Fix 1 — S3 Versioning (enable before first upload):**
```
S3 → Bucket → Properties → Bucket Versioning → Enable
```

**Fix 2 — MFA Delete:**
```bash
aws s3api put-bucket-versioning \
  --bucket sloff-electronics-kb-docs \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::ACCOUNT_ID:mfa/device TOTP_CODE"
```

**Fix 3 — Object Lock in Compliance mode:**
```
S3 → Bucket → Properties → Object Lock → Enable
→ Mode: Compliance · Retention: 365 days
```

**Fix 4 — Cross-region replication:**
```
S3 → Bucket → Management → Replication rules → Create rule
→ Destination: bucket in a different region
```

**Fix 5 — Never auto-sync on upload:**
```
Document uploaded → human review → approved → sync triggered manually
```

---

## Known Build Blockers

| Issue | Cause | Fix |
|---|---|---|
| KB creation blocked | Authenticated as root user | Create IAM user, re-authenticate |
| Bedrock features missing | Wrong region (eu-west-2) | Switch to us-east-1 before building |
| Claude access denied | Anthropic use case not submitted | Submit via Model Catalog — do this first |
| Empty KB results | Sync not triggered after creation | Always run sync explicitly |
| Legacy model error | Old model version defaulted | Switch to Claude Sonnet 4.6 in model selector |
| IAM role misunderstood | Assumed it covers Claude | IAM role = S3 + OpenSearch only |

---

## Cleanup — Cost Control

```
1. Bedrock → Knowledge Bases → Delete sloff-electronics-kb
2. OpenSearch → Serverless → Collections → Delete collection
3. S3 → Empty bucket → Delete bucket
4. IAM → Roles → Delete AmazonBedrockExecutionRoleForKnowledgeBase (optional)
```

---

## Deliverables

- `docs/Sloff_RAG_Documentation_v3.docx` — Technical documentation with architecture diagram
- `slides/Sloff_RAG_Presentation_v2.pptx` — 8-slide presentation deck
- `knowledge-base/` — All 5 source PDF documents
- `architecture/Blank_diagram.png` — draw.io architecture diagram

---

## Team

**CloudBoosta Academy — Group B**

| Name | Role |
|---|---|
| Osahon Igbogbo | Technical Lead · Cloud Engineer |
| Emmanuel Onuoha | Cloud Engineer |
| Bolarinwa Akinde | Cloud Engineer |
| Chinedu Izuelu | Cloud Engineer |
| Israel Olayinka | Cloud Engineer |

[LinkedIn](https://linkedin.com/in/your-profile) · May 2026
