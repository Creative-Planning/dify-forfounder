# ForFounders Dify Deployment Documentation

## Overview

This CDK project deploys Dify (an AI workflow automation platform) on AWS for ForFounders organization. The deployment uses AWS CDK to provision all necessary infrastructure including ECS, RDS Aurora, ElastiCache Redis, CloudFront, and ALB.

## Domain Configuration

**Production URL:** https://dify-prod.forfounder.com

### Domain Setup
- **Domain:** forfounder.com (hosted in Route53)
- **Subdomain:** dify-prod
- **CloudFront Distribution:** d2v8xc2bgav7gf.cloudfront.net

The domain configuration is set in `bin/cdk.ts`:
```typescript
domainName: 'forfounder.com',
subDomain: 'dify-prod',
```

This ensures that:
- All Dify environment variables (CONSOLE_WEB_URL, CONSOLE_API_URL, APP_WEB_URL) are set to `https://dify-prod.forfounder.com`
- Route53 A record is automatically created pointing to CloudFront
- SSL/TLS certificate is provisioned via ACM

**IMPORTANT:** Always access Dify via https://dify-prod.forfounder.com, NOT via the CloudFront URL directly. The app is configured to expect the custom domain for proper authentication and session management.

## Infrastructure Configuration

### Cost Optimization Settings
The deployment uses cost-optimized settings suitable for production:

```typescript
isRedisMultiAz: false,           // Single AZ Redis (lower cost)
useNatInstance: true,            // NAT instance instead of NAT Gateway
enableAuroraScalesToZero: true,  // Aurora Serverless v2 scales to 0 ACU
useFargateSpot: true,            // Use Fargate Spot instances
```

### Deployed Resources
- **ECS Cluster:** Fargate-based container orchestration
- **RDS Aurora PostgreSQL:** Serverless v2 (scales 0-2 ACU)
  - Main database: `main`
  - Vector database: `pgvector`
- **ElastiCache Redis:** Single AZ cluster for sessions and caching
- **S3 Buckets:** Storage for uploads and access logs
- **CloudFront:** CDN with custom domain support
- **ALB:** Internal load balancer for ECS services

### Container Images
- **API/Web:** `langgenius/dify-api:1.9.2` and `langgenius/dify-web:1.9.2`
- **Sandbox:** Official Dify sandbox image
- **Plugin Daemon:** `0.3.3-local`

## Deployment Instructions

### Prerequisites
1. AWS CLI configured with `forfounder-prod-admin` profile
2. Node.js and npm installed
3. AWS CDK installed: `npm install -g aws-cdk`

### Initial Deployment
```bash
cd /home/acb/WSL2-Client-Work/CreativePlanning/ForFounder/dify-forfounder

# Install dependencies
npm install

# Set AWS profile
export AWS_PROFILE=forfounder-prod-admin

# Bootstrap CDK (first time only)
cdk bootstrap

# Deploy
cdk deploy --all
```

### Updates
```bash
# Set AWS profile
export AWS_PROFILE=forfounder-prod-admin

# Deploy changes
cdk deploy --all
```

## Related Repositories

### forfounder-dify-deploy
**Location:** `/home/acb/WSL2-Client-Work/CreativePlanning/ForFounder/forfounder-dify-deploy`

This separate repository is used for **app/workflow migration** between dev and prod environments, NOT infrastructure deployment.

- **Purpose:** Export/import Dify apps and workflows
- **GitHub Actions:** Automated workflow deployment pipeline
- **CloudFormation Templates:** Snapshots only (in `deployAWS/`), not for direct deployment

**Do NOT use the CloudFormation templates in forfounder-dify-deploy for infrastructure changes.** Always use this CDK repository as the source of truth for infrastructure.

## Troubleshooting

### Login Lock Issues

If an account gets locked out after too many failed login attempts, you can clear the lock via ECS Exec:

**Step 1:** Access the ECS container via AWS Console
1. Go to ECS → Clusters → DifyOnAwsStack-ClusterEB0386A7-m4QQ0cTROlAp
2. Go to Tasks tab → Find the API Worker task
3. Click "Exec" tab → Select "Worker" container
4. Click "Execute"

**Step 2:** Run this Python script in the terminal:
```bash
printf 'from app_factory import create_app\nfrom services.account_service import AccountService\n\napp = create_app()\nemail = "USER@EMAIL.COM"\n\nwith app.app_context():\n    try:\n        AccountService.reset_login_error_rate_limit(email)\n        print(f"SUCCESS: Login lock cleared for {email}")\n        is_locked = AccountService.is_login_error_rate_limit(email)\n        print(f"Account still locked: {is_locked}")\n    except Exception as e:\n        print(f"ERROR: {e}")\n' > clear.py

python3 clear.py
```

