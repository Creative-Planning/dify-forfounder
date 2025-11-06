## Deployment Details

- **Date**: 2025-11-06 (Redeployed fresh - resolved login loop issue)
- **Environment**: ForFounders Dify (CDK)
- **AWS Profile**: `forfounder-prod-admin`
- **Commands Executed**:
  - `npx cdk destroy DifyOnAwsStack --force` (cleanup)
  - `npx cdk deploy DifyOnAwsStack` (fresh deployment)
- **Primary Endpoint**: https://d2v8xc2bgav7gf.cloudfront.net
- **CloudFormation Stack ARN**: `arn:aws:cloudformation:us-east-1:935775993565:stack/DifyOnAwsStack/ebc2d110-bb26-11f0-9f3d-0affed8d4799`
- **CDK Output (execute-command helper)**:
  ```
  aws ecs execute-command --region us-east-1 --cluster DifyOnAwsStack-ClusterEB0386A7-m4QQ0cTROlAp --container Main --interactive --command "bash" --task $(aws ecs list-tasks --region us-east-1 --cluster DifyOnAwsStack-ClusterEB0386A7-m4QQ0cTROlAp --service-name DifyOnAwsStack-ApiServiceFargateServiceE4EA9E4E-nG9AzoeSYViO --desired-status RUNNING --query 'taskArns[0]' --output text)
  ```
- **Initial Admin Setup**: Visit the primary endpoint, create the first admin account on first login, then configure providers as needed.

