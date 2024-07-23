# Project Name: Multi-Region CI/CD Pipeline with AWS

## Overview
This project involves setting up a three-stage CI/CD pipeline to deploy a given application across multiple AWS Regions. The pipeline will utilize the following AWS services:
1. **AWS CodeCommit**: For source code management and version control.
2. **AWS CodeBuild**: To build the code and run tests.
3. **AWS CloudFormation**: As the deploy provider to provision and manage the infrastructure.

## Table of Contents
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [Step 1: Creating a Local Testing Environment](#step-1-creating-a-local-testing-environment)
  - [Step 2: Setting Up AWS CodeCommit](#step-2-setting-up-aws-codecommit)
  - [Step 3: Configuring AWS CodeBuild](#step-3-configuring-aws-codebuild)
  - [Step 4: Creating AWS CloudFormation Templates](#step-4-creating-aws-cloudformation-templates)
  - [Step 5: Configuring AWS CodePipeline](#step-5-configuring-aws-codepipeline)
- [Usage](#usage)
- [Contributing](#contributing)
- [License](#license)

## Architecture
The CI/CD pipeline consists of three main stages:
1. **Source Stage**: Uses AWS CodeCommit to manage and version control the source code.
2. **Build Stage**: Uses AWS CodeBuild to build the code and run tests.
3. **Deploy Stage**: Uses AWS CloudFormation to provision and manage the infrastructure in multiple AWS Regions.

## Prerequisites
Before setting up the CI/CD pipeline, ensure you have the following:
- An AWS account with appropriate permissions.
- AWS CLI configured on your local machine.
- Basic knowledge of AWS services: CodeCommit, CodeBuild, CloudFormation, and CodePipeline.
- The application codebase that you intend to deploy.
- Create necessary IAM roles and policies and choose Cloudformation as trustee:
  - **EC2FullAccess**
  - **SSM AutomationRole**
  - **IAMFullAccess**
- Create an SNS Topic for notifications:
  - Topic name: `Code_Ready_4_Prod`
  - Subscription using the email protocol to receive notifications.

## Setup Instructions

### Step 1: Creating a Local Testing Environment
1. **Create an EC2 Instance** in `us-east-1` named `local-test-env`.
2. Attach an admin role to the EC2 instance.

3. **Check, if not present Install necessary tools**:
   ```bash

   python3 --version    (check if python3 is present)
   cfn-lint --version    (check if cfn-lint is present) if not
   sudo yum update -y
   sudo yum install -y python3 python3-pip
   pip3 install cfn-lint
   ```

4. **Create and upload your CloudFormation template**:
   - Create an S3 bucket named `3stages-cicd-bucket` in `us-east-1`.
   - Upload your `Ec2_cfn_template.yml` to the S3 bucket.

5. **Copy the template to your EC2 instance**:
   ```bash
   aws s3 cp s3://3stages-cicd-bucket/Ec2_cfn_template.yml .
   ls -l
   ```

6. **Test the CloudFormation template**:
   ```bash
   cfn-lint Ec2_cfn_template.yml
   ```

7. **Package the CloudFormation template**:
   ```bash
   aws cloudformation package --template-file Ec2_cfn_template.yml --s3-bucket 3stages-cicd-bucket --output-template-file build_template_Artifact.yml
   ls -l
   ```

8. **Deploy to the test environment**:
   - Use the CloudFormation console in `us-east-1` to create a stack using the `build_template_Artifact.yml`.

### Step 2: Setting Up AWS CodeCommit
1. Go to the **CodeCommit** console and create a repository named `3stage-cicd-source-repo`.
2. Clone the repository to your local machine:
   ```bash
   git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/3stage-cicd-source-repo
   cd 3stage-cicd-source-repo
   ```

### Step 3: Configuring AWS CodeBuild
1. Open the **CodeBuild** console and create a new build project:
   - Project name: `3stage-cicd-Buildproject`
   - Source: CodeCommit repository `3stage-cicd-source-repo`
   - Environment settings:
     - Managed image: Amazon Linux 2023
     - Runtime: Standard
     - Image: `aws/codebuild/amazonlinux2-x86_64-standard:3.0`
     - Buildspec file: `buildspec.yml`

Example `buildspec.yml`: see documentation 


### Step 4: Creating AWS CloudFormation Templates
1. Create CloudFormation templates to define the infrastructure for your application.
2. Ensure the templates are parameterized to support multiple AWS Regions.

### Step 5: Configuring AWS CodePipeline
1. Navigate to the **CodePipeline** console and create a pipeline named `3stage-cicd-pipeline`.

#### Source Stage
1. Select CodeCommit and choose your repository `3stage-cicd-source-repo`.
2. Specify the branch (e.g., master).

#### Build Stage
1. Select AWS CodeBuild and choose the build project `3stage-cicd-Buildproject`.

#### Deploy Stage
1. Add a new stage for deployment.
2. Select AWS CloudFormation as the deploy provider.

**Deploy to Test Environment**:
- Action name: `DeployToTest`
- Region: `us-east-1`
- Stack name: `cfn-test-env-stack`
- Template file: `build_template_Artifact.yml`
- Capabilities: `CAPABILITY_NAMED_IAM`
- Role name: `CloudFormationServiceRoleForMyTestPipeline`
- Parameter overrides: `{ "KeyName": "cfn-Key_pair", "VPCStackName": "SampleNetworkCrossStack" }`

**Manual Approval**:
1. Add an action for manual approval.
2. Action provider: Manual approval
3. SNS topic ARN: Use the SNS topic created in the prerequisites.
4. Comments: "Please review and approve the changes."

**Deploy to Production**:
- Action name: `DeployToProd`
- Region: `us-west-1`
- Stack name: `cfn-prod-env-stack`
- Template file: `build_template_Artifact.yml`
- Capabilities: `CAPABILITY_NAMED_IAM`
- Role name: `CloudFormationServiceRoleForMyTestPipeline`
- Parameter overrides: `{ "KeyName": "cfn_kp-us-west-1", "VPCStackName": "SampleNetworkCrossStack" }`

### Detailed Configuration in the Console
1. **Creating the Pipeline**:
   - Open the **AWS CodePipeline** console.
   - Click on **Create pipeline**.
   - Enter a pipeline name and select the appropriate role.
   - Choose **Amazon S3** as the artifact store and create a new bucket if needed. Ensure this bucket is in `us-east-1` or a region that supports multi-region deployment.

2. **Source Stage**:
   - For the source provider, choose **AWS CodeCommit**.
   - Select the repository and branch that contains your application code.

3. **Build Stage**:
   - For the build provider, choose **AWS CodeBuild**.
   - Select the build project you created earlier.
   - This stage will take the source code and produce build artifacts which are stored in the S3 bucket specified in the artifact store.

4. **Deploy Stage**:
   - Add a new stage for deployment.
   - Select **AWS CloudFormation** as the deploy provider.
   - Add multiple actions within the stage to deploy to different regions.

## Usage
1. Commit your source code to the CodeCommit repository.
2. The CodePipeline will automatically trigger, building and deploying your application across the specified AWS Regions.

## Contributing
Contributions are welcome! Please fork the repository and use a feature branch. Pull requests are accepted.

## License
This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

By following the above instructions, you will have a fully functional CI/CD pipeline that deploys your application across multiple AWS Regions using AWS CodeCommit, AWS CodeBuild

, and AWS CloudFormation.