Replace `USER@EMAIL.COM` with the actual locked email address.

**Login Lock Details:**
- **Max Failed Attempts:** 5
- **Lock Duration:** Configured in REDIS (LOGIN_LOCKOUT_DURATION)
- **Storage:** Redis key format: `login_error_rate_limit:{email}`

### Authentication/Session Issues

**Symptoms:** Login succeeds but redirects back to login page, or 401 Unauthorized errors in browser console.

**Causes:**
1. Accessing via CloudFront URL instead of custom domain
2. Stale cookies/tokens in browser
3. Domain mismatch in environment variables

**Solutions:**
1. Always use https://dify-prod.forfounder.com
2. Clear browser cookies and local storage for the domain
3. Try incognito/private browsing window
4. Verify environment variables in ECS task definition match the custom domain

### Environment Variables Check

To verify the correct URLs are set in the running containers:

**Via AWS Console:**
1. Go to ECS → Task Definitions
2. Find latest revision of ApiServiceTask or WebServiceTask
3. Check Container Definitions → Environment Variables
4. Verify CONSOLE_WEB_URL, CONSOLE_API_URL, APP_WEB_URL all point to `https://dify-prod.forfounder.com`

**Via ECS Exec:**
```bash
env | grep -i url
```

Should show:
```
CONSOLE_WEB_URL=https://dify-prod.forfounder.com
CONSOLE_API_URL=https://dify-prod.forfounder.com
APP_WEB_URL=https://dify-prod.forfounder.com
SERVICE_API_URL=https://dify-prod.forfounder.com
```

## Database Access

### PostgreSQL Connection
The RDS Aurora cluster has Data API enabled for administrative tasks without direct network access.

**Get database credentials:**
```bash
aws secretsmanager get-secret-value \
  --secret-id PostgresClusterSecretC5EAFD-1r56BG049FEp \
  --region us-east-1 \
  --query SecretString \
  --output text
```

**Database Details:**
- **Endpoint:** difyonawsstack-postgrescluster53e5bdab-ks93a8enaczn.cluster-cgv4ikwc45eq.us-east-1.rds.amazonaws.com
- **Port:** 5432
- **Main Database:** main
- **Vector Database:** pgvector

### Redis Access
Redis is only accessible from within the VPC (ECS containers).

**Endpoint:** master.dir17b9jbzlezx46.2wqsha.use1.cache.amazonaws.com:6379

Redis requires password authentication. The password is stored as an environment variable in the ECS tasks.

## Admin Account Information

**Default Admin Email:** andrew.box@creativeplanning.com

This is the only admin account for the ForFounders Dify instance. If locked out or password forgotten, use the troubleshooting steps above to reset.

## Monitoring and Logs

### CloudWatch Logs
- **API Logs:** /aws/ecs/DifyOnAwsStack-ApiService
- **Web Logs:** /aws/ecs/DifyOnAwsStack-WebService

### Access Logs
- **ALB Logs:** S3 bucket with prefix `dify-alb/`
- **CloudFront Logs:** S3 bucket with prefix `dify-cloudfront/`

### Container Insights
ECS Container Insights is enabled for performance monitoring via CloudWatch.

## Security Notes

1. **Redis Password:** Automatically generated and stored in ECS task environment
2. **Database Credentials:** Stored in AWS Secrets Manager
3. **Encryption:**
   - RDS storage encrypted
   - S3 buckets enforce SSL
   - CloudFront uses HTTPS only
4. **Network Security:**
   - ALB is internal (private subnets only)
   - CloudFront provides public access with WAF protection
   - ECS tasks have no public IPs

## Cost Management

**Estimated Monthly Cost (as of deployment):**
- ECS Fargate Spot: ~$30-50
- RDS Aurora Serverless v2 (scales to 0): ~$20-80 depending on usage
- ElastiCache (single AZ): ~$15-25
- NAT Instance (t3.nano): ~$5
- Data Transfer & Storage: Variable

**Cost Optimization Tips:**
- Aurora scales to 0 ACU when inactive (60 min idle timeout)
- Use Fargate Spot for additional ~70% savings
- Single AZ Redis acceptable for non-critical workloads
- Review CloudWatch logs retention policies

## Version History

- **2025-11-07:** Added domain configuration (dify-prod.forfounder.com)
- **Initial Deployment:** Dify 1.9.2 with cost-optimized settings
