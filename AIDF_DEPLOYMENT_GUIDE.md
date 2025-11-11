# Complete Guide: Deploying F5 AI Agent Workflow Design Hub (Flowise) to AIDF as Containerized Service

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Understanding AIDF Architecture](#understanding-aidf-architecture)
4. [One-Time Setup](#one-time-setup)
5. [Preparing Flowise for AIDF](#preparing-flowise-for-aidf)
6. [Deploying to AIDF](#deploying-to-aidf)
7. [Testing Your Deployment](#testing-your-deployment)
8. [Troubleshooting](#troubleshooting)
9. [Best Practices](#best-practices)

---

## Overview

This guide walks you through deploying **F5 AI Agent Workflow Design Hub (Flowise)** to **AIDF (AI Data Fabric)**, F5's internal AI deployment platform that runs on Databricks with the XCDF SDK.

**What is Flowise?**
Flowise is a low-code/no-code visual workflow builder for creating AI agent applications. It provides:
- Drag-and-drop interface for building LLM workflows (chatflows)
- Integration with multiple LLM providers (Azure OpenAI, Anthropic, etc.)
- Built-in vector databases, memory systems, and tools
- Multi-tenant architecture (Organizations → Workspaces → Users)
- REST API for workflow execution

**What you'll accomplish:**
- ✅ Set up your local development environment
- ✅ Configure Databricks CLI and secrets
- ✅ Prepare Flowise application for AIDF deployment
- ✅ Deploy complete Flowise service to AIDF infrastructure
- ✅ Test your deployed endpoint

**Time estimate:**
- First deployment: ~2-3 hours (including setup)
- Subsequent deployments: ~15-30 minutes

---

## Prerequisites

### Required Software

```bash
# 1. Python 3.11+ (check your version)
python3 --version  # Should show 3.11 or higher

# 2. Databricks CLI (install if needed)
pip install databricks-cli

# 3. Docker and Docker Compose (for containerization)
docker --version
docker-compose --version

# 4. Git (for version control)
git --version

# 5. Node.js 20+ and pnpm (for Flowise)
node --version  # Should show 20 or higher
npm install -g pnpm
```

### Required Access & Credentials

Before starting, ensure you have:

- [ ] **Databricks workspace access** at F5
- [ ] **Databricks CLI profile configured** (DEV or PROD)
- [ ] **Azure OpenAI API key** (prod-use-aicoe-openai endpoint)
- [ ] **AIDF namespace** (typically `xc-gpt`)
- [ ] **AIDF Organization API key** for testing endpoints
- [ ] **Databricks cluster ID** (e.g., `1008-232917-vlk982m9`)

### Verify Databricks CLI Setup

```bash
# List available profiles
databricks auth profiles

# Expected output:
# DEV
# PROD

# Test your profile
databricks clusters list --profile DEV

# Should show list of available clusters without errors
```

---

## Understanding AIDF Architecture

### What is AIDF?

**AIDF (AI Data Fabric)** is F5's proprietary AI deployment platform built on top of Databricks:

```
┌─────────────────────────────────────────────────┐
│    Flowise Application (Local Development)      │
│  ┌──────────────┐     ┌──────────────┐         │
│  │ Dockerfile   │     │ docker-      │         │
│  │              │     │ compose.yml  │         │
│  └──────────────┘     └──────────────┘         │
│  ┌──────────────────────────────────┐          │
│  │ Flowise Source Code              │          │
│  │  • packages/ui/ (Frontend)       │          │
│  │  • packages/server/ (Backend)    │          │
│  │  • packages/components/ (Nodes)  │          │
│  └──────────────────────────────────┘          │
└──────────────────┬──────────────────────────────┘
                   │
                   │ deploy_flowise.py
                   ▼
┌─────────────────────────────────────────────────┐
│              Databricks Platform                │
│  ┌───────────────────────────────────────────┐  │
│  │  Deployment Job (executes on cluster)    │  │
│  │  • Loads XCDF SDK                        │  │
│  │  • Registers Flowise as service          │  │
│  │  • Builds container image                │  │
│  │  • Deploys to AIDF infrastructure       │  │
│  └───────────────────────────────────────────┘  │
└──────────────────┬──────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────┐
│         AIDF Deployed Flowise Service           │
│  Endpoint: https://{namespace}.ml.df.f5sdc.com/ │
│            flowise-f5/v1/model/external         │
│                                                  │
│  • Full Flowise UI (port 3000)                  │
│  • Multi-tenant workspace management            │
│  • Visual workflow builder                      │
│  • Chatflow execution API                       │
│  • Azure OpenAI integration                     │
│  • Managed database and Redis                   │
└─────────────────────────────────────────────────┘
```

### Deployment Flow

```
Local Dev → Build Container → Upload to DBFS → Run Databricks Job
→ Register with XCDF → Deploy Service → Live Flowise Instance
```

---

## One-Time Setup

These steps only need to be done once per development environment.

### Step 1: Clone the Flowise Repository

```bash
# Navigate to your projects directory
cd ~/projects

# Clone the F5-customized Flowise repository
git clone https://github.com/maximuslee1226/Flowise.git
cd Flowise

# Verify F5 branding is present
ls -la images/f5-logo.png
ls -la docker-compose-f5.yml
```

### Step 2: Install Dependencies

```bash
# Install Node.js dependencies
pnpm install

# Verify Flowise builds successfully
pnpm build

# Expected output: Build completed without errors
```

### Step 3: Configure Environment Variables

```bash
# Copy the example environment file
cp .env.example .env

# Open .env in your text editor
nano .env  # or: code .env, vim .env, etc.
```

**Edit `.env` with your actual credentials:**

```bash
# ===========================================
# AIDF Deployment Configuration
# ===========================================

# Databricks Configuration
AIDF_DATABRICKS_USERNAME=your.name@f5.com
AIDF_DATABRICKS_PROFILE=DEV                    # or PROD
AIDF_DATABRICKS_SECRETS_SCOPE=your-xc-gpt      # e.g., tfotherby-xc-gpt

# AIDF Platform Configuration
AIDF_NAMESPACE=xc-gpt                          # Your AIDF namespace

# API Keys for Testing (DO NOT commit this file!)
AIDF_NAMESPACE_ORG_API_KEY=your-org-api-key-here

# Azure OpenAI API Key (will be uploaded to Databricks Secrets)
F5_DEVELOPER_NODE_OPENAI_API_KEY=your-azure-openai-key-here

# ===========================================
# Flowise Configuration
# ===========================================

# Database Configuration (managed by AIDF)
DATABASE_TYPE=postgres
DATABASE_HOST=aidf-postgres.internal.f5sdc.com
DATABASE_PORT=5432
DATABASE_USER=flowise
DATABASE_PASSWORD=<will-be-set-by-aidf>
DATABASE_NAME=flowise_f5

# Redis Configuration (for caching and sessions)
REDIS_URL=redis://aidf-redis.internal.f5sdc.com:6379

# Application Settings
PORT=3000
FLOWISE_USERNAME=admin
FLOWISE_PASSWORD=<will-be-set-by-aidf>

# Azure OpenAI Integration
AZURE_OPENAI_API_KEY=${F5_DEVELOPER_NODE_OPENAI_API_KEY}
AZURE_OPENAI_ENDPOINT=https://prod-use-aicoe-openai.openai.azure.com

# Security
PASSPHRASE=<will-be-generated-by-aidf>
TOOL_FUNCTION_BUILTIN_DEP=crypto,fs,path
TOOL_FUNCTION_EXTERNAL_DEP=moment,lodash

# Logging
LOG_LEVEL=info
LOG_PATH=/root/.flowise/logs

# Model Cache
LANGCHAIN_TRACING_V2=false
```

**⚠️ IMPORTANT:**
- Never commit `.env` to git
- Verify `.env` is in `.gitignore`
- Keep your API keys secure

```bash
# Verify .env is ignored by git
grep "\.env" .gitignore

# Should show: .env or *.env
```

### Step 4: Set Up Databricks Secrets

Databricks secrets securely store your API keys for use during deployment jobs.

**Create the secrets setup script:**

```bash
# Navigate to scripts directory (create if needed)
mkdir -p scripts/aidf
cd scripts/aidf

# Create setup_databricks_secrets.py
nano setup_databricks_secrets.py
```

**Paste this script:**

```python
#!/usr/bin/env python3
"""
Setup Databricks secrets for Flowise AIDF deployment.
"""
import os
import sys
import argparse
from dotenv import load_dotenv
from databricks_cli.sdk import ApiClient
from databricks_cli.secrets.api import SecretApi

def setup_secrets(scope: str, profile: str):
    """Upload secrets from .env to Databricks."""

    # Load environment variables
    env_path = os.path.join(os.path.dirname(__file__), '../../.env')
    load_dotenv(env_path)

    # Required secrets mapping
    secrets = {
        'AZURE_OPENAI_API_KEY_PROD_USE_AICOE': os.getenv('F5_DEVELOPER_NODE_OPENAI_API_KEY'),
        'AI_ASSISTANT_ORG_KEY': os.getenv('AIDF_NAMESPACE_ORG_API_KEY'),
        'FLOWISE_DB_PASSWORD': os.getenv('DATABASE_PASSWORD', 'flowise_secure_pass_' + os.urandom(8).hex()),
        'FLOWISE_ADMIN_PASSWORD': os.getenv('FLOWISE_PASSWORD', 'admin_' + os.urandom(12).hex()),
        'FLOWISE_PASSPHRASE': os.urandom(32).hex(),
    }

    # Validate secrets
    missing = [k for k, v in secrets.items() if not v]
    if missing:
        print(f"❌ Missing required environment variables: {', '.join(missing)}")
        sys.exit(1)

    print(f"✓ Loaded {len(secrets)} secrets from .env file")

    # Initialize Databricks API client
    api_client = ApiClient(profile=profile)
    secret_api = SecretApi(api_client)

    # Create scope (ignore if exists)
    try:
        secret_api.create_scope(scope, None)
        print(f"✓ Created Databricks secret scope: {scope}")
    except Exception as e:
        if "already exists" in str(e).lower():
            print(f"✓ Using existing secret scope: {scope}")
        else:
            raise

    # Upload each secret
    for key, value in secrets.items():
        try:
            secret_api.put_secret(scope, key, value, None)
            print(f"✓ Uploaded secret: {key}")
        except Exception as e:
            print(f"❌ Failed to upload {key}: {e}")
            sys.exit(1)

    print(f"\n✓ All secrets uploaded successfully to scope: {scope}")
    print(f"\nVerify with: databricks secrets list-secrets {scope} --profile {profile}")

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Setup Databricks secrets for Flowise")
    parser.add_argument("--scope", required=True, help="Databricks secret scope name")
    parser.add_argument("--profile", default="DEV", help="Databricks CLI profile")
    args = parser.parse_args()

    setup_secrets(args.scope, args.profile)
```

**Run the secrets setup script:**

```bash
# Make script executable
chmod +x setup_databricks_secrets.py

# Run the script
python3 setup_databricks_secrets.py \
  --scope your-xc-gpt \
  --profile DEV

# Expected output:
# ✓ Loaded 5 secrets from .env file
# ✓ Creating Databricks secret scope: your-xc-gpt
# ✓ Uploading secret: AZURE_OPENAI_API_KEY_PROD_USE_AICOE
# ✓ Uploading secret: AI_ASSISTANT_ORG_KEY
# ✓ Uploading secret: FLOWISE_DB_PASSWORD
# ✓ Uploading secret: FLOWISE_ADMIN_PASSWORD
# ✓ Uploading secret: FLOWISE_PASSPHRASE
# ✓ All secrets uploaded successfully
```

**Verify secrets were uploaded:**

```bash
databricks secrets list-secrets your-xc-gpt --profile DEV

# Expected output:
# Key name                                   Last updated
# AZURE_OPENAI_API_KEY_PROD_USE_AICOE       2025-01-27
# AI_ASSISTANT_ORG_KEY                       2025-01-27
# FLOWISE_DB_PASSWORD                        2025-01-27
# FLOWISE_ADMIN_PASSWORD                     2025-01-27
# FLOWISE_PASSPHRASE                         2025-01-27
```

### Step 5: Get Your Databricks Cluster ID

You need an existing cluster ID for deployment jobs.

```bash
# List available clusters
databricks clusters list --profile DEV

# Find a running cluster and note its ID
# Example output:
# 1008-232917-vlk982m9  my-cluster  RUNNING

# Copy the cluster ID (e.g., 1008-232917-vlk982m9)
```

**⚠️ Note:** If no clusters are available, contact your Databricks admin to create one or start an existing cluster.

---

## Preparing Flowise for AIDF

Now let's prepare the Flowise application for AIDF deployment.

### Application Structure

Flowise AIDF deployment uses this structure:

```
Flowise/
├── Dockerfile                      # Container build configuration
├── docker-compose-f5.yml           # F5-customized Docker Compose
├── packages/
│   ├── ui/                         # Frontend (F5 branded)
│   ├── server/                     # Backend API
│   └── components/                 # Workflow nodes
├── scripts/
│   └── aidf/
│       ├── deploy_flowise.py       # AIDF deployment script
│       └── setup_databricks_secrets.py
└── .env                            # Configuration (not committed)
```

### Step 1: Review Dockerfile

The Dockerfile at `/Users/br.lee/projects/Flowise/Dockerfile` is already configured for AIDF deployment.

**Key configurations:**
- Node.js 20 Alpine base image
- Chromium for PDF/document processing
- Health check support (curl/wget)
- Optimized memory settings
- Port 3000 for Flowise UI

### Step 2: Review Docker Compose Configuration

The `docker-compose-f5.yml` at `/Users/br.lee/projects/Flowise/docker-compose-f5.yml` is configured for AIDF.

**Key features:**
- Persistent volume for data
- Health check endpoint: `/api/v1/ping`
- Production configuration
- Restart policy

### Step 3: Create AIDF Deployment Script

Create `scripts/aidf/deploy_flowise.py`:

```bash
cd /Users/br.lee/projects/Flowise
mkdir -p scripts/aidf
nano scripts/aidf/deploy_flowise.py
```

**Paste this deployment script:**

```python
#!/usr/bin/env python3
"""
Deploy Flowise application to AIDF (AI Data Fabric).
"""
import os
import sys
import json
import argparse
import subprocess
from pathlib import Path
from dotenv import load_dotenv

# Load environment
load_dotenv()

def deploy_flowise(cluster_id: str, profile: str):
    """Deploy Flowise to AIDF via Databricks job."""

    # Configuration
    namespace = os.getenv('AIDF_NAMESPACE', 'xc-gpt')
    secrets_scope = os.getenv('AIDF_DATABRICKS_SECRETS_SCOPE')
    username = os.getenv('AIDF_DATABRICKS_USERNAME')

    if not all([secrets_scope, username]):
        print("❌ Missing required environment variables in .env")
        sys.exit(1)

    print("=" * 60)
    print("F5 AI Agent Workflow Design Hub - AIDF Deployment")
    print("=" * 60)
    print(f"Namespace: {namespace}")
    print(f"Profile: {profile}")
    print(f"Cluster: {cluster_id}")
    print(f"Secrets: {secrets_scope}")
    print("=" * 60)

    # Generate deployment script
    deployment_script = f"""
import xcdf_sdk
from xcdf_sdk import ModelConfig, DeploymentConfig

# Configure Flowise service
config = ModelConfig(
    name="flowise-f5",
    version="1.0.0",
    description="F5 AI Agent Workflow Design Hub - Visual LLM Workflow Builder",
    framework="custom",
    runtime="docker"
)

deployment = DeploymentConfig(
    namespace="{namespace}",
    replicas=2,
    resources={{
        "cpu": "2",
        "memory": "4Gi"
    }},
    environment={{
        "DATABASE_TYPE": "postgres",
        "DATABASE_HOST": "${{AIDF_DB_HOST}}",
        "DATABASE_USER": "flowise",
        "DATABASE_PASSWORD": "${{{{dbsecret:{secrets_scope}/FLOWISE_DB_PASSWORD}}}}",
        "DATABASE_NAME": "flowise_f5",
        "REDIS_URL": "${{AIDF_REDIS_URL}}",
        "PORT": "3000",
        "AZURE_OPENAI_API_KEY": "${{{{dbsecret:{secrets_scope}/AZURE_OPENAI_API_KEY_PROD_USE_AICOE}}}}",
        "AZURE_OPENAI_ENDPOINT": "https://prod-use-aicoe-openai.openai.azure.com",
        "FLOWISE_USERNAME": "admin",
        "FLOWISE_PASSWORD": "${{{{dbsecret:{secrets_scope}/FLOWISE_ADMIN_PASSWORD}}}}",
        "PASSPHRASE": "${{{{dbsecret:{secrets_scope}/FLOWISE_PASSPHRASE}}}}",
        "LOG_LEVEL": "info",
        "NODE_ENV": "production"
    }},
    ports=[3000],
    health_check={{
        "path": "/api/v1/ping",
        "interval": 30,
        "timeout": 10
    }}
)

# Build and deploy
print("Building Flowise container image...")
xcdf_sdk.build_container("Dockerfile", "flowise-f5:1.0.0")

print("Registering with XCDF...")
xcdf_sdk.register_model(config)

print("Deploying to AIDF...")
xcdf_sdk.deploy(config, deployment)

print("✓ Deployment complete!")
print(f"Endpoint: https://{namespace}.ml.df.f5sdc.com/flowise-f5/")
"""

    # Write script to temporary file
    script_path = "/tmp/deploy_flowise_script.py"
    with open(script_path, 'w') as f:
        f.write(deployment_script)

    print("\n✓ Generated deployment script")

    # Upload to DBFS
    dbfs_path = f"dbfs:/FileStore/{username}/flowise_deploy.py"
    cmd = f"databricks fs cp {script_path} {dbfs_path} --overwrite --profile {profile}"

    print(f"\n📤 Uploading to DBFS: {dbfs_path}")
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)

    if result.returncode != 0:
        print(f"❌ Upload failed: {result.stderr}")
        sys.exit(1)

    print("✓ Upload complete")

    # Submit Databricks job
    job_config = {
        "run_name": f"Deploy Flowise - {username}",
        "existing_cluster_id": cluster_id,
        "spark_python_task": {
            "python_file": dbfs_path
        },
        "libraries": [
            {"pypi": {"package": "xcdf-sdk"}},
            {"pypi": {"package": "docker"}},
        ]
    }

    job_file = "/tmp/flowise_job.json"
    with open(job_file, 'w') as f:
        json.dump(job_config, f, indent=2)

    print("\n🚀 Submitting deployment job to Databricks...")
    cmd = f"databricks jobs run-now --json-file {job_file} --profile {profile}"
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)

    if result.returncode != 0:
        print(f"❌ Job submission failed: {result.stderr}")
        sys.exit(1)

    run_info = json.loads(result.stdout)
    run_id = run_info.get('run_id')

    print(f"✓ Job submitted: Run ID {run_id}")
    print(f"\n📊 Monitor progress:")
    print(f"   https://databricks.f5sdc.com/#job/{run_id}")
    print(f"\n⏳ Deployment typically takes 5-10 minutes")
    print(f"   Phase 1: Build container (2-3 min)")
    print(f"   Phase 2: Register with XCDF (1-2 min)")
    print(f"   Phase 3: Deploy service (2-5 min)")
    print(f"\n✓ When complete, access Flowise at:")
    print(f"   https://{namespace}.ml.df.f5sdc.com/flowise-f5/")

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Deploy Flowise to AIDF")
    parser.add_argument("--cluster-id", required=True, help="Databricks cluster ID")
    parser.add_argument("--profile", default="DEV", help="Databricks profile")
    args = parser.parse_args()

    deploy_flowise(args.cluster_id, args.profile)
```

**Make script executable:**

```bash
chmod +x scripts/aidf/deploy_flowise.py
```

---

## Deploying to AIDF

Now you're ready to deploy Flowise to AIDF!

### Step 1: Navigate to Flowise Directory

```bash
cd /Users/br.lee/projects/Flowise
```

### Step 2: Run Deployment Script

```bash
# Deploy Flowise to AIDF
./scripts/aidf/deploy_flowise.py \
  --cluster-id 1008-232917-vlk982m9 \
  --profile DEV

# Replace:
# - 1008-232917-vlk982m9 → your Databricks cluster ID
# - DEV → your Databricks profile (DEV or PROD)
```

### Step 3: Monitor Deployment Progress

The script will show detailed progress. Here's what to expect:

**Phase 1: Configuration (10 seconds)**
- Loads `.env` configuration
- Validates settings
- Shows deployment parameters

**Phase 2: Script Generation (5 seconds)**
- Generates XCDF deployment script
- Configures environment variables
- Sets up database connections

**Phase 3: Upload to DBFS (10 seconds)**
- Uploads deployment script to Databricks

**Phase 4: Job Submission (5 seconds)**
- Submits Databricks job
- Returns Run ID

**Phase 5: Job Execution (5-10 minutes)**
- Building container image (120-180s)
- Registering with XCDF (60-120s)
- Deploying service (120-300s)
- Starting health checks (30-60s)

**Phase 6: Final Output**
- Shows endpoint URL
- Provides access instructions
- Confirms success

**Expected total time: 6-11 minutes**

### Step 4: Save Endpoint URL

Copy the endpoint URL from the output:

```
https://xc-gpt.ml.df.f5sdc.com/flowise-f5/
```

**URL Format:**
```
https://{namespace}.ml.df.f5sdc.com/{service-id}/
      └─────┬─────┘                  └────┬────┘
         AIDF namespace              Service ID (flowise-f5)
```

---

## Testing Your Deployment

### Method 1: Access Flowise UI

```bash
# Open browser to Flowise UI
open https://xc-gpt.ml.df.f5sdc.com/flowise-f5/

# Login with credentials from Databricks secrets
Username: admin
Password: <from FLOWISE_ADMIN_PASSWORD secret>
```

**What you can do:**
- ✅ Create new chatflows with visual builder
- ✅ Configure LLM nodes (Azure OpenAI)
- ✅ Add vector databases, memory, tools
- ✅ Test workflows in real-time
- ✅ Manage workspaces and users
- ✅ Export/import chatflows

### Method 2: Test Chatflow API

```bash
# Create a test chatflow first in the UI, then get its ID
CHATFLOW_ID="your-chatflow-id"

# Test the chatflow
curl -X POST "https://xc-gpt.ml.df.f5sdc.com/flowise-f5/api/v1/prediction/${CHATFLOW_ID}" \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_ORG_API_KEY" \
  -d '{
    "question": "What is the purpose of this chatflow?",
    "overrideConfig": {}
  }'
```

### Method 3: Health Check

```bash
# Check service health
curl "https://xc-gpt.ml.df.f5sdc.com/flowise-f5/api/v1/ping"

# Expected response: {"status": "ok"}
```

### Method 4: List All Chatflows

```bash
# List available chatflows
curl -X GET "https://xc-gpt.ml.df.f5sdc.com/flowise-f5/api/v1/chatflows" \
  -H "x-api-key: YOUR_ORG_API_KEY"
```

---

## Troubleshooting

### Issue: "Configuration file not found"

```bash
# Verify .env exists
ls -la .env

# Check you're in correct directory
pwd  # Should end with: /Flowise
```

### Issue: "Databricks secrets not found"

```bash
# List secrets
databricks secrets list-secrets your-xc-gpt --profile DEV

# Re-run setup if empty
cd scripts/aidf
python3 setup_databricks_secrets.py --scope your-xc-gpt --profile DEV
```

### Issue: "Build failed"

- Check Databricks job logs (click "View progress" URL)
- Verify XCDF package exists in `/Workspace/Shared/packages/`
- Contact Databricks admin if package is missing
- Ensure Docker daemon is available on cluster

### Issue: "Database connection failed"

```bash
# Verify database credentials in secrets
databricks secrets get your-xc-gpt FLOWISE_DB_PASSWORD --profile DEV

# Check AIDF database service status
# Contact AIDF admin if database is unavailable
```

### Issue: "Endpoint returns 401"

```bash
# Verify API key
echo $AIDF_NAMESPACE_ORG_API_KEY

# Test with verbose output
curl -v -X GET "https://xc-gpt.ml.df.f5sdc.com/flowise-f5/api/v1/ping" \
  -H "x-api-key: YOUR_ORG_API_KEY"
```

### Issue: "Endpoint returns 404"

If you get `404 Not Found`:
1. Verify deployment completed successfully
2. Check service name matches: `flowise-f5`
3. Confirm namespace is correct: `xc-gpt`
4. Wait 1-2 minutes after deployment completes

---

## Best Practices

### Security ✅

- Store API keys in `.env` (never commit)
- Use Databricks Secrets for production
- Rotate keys regularly
- Use separate scopes for DEV/PROD
- Enable authentication in production
- Restrict workspace access by user roles

### Configuration ✅

- Use descriptive service IDs
- Set appropriate resource limits (CPU/memory)
- Configure persistent volumes for data
- Enable health checks
- Set up logging and monitoring

### Deployment ✅

- Test locally with `docker-compose -f docker-compose-f5.yml up`
- Deploy to DEV before PROD
- Monitor first deployment carefully
- Verify UI loads and chatflows execute
- Test Azure OpenAI integration

### Workflow Management ✅

- Create separate workspaces for teams
- Use version control for chatflow exports
- Document complex workflows
- Test chatflows thoroughly before production
- Monitor API usage and performance

---

## Quick Reference

### Essential Commands

```bash
# Build locally
docker-compose -f docker-compose-f5.yml up --build

# Deploy to AIDF
./scripts/aidf/deploy_flowise.py --cluster-id CLUSTER_ID --profile DEV

# Test health
curl https://xc-gpt.ml.df.f5sdc.com/flowise-f5/api/v1/ping

# Access UI
open https://xc-gpt.ml.df.f5sdc.com/flowise-f5/

# Secrets
python3 scripts/aidf/setup_databricks_secrets.py --scope SCOPE --profile DEV
databricks secrets list-secrets SCOPE --profile DEV
```

### Key Files

```
Flowise/
├── .env                              # Credentials (don't commit!)
├── Dockerfile                        # Container build
├── docker-compose-f5.yml             # F5 Docker config
├── packages/
│   ├── ui/                           # Frontend (F5 branded)
│   └── server/                       # Backend API
└── scripts/aidf/
    ├── deploy_flowise.py             # Deploy script
    └── setup_databricks_secrets.py   # Secrets setup
```

### API Endpoints

```
GET  /                                 # Flowise UI
GET  /api/v1/ping                      # Health check
GET  /api/v1/chatflows                 # List chatflows
GET  /api/v1/chatflows/:id             # Get chatflow
POST /api/v1/prediction/:id            # Execute chatflow (streaming)
```

---

**🎉 You're ready to use Flowise on AIDF!**

For questions, contact the F5 AI CoE team.

---

*Last updated: January 2025*
