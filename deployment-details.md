## Deployment Details

- **Date**: 2024-10-24
- **Environment**: ForFounders Dify (CDK)
- **AWS Profile**: `forfounder-prod-admin`
- **Commands Executed**:
  - `npx cdk bootstrap`
  - `npx cdk deploy --all`
- **Primary Endpoint**: https://dpxojkfx6z8mo.cloudfront.net
- **CloudFormation Stack ARN**: `arn:aws:cloudformation:us-east-1:935775993565:stack/DifyOnAwsStack/d26b1fe0-b119-11f0-88e4-12d4d00385d3`
- **CDK Output (execute-command helper)**:
  ```
  aws ecs execute-command --region us-east-1 --cluster DifyOnAwsStack-ClusterEB0386A7-sFGnlPXdQ49e --container Main --interactive --command "bash" --task $(aws ecs list-tasks --region us-east-1 --cluster DifyOnAwsStack-ClusterEB0386A7-sFGnlPXdQ49e --service-name DifyOnAwsStack-ApiServiceFargateServiceE4EA9E4E-on9TtxjiNb6y --desired-status RUNNING --query 'taskArns[0]' --output text)
  ```
- **Initial Admin Setup**: Visit the primary endpoint, create the first admin account on first login, then configure providers as needed.

