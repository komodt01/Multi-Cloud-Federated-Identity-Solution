# Multi-Cloud Federated Identity: Linux Commands Reference

## Table of Contents
1. [Prerequisites Setup](#prerequisites-setup)
2. [AWS Commands](#aws-commands)
3. [Azure Commands](#azure-commands)
4. [GCP Commands](#gcp-commands)
5. [GitHub Actions Commands](#github-actions-commands)
6. [Verification and Testing](#verification-and-testing)
7. [Monitoring and Troubleshooting](#monitoring-and-troubleshooting)
8. [Cleanup Commands](#cleanup-commands)

## Prerequisites Setup

### Install Cloud CLIs

```bash
# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version

# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az --version

# Install Google Cloud SDK
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud --version

# Install additional tools
sudo apt update
sudo apt install -y jq curl wget git
```

### Verify Installation

```bash
# Check all CLI tools
which aws && aws --version
which az && az --version  
which gcloud && gcloud --version
which jq && jq --version
```

### Authentication Setup

```bash
# AWS Authentication
aws configure
# Enter Access Key ID, Secret Access Key, Region, Output format

# Azure Authentication
az login
# Opens browser for authentication

# GCP Authentication
gcloud auth login
# Opens browser for authentication
gcloud config set project YOUR_PROJECT_ID
```

## AWS Commands

### OIDC Provider Management

```bash
# Create OIDC Identity Provider
aws iam create-open-id-connect-provider \
    --url "https://token.actions.githubusercontent.com" \
    --client-id-list "sts.amazonaws.com" \
    --thumbprint-list "6938fd4d98bab03faadb97b34396831e3780aea1" "1c58a3a8518e8759bf075b76b750d4f2df264fcd"

# List OIDC providers
aws iam list-open-id-connect-providers

# Get OIDC provider details
aws iam get-open-id-connect-provider \
    --open-id-connect-provider-arn "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com"

# Update OIDC provider thumbprints
aws iam update-open-id-connect-provider-thumbprint \
    --open-id-connect-provider-arn "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com" \
    --thumbprint-list "6938fd4d98bab03faadb97b34396831e3780aea1"

# Delete OIDC provider
aws iam delete-open-id-connect-provider \
    --open-id-connect-provider-arn "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com"
```

### IAM Role Management

```bash
# Create trust policy document
cat > trust-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
                    "token.actions.githubusercontent.com:sub": "repo:komodt01/Federated-Identity:ref:refs/heads/main"
                }
            }
        }
    ]
}
EOF

# Create IAM role
aws iam create-role \
    --role-name "FederatedIdentityRole" \
    --assume-role-policy-document file://trust-policy.json \
    --description "Role for GitHub Actions federated identity"

# Get role details
aws iam get-role --role-name "FederatedIdentityRole"

# List role policies
aws iam list-attached-role-policies --role-name "FederatedIdentityRole"
aws iam list-role-policies --role-name "FederatedIdentityRole"

# Attach managed policy
aws iam attach-role-policy \
    --role-name "FederatedIdentityRole" \
    --policy-arn "arn:aws:iam::aws:policy/ReadOnlyAccess"

# Create and attach inline policy
cat > s3-policy.json << 'EOF'
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
                "arn:aws:s3:::federated-identity-bucket-*",
                "arn:aws:s3:::federated-identity-bucket-*/*"
            ]
        }
    ]
}
EOF

aws iam put-role-policy \
    --role-name "FederatedIdentityRole" \
    --policy-name "S3AccessPolicy" \
    --policy-document file://s3-policy.json
```

### S3 Bucket Management

```bash
# Create S3 bucket
aws s3 mb s3://federated-identity-bucket-$(aws sts get-caller-identity --query Account --output text)

# Enable versioning
aws s3api put-bucket-versioning \
    --bucket "federated-identity-bucket-$(aws sts get-caller-identity --query Account --output text)" \
    --versioning-configuration Status=Enabled

# Block public access
aws s3api put-public-access-block \
    --bucket "federated-identity-bucket-$(aws sts get-caller-identity --query Account --output text)" \
    --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# List bucket contents
aws s3 ls s3://federated-identity-bucket-$(aws sts get-caller-identity --query Account --output text)

# Upload test file
echo "Hello from federated identity" > test-file.txt
aws s3 cp test-file.txt s3://federated-identity-bucket-$(aws sts get-caller-identity --query Account --output text)/
```

### CloudTrail Setup

```bash
# Create CloudTrail
aws cloudtrail create-trail \
    --name "FederatedIdentityTrail" \
    --s3-bucket-name "federated-identity-bucket-$(aws sts get-caller-identity --query Account --output text)" \
    --s3-key-prefix "cloudtrail-logs/" \
    --include-global-service-events \
    --is-multi-region-trail

# Start logging
aws cloudtrail start-logging --name "FederatedIdentityTrail"

# Get trail status
aws cloudtrail get-trail-status --name "FederatedIdentityTrail"
```

## Azure Commands

### Authentication and Setup

```bash
# Login and set subscription
az login
az account set --subscription "YOUR_SUBSCRIPTION_ID"
az account show

# Create resource group
az group create --name "rg-federated-identity" --location "East US"
```

### Application Registration

```bash
# Create application registration
az ad app create \
    --display-name "Multi-Cloud Federated Identity Application" \
    --sign-in-audience "AzureADMyOrg" \
    --web-redirect-uris "https://localhost:8080/callback"

# Get application details
APP_ID=$(az ad app list --display-name "Multi-Cloud Federated Identity Application" --query "[0].appId" -o tsv)
echo "Application ID: $APP_ID"

# Create service principal
az ad sp create --id "$APP_ID"

# Get service principal details
SP_ID=$(az ad sp list --display-name "Multi-Cloud Federated Identity Application" --query "[0].id" -o tsv)
echo "Service Principal ID: $SP_ID"
```

### Federated Identity Credentials

```bash
# Create federated identity credential
az ad app federated-credential create \
    --id "$APP_ID" \
    --parameters '{
        "name": "github-actions-credential",
        "issuer": "https://token.actions.githubusercontent.com",
        "subject": "repo:komodt01/Federated-Identity:ref:refs/heads/main",
        "audiences": ["api://AzureADTokenExchange"],
        "description": "Federated credential for GitHub Actions OIDC"
    }'

# List federated credentials
az ad app federated-credential list --id "$APP_ID"

# Show specific federated credential
az ad app federated-credential show \
    --id "$APP_ID" \
    --federated-credential-id "github-actions-credential"

# Delete federated credential
az ad app federated-credential delete \
    --id "$APP_ID" \
    --federated-credential-id "github-actions-credential"
```

### Resource Management

```bash
# Create Key Vault
VAULT_NAME="kv-fed-$(openssl rand -hex 4)"
az keyvault create \
    --name "$VAULT_NAME" \
    --resource-group "rg-federated-identity" \
    --location "East US"

# Create Storage Account
STORAGE_NAME="stfed$(openssl rand -hex 4)"
az storage account create \
    --name "$STORAGE_NAME" \
    --resource-group "rg-federated-identity" \
    --location "East US" \
    --sku Standard_LRS \
    --kind StorageV2 \
    --https-only true

# Set Key Vault permissions
az keyvault set-policy \
    --name "$VAULT_NAME" \
    --object-id "$SP_ID" \
    --secret-permissions get list \
    --key-permissions get list
```

### RBAC Management

```bash
# Assign Contributor role to resource group
az role assignment create \
    --assignee "$SP_ID" \
    --role "Contributor" \
    --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-federated-identity"

# Assign Key Vault access
az role assignment create \
    --assignee "$SP_ID" \
    --role "Key Vault Secrets User" \
    --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-federated-identity/providers/Microsoft.KeyVault/vaults/$VAULT_NAME"

# List role assignments
az role assignment list --assignee "$SP_ID" --output table
```

## GCP Commands

### Project Setup

```bash
# Set project and region
export PROJECT_ID="federated-465114"
gcloud config set project $PROJECT_ID
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# Verify project settings
gcloud config list
gcloud projects describe $PROJECT_ID
```

### API Management

```bash
# Enable required APIs
REQUIRED_APIS=(
    "iam.googleapis.com"
    "cloudresourcemanager.googleapis.com"
    "storage.googleapis.com"
    "cloudkms.googleapis.com"
    "secretmanager.googleapis.com"
    "cloudfunctions.googleapis.com"
    "pubsub.googleapis.com"
    "monitoring.googleapis.com"
    "logging.googleapis.com"
    "compute.googleapis.com"
)

for api in "${REQUIRED_APIS[@]}"; do
    echo "Enabling $api..."
    gcloud services enable "$api"
done

# List enabled services
gcloud services list --enabled

# Check API status
gcloud services list --enabled --filter="name:iam.googleapis.com"
```

### Workload Identity Federation

```bash
# Create Workload Identity Pool
gcloud iam workload-identity-pools create "federated-identity-pool" \
    --location="global" \
    --display-name="Multi-Cloud Federated Identity Pool" \
    --description="Workload identity pool for federated authentication"

# List workload identity pools
gcloud iam workload-identity-pools list --location=global

# Create GitHub Actions provider
gcloud iam workload-identity-pools providers create-oidc "github-actions" \
    --workload-identity-pool="federated-identity-pool" \
    --location="global" \
    --issuer-uri="https://token.actions.githubusercontent.com" \
    --allowed-audiences="https://iam.googleapis.com/projects/$PROJECT_ID/locations/global/workloadIdentityPools/federated-identity-pool/providers/github-actions" \
    --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository,attribute.ref=assertion.ref" \
    --attribute-condition="assertion.repository_owner == 'komodt01' && assertion.repository == 'komodt01/Federated-Identity' && assertion.ref == 'refs/heads/main'"

# List providers
gcloud iam workload-identity-pools providers list \
    --workload-identity-pool="federated-identity-pool" \
    --location="global"

# Get provider details
gcloud iam workload-identity-pools providers describe "github-actions" \
    --workload-identity-pool="federated-identity-pool" \
    --location="global"
```

### Service Account Management

```bash
# Create service account
gcloud iam service-accounts create "federated-identity-sa" \
    --display-name="Federated Identity Service Account" \
    --description="Service account for GitHub Actions federated identity"

# Get service account email
SA_EMAIL="federated-identity-sa@$PROJECT_ID.iam.gserviceaccount.com"

# Add IAM policy binding for workload identity
gcloud iam service-accounts add-iam-policy-binding "$SA_EMAIL" \
    --role="roles/iam.workloadIdentityUser" \
    --member="principalSet://iam.googleapis.com/projects/$PROJECT_ID/locations/global/workloadIdentityPools/federated-identity-pool/attribute.repository/komodt01/Federated-Identity"

# Grant permissions to service account
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
    --member="serviceAccount:$SA_EMAIL" \
    --role="roles/storage.admin"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
    --member="serviceAccount:$SA_EMAIL" \
    --role="roles/secretmanager.admin"

# List service account bindings
gcloud iam service-accounts get-iam-policy "$SA_EMAIL"
```

### Storage and Secrets

```bash
# Create storage bucket
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
BUCKET_NAME="federated-identity-bucket-$PROJECT_NUMBER"

gsutil mb -p "$PROJECT_ID" -l "us-central1" "gs://$BUCKET_NAME"
gsutil uniformbucketlevelaccess set on "gs://$BUCKET_NAME"
gsutil versioning set on "gs://$BUCKET_NAME"

# Create secret
echo '{"github_org":"komodt01","github_repo":"Federated-Identity"}' | \
gcloud secrets create "federated-identity-config" --data-file=-

# Access secret
gcloud secrets versions access latest --secret="federated-identity-config"

# List storage buckets
gsutil ls -p "$PROJECT_ID"

# Upload test file
echo "Hello from GCP" > test-file.txt
gsutil cp test-file.txt "gs://$BUCKET_NAME/"
```

## GitHub Actions Commands

### Repository Setup

```bash
# Clone repository
git clone https://github.com/komodt01/Federated-Identity.git
cd Federated-Identity

# Create workflows directory
mkdir -p .github/workflows

# Create multi-cloud workflow
cat > .github/workflows/multi-cloud-test.yml << 'EOF'
name: Multi-Cloud Federated Identity Test
on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  test-aws:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::248830917924:role/FederatedIdentityRole
          aws-region: us-east-1
      - run: aws sts get-caller-identity

  test-azure:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v1
        with:
          client-id: dadacdd1-d1d1-4e91-ad5b-f73c0906c0af
          tenant-id: 94e6e1f2-3fce-43af-88d0-8fc30a31c1d6
          subscription-id: 7d939770-deef-433d-9283-5bd5eb79aeaf
      - run: az account show

  test-gcp:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: google-github-actions/auth@v1
        with:
          workload_identity_provider: 'projects/596237948242/locations/global/workloadIdentityPools/federated-identity-pool/providers/github-actions'
          service_account: 'federated-identity-sa@federated-465114.iam.gserviceaccount.com'
      - run: gcloud auth list
EOF

# Commit and push
git add .github/workflows/multi-cloud-test.yml
git commit -m "Add multi-cloud federated identity test workflow"
git push origin main
```

### Manual Token Testing

```bash
# Test OIDC token generation (run within GitHub Actions)
curl -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
     -H "Accept: application/json; api-version=2.0" \
     -H "Content-Type: application/json" \
     "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=sts.amazonaws.com" | jq

# Decode JWT token (for debugging)
# Note: Replace TOKEN with actual token
TOKEN="eyJhbGciOiJSUzI1NiJ9..."
echo "$TOKEN" | cut -d. -f2 | base64 -d | jq

# Get GitHub OIDC configuration
curl -s https://token.actions.githubusercontent.com/.well-known/openid_configuration | jq

# Get GitHub JWKS (public keys)
curl -s https://token.actions.githubusercontent.com/.well-known/jwks | jq
```

## Verification and Testing

### Complete System Test Script

```bash
#!/bin/bash
# complete-test.sh - Test all cloud providers

set -e

echo "🚀 Starting multi-cloud federated identity tests..."

# Test AWS
echo "Testing AWS..."
AWS_ACCOUNT_ID="248830917924"
AWS_ROLE_ARN="arn:aws:iam::$AWS_ACCOUNT_ID:role/FederatedIdentityRole"
AWS_BUCKET="federated-identity-bucket-$AWS_ACCOUNT_ID"

aws iam get-role --role-name "FederatedIdentityRole" >/dev/null && echo "✅ AWS IAM Role exists"
aws s3 ls "s3://$AWS_BUCKET" >/dev/null && echo "✅ AWS S3 Bucket accessible"
aws iam get-open-id-connect-provider --open-id-connect-provider-arn "arn:aws:iam::$AWS_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com" >/dev/null && echo "✅ AWS OIDC Provider exists"

# Test Azure
echo "Testing Azure..."
AZURE_APP_ID="dadacdd1-d1d1-4e91-ad5b-f73c0906c0af"
AZURE_SP_ID="97f75a53-b22e-4c3f-968e-95ecab4df516"

az ad app show --id "$AZURE_APP_ID" >/dev/null && echo "✅ Azure App Registration exists"
az ad sp show --id "$AZURE_SP_ID" >/dev/null && echo "✅ Azure Service Principal exists"
az ad app federated-credential list --id "$AZURE_APP_ID" --query "length(@)" -o tsv | grep -q "1" && echo "✅ Azure Federated Credentials configured"

# Test GCP
echo "Testing GCP..."
GCP_PROJECT="federated-465114"
GCP_SA="federated-identity-sa@$GCP_PROJECT.iam.gserviceaccount.com"

gcloud iam workload-identity-pools describe "federated-identity-pool" --location=global >/dev/null && echo "✅ GCP Workload Identity Pool exists"
gcloud iam service-accounts describe "$GCP_SA" >/dev/null && echo "✅ GCP Service Account exists"
gcloud iam workload-identity-pools providers describe "github-actions" --workload-identity-pool="federated-identity-pool" --location=global >/dev/null && echo "✅ GCP OIDC Provider exists"

echo "🎉 All tests completed successfully!"
```

### Individual Cloud Tests

```bash
# Test AWS components
test-aws.sh() {
    echo "Testing AWS federated identity..."
    
    # Test OIDC provider
    aws iam get-open-id-connect-provider \
        --open-id-connect-provider-arn "arn:aws:iam::248830917924:oidc-provider/token.actions.githubusercontent.com"
    
    # Test IAM role
    aws iam get-role --role-name "FederatedIdentityRole"
    
    # Test S3 bucket
    aws s3 ls s3://federated-identity-bucket-248830917924
    
    # Test role policies
    aws iam list-attached-role-policies --role-name "FederatedIdentityRole"
    aws iam list-role-policies --role-name "FederatedIdentityRole"
}

# Test Azure components  
test-azure.sh() {
    echo "Testing Azure federated identity..."
    
    APP_ID="dadacdd1-d1d1-4e91-ad5b-f73c0906c0af"
    
    # Test app registration
    az ad app show --id "$APP_ID"
    
    # Test federated credentials
    az ad app federated-credential list --id "$APP_ID"
    
    # Test resources
    az group show --name "rg-federated-identity"
    az keyvault list --resource-group "rg-federated-identity"
    az storage account list --resource-group "rg-federated-identity"
}

# Test GCP components
test-gcp.sh() {
    echo "Testing GCP federated identity..."
    
    # Test workload identity pool
    gcloud iam workload-identity-pools describe "federated-identity-pool" --location=global
    
    # Test provider
    gcloud iam workload-identity-pools providers describe "github-actions" \
        --workload-identity-pool="federated-identity-pool" --location=global
    
    # Test service account
    gcloud iam service-accounts describe "federated-identity-sa@federated-465114.iam.gserviceaccount.com"
    
    # Test bucket
    gsutil ls gs://federated-identity-bucket-596237948242
}
```

### Network Connectivity Tests

```bash
# Test OIDC endpoints connectivity
test-oidc-connectivity() {
    echo "Testing OIDC endpoint connectivity..."
    
    # GitHub OIDC endpoints
    curl -sI https://token.actions.githubusercontent.com/.well-known/openid_configuration
    curl -sI https://token.actions.githubusercontent.com/.well-known/jwks
    
    # Cloud provider STS endpoints
    curl -sI https://sts.amazonaws.com/
    curl -sI https://login.microsoftonline.com/common/oauth2/v2.0/token
    curl -sI https://sts.googleapis.com/v1/token
    
    echo "✅ All endpoints reachable"
}

# Test DNS resolution
test-dns-resolution() {
    echo "Testing DNS resolution..."
    
    dig token.actions.githubusercontent.com +short
    dig sts.amazonaws.com +short
    dig login.microsoftonline.com +short
    dig sts.googleapis.com +short
    
    echo "✅ DNS resolution working"
}

# Test SSL/TLS connectivity
test-ssl-connectivity() {
    echo "Testing SSL/TLS connectivity..."
    
    openssl s_client -connect token.actions.githubusercontent.com:443 -verify_return_error < /dev/null
    openssl s_client -connect sts.amazonaws.com:443 -verify_return_error < /dev/null
    openssl s_client -connect login.microsoftonline.com:443 -verify_return_error < /dev/null
    
    echo "✅ SSL/TLS connectivity working"
}
```

## Monitoring and Troubleshooting

### Log Analysis Commands

```bash
# AWS CloudTrail logs analysis
aws logs describe-log-groups --log-group-name-prefix "/aws/cloudtrail"

# Filter CloudTrail events for STS AssumeRoleWithWebIdentity
aws logs filter-log-events \
    --log-group-name "/aws/cloudtrail/insights" \
    --filter-pattern "{ $.eventName = AssumeRoleWithWebIdentity }"

# Azure Activity Log analysis
az monitor activity-log list \
    --resource-group "rg-federated-identity" \
    --start-time "2025-01-01T00:00:00Z" \
    --end-time "2025-12-31T23:59:59Z"

# Filter for authentication events
az monitor activity-log list \
    --caller "dadacdd1-d1d1-4e91-ad5b-f73c0906c0af" \
    --status "Succeeded"

# GCP Cloud Audit Logs
gcloud logging read "protoPayload.serviceName=iam.googleapis.com AND protoPayload.methodName=google.iam.v1.IAMCredentialsService.GenerateStsAccessToken" \
    --limit=50 \
    --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.resourceName)"

# Real-time log streaming
gcloud logging tail "resource.type=gce_instance" --format="value(jsonPayload.message)"
```

### Performance Monitoring

```bash
# Measure authentication latency
measure-auth-latency() {
    local start_time=$(date +%s%3N)
    
    # AWS
    aws sts get-caller-identity >/dev/null
    local aws_time=$(($(date +%s%3N) - start_time))
    
    # Azure
    local start_time=$(date +%s%3N)
    az account show >/dev/null
    local azure_time=$(($(date +%s%3N) - start_time))
    
    # GCP
    local start_time=$(date +%s%3N)
    gcloud auth list >/dev/null
    local gcp_time=$(($(date +%s%3N) - start_time))
    
    echo "Authentication latency (illustrative measurement):"
    echo "AWS: ${aws_time}ms"
    echo "Azure: ${azure_time}ms"
    echo "GCP: ${gcp_time}ms"
    echo "Note: Actual latency varies by network conditions and geographic location"
}

# Monitor token usage
monitor-token-usage() {
    echo "Monitoring token usage patterns..."
    
    # AWS token usage
    aws cloudtrail lookup-events \
        --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity \
        --start-time "$(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%SZ)" \
        --query 'Events[].{Time:EventTime,User:Username,Source:SourceIPAddress}'
    
    # Azure token usage
    az monitor activity-log list \
        --resource-group "rg-federated-identity" \
        --start-time "$(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%SZ)" \
        --query '[].{Time:eventTimestamp,User:caller,Operation:operationName.value}'
}

# Check service health
check-service-health() {
    echo "Checking cloud service health..."
    
    # AWS Service Health
    aws health describe-events --filter "services=iam,sts"
    
    # Azure Service Health
    az rest --method GET --url "https://management.azure.com/subscriptions/$(az account show --query id -o tsv)/providers/Microsoft.ResourceHealth/events?api-version=2020-05-01"
    
    # GCP Service Health
    gcloud compute operations list --filter="operationType=compute.globalOperations.get"
}
```

### Troubleshooting Commands

```bash
# Debug token validation issues
debug-token-validation() {
    echo "Debugging token validation..."
    
    # Check GitHub OIDC configuration
    curl -s https://token.actions.githubusercontent.com/.well-known/openid_configuration | jq .
    
    # Check JWKS keys
    curl -s https://token.actions.githubusercontent.com/.well-known/jwks | jq .
    
    # Validate token structure (replace with actual token)
    local token="$1"
    echo "$token" | cut -d. -f1 | base64 -d | jq . # Header
    echo "$token" | cut -d. -f2 | base64 -d | jq . # Payload
}

# Check permissions
debug-permissions() {
    echo "Debugging permissions..."
    
    # AWS permissions
    aws iam simulate-principal-policy \
        --policy-source-arn "arn:aws:iam::248830917924:role/FederatedIdentityRole" \
        --action-names "s3:GetObject" \
        --resource-arns "arn:aws:s3:::federated-identity-bucket-248830917924/*"
    
    # Azure permissions
    az role assignment list \
        --assignee "97f75a53-b22e-4c3f-968e-95ecab4df516" \
        --output table
    
    # GCP permissions
    gcloud projects get-iam-policy "federated-465114" \
        --flatten="bindings[].members" \
        --format="table(bindings.role)" \
        --filter="bindings.members:federated-identity-sa@federated-465114.iam.gserviceaccount.com"
}

# Network troubleshooting
debug-network() {
    echo "Debugging network connectivity..."
    
    # Check DNS resolution
    nslookup token.actions.githubusercontent.com
    nslookup sts.amazonaws.com
    nslookup login.microsoftonline.com
    nslookup sts.googleapis.com
    
    # Check port connectivity
    nc -zv token.actions.githubusercontent.com 443
    nc -zv sts.amazonaws.com 443
    nc -zv login.microsoftonline.com 443
    nc -zv sts.googleapis.com 443
    
    # Trace route to endpoints
    traceroute token.actions.githubusercontent.com
    traceroute sts.amazonaws.com
}
```

### Automated Monitoring Scripts

```bash
# Continuous monitoring script
continuous-monitor.sh() {
    while true; do
        echo "$(date): Checking federated identity health..."
        
        # Quick health check
        aws sts get-caller-identity >/dev/null 2>&1 && echo "✅ AWS OK" || echo "❌ AWS FAIL"
        az account show >/dev/null 2>&1 && echo "✅ Azure OK" || echo "❌ Azure FAIL"  
        gcloud auth list --filter="status:ACTIVE" >/dev/null 2>&1 && echo "✅ GCP OK" || echo "❌ GCP FAIL"
        
        sleep 300 # Check every 5 minutes
    done
}

# Alert on failures
alert-on-failure.sh() {
    local service="$1"
    local command="$2"
    
    if ! eval "$command" >/dev/null 2>&1; then
        echo "ALERT: $service authentication failed at $(date)"
        # Send alert (webhook, email, etc.)
        curl -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"🚨 $service federated identity failure at $(date)\"}" \
            "$SLACK_WEBHOOK_URL"
    fi
}
```

## Cleanup Commands

### AWS Cleanup

```bash
# Complete AWS cleanup
cleanup-aws() {
    echo "Cleaning up AWS resources..."
    
    # Detach policies from role
    aws iam detach-role-policy \
        --role-name "FederatedIdentityRole" \
        --policy-arn "arn:aws:iam::aws:policy/ReadOnlyAccess"
    
    # Delete inline policies
    aws iam delete-role-policy \
        --role-name "FederatedIdentityRole" \
        --policy-name "S3AccessPolicy"
    
    # Delete managed policy
    aws iam delete-policy \
        --policy-arn "arn:aws:iam::248830917924:policy/FederatedIdentityS3Policy"
    
    # Delete IAM role
    aws iam delete-role --role-name "FederatedIdentityRole"
    
    # Delete OIDC provider
    aws iam delete-open-id-connect-provider \
        --open-id-connect-provider-arn "arn:aws:iam::248830917924:oidc-provider/token.actions.githubusercontent.com"
    
    # Delete S3 bucket (empty first)
    aws s3 rm s3://federated-identity-bucket-248830917924 --recursive
    aws s3 rb s3://federated-identity-bucket-248830917924
    
    # Delete CloudTrail
    aws cloudtrail stop-logging --name "FederatedIdentityTrail"
    aws cloudtrail delete-trail --name "FederatedIdentityTrail"
    
    echo "✅ AWS cleanup completed"
}
```

### Azure Cleanup

```bash
# Complete Azure cleanup
cleanup-azure() {
    echo "Cleaning up Azure resources..."
    
    APP_ID="dadacdd1-d1d1-4e91-ad5b-f73c0906c0af"
    
    # Delete resource group (removes all resources)
    az group delete --name "rg-federated-identity" --yes --no-wait
    
    # Delete federated credentials
    az ad app federated-credential delete \
        --id "$APP_ID" \
        --federated-credential-id "github-actions-credential"
    
    # Delete service principal
    az ad sp delete --id "$APP_ID"
    
    # Delete application registration
    az ad app delete --id "$APP_ID"
    
    echo "✅ Azure cleanup completed"
}
```

### GCP Cleanup

```bash
# Complete GCP cleanup
cleanup-gcp() {
    echo "Cleaning up GCP resources..."
    
    PROJECT_ID="federated-465114"
    SA_EMAIL="federated-identity-sa@$PROJECT_ID.iam.gserviceaccount.com"
    
    # Remove IAM bindings
    gcloud projects remove-iam-policy-binding "$PROJECT_ID" \
        --member="serviceAccount:$SA_EMAIL" \
        --role="roles/storage.admin"
    
    gcloud projects remove-iam-policy-binding "$PROJECT_ID" \
        --member="serviceAccount:$SA_EMAIL" \
        --role="roles/secretmanager.admin"
    
    # Delete service account
    gcloud iam service-accounts delete "$SA_EMAIL" --quiet
    
    # Delete workload identity provider
    gcloud iam workload-identity-pools providers delete "github-actions" \
        --workload-identity-pool="federated-identity-pool" \
        --location="global" --quiet
    
    # Delete workload identity pool
    gcloud iam workload-identity-pools delete "federated-identity-pool" \
        --location="global" --quiet
    
    # Delete storage bucket
    gsutil rm -r gs://federated-identity-bucket-596237948242
    
    # Delete secrets
    gcloud secrets delete "federated-identity-config" --quiet
    
    echo "✅ GCP cleanup completed"
}
```

### Complete Cleanup Script

```bash
#!/bin/bash
# complete-cleanup.sh - Remove all federated identity resources

cleanup-all() {
    echo "🧹 Starting complete cleanup of multi-cloud federated identity..."
    
    read -p "Are you sure you want to delete ALL resources? (yes/no): " confirm
    if [[ $confirm != "yes" ]]; then
        echo "Cleanup cancelled"
        exit 0
    fi
    
    # Cleanup in parallel
    cleanup-aws &
    cleanup-azure &
    cleanup-gcp &
    
    wait # Wait for all cleanup jobs to complete
    
    echo "🎉 Complete cleanup finished!"
    echo "All federated identity resources have been removed."
}

# Emergency cleanup (force delete everything)
emergency-cleanup() {
    echo "🚨 Emergency cleanup - force deleting all resources..."
    
    # Force delete with minimal error checking
    aws iam delete-role --role-name "FederatedIdentityRole" 2>/dev/null || true
    az group delete --name "rg-federated-identity" --yes --no-wait 2>/dev/null || true
    gcloud iam workload-identity-pools delete "federated-identity-pool" --location="global" --quiet 2>/dev/null || true
    
    echo "Emergency cleanup completed"
}
```

### Verification After Cleanup

```bash
# Verify cleanup completion
verify-cleanup() {
    echo "Verifying cleanup completion..."
    
    # Verify AWS cleanup
    if aws iam get-role --role-name "FederatedIdentityRole" 2>/dev/null; then
        echo "❌ AWS role still exists"
    else
        echo "✅ AWS resources cleaned up"
    fi
    
    # Verify Azure cleanup
    if az group show --name "rg-federated-identity" 2>/dev/null; then
        echo "❌ Azure resource group still exists"
    else
        echo "✅ Azure resources cleaned up"
    fi
    
    # Verify GCP cleanup
    if gcloud iam workload-identity-pools describe "federated-identity-pool" --location=global 2>/dev/null; then
        echo "❌ GCP workload identity pool still exists"
    else
        echo "✅ GCP resources cleaned up"
    fi
    
    echo "Cleanup verification completed"
}
```

This comprehensive Linux commands reference provides all the necessary commands to deploy, manage, monitor, troubleshoot, and clean up the multi-cloud federated identity solution. Each section includes practical examples and real commands that were used in the actual implementation.
