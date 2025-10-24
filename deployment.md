# Dify on AWS - Deployment Guide

A step-by-step guide to deploy Dify using AWS CDK with or without a custom domain.

## Branch Defaults

This branch is preconfigured for a simple, public deployment with CloudFront in front of an internal ALB:

- CloudFront: enabled and terminates TLS for the domain; Route 53 aliases the domain to CloudFront.
- ALB: private (not internet-facing) and only accepts traffic from CloudFront; origin uses HTTP.
- Access: public by default; no IP CIDR allowlists are configured. To restrict, add `allowedIPv4Cidrs` in `bin/cdk.ts`.

## Prerequisites

Before you begin, ensure you have:

1. **AWS CLI installed and configured**
   ```bash
   aws configure
   # Enter your Access Key ID, Secret Access Key, and default region
   ```

2. **Node.js (v18 or newer)**
   ```bash
   node --version  # Should be 18.x or higher
   ```

3. **Docker installed and running**
   ```bash
   docker --version
   ```

4. **AWS Account with Administrator permissions**

## Step 1: Clone and Setup

```bash
git clone https://github.com/aws-samples/dify-self-hosted-on-aws.git
cd dify-self-hosted-on-aws
npm install
```

## Step 2: Domain Setup

For this deployment, we do not use a Route 53 hosted zone for the subdomain but if you did, follow below.

### DNS Configuration

**Step 1**: Create Route 53 hosted zone for the subdomain:
```bash
# Create hosted zone for the subdomain
aws route53 create-hosted-zone \
  --name dify.geoprofessional.org \
  --caller-reference $(date +%s)
```

**Step 2**: At your **domain provider** (not AWS), create NS records to delegate the subdomain to Route 53:
```
Name: dify
Type: NS
Values: 
  ns-3.awsdns-00.com.
  ns-948.awsdns-54.net.
  ns-1410.awsdns-48.org.
  ns-1808.awsdns-34.co.uk.
```

**Step 3**: Configure CDK for the subdomain:
```typescript
export const props: EnvironmentProps = {
  awsRegion: 'us-west-2',
  awsAccount: process.env.CDK_DEFAULT_ACCOUNT!,
  difyImageTag: '1.9.1',
  
  // Domain settings
  domainName: 'dify.geoprofessional.org',  // The delegated subdomain
  
  // Cost optimizations (recommended)
  enableAuroraScalesToZero: true,
  useNatInstance: true,
  isRedisMultiAz: false,
  useFargateSpot: true,
};
```

**Result**: `https://dify.mydomain.org`

## Step 3: Optional Access Control

By default, the deployment is publicly accessible on the Internet. Use the options
below only if you want to restrict or change access behavior. To re-open access,
remove any IP allowlists you previously added.

### Restrict Access by IP Address (optional)

Add to your `bin/cdk.ts`:
```typescript
export const props: EnvironmentProps = {
  // ... your other config ...
  
  // Only allow access from specific IP addresses
  allowedIPv4Cidrs: [
    '203.0.113.0/24',    // Your office IP range
    '198.51.100.5/32',   // Your home IP
  ],
};
```

Notes:
- With CloudFront enabled (default), these ranges are enforced by a CloudFront WAF allow rule.
- Without CloudFront, these ranges are applied to the ALB listener security rules.
- To make access public again, remove `allowedIPv4Cidrs` (and `allowedIPv6Cidrs` if present),
  or set them to open ranges like `['0.0.0.0/0']` and `['::/0']`.

### Use Direct ALB (No CloudFront)

For internal/private deployments:
```typescript
export const props: EnvironmentProps = {
  // ... your other config ...
  
  useCloudFront: false,  // Direct ALB access
  // Requires domainName for HTTPS, otherwise HTTP only
};
```

Notes:
- With `useCloudFront: false` and no IP allowlist configured, the ALB listener allows
  public access (IPv4 `0.0.0.0/0`, IPv6 `::/0`). Add `allowedIPv4Cidrs`/`allowedIPv6Cidrs`
  to restrict if needed.

## Step 4: Deploy

### First Time Deployment

