# compose-ecs-cfn-cicd-demo-app
Deploys a 2-tier webapp (flask frontend, redis backend, efs volumes, aws sdn) on ecs fargate

This repository includes a sample a 2-tier webapp (flask frontend, redis backend, efs volumes, aws sdn) on ecs fargate to be used in Docker Compose demonstrations, specifically demos involving the [Docker Compose for Amazon ECS Plugin](https://docs.docker.com/cloud/ecs-integration/)

This repository also includes CloudFormation templates to:
- Build the foundational AWS resources (VPCs, ECS Clusters).
- Build a CodePipeline to deploy the Docker Compose file to Amazon ECS in an
  automated fashion.
