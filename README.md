# compose-ecs-cfn-cicd-demo-app
Deploys a 2-tier webapp (flask frontend, redis backend, efs volumes, aws sdn) on ecs fargate

This repository includes a sample a 2-tier webapp (flask frontend, redis backend, efs volumes, aws sdn) on ecs fargate to be used in Docker Compose demonstrations, specifically demos involving the [Docker Compose for Amazon ECS Plugin](https://docs.docker.com/cloud/ecs-integration/)

This repository also includes CloudFormation templates to:
- Build the foundational AWS resources (VPCs, ECS Clusters).
- Build a CodePipeline to deploy the Docker Compose file to Amazon ECS in an
  automated fashion.

## deploying example

1. Deploy a Cloud9 IDE environment, i.e. using the following specs:
- t3.large (8 GiB RAM + 2 vCPU) on Ubuntu Server 18.04 LTS

1. Prepare the Cloud9 env

optional: set the default region:

```bash
export AWS_DEFAULT_REGION=eu-central-1
```

Install requirements:

```bash
echo "alias l='ls -alrth'" >> .bashrc  
sudo pip install --upgrade awscli && hash -r
sudo apt update
sudo apt install jq gettext bash-completion moreutils -y
```

Install docker compose cli Option 1:
```bash
curl -L -o docker-linux-amd64 https://github.com/docker/compose-cli/releases/download/v1.0.31/docker-linux-amd64
mv docker-linux-amd64 docker
chmod +x docker
which docker
sudo ln -s $(which docker) /usr/local/bin/com.docker.cli
./docker/docker —context default ps
sudo mv docker /usr/local/bin/docker
docker version && docker compose version
```

Install docker compose cli Option 2:
```bash
curl -L https://raw.githubusercontent.com/docker/compose-cli/main/scripts/install/install_linux.sh | sh
docker version && docker compose version
```

1. Allow inbound access into the Cloud9 instance by updating the EC2 instance SG. This is in order to test the app running 'locally' on the Cloud9 IDE with compose up

1. Clone this repository
```bash
git clone https://github.com/henrybravo/compose-ecs-cfn-cicd-demo-app.git
cd compose-ecs-cfn-cicd-demo-app
```

1. Deploy the infrastructure

```bash
# Navigate to the Infrastructure Directory 
cd infrastructure/

# Deploy the CloudFormation Template 
aws cloudformation create-stack --stack-name compose-infrastructure --template-body file://cloudformation.yaml --capabilities CAPABILITY_IAM

# get notified when ready
until [[ `aws cloudformation describe-stacks  --stack-name "compose-infrastructure"  --query "Stacks[0].[StackStatus]"   --output text` == "CREATE_COMPLETE" ]]; do  echo "The stack is NOT in a state of CREATE_COMPLETE at `date`";  sleep 30; done && echo "The Stack is built at `date` - Please proceed"
```

1. Deploy the CodePipeline and CI/CD building blocks

```bash
# Set the VPC Id $ 
VPC_ID=$(aws cloudformation describe-stacks --stack-name compose-infrastructure --query "Stacks[0].Outputs[?OutputKey=='VpcId'].OutputValue" --output text)

# Set the ECS Cluster Name $ 
ECS_CLUSTER=$(aws cloudformation describe-stacks --stack-name compose-infrastructure --query "Stacks[0].Outputs[?OutputKey=='ClusterName'].OutputValue" --output text)

# The Loadbalancer Arn $ 
LOADBALANCER_ARN=$(aws cloudformation describe-stacks --stack-name compose-infrastructure --query "Stacks[0].Outputs[?OutputKey=='LoadbalancerId'].OutputValue" --output text)

# Navigate to the directory that stores the Pipeline Template 
cd ../pipeline/ 

# Deploy the AWS CloudFormation Template, passing in the existing AWS Resource Paramaters 
aws cloudformation create-stack --stack-name compose-pipeline --template-body file://cloudformation.yaml --capabilities CAPABILITY_IAM --parameters ParameterKey=ExistingAwsVpc,ParameterValue=$VPC_ID ParameterKey=ExistingEcsCluster,ParameterValue=$ECS_CLUSTER ParameterKey=ExistingLoadbalancer,ParameterValue=$LOADBALANCER_ARN

# get notified when ready
until [[ `aws cloudformation describe-stacks  --stack-name "compose-pipeline" --query "Stacks[0].[StackStatus]"   --output text` == "CREATE_COMPLETE" ]]; do  echo "The stack is NOT in a state of CREATE_COMPLETE at `date`";  sleep 30; done && echo "The Stack is built at `date` - Please proceed"
```

1. Deploy the application
```bash
# Ensure you are in the Application directory of the cloned repository $ 
cd ../application/

# Retrieve the S3 Bucket Name $ 
BUCKET_NAME=$(aws cloudformation describe-stacks --stack-name compose-pipeline --query "Stacks[0].Outputs[?OutputKey=='S3BucketName'].OutputValue" --output text)

# Zip up the Code and Upload to S3 $ 
zip -r compose-bundle.zip . 
aws s3 cp compose-bundle.zip s3://$BUCKET_NAME/compose-bundle.zip
```