```bash
# Bootstrap your AWS account for CDK (one-time per account/region)
npx cdk bootstrap

# Deploy the stack
npx cdk deploy --all
```

The deployment takes about 15-20 minutes. You'll see progress updates in the terminal.

### Subsequent Deployments

```bash
# For configuration changes
npx cdk deploy --all
```

## Step 5: Get Your Dify URL

After successful deployment, you'll see output like:
```
Outputs:
DifyOnAwsStack.DifyUrl = https://dify.yourdomain.com
```

**Or without custom domain:**
```
DifyOnAwsStack.DifyUrl = https://d1234567890123.cloudfront.net
```

Visit this URL to access your Dify installation!

## Step 6: Initial Dify Setup

1. **Open your Dify URL** in a browser
2. **Create admin account** on first visit
3. **Configure Bedrock models** (optional):
   - Go to Settings → Model Provider → AWS Bedrock
   - Select your region and save
   - Models must be enabled in Bedrock console first

## Monthly Cost Estimate

With recommended cost optimizations:
- **Minimum**: ~$32-35/month (with auto-scaling to zero)
- **Typical light usage**: ~$40-50/month
- **Heavier usage**: Scales up based on ECS/database usage

## Common Issues & Solutions

### Certificate Validation Timeout
**Problem**: "Certificate validation timed out"
**Solution**: Ensure your domain's nameservers point to Route 53 and wait for DNS propagation (up to 48 hours).

### Permission Errors
**Problem**: "Access denied" during deployment
**Solution**: Ensure your AWS user/role has Administrator permissions or the specific permissions listed in the CDK documentation.

### Domain Not Found
**Problem**: "Hosted zone not found for domain"
**Solution**: Verify the domain exists in Route 53 in your account and region.

### CloudFormation Rollback
**Problem**: Stack creation failed and rolled back
**Solution**: Check CloudFormation console for specific error details. Common fixes:
- Verify domain ownership
- Check service quotas
- Ensure region has required services

## Using Custom Dify Images

If you've made custom modifications to Dify's source code, you can deploy your custom images instead of the official Docker Hub images.

### Quick Guide

1. **Build and push custom images to ECR**:
   ```bash
   cd /path/to/your/dify/docker
   ./push-to-ecr.sh 1.9.1-custom
   ```

2. **Enable custom images in `bin/cdk.ts`**:
   ```typescript
   const USE_CUSTOM_IMAGES = true;  // Change from false to true
   ```

3. **Deploy**:
   ```bash
   npx cdk deploy --all
   ```

### Switching Back to Official Images

To roll back to official Docker Hub images, simply change the flag:

```typescript
const USE_CUSTOM_IMAGES = false;  // Change from true to false
```

Then deploy again. This provides a quick rollback mechanism if needed.

### Full Documentation

For complete details on using custom images, including troubleshooting and best practices, see **[CUSTOM_IMAGES.md](CUSTOM_IMAGES.md)**.

## Upgrading Dify

### Upgrading to a New Dify Version

The CDK deployment makes upgrading Dify versions straightforward with automatic database migrations and zero-downtime rolling updates.

#### Step 1: Understand Aurora Automatic Backups

Your Aurora PostgreSQL cluster **automatically backs up continuously** to Amazon S3 with:
- **Default retention**: 1 day (configurable up to 35 days)
- **Point-in-time restore**: Restore to any moment within retention period (typically within 5 minutes of current time)
- **No performance impact**: Backups run continuously without affecting your database
- **Always enabled**: Cannot be disabled for Aurora clusters

**For routine upgrades**, the automatic backups are usually sufficient. You can restore to any point before the upgrade if needed.

#### Step 1a: Optional Manual Snapshot (Major Upgrades Only)

For **major version upgrades** (e.g., Dify v0.x to v1.x) or if you want extra safety, create a manual snapshot:

