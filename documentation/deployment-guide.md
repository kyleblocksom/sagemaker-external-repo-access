# Deployment Guide
---
## Deployment Workflow
The below workflow diagram highlights the end-to-end deployment process that is detailed within this guide:
![](../img/deployment-workflow.svg)

## Pre-Deployment
By default, AWS CloudFormation uses a temporary session that it generates from your user credentials for stack operations. If you specify a service role, CloudFormation will instead use that role's credentials.

To deploy this solution, your IAM user/role or service role must have permissions to deploy the resources specified in the CloudFormation template. For more details on AWS Identity and Access Management (IAM) with CloudFormation, please refer to the [AWS CloudFormation User Guide](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-iam-template.html).

You must also have [AWS CLI](https://aws.amazon.com/cli/) installed. For instructions on installing AWS CLI, please see [Installing, updating, and uninstalling the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html). If you would like to use the multi-account model deployment option, you need access to minimum two AWS accounts, recommended three accounts for development, staging and production environments.

### Gather Third-Party Repository Configuration Settings
The CloudFormation template requires a total of eleven user-defined parameters, 2 of which are specific to the third-party repository that you wish to clone.

Navigate to the third-party repository and note the branch name and SSH URL:

![](../img/git-branch-ssh-url.svg)

### Establish VPC Networking Configuration
This solution requires private VPC subnets into which you can deploy your Lambda Function and CodeBuild Project. These private subnets must be deployed into a VPC that is also configured with a NAT Gateway (NGW) in a public subnet to facilitate intenret ingress and egress through an Internet Gateway.

If your environment does not have the required VPC, subnets, NAT Gateway, and Internet Gateway configuration, you can create those by launching the following [CloudFormation template](https://github.com/awsdocs/aws-lambda-developer-guide/blob/master/templates/vpc-privatepublic.yaml).

### Create SSH Key Pair
To authenticate with the third-party Git repository, you will use an SSH key pair. To create the key pair, use the below ssh-keygen command in your local terminal:

```sh
ssh-keygen -t rsa -b 4096 -C "your_git_email@example.com"
```

Follow the commands prompts to create a name (ex: git-codepipeline-rsa) and optional paraphrase for the key. This generates your private/public rsa key pair in the current directory: git-codepipeline-rsa and git-codepipeline-rsa.pub.

Make a note of the contents of git-codepipeline-rsa.pub and add it as an authorized key for your Git user. For instructions, please see [Adding an SSH key to your GitLab account](https://docs.gitlab.com/ee/ssh/#adding-an-ssh-key-to-your-gitlab-account).

Next, publish your private key to AWS Secrets Manager using the AWS Command Line Interface (AWS CLI):

```sh
export SecretsManagerArn=$(aws secretsmanager create-secret --name git-codepipeline-rsa \
--secret-string file://git-codepipeline-rsa --query ARN --output text)
```
Make a note of the Secret ARN, which you will input later as the SecretsManagerArnForSSHPrivateKey CloudFormation parameter.

### Create S3 and Upload Lambda Function
This solution leverages a Lambda function that polls for the job details of the CodePipeline custom source action. Once there is a change on the source branch, the Lambda function triggers AWS CodeBuild execution and passes all the job-related information.
Once CodeBuild complete its execution, the Lambda function then sends a success message to the CodePipeline source stage so it can proceed to the next stage.

The Lambda function code needs to be uploaded to an S3 bucket in the same Region where the CloudFormation stack is being deployed. To create a new S3 bucket, use the following AWS CLI commands:

```sh
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export S3_BUCKET_NAME=git-codepipeline-lambda-${ACCOUNT_ID} 
aws s3 mb s3://${S3_BUCKET_NAME} --region us-east-1
```
Download the compressesed [Lambda code file](../lambda/git-clone-lambda.zip) to your working directory, then upload the file to S3 bucket you just created using the following AWS CLI commands:

```sh
aws s3 cp git-clone-lambda.zip s3://${S3_BUCKET_NAME}/git-clone-lambda.zip
```

#### Optional - Run Security Scan on the CloudFormation Templates
If you would like to run a security scan on the CloudFormation templates using [`cfn_nag`](https://github.com/stelligent/cfn_nag) (recommended), you have to install `cfn_nag`:
```sh
brew install ruby brew-gem
brew gem install cfn-nag
```

To initiate the security scan, run the following command:
```sh
cfn_nag_scan --input-path <path to cloudformation json>
```

## Deployment 

The following section provides step-by-step deployment instructions
You can also find all CLI commands in the delivered shell scripts in the project folder `test`.

### Two-step deployment via CloudFormation
With this option you provision a data science environment in two steps, each with its own CloudFormation template. You can control all deployment parameters.  

📜 Use this option if you want to parametrize every aspect of the deployment based on your specific requirements and environment.

❗ You can select your existing VPC and network resources (subnets, NAT gateways, route tables) and existing IAM resources to be used for stack set deployment. Set the corresponding CloudFormation parameters to names and ARNs or your existing resources.

❗ You must specify the valid OU IDs for the `OrganizationalUnitStagingId`/`OrganizationalUnitProdId` **or** `StagingAccountList`/`ProductionAccountList` parameters for the `env-main.yaml` template to enable multi-account model deployment.

You can use the provided [shell script](../test/cfn-test-e2e.sh) to run this deployment type or follow the commands below.

#### Step 1: Deploy the core infrastructure
❗ Make sure you package the CloudFormation templates as described in [these instructions]((../package-cfn.md)).

In this step you deploy the _shared core infrastructure_ into your AWS Account. The stack (`core-main.yaml`) provisions:
1. Shared IAM roles for data science personas and services (optional if you bring your own IAM roles)
2. A shared services VPC and related networking resources (optional if you bring your own network configuration)
3. An AWS Service Catalog portfolio to provide a self-service deployment for the **data science administrator** user role
4. An ECS Fargate cluster to run a private PyPi mirror (_not implemented at this stage_)
5. Security guardrails for your data science environment (_detective controls are not implemented at this stage_)

The deployment options you can use are:
+ `CreateIAMRoles`: default `YES`. Set to `NO` if you have created the IAM roles outside of the stack (e.g. via a separate process) - such as "Bring Your Own IAM Role (BYOR IAM)" use case
+ `CreateSharedServices`: default `NO`. Set to `YES` if you would like to create a shared services VPC and an ECS Fargate cluster for a private PyPi mirror (_not implemented at this stage_)
+ `CreateSCPortfolio`: default `YES`. Set to `NO`if you don't want to to deploy an AWS Service Catalog portfolio with data science environment products
+ `DSAdministratorRoleArn`: **required** if `CreateIAMRoles=NO`, otherwise automatically provisioned
+ `SCLaunchRoleArn`: **required** if `CreateIAMRoles=NO`, otherwise automatically provisioned
+ `SecurityControlExecutionRoleArn`: **required** if `CreateIAMRoles=NO`, otherwise automatically provisioned

The following command uses the default values for the deployment options. You can specify parameters via `ParameterKey=<ParameterKey>,ParameterValue=<Value>` pairs in the `aws cloudformation create-stack` call:
```sh
STACK_NAME="sm-mlops-core"

[[ unset"${S3_BUCKET_NAME}" == "unset" ]] && echo "\e[1;31mERROR: S3_BUCKET_NAME is not set" || echo "\e[1;32mS3_BUCKET_NAME is set to ${S3_BUCKET_NAME}"

aws cloudformation create-stack \
    --template-url https://s3.$AWS_DEFAULT_REGION.amazonaws.com/$S3_BUCKET_NAME/sagemaker-mlops/core-main.yaml \
    --region $AWS_DEFAULT_REGION \
    --stack-name $STACK_NAME  \
    --disable-rollback \
    --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
    --parameters \
        ParameterKey=StackSetName,ParameterValue=$STACK_NAME
```

The previous command launches the stack deployment and returns the `StackId`. You can track the stack deployment status in [AWS CloudFormation console](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks?filteringStatus=active&filteringText=&viewNested=true&hideStacks=false) or in your terminal with the following command:
```sh
aws cloudformation describe-stacks \
    --stack-name $STACK_NAME \
    --query "Stacks[0].StackStatus"
```

After a successful stack deployment, the status turns to from `CREATE_IN_PROGRESS` to `CREATE_COMPLETE`. You can print the stack output:
```sh
aws cloudformation describe-stacks \
    --stack-name $STACK_NAME  \
    --output table \
    --query "Stacks[0].Outputs[*].[OutputKey, OutputValue]"
```

## Post-Deployment
After you successfully deploy all CloudFormation stacks and created the needed infrastructure, you can start Studio and start working with the delivered notebooks.

### Start Studio
To launch Studio you must go to [SageMaker console](https://console.aws.amazon.com/sagemaker/home?#/dashboard), click **Control panel** and click on the **Studio** in **Launch app** button on the **Users** panel:

![](../img/open-studio.png)

### Clone code repository
To use the provided notebooks you must clone the source code repository into your Studio environment.
Open a system terminal in Studio in the **Launcher** window:

![](../img/studio-system-terminal.png)

Run the following command in the terminal:
```sh
git clone https://github.com/aws-samples/amazon-sagemaker-secure-mlops.git
```

This command downloads and saves the code repository in your home directory in Studio.
Now go to the file browser and open [`00-setup` notebook](../sm-notebooks/00-setup.ipynb):

![](../img/file-browser-setup.png)

The first start of the notebook kernel on a new KernelGateway app takes about 5 minutes. Continue with the setup instructions in the notebook after Kernel is ready.

❗ You have to run the whole notebook to setup your SageMaker environment.

Now you setup your data science environment and can start experimenting.

---

[Back to README](../README.md)

---

Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
SPDX-License-Identifier: MIT-0
