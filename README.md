# 🤖Digital Portfolio AI — Your AI-Powered Digital Resume

> Instead of a static resume, give visitors a chatbot that *knows you* — your career history, skills, and personality. Recruiters and managers can have real conversations with your digital twin.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Step 1: Personalize Your Data](#step-1-personalize-your-data)
- [Step 2: Local Development](#step-2-local-development)
- [Step 3: Deploy Lambda Function](#step-3-deploy-lambda-function)
- [Step 4: Set Up API Gateway](#step-4-set-up-api-gateway)
- [Step 5: Deploy Frontend to S3](#step-5-deploy-frontend-to-s3)
- [Step 6: Set Up CloudFront](#step-6-set-up-cloudfront)
- [Step 7: Test Everything](#step-7-test-everything)
- [Troubleshooting](#troubleshooting)

---

## Overview

Twin AI is a full-stack serverless application that replaces your resume with an interactive AI chatbot. It uses:

- **Frontend**: Next.js static site hosted on S3 + CloudFront
- **Backend**: FastAPI running on AWS Lambda via Mangum
- **AI**: AWS Bedrock (Amazon Nova Lite) for intelligent responses
- **Memory**: S3 for persistent conversation history
- **API**: AWS API Gateway (HTTP API)

---

## Prerequisites

Make sure you have the following installed before you begin:

| Tool | Purpose | Install Guide |
|------|---------|--------------|
| Node.js | Frontend build | [nodejs.org](https://nodejs.org) |
| Python 3.12 | Backend runtime | [python.org](https://python.org) |
| uv | Fast Python package manager | [docs.astral.sh/uv](https://docs.astral.sh/uv/getting-started/installation/) |
| AWS CLI | Deploy to AWS | [aws.amazon.com/cli](https://aws.amazon.com/cli/) |
| Docker | Optional, for local testing | [docker.com](https://docker.com) |

**Install uv (Python package manager):**

```bash
# Mac/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Verify: `uv --version`

---

## Project Structure

```
twin-ai/
├── backend/
│   ├── data/
│   │   ├── facts.json        ← Your personal facts
│   │   ├── linkedin.pdf      ← Your LinkedIn profile PDF
│   │   ├── style.txt         ← Your communication style
│   │   └── summary.txt       ← Your career summary
│   ├── me.txt                ← Your full personality prompt
│   ├── server.py             ← FastAPI application
│   ├── context.py            ← Prompt builder
│   ├── resources.py          ← Data loader
│   └── requirements.txt
├── frontend/
│   ├── components/
│   │   └── twin.tsx          ← Chat UI component
│   ├── next.config.ts
│   └── package.json
```

---

## Step 1: Personalize Your Data

Update these files with **your own information** before deploying:

### `backend/data/facts.json`
```json
{
  "full_name": "Your Full Name",
  "name": "Your First Name",
  "title": "Your Job Title",
  "location": "Your City, Country",
  "email": "your@email.com"
}
```

### `backend/data/summary.txt`
Write 2-3 paragraphs about your career background, key achievements, and what makes you unique.

### `backend/data/style.txt`
Describe how you communicate — formal/casual, concise/detailed, etc.

### `backend/data/linkedin.pdf`
Export your LinkedIn profile as PDF and place it here.

### `backend/me.txt`
This is the main personality file. Write it in first person as if you're briefing the AI on who you are.

---

## Step 2: Local Development

### Configure Environment

Create `backend/.env`:
```env
DEFAULT_AWS_REGION=us-east-1
BEDROCK_MODEL_ID=global.amazon.nova-lite-v1:0
CORS_ORIGINS=http://localhost:3000
USE_S3=false
```

> **Note**: AWS Bedrock uses your AWS credentials directly — no separate API key needed. Make sure your AWS CLI is configured with `aws configure`.

### Enable Bedrock Model Access

Before running locally or deploying, you must request access to the Bedrock model:

1. Go to **AWS Console → Bedrock → Model access**
2. Click **Manage model access**
3. Enable **Amazon Nova Lite** (or your chosen model)
4. Click **Save changes** — access is usually granted within a few minutes

### Run Backend

```bash
cd backend
uv add -r requirements.txt
uv run uvicorn server:app --reload
```

Backend runs at `http://localhost:8000`. Test it:
```bash
curl http://localhost:8000/health
# {"status": "healthy", "use_s3": false}
```

### Run Frontend

```bash
cd frontend
npm install
npm run dev
```

Frontend runs at `http://localhost:3000`.

---

## Step 3: Deploy Lambda Function

### 3.1 Create S3 Memory Bucket

Go to **S3 → Create bucket**:
- Bucket name: `twin-memory` (or your preferred name, **no spaces**)
- Region: your preferred region
- Keep all other defaults
- Click **Create bucket**

### 3.2 Create Lambda Function

Go to **Lambda → Create function**:
- **Function name**: `portfolio-ai-bot`
- **Runtime**: Python 3.12
- **Architecture**: x86_64
- Click **Create function**

### 3.3 Build and Upload Deployment Package

```bash
cd backend
uv run deploy.py
```

This creates `lambda-deployment.zip`. Then in Lambda Console:
- Click **Upload from → .zip file**
- Upload `lambda-deployment.zip`
- Click **Save**

### 3.4 Configure Environment Variables

Go to **Lambda → Configuration → Environment variables → Edit**:

| Key | Value |
|-----|-------|
| `DEFAULT_AWS_REGION` | `us-east-1` (or your region) |
| `BEDROCK_MODEL_ID` | `global.amazon.nova-lite-v1:0` |
| `CORS_ORIGINS` | `*` (update to CloudFront URL later) |
| `USE_S3` | `true` |
| `S3_BUCKET` | `twin-memory` (**no spaces!**) |

### 3.5 Configure Lambda Handler and Timeout

- Go to **Configuration → General configuration → Edit**
- Set **Handler** to: `lambda_handler.lambda_handler`
- Set **Timeout** to: `30 seconds`
- Click **Save**

### 3.6 Add Permissions to Lambda Role

Go to **IAM → Roles → your-lambda-role → Add permissions → Create inline policy**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::twin-memory",
        "arn:aws:s3:::twin-memory/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": "*"
    }
  ]
}
```

> **Important**: Lambda needs both S3 permissions (for memory) and Bedrock permissions (to call the AI model).

### 3.7 Test Lambda

Go to **Lambda → Test tab**, create a new test event:

```json
{
  "version": "2.0",
  "routeKey": "GET /health",
  "rawPath": "/health",
  "headers": {
    "content-type": "application/json"
  },
  "requestContext": {
    "http": {
      "method": "GET",
      "path": "/health",
      "protocol": "HTTP/1.1",
      "sourceIp": "127.0.0.1",
      "userAgent": "test-invoke"
    },
    "routeKey": "GET /health",
    "stage": "$default"
  },
  "isBase64Encoded": false
}
```

Expected response: `{"status": "healthy", "use_s3": true}`

---

## Step 4: Set Up API Gateway

### 4.1 Create HTTP API

Go to **API Gateway → Create API → HTTP API → Build**:
- Click **Add integration**
- Integration type: **Lambda**
- Lambda function: select `portfolio-ai-bot`
- API name: `twin-api-gateway`
- Click **Next**

### 4.2 Configure Routes

Add these routes (click **Add route** for each):

| Method | Resource Path | Integration |
|--------|--------------|-------------|
| `ANY` | `/{proxy+}` | twin-api |
| `GET` | `/` | twin-api |
| `GET` | `/health` | twin-api |
| `POST` | `/chat` | twin-api |
| `OPTIONS` | `/{proxy+}` | twin-api |

### 4.3 Configure CORS

Go to **your API → CORS → Configure**:

> ⚠️ **Important**: For each field, you must type the value AND click **Add**. Simply typing won't save it.

| Setting | Value |
|---------|-------|
| Access-Control-Allow-Origin | `*` |
| Access-Control-Allow-Headers | `*` |
| Access-Control-Allow-Methods | `*` |
| Access-Control-Max-Age | `300` |

Click **Save**.

### 4.4 Get Your API URL

Go to **Stages → $default** and copy your **Invoke URL**:
```
https://YOUR-API-ID.execute-api.us-east-1.amazonaws.com
```

Test in terminal:
```bash
curl https://YOUR-API-ID.execute-api.us-east-1.amazonaws.com/health
# {"status": "healthy", "use_s3": true}
```

### 4.5 Update Frontend API URL

In `frontend/components/twin.tsx`, update the API endpoint:
```ts
const response = await fetch('https://YOUR-API-ID.execute-api.us-east-1.amazonaws.com/chat', {
```

---

## Step 5: Deploy Frontend to S3

### 5.1 Configure Next.js for Static Export

In `frontend/next.config.ts`:
```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: 'export',
};

export default nextConfig;
```

### 5.2 Build

```bash
cd frontend
npm run build
```

This creates an `out/` directory with your static files.

### 5.3 Create Frontend S3 Bucket

Go to **S3 → Create bucket**:
- Bucket name: `twin-frontend` (or your preferred name)
- **Uncheck** "Block all public access"
- Enable **Static website hosting**:
  - Index document: `index.html`
  - Error document: `index.html`
- Add this bucket policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::twin-frontend/*"
  }]
}
```

### 5.4 Upload Files

```bash
cd frontend
aws s3 sync out/ s3://twin-frontend/ --delete
```

---

## Step 6: Set Up CloudFront

### 6.1 Get Your S3 Website Endpoint

Go to **S3 → your frontend bucket → Properties → Static website hosting**. Copy the **Bucket website endpoint** (looks like `http://twin-frontend.s3-website-us-east-1.amazonaws.com`).

### 6.2 Create CloudFront Distribution

Go to **CloudFront → Create distribution**:

**Origin settings:**
- Choose origin: **Other** (NOT Amazon S3)
- Origin domain: paste your S3 website endpoint **without** `http://`
  - Example: `twin-frontend.s3-website-us-east-1.amazonaws.com`
- Origin protocol: **HTTP only** ← Critical! S3 static hosting does not support HTTPS

**Cache behavior:**
- Viewer protocol policy: **Redirect HTTP to HTTPS**
- Allowed HTTP methods: `GET, HEAD`

**Settings:**
- Default root object: `index.html`
- Price class: Use only North America and Europe (saves cost)

Click **Create distribution**. Wait 5–15 minutes for deployment.

### 6.3 Update CORS for CloudFront

Once deployed, copy your **CloudFront distribution domain** (like `d1234abcd.cloudfront.net`).

Go to **Lambda → Configuration → Environment variables** and update:

```
CORS_ORIGINS = https://d1234abcd.cloudfront.net
```

> ⚠️ **Critical**: The URL must:
> - Start with `https://`
> - Match your CloudFront domain exactly
> - Have **no trailing slash**

### 6.4 Invalidate Cache

Go to **CloudFront → your distribution → Invalidations → Create invalidation**:
- Path: `/*`
- Click **Create invalidation**

---

## Step 7: Test Everything

### Check your Twin is live

Visit: `https://YOUR-CLOUDFRONT-DOMAIN.cloudfront.net`

### Verify memory is working

Go to **S3 → twin-memory bucket** — you should see JSON files for each conversation session.

### Monitor logs

Go to **CloudWatch → Log groups → /aws/lambda/portfolio-ai-bot** to debug any issues.

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `Handler 'lambda_handler' missing` | Wrong handler config | Set handler to `lambda_handler.lambda_handler` |
| `Method Not Allowed` on `/chat` | Hitting GET in browser | Use POST — this is expected browser behavior |
| `Failed to fetch` in browser | CORS not configured | Configure CORS in API Gateway and set `CORS_ORIGINS` in Lambda |
| `AccessDenied on S3` | Missing IAM permissions | Add S3 inline policy to Lambda role |
| `AccessDenied on Bedrock` | Missing Bedrock permissions | Add `bedrock:InvokeModel` to Lambda role policy |
| `Model not available` | Bedrock model access not enabled | Enable model access in **Bedrock → Model access** |
| `Invalid bucket name` | Trailing space in env var | Check `S3_BUCKET` env var for trailing spaces |
| `sourceIp KeyError` | Incomplete test event | Add `sourceIp` and `userAgent` to test event's `requestContext.http` |
| CloudFront 504 errors | Wrong origin protocol | Set origin protocol to **HTTP only** in CloudFront |

---

## Architecture

```
Browser → CloudFront → S3 (static frontend)
                ↓
Browser → API Gateway → Lambda (FastAPI + Mangum)
                              ↓
                        AWS Bedrock (Nova Lite)
                              ↓
                           S3 (conversation memory)
```

---