```bash
# Get your RDS cluster identifier
aws rds describe-db-clusters --region us-west-2 \
  --query 'DBClusters[?contains(DBClusterIdentifier, `DifyOnAwsStack`)].DBClusterIdentifier' \
  --output text

# Create a manual snapshot (replace <CLUSTER_IDENTIFIER> with the output above)
aws rds create-db-cluster-snapshot \
  --region us-west-2 \
  --db-cluster-snapshot-identifier dify-pre-upgrade-$(date +%s) \
  --db-cluster-identifier <CLUSTER_IDENTIFIER>
```

**Note**: Manual snapshots are stored until you explicitly delete them and incur storage costs (~$0.095/GB-month in us-west-2). For most upgrades, relying on automatic backups is more cost-effective.

#### Step 2: Update the Dify Version

Edit `bin/cdk.ts` and update the version tags:

```typescript
export const props: EnvironmentProps = {
  awsRegion: 'us-west-2',
  awsAccount: process.env.CDK_DEFAULT_ACCOUNT!,

  // Update to the desired version
  difyImageTag: '1.9.1',  // Change this to your target version
  difyPluginDaemonImageTag: '0.2.0-local',  // Update if needed

  // ... rest of your config
};
```

#### Step 3: Deploy the Update

```bash
npx cdk deploy --all
```

#### What Happens During Upgrade

The CDK deployment automatically handles:

1. ✅ **Pulls new Docker images** with the updated version tags
2. ✅ **Runs database migrations automatically** (via `MIGRATION_ENABLED=true`)
3. ✅ **Performs zero-downtime rolling update**:
   - New ECS tasks start with the new version
   - Health checks ensure new tasks are healthy
   - Old tasks are gracefully stopped
4. ✅ **Preserves all data** - Aurora database and S3 storage remain untouched

The upgrade typically completes in 5-10 minutes.

#### Special Case: Major Version Upgrades (v0 to v1)

For major version upgrades like Dify v0.x to v1.x, follow the special instructions in the [README](README.md#upgrading-dify-v0-to-v1), which require manual plugin migration steps.

#### Verification After Upgrade

1. Visit your Dify URL and confirm it loads successfully
2. Check that your existing workflows and data are intact
3. Review the Dify [release notes](https://github.com/langgenius/dify/releases) for any breaking changes

#### Rollback (If Needed)

If you need to rollback after an upgrade:

**Option 1: Rollback application only (data remains current)**
1. Edit `bin/cdk.ts` and set `difyImageTag` back to the previous version
2. Run `npx cdk deploy --all`
3. This keeps your current database state but runs the older Dify version

**Option 2: Point-in-time restore using automatic backups (FREE)**
```bash
# Restore Aurora cluster to a specific time before the upgrade
aws rds restore-db-cluster-to-point-in-time \
  --source-db-cluster-identifier <CLUSTER_IDENTIFIER> \
  --db-cluster-identifier dify-restored-cluster \
  --restore-to-time 2025-10-06T10:30:00Z \
  --region us-west-2
```

**Option 3: Restore from manual snapshot (if created)**
```bash
aws rds restore-db-cluster-from-snapshot \
  --db-cluster-identifier dify-restored-cluster \
  --snapshot-identifier <SNAPSHOT_ID> \
  --region us-west-2
```

**Important**: When restoring to a new cluster, you'll need to update your CDK stack to point to the new cluster or manually reconfigure the connection.

## Cleanup

To avoid ongoing charges:
```bash
npx cdk destroy --force
```

**Note**: This will delete all data. Ensure you have backups if needed.

## GitHub Actions Deployment

Create `.github/workflows/deploy.yml`:
```yaml
name: Deploy Dify
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
      - run: npm ci
      - run: npx cdk bootstrap
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: ${{ secrets.AWS_DEFAULT_REGION }}
      - run: npx cdk deploy --require-approval=never --all
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: ${{ secrets.AWS_DEFAULT_REGION }}
```

Set these GitHub Secrets:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY` 
- `AWS_DEFAULT_REGION`

## Next Steps

- [Setup Bedrock Models](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html)
- [Configure SMTP/Email](README.md#setup-email-smtp-for-user-invitation)
- [Connect to Notion](README.md#connect-to-notion)
- [Add Python packages](README.md#add-python-packages-available-in-code-execution)

For advanced configuration options, see the main [README.md](README.md).
