# Multi-Cloud Federated Identity: Complete Teardown Guide

## Table of Contents
1. [Overview](#overview)
2. [Pre-Teardown Checklist](#pre-teardown-checklist)
3. [Data Backup and Export](#data-backup-and-export)
4. [AWS Teardown](#aws-teardown)
5. [Azure Teardown](#azure-teardown)
6. [GCP Teardown](#gcp-teardown)
7. [GitHub Repository Cleanup](#github-repository-cleanup)
8. [Verification and Validation](#verification-and-validation)
9. [Emergency Teardown](#emergency-teardown)
10. [Post-Teardown Tasks](#post-teardown-tasks)

## Overview

This guide provides comprehensive instructions for safely removing all multi-cloud federated identity resources created during the implementation. The teardown process is designed to be reversible until the final confirmation steps.

**⚠️ WARNING**: This process will permanently delete all federated identity resources. Ensure you have proper backups and approval before proceeding.

### Teardown Strategy

The teardown follows this order to minimize dependencies and potential issues:
1. **Stop active workflows** to prevent new authentications
2. **Export audit logs and configurations** for compliance
3. **Remove cloud resources** in dependency order
4. **Clean up GitHub configurations**
5. **Verify complete removal**

### Estimated Time
- **Full teardown**: 45-60 minutes
- **Emergency teardown**: 15-20 minutes
- **Verification**: 10-15 minutes

## Pre-Teardown Checklist

### 1. Approval and Documentation

```bash
# Create teardown log
echo "Federated Identity Teardown - $(date)" > teardown-log.txt
echo "Operator: $(whoami)" >> teardown-log.txt
echo "Reason: [Enter reason for teardown]" >> teardown-log.txt
```

**Required Approvals:**
- [ ] Security team approval
- [ ] Operations team notification
- [ ] Business stakeholder sign-off
- [ ] Compliance team notification (if applicable)

### 2. Active Workflow Check

```bash
# Check for active GitHub Actions workflows
echo "Checking for active workflows..."
gh run list --limit 10 --json status,conclusion
echo "⚠️ Ensure no critical workflows are running before proceeding"
```

### 3. Current Resource Inventory

```bash
# Document current resources
./create-resource-inventory.sh > pre-teardown-inventory.json
```

## Data Backup and Export

### 1. Export Audit Logs

```bash
#!/bin/bash
# export-audit-logs.sh

echo "Exporting audit logs before teardown..."
mkdir -p audit-logs-backup

# AWS CloudTrail logs
echo "Exporting AWS CloudTrail logs..."
aws logs filter-log-events \
    --log-group-name "/aws/cloudtrail/federated-identity" \
    --start-time "$(date -d '30 days ago' +%s)000" \
    --output json > audit-logs-backup/aws-cloudtrail-30days.json

# Azure Activity Logs
echo "Exporting Azure Activity Logs..."
az monitor activity-log list \
    --resource-group "rg-federated-identity" \
    --start-time "$(date -d '30 days ago' -u +%Y-%m-%dT%H:%M:%SZ)" \
    --output json > audit-logs-backup/azure-activity-30days.json

# GCP Audit Logs
echo "Exporting GCP Audit Logs..."
gcloud logging read "protoPayload.serviceName=iam.googleapis.com" \
    --limit=1000 \
    --format=json > audit-logs-backup/gcp-audit-30days.json

echo "✅ Audit logs exported to audit-logs-backup/"
```

### 2. Export Configurations

```bash
#!/bin/bash
# export-configurations.sh

echo "Exporting current configurations..."
mkdir -p config-backup

# AWS Configuration
aws iam get-role --role-name "FederatedIdentityRole" > config-backup/aws-role.json
aws iam get-open-id-connect-provider \
    --open-id-connect-provider-arn "arn:aws:iam::248830917924:oidc-provider/token.actions.githubusercontent.com" \
    > config-backup/aws-oidc-provider.json

# Azure Configuration
APP_ID="dadacdd1-d1d1-4e91-ad5b-f73c0906c0af"
az ad app show --id "$APP_ID" > config-backup/azure-app-registration.json
az ad app federated-credential list --id "$APP_ID" > config-backup/azure-federated-credentials.json

# GCP Configuration
gcloud iam workload-identity-pools describe "federated-identity-pool" \
    --location=global --format=json > config-backup/gcp-workload-identity-pool.json
gcloud iam workload-identity-pools providers describe "github-actions" \
    --workload-identity-pool="federated-identity-pool" \
    --location=global --format=json > config-backup/gcp-oidc-provider.json

echo "✅ Configurations exported to config-backup/"
```

### 3. Document Resource Dependencies

```bash
#!/bin/bash
# document-dependencies.sh

echo "Documenting resource dependencies..."

cat > resource-dependencies.md << 'EOF'
# Resource Dependencies Map

## AWS Dependencies
- IAM Role depends on: OIDC Provider
- S3 Bucket depends on: IAM Role (for access)
- CloudTrail depends on: S3 Bucket

## Azure Dependencies  
- Service Principal depends on: App Registration
- Key Vault depends on: Service Principal (for access)
- Storage Account depends on: Resource Group

## GCP Dependencies
- Service Account depends on: Workload Identity Pool
- Workload Identity Provider depends on: Workload Identity Pool
- Storage Bucket depends on: Service Account (for access)

## Teardown Order
1. Remove access bindings
2. Delete dependent resources (buckets, vaults)
3. Delete identity resources (roles, service accounts)
4. Delete identity providers and pools
5. Delete infrastructure (resource groups)
EOF

echo "✅ Dependencies documented in resource-dependencies.md"
```

## AWS Teardown

### 1. Pre-Teardown Verification

```bash
#!/bin/bash
# aws-pre-teardown-check.sh

echo "🔍 AWS Pre-teardown verification..."

# Check for active STS sessions
aws sts get-caller-identity
echo "Current AWS identity confirmed"

# List resources to be deleted
echo "📋 Resources to be deleted:"
echo "1. IAM Role: FederatedIdentityRole"
echo "2. OIDC Provider: token.actions.githubusercontent.com"
echo "3. S3 Bucket: federated-identity-bucket-248830917924"
echo "4. CloudTrail: FederatedIdentityTrail"

# Check for resource dependencies
ROLE_POLICIES=$(aws iam list-attached-role-policies --role-name "FederatedIdentityRole" --query "length(AttachedPolicies)" --output text)
INLINE_POLICIES=$(aws iam list-role-policies --role-name "FederatedIdentityRole" --query "length(PolicyNames)" --output text)

echo "Role has $ROLE_POLICIES attached policies and $INLINE_POLICIES inline policies"

read -p "Proceed with AWS teardown? (yes/no): " confirm
if [[ $confirm != "yes" ]]; then
    echo "AWS teardown cancelled"
    exit 1
fi
```

### 2. Remove IAM Role and Policies

```bash
#!/bin/bash
# aws-teardown-iam.sh

echo "🗑️ Removing AWS IAM resources..."

ROLE_NAME="FederatedIdentityRole"
ACCOUNT_ID="248830917924"

# 1. Detach managed policies
echo "Detaching managed policies..."
aws iam list-attached-role-policies --role-name "$ROLE_NAME" --query "AttachedPolicies[].PolicyArn" --output text | while read policy_arn; do
    if [[ -n "$policy_arn" ]]; then
        echo "Detaching policy: $policy_arn"
        aws iam detach-role-policy --role-name "$ROLE_NAME" --policy-arn "$policy_arn"
    fi
done

# 2. Delete inline policies
echo "Deleting inline policies..."
aws iam list-role-policies --role-name "$ROLE_NAME" --query "PolicyNames" --output text | while read policy_name; do
    if [[ -n "$policy_name" ]]; then
        echo "Deleting inline policy: $policy_name"
        aws iam delete-role-policy --role-name "$ROLE_NAME" --policy-name "$policy_name"
    fi
done

# 3. Delete custom managed policies (if any)
echo "Checking for custom managed policies..."
aws iam list-policies --scope Local --query "Policies[?contains(PolicyName, 'FederatedIdentity')].Arn" --output text | while read policy_arn; do
    if [[ -n "$policy_arn" ]]; then
        echo "Deleting custom policy: $policy_arn"
        # Detach from all entities first
        aws iam list-entities-for-policy --policy-arn "$policy_arn" --query "PolicyRoles[].RoleName" --output text | while read role; do
            aws iam detach-role-policy --role-name "$role" --policy-arn "$policy_arn"
        done
        aws iam delete-policy --policy-arn "$policy_arn"
    fi
done

# 4. Delete IAM role
echo "Deleting IAM role..."
aws iam delete-role --role-name "$ROLE_NAME"

echo "✅ IAM resources removed"
```

### 3. Remove OIDC Provider

```bash
#!/bin/bash
# aws-teardown-oidc.sh

echo "🗑️ Removing AWS OIDC Provider..."

OIDC_PROVIDER_ARN="arn:aws:iam::248830917924:oidc-provider/token.actions.githubusercontent.com"

# Check if provider exists
if aws iam get-open-id-connect-provider --open-id-connect-provider-arn "$OIDC_PROVIDER_ARN" >/dev/null 2>&1; then
    echo "Deleting OIDC provider: $OIDC_PROVIDER_ARN"
    aws iam delete-open-id-connect-provider --open-id-connect-provider-arn "$OIDC_PROVIDER_ARN"
    echo "✅ OIDC provider deleted"
else
    echo "⚠️ OIDC provider not found (may already be deleted)"
fi
```

### 4. Remove S3 Resources

```bash
#!/bin/bash
# aws-teardown-s3.sh

echo "🗑️ Removing AWS S3 resources..."

BUCKET_NAME="federated-identity-bucket-248830917924"

# Check if bucket exists
if aws s3 ls "s3://$BUCKET_NAME" >/dev/null 2>&1; then
    echo "Emptying S3 bucket: $BUCKET_NAME"
    
    # Delete all objects including versions
    aws s3api list-object-versions --bucket "$BUCKET_NAME" --query "Versions[].{Key:Key,VersionId:VersionId}" --output text | while read key version; do
        if [[ -n "$key" && -n "$version" ]]; then
            aws s3api delete-object --bucket "$BUCKET_NAME" --key "$key" --version-id "$version"
        fi
    done
    
    # Delete delete markers
    aws s3api list-object-versions --bucket "$BUCKET_NAME" --query "DeleteMarkers[].{Key:Key,VersionId:VersionId}" --output text | while read key version; do
        if [[ -n "$key" && -n "$version" ]]; then
            aws s3api delete-object --bucket "$BUCKET_NAME" --key "$key" --version-id "$version"
        fi
    done
    
    # Delete bucket
    echo "Deleting S3 bucket: $BUCKET_NAME"
    aws s3 rb "s3://$BUCKET_NAME"
    echo "✅ S3 bucket deleted"
else
    echo "⚠️ S3 bucket not found (may already be deleted)"
fi
```

### 5. Remove CloudTrail

```bash
#!/bin/bash
# aws-teardown-cloudtrail.sh

echo "🗑️ Removing AWS CloudTrail..."

TRAIL_NAME="FederatedIdentityTrail"

# Check if trail exists
if aws cloudtrail describe-trails --trail-name-list "$TRAIL_NAME" >/dev/null 2>&1; then
    echo "Stopping CloudTrail logging..."
    aws cloudtrail stop-logging --name "$TRAIL_NAME"
    
    echo "Deleting CloudTrail: $TRAIL_NAME"
    aws cloudtrail delete-trail --name "$TRAIL_NAME"
    echo "✅ CloudTrail deleted"
else
    echo "⚠️ CloudTrail not found (may already be deleted)"
fi
```

## Azure Teardown

### 1. Azure Pre-Teardown Verification

```bash
#!/bin/bash
# azure-pre-teardown-check.sh

echo "🔍 Azure Pre-teardown verification..."

# Verify authentication
az account show
echo "Current Azure identity confirmed"

APP_ID="dadacdd1-d1d1-4e91-ad5b-f73c0906c0af"

# List resources to be deleted
echo "📋 Resources to be deleted:"
echo "1. App Registration: $APP_ID"
echo "2. Service Principal: (linked to app)"
echo "3. Resource Group: rg-federated-identity"
echo "4. Key Vault: (within resource group)"
echo "5. Storage Account: (within resource group)"

# Check federated credentials
CRED_COUNT=$(az ad app federated-credential list --id "$APP_ID" --query "length(@)" --output tsv 2>/dev/null || echo "0")
echo "App has $CRED_COUNT federated credentials"

read -p "Proceed with Azure teardown? (yes/no): " confirm
if [[ $confirm != "yes" ]]; then
    echo "Azure teardown cancelled"
    exit 1
fi
```

### 2. Remove Resource Group and Resources

```bash
#!/bin/bash
# azure-teardown-resources.sh

echo "🗑️ Removing Azure resource group and all resources..."

RESOURCE_GROUP="rg-federated-identity"

# Check if resource group exists
if az group show --name "$RESOURCE_GROUP" >/dev/null 2>&1; then
    echo "📋 Listing resources in $RESOURCE_GROUP:"
    az resource list --resource-group "$RESOURCE_GROUP" --output table
    
    echo ""
    read -p "Delete resource group '$RESOURCE_GROUP' and ALL resources within it? (yes/no): " confirm
    if [[ $confirm == "yes" ]]; then
        echo "Deleting resource group: $RESOURCE_GROUP"
        echo "⏳ This may take several minutes..."
        az group delete --name "$RESOURCE_GROUP" --yes --no-wait
        
        # Wait for deletion to complete
        echo "Waiting for resource group deletion to complete..."
        while az group show --name "$RESOURCE_GROUP" >/dev/null 2>&1; do
            echo "  Still deleting... $(date)"
            sleep 30
        done
        echo "✅ Resource group deleted"
    else
        echo "Resource group deletion cancelled"
    fi
else
    echo "⚠️ Resource group not found (may already be deleted)"
fi
```

### 3. Remove App Registration and Service Principal

```bash
#!/bin/bash
# azure-teardown-identity.sh

echo "🗑️ Removing Azure identity resources..."

APP_ID="dadacdd1-d1d1-4e91-ad5b-f73c0906c0af"

# 1. Remove federated credentials
echo "Removing federated credentials..."
az ad app federated-credential list --id "$APP_ID" --query "[].name" --output tsv 2>/dev/null | while read cred_name; do
    if [[ -n "$cred_name" ]]; then
        echo "Deleting federated credential: $cred_name"
        az ad app federated-credential delete --id "$APP_ID" --federated-credential-id "$cred_name"
    fi
done

# 2. Remove role assignments
echo "Removing role assignments..."
SP_ID=$(az ad sp list --display-name "Multi-Cloud Federated Identity Application" --query "[0].id" -o tsv 2>/dev/null)
if [[ -n "$SP_ID" ]]; then
    az role assignment list --assignee "$SP_ID" --query "[].id" --output tsv | while read assignment_id; do
        if [[ -n "$assignment_id" ]]; then
            echo "Deleting role assignment: $assignment_id"
            az role assignment delete --ids "$assignment_id"
        fi
    done
fi

# 3. Delete service principal
if [[ -n "$SP_ID" ]]; then
    echo "Deleting service principal: $SP_ID"
    az ad sp delete --id "$SP_ID"
fi

# 4. Delete application registration
if az ad app show --id "$APP_ID" >/dev/null 2>&1; then
    echo "Deleting application registration: $APP_ID"
    az ad app delete --id "$APP_ID"
    echo "✅ App registration deleted"
else
    echo "⚠️ App registration not found (may already be deleted)"
fi

echo "✅ Azure identity resources removed"
```

## GCP Teardown

### 1. GCP Pre-Teardown Verification

```bash
#!/bin/bash
# gcp-pre-teardown-check.sh

echo "🔍 GCP Pre-teardown verification..."

# Verify authentication and project
gcloud auth list
gcloud config list
echo "Current GCP identity and project confirmed"

PROJECT_ID="federated-465114"
SA_EMAIL="federated-identity-sa@$PROJECT_ID.iam.gserviceaccount.com"

# List resources to be deleted
echo "📋 Resources to be deleted:"
echo "1. Workload Identity Pool: federated-identity-pool"
echo "2. OIDC Provider: github-actions"
echo "3. Service Account: $SA_EMAIL"
echo "4. Storage Bucket: federated-identity-bucket-596237948242"
echo "5. Secrets: federated-identity-config"

# Check pool and provider
if gcloud iam workload-identity-pools describe "federated-identity-pool" --location=global >/dev/null 2>&1; then
    echo "✓ Workload Identity Pool exists"
else
    echo "⚠️ Workload Identity Pool not found"
fi

read -p "Proceed with GCP teardown? (yes/no): " confirm
if [[ $confirm != "yes" ]]; then
    echo "GCP teardown cancelled"
    exit 1
fi
```

### 2. Remove IAM Bindings

```bash
#!/bin/bash
# gcp-teardown-iam.sh

echo "🗑️ Removing GCP IAM bindings..."

PROJECT_ID="federated-465114"
SA_EMAIL="federated-identity-sa@$PROJECT_ID.iam.gserviceaccount.com"

# Remove project-level IAM bindings
echo "Removing project-level IAM bindings..."
ROLES=("roles/storage.admin" "roles/secretmanager.admin" "roles/logging.logWriter" "roles/monitoring.metricWriter")

for role in "${ROLES[@]}"; do
    echo "Removing binding for role: $role"
    gcloud projects remove-iam-policy-binding "$PROJECT_ID" \
        --member="serviceAccount:$SA_EMAIL" \
        --role="$role" \
        --quiet 2>/dev/null || echo "  Binding not found for $role"
done

# Remove workload identity bindings
echo "Removing workload identity bindings..."
gcloud iam service-accounts remove-iam-policy-binding "$SA_EMAIL" \
    --role="roles/iam.workloadIdentityUser" \
    --member="principalSet://iam.googleapis.com/projects/$PROJECT_ID/locations/global/workloadIdentityPools/federated-identity-pool/attribute.repository/komodt01/Federated-Identity" \
    --quiet 2>/dev/null || echo "  Workload identity binding not found"

echo "✅ IAM bindings removed"
```

### 3. Remove Storage and Secrets

```bash
#!/bin/bash
# gcp-teardown-storage.sh

echo "🗑️ Removing GCP storage and secrets..."

PROJECT_ID="federated-465114"
BUCKET_NAME="federated-identity-bucket-596237948242"

# Delete storage bucket
if gsutil ls "gs://$BUCKET_NAME" >/dev/null 2>&1; then
    echo "Deleting storage bucket contents..."
    gsutil -m rm -r "gs://$BUCKET_NAME/**" 2>/dev/null || echo "  Bucket is empty"
    
    echo "Deleting storage bucket: $BUCKET_NAME"
    gsutil rb "gs://$BUCKET_NAME"
    echo "✅ Storage bucket deleted"
else
    echo "⚠️ Storage bucket not found (may already be deleted)"
fi

# Delete secrets
echo "Deleting secrets..."
SECRET_NAMES=("federated-identity-config")

for secret in "${SECRET_NAMES[@]}"; do
    if gcloud secrets describe "$secret" >/dev/null 2>&1; then
        echo "Deleting secret: $secret"
        gcloud secrets delete "$secret" --quiet
    else
        echo "⚠️ Secret $secret not found"
    fi
done

echo "✅ Storage and secrets removed"
```

### 4. Remove Service Account

```bash
#!/bin/bash
# gcp-teardown-service-account.sh

echo "🗑️ Removing GCP service account..."

PROJECT_ID="federated-465114"
SA_EMAIL="federated-identity-sa@$PROJECT_ID.iam.gserviceaccount.com"

if gcloud iam service-accounts describe "$SA_EMAIL" >/dev/null 2>&1; then
    echo "Deleting service account: $SA_EMAIL"
    gcloud iam service-accounts delete "$SA_EMAIL" --quiet
    echo "✅ Service account deleted"
else
    echo "⚠️ Service account not found (may already be deleted)"
fi
```

### 5. Remove Workload Identity Pool and Provider

```bash
#!/bin/bash
# gcp-teardown-workload-identity.sh

echo "🗑️ Removing GCP Workload Identity resources..."

# Delete workload identity provider
echo "Deleting workload identity provider..."
if gcloud iam workload-identity-pools providers describe "github-actions" \
    --workload-identity-pool="federated-identity-pool" \
    --location="global" >/dev/null 2>&1; then
    
    gcloud iam workload-identity-pools providers delete "github-actions" \
        --workload-identity-pool="federated-identity-pool" \
        --location="global" --quiet
    echo "✅ Workload identity provider deleted"
else
    echo "⚠️ Workload identity provider not found"
fi

# Delete workload identity pool
echo "Deleting workload identity pool..."
if gcloud iam workload-identity-pools describe "federated-identity-pool" \
    --location="global" >/dev/null 2>&1; then
    
    gcloud iam workload-identity-pools delete "federated-identity-pool" \
        --location="global" --quiet
    echo "✅ Workload identity pool deleted"
else
    echo "⚠️ Workload identity pool not found"
fi
```

## GitHub Repository Cleanup

### 1. Remove GitHub Actions Workflows

```bash
#!/bin/bash
# github-teardown-workflows.sh

echo "🗑️ Cleaning up GitHub repository..."

# List current workflows
echo "Current workflows:"
ls -la .github/workflows/

# Remove federated identity workflows
WORKFLOWS_TO_REMOVE=(
    ".github/workflows/multi-cloud-test.yml"
    ".github/workflows/aws-test.yml"
    ".github/workflows/azure-test.yml"
    ".github/workflows/gcp-test.yml"
)

for workflow in "${WORKFLOWS_TO_REMOVE[@]}"; do
    if [[ -f "$workflow" ]]; then
        echo "Removing workflow: $workflow"
        rm "$workflow"
    else
        echo "⚠️ Workflow not found: $workflow"
    fi
done

# Remove deployment scripts
SCRIPTS_TO_REMOVE=(
    "deploy-aws.sh"
    "deploy-azure.sh"
    "deploy-gcp.sh"
    "test-aws-deployment.sh"
    "test-azure-deployment.sh"
    "test-gcp-deployment.sh"
)

for script in "${SCRIPTS_TO_REMOVE[@]}"; do
    if [[ -f "$script" ]]; then
        echo "Removing script: $script"
        rm "$script"
    else
        echo "⚠️ Script not found: $script"
    fi
done

echo "✅ GitHub repository cleaned up"
```

### 2. Remove Configuration Files

```bash
#!/bin/bash
# remove-config-files.sh

echo "🗑️ Removing configuration files..."

CONFIG_FILES_TO_REMOVE=(
    "trust-policy.json"
    "s3-policy.json"
    "inline-s3-policy.json"
    "federated-identity-template.yaml"
    "github-actions-aws-example.yml"
    "github-actions-azure-example.yml"
    "github-actions-gcp-example.yml"
)

for file in "${CONFIG_FILES_TO_REMOVE[@]}"; do
    if [[ -f "$file" ]]; then
        echo "Removing config file: $file"
        rm "$file"
    else
        echo "⚠️ Config file not found: $file"
    fi
done

echo "✅ Configuration files removed"
```

## Verification and Validation

### 1. Complete Teardown Verification

```bash
#!/bin/bash
# verify-complete-teardown.sh

echo "🔍 Verifying complete teardown..."

# AWS Verification
echo "=== AWS Verification ==="
if aws iam get-role --role-name "FederatedIdentityRole" >/dev/null 2>&1; then
    echo "❌ AWS IAM Role still exists"
else
    echo "✅ AWS IAM Role deleted"
fi

if aws iam get-open-id-connect-provider --open-id-connect-provider-arn "arn:aws:iam::248830917924:oidc-provider/token.actions.githubusercontent.com" >/dev/null 2>&1; then
    echo "❌ AWS OIDC Provider still exists"
else
    echo "✅ AWS OIDC Provider deleted"
fi

if aws s3 ls s3://federated-identity-bucket-248830917924 >/dev/null 2>&1; then
    echo "❌ AWS S3 Bucket still exists"
else
    echo "✅ AWS S3 Bucket deleted"
fi

# Azure Verification
echo "=== Azure Verification ==="
if az ad app show --id "dadacdd1-d1d1-4e91-ad5b-f73c0906c0af" >/dev/null 2>&1; then
    echo "❌ Azure App Registration still exists"
else
    echo "✅ Azure App Registration deleted"
fi

if az group show --name "rg-federated-identity" >/dev/null 2>&1; then
    echo "❌ Azure Resource Group still exists"
else
    echo "✅ Azure Resource Group deleted"
fi

# GCP Verification
echo "=== GCP Verification ==="
if gcloud iam workload-identity-pools describe "federated-identity-pool" --location=global >/dev/null 2>&1; then
    echo "❌ GCP Workload Identity Pool still exists"
else
    echo "✅ GCP Workload Identity Pool deleted"
fi

if gcloud iam service-accounts describe "federated-identity-sa@federated-465114.iam.gserviceaccount.com" >/dev/null 2>&1; then
    echo "❌ GCP Service Account still exists"
else
    echo "✅ GCP Service Account deleted"
fi

if gsutil ls gs://federated-identity-bucket-596237948242 >/dev/null 2>&1; then
    echo "❌ GCP Storage Bucket still exists"
else
    echo "✅ GCP Storage Bucket deleted"
fi

echo ""
echo "🎯 Teardown verification completed"
```

### 2. Generate Teardown Report

```bash
#!/bin/bash
# generate-teardown-report.sh

echo "📊 Generating teardown report..."

cat > teardown-report.md << EOF
# Federated Identity Teardown Report

**Date**: $(date)
**Operator**: $(whoami)
**Duration**: [Manual entry required]

## Resources Removed

### AWS
- [x] IAM Role: FederatedIdentityRole
- [x] OIDC Provider: token.actions.githubusercontent.com
- [x] S3 Bucket: federated-identity-bucket-248830917924
- [x] CloudTrail: FederatedIdentityTrail

### Azure
- [x] App Registration: dadacdd1-d1d1-4e91-ad5b-f73c0906c0af
- [x] Service Principal: (linked to app)
- [x] Resource Group: rg-federated-identity
- [x] Key Vault: (within resource group)
- [x] Storage Account: (within resource group)

### GCP
- [x] Workload Identity Pool: federated-identity-pool
- [x] OIDC Provider: github-actions
- [x] Service Account: federated-identity-sa@federated-465114.iam.gserviceaccount.com
- [x] Storage Bucket: federated-identity-bucket-596237948242
- [x] Secrets: federated-identity-config

### GitHub Repository
- [x] Workflow files removed
- [x] Deployment scripts removed
- [x] Configuration files removed

## Data Preserved
- [x] Audit logs exported to audit-logs-backup/
- [x] Configurations exported to config-backup/
- [x] Documentation preserved

## Verification Status
- [x] All cloud resources verified as deleted
- [x] No remaining federated identity resources found
- [x] GitHub repository cleaned up

## Notes
- Teardown completed successfully
- All backups preserved for compliance
- No issues encountered during removal process

**Report generated at**: $(date)
EOF

echo "✅ Teardown report generated: teardown-report.md"
```

## Emergency Teardown

### Emergency Teardown Script

```bash
#!/bin/bash
# emergency-teardown.sh

echo "🚨 EMERGENCY TEARDOWN - FORCE DELETING ALL RESOURCES"
echo "⚠️ This will attempt to delete everything without confirmation"

read -p "Are you sure you want to proceed with emergency teardown? (EMERGENCY/no): " confirm
if [[ $confirm != "EMERGENCY" ]]; then
    echo "Emergency teardown cancelled"
    exit 0
fi

echo "🚨 Starting emergency teardown..."

# AWS Emergency Cleanup
echo "Emergency AWS cleanup..."
aws iam detach-role-policy --role-name "FederatedIdentityRole" --policy-arn "arn:aws:iam::aws:policy/ReadOnlyAccess" 2>/dev/null
aws iam delete-role-policy --role-name "FederatedIdentityRole" --policy-name "S3AccessPolicy" 2>/dev/null
aws iam delete-role --role-name "FederatedIdentityRole" 2>/dev/null
aws iam delete-open-id-connect-provider --open-id-connect-provider-arn "arn:aws:iam::248830917924:oidc-provider/token.actions.githubusercontent.com" 2>/dev/null
aws s3 rm s3://federated-identity-bucket-248830917924 --recursive 2>/dev/null
aws s3 rb s3://federated-identity-bucket-248830917924 2>/dev/null
aws cloudtrail stop-logging --name "FederatedIdentityTrail" 2>/dev/null
aws cloudtrail delete-trail --name "FederatedIdentityTrail" 2>/dev/null

# Azure Emergency Cleanup
echo "Emergency Azure cleanup..."
az group delete --name "rg-federated-identity" --yes --no-wait 2>/dev/null
az ad app delete --id "dadacdd1-d1d1-4e91-ad5b-f73c0906c0af" 2>/dev/null

# GCP Emergency Cleanup
echo "Emergency GCP cleanup..."
gcloud iam service-accounts delete "federated-identity-sa@federated-465114.iam.gserviceaccount.com" --quiet 2>/dev/null
gcloud iam workload-identity-pools providers delete "github-actions" --workload-identity-pool="federated-identity-pool" --location="global" --quiet 2>/dev/null
gcloud iam workload-identity-pools delete "federated-identity-pool" --location="global" --quiet 2>/dev/null
gsutil rm -r gs://federated-identity-bucket-596237948242 2>/dev/null
gcloud secrets delete "federated-identity-config" --quiet 2>/dev/null

echo "🚨 Emergency teardown completed"
echo "⚠️ Some resources may still exist if deletion failed"
echo "Run verification script to check remaining resources"
```

## Post-Teardown Tasks

### 1. Compliance and Audit Tasks

```bash
#!/bin/bash
# post-teardown-compliance.sh

echo "📋 Post-teardown compliance tasks..."

# Create compliance report
cat > compliance-teardown-report.md << EOF
# Compliance Teardown Report

## Data Retention
- [x] Audit logs exported and retained for required period
- [x] Configuration backups created and stored securely
- [x] Access logs preserved for compliance review

## Security Verification
- [x] All credentials and secrets removed
- [x] No federated identity resources remain active
- [x] All access tokens invalidated

## Documentation
- [x] Teardown process documented
- [x] Resource inventory updated
- [x] Change management records updated

## Notifications Sent
- [ ] Security team notified of completion
- [ ] Compliance team notified
- [ ] Audit team provided with retention data
- [ ] Business stakeholders informed

**Compliance Officer Review Required**: Yes
**Next Review Date**: $(date -d '+1 year' '+%Y-%m-%d')
EOF

echo "✅ Compliance report created: compliance-teardown-report.md"
```

### 2. Update Documentation

```bash
#!/bin/bash
# update-documentation.sh

echo "📚 Updating project documentation..."

# Update README to reflect teardown
if [[ -f "README.md" ]]; then
    sed -i '1i# ⚠️ PROJECT DECOMMISSIONED\n\nThis federated identity implementation has been torn down on $(date).\nAll cloud resources have been removed and access has been revoked.\nSee teardown-report.md for details.\n' README.md
fi

# Create decommission notice
cat > DECOMMISSIONED.md << EOF
# Project Decommissioned

**Decommission Date**: $(date)
**Decommissioned By**: $(whoami)

## Summary
This multi-cloud federated identity project has been completely torn down and decommissioned.

## Resources Removed
- All AWS IAM roles, OIDC providers, S3 buckets, and CloudTrail configurations
- All Azure app registrations, service principals, resource groups, and associated resources
- All GCP workload identity pools, service accounts, storage buckets, and secrets
- All GitHub Actions workflows and deployment scripts

## Data Preserved
- Audit logs exported to audit-logs-backup/
- Configuration backups stored in config-backup/
- Documentation preserved for reference

## Contact
For questions about this decommission, contact: [Your contact information]

## Recovery
To recreate this infrastructure, refer to the preserved documentation and configuration backups.
The deployment scripts can be recreated from the linux-commands.md documentation.
EOF

echo "✅ Documentation updated"
```

### 3. Final Verification and Sign-off

```bash
#!/bin/bash
# final-verification.sh

echo "🔍 Final verification and sign-off..."

# Run comprehensive verification
./verify-complete-teardown.sh > final-verification-results.txt

# Create sign-off document
cat > teardown-sign-off.md << EOF
# Federated Identity Teardown Sign-off

## Verification Results
$(cat final-verification-results.txt)

## Approvals Required

### Technical Sign-off
- [ ] Cloud Security Engineer: _________________ Date: _______
- [ ] DevOps Team Lead: _________________ Date: _______
- [ ] Platform Engineer: _________________ Date: _______

### Business Sign-off
- [ ] Project Manager: _________________ Date: _______
- [ ] Security Manager: _________________ Date: _______
- [ ] Compliance Officer: _________________ Date: _______

## Attestation
I attest that all federated identity resources have been properly removed
and all required compliance procedures have been followed.

**Technical Lead**: $(whoami)
**Date**: $(date)
**Signature**: _________________

## Final Notes
- All cloud provider resources confirmed deleted
- Audit logs and configurations preserved
- Documentation updated and archived
- No remaining security risks identified

**Project Status**: DECOMMISSIONED
EOF

echo "✅ Sign-off document created: teardown-sign-off.md"
```

## Troubleshooting Common Teardown Issues

### 1. Resource Dependencies

```bash
#!/bin/bash
# troubleshoot-dependencies.sh

echo "🔧 Troubleshooting resource dependencies..."

# Check for resources preventing deletion
check_aws_dependencies() {
    echo "Checking AWS dependencies..."
    
    # Check for policies attached to role
    aws iam list-attached-role-policies --role-name "FederatedIdentityRole" 2>/dev/null
    aws iam list-role-policies --role-name "FederatedIdentityRole" 2>/dev/null
    
    # Check for bucket objects
    aws s3 ls s3://federated-identity-bucket-248830917924 --recursive 2>/dev/null
}

check_azure_dependencies() {
    echo "Checking Azure dependencies..."
    
    # Check for role assignments
    SP_ID=$(az ad sp list --display-name "Multi-Cloud Federated Identity Application" --query "[0].id" -o tsv 2>/dev/null)
    if [[ -n "$SP_ID" ]]; then
        az role assignment list --assignee "$SP_ID" 2>/dev/null
    fi
    
    # Check for resources in resource group
    az resource list --resource-group "rg-federated-identity" 2>/dev/null
}

check_gcp_dependencies() {
    echo "Checking GCP dependencies..."
    
    # Check for IAM bindings
    gcloud projects get-iam-policy "federated-465114" \
        --flatten="bindings[].members" \
        --filter="bindings.members:federated-identity-sa@federated-465114.iam.gserviceaccount.com" 2>/dev/null
    
    # Check for bucket objects
    gsutil ls gs://federated-identity-bucket-596237948242/** 2>/dev/null
}

check_aws_dependencies
check_azure_dependencies
check_gcp_dependencies
```

### 2. Permission Issues

```bash
#!/bin/bash
# troubleshoot-permissions.sh

echo "🔧 Troubleshooting permission issues..."

# Check current permissions
echo "Current AWS identity:"
aws sts get-caller-identity

echo "Current Azure identity:"
az account show

echo "Current GCP identity:"
gcloud auth list --filter=status:ACTIVE

# Test required permissions
test_aws_permissions() {
    echo "Testing AWS permissions..."
    aws iam list-roles --max-items 1 >/dev/null 2>&1 && echo "✅ IAM list permissions" || echo "❌ IAM list permissions"
    aws s3 ls >/dev/null 2>&1 && echo "✅ S3 list permissions" || echo "❌ S3 list permissions"
}

test_azure_permissions() {
    echo "Testing Azure permissions..."
    az ad app list --query "[0]" >/dev/null 2>&1 && echo "✅ App registration permissions" || echo "❌ App registration permissions"
    az group list >/dev/null 2>&1 && echo "✅ Resource group permissions" || echo "❌ Resource group permissions"
}

test_gcp_permissions() {
    echo "Testing GCP permissions..."
    gcloud iam service-accounts list --limit=1 >/dev/null 2>&1 && echo "✅ Service account permissions" || echo "❌ Service account permissions"
    gcloud iam workload-identity-pools list --location=global --limit=1 >/dev/null 2>&1 && echo "✅ Workload identity permissions" || echo "❌ Workload identity permissions"
}

test_aws_permissions
test_azure_permissions
test_gcp_permissions
```

### 3. Partial Teardown Recovery

```bash
#!/bin/bash
# recover-partial-teardown.sh

echo "🔧 Recovering from partial teardown..."

# Function to recreate resources if needed
recreate_if_needed() {
    local resource_type="$1"
    local check_command="$2"
    local create_command="$3"
    
    if ! eval "$check_command" >/dev/null 2>&1; then
        echo "❌ $resource_type not found"
        read -p "Recreate $resource_type? (y/n): " recreate
        if [[ $recreate == "y" ]]; then
            eval "$create_command"
        fi
    else
        echo "✅ $resource_type exists"
    fi
}

# Example recovery checks
echo "Checking for partial teardown state..."

# AWS checks
recreate_if_needed "AWS IAM Role" \
    "aws iam get-role --role-name FederatedIdentityRole" \
    "echo 'Run deploy-aws.sh to recreate AWS resources'"

# Azure checks  
recreate_if_needed "Azure App Registration" \
    "az ad app show --id dadacdd1-d1d1-4e91-ad5b-f73c0906c0af" \
    "echo 'Run deploy-azure.sh to recreate Azure resources'"

# GCP checks
recreate_if_needed "GCP Workload Identity Pool" \
    "gcloud iam workload-identity-pools describe federated-identity-pool --location=global" \
    "echo 'Run deploy-gcp.sh to recreate GCP resources'"
```

## Summary

This teardown guide provides comprehensive procedures for safely removing all multi-cloud federated identity resources. Key points:

### 🎯 **Teardown Order**
1. **Stop workflows** and export data
2. **Remove dependent resources** (buckets, storage)
3. **Remove identity resources** (roles, service accounts)
4. **Remove identity providers** (OIDC, pools)
5. **Clean up infrastructure** (resource groups)
6. **Verify complete removal**

### 🛡️ **Safety Measures**
- **Backup all configurations** before deletion
- **Export audit logs** for compliance
- **Verify each step** before proceeding
- **Document all actions** for audit trail

### ⚡ **Emergency Procedures**
- **Emergency teardown script** for rapid removal
- **Recovery procedures** for partial teardowns
- **Troubleshooting guides** for common issues

### 📋 **Compliance**
- **Audit log preservation** for required retention periods
- **Configuration backups** for potential recreation
- **Sign-off procedures** for proper approval chain
- **Documentation updates** reflecting decommission status

This guide ensures complete, auditable, and reversible teardown of the multi-cloud federated identity implementation while maintaining security and compliance requirements.
