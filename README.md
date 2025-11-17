# FortiCNAPP Cloud Integration Setup

Setup instructions for FortiCNAPP (Lacework) across AWS (Control Tower), GCP, and Azure.

## Prerequisites

- Access to FortiCNAPP Console
- Access to AWS CloudShell, Google Cloud Shell, and Azure Cloud Shell

## Permissions

Ensure you have the necessary permissions configured for each cloud provider:

- **AWS**: [AWS Configuration Integration Prerequisites](https://docs.fortinet.com/document/forticnapp/latest/administration-guide/618119/aws-configuration-integration-prerequisites) | [AWS Integration - Terraform from AWS CloudShell](https://docs.fortinet.com/document/forticnapp/latest/administration-guide/283460/aws-integration-terraform-from-aws-cloudshell)
- **GCP**: [Storage-Based Google Cloud Integration - Terraform from Google Cloud Shell](https://docs.fortinet.com/document/forticnapp/latest/administration-guide/305117/storage-based-google-cloud-integration-terraform-from-google-cloud-shell)
- **Azure**: [Create an Azure App for Integration](https://docs.fortinet.com/document/forticnapp/latest/administration-guide/644235)


## FortiCNAPP Lacework CLI Setup

Setup uses each provider's cloud shell. Install Terraform and the Lacework CLI to generate configurations for deployment.

### 1. Create an API Key

1. Log in to FortiCNAPP Console (eg via https://forticloud.com)
2. In FortiCNAPP, once logged in, navigate to **Settings** > **API Keys** > **Add New**
3. Download the API key JSON file
4. Transfer API Key to Cloud Shell

- **AWS CloudShell**: Click the Actions menu (☰) > Upload file
- **Google Cloud Shell**: Click the three-dot menu > Upload file
- **Azure Cloud Shell**: Click the Upload/Download files icon > Upload

### 3. Install Lacework CLI in Cloud Shell (AWS, Azure, Google Cloud)

Docs: [Get started with the Lacework FortiCNAPP CLI](https://docs.fortinet.com/document/forticnapp/latest/cli-reference/68020/get-started-with-the-lacework-forticnapp-cli)

```bash
# Create bin directory and add to PATH
mkdir -p "$HOME/bin"
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc

# 2. Install the Lacework CLI binary
curl -sSL https://raw.githubusercontent.com/lacework/go-sdk/main/cli/install.sh | sudo bash -s -- -d "$HOME/bin"

# 4. Verify Lacework CLI Installation
lacework version

# 5. Configure Lacework CLI with API Key
lacework configure -j [path_to_api_key.json]

# 6. Verify Lacework CLI Configuration
lacework account list
```

## Terraform For AWS CloudShell (Azure and GCP have Terraform pre-installed in their cloud shells)

### AWS CloudShell

```bash
# Download and install Terraform (replace VERSION with latest, e.g., 1.6.0)
VERSION=$(curl -s https://checkpoint-api.hashicorp.com/v1/check/terraform | jq -r -M '.current_version')
wget https://releases.hashicorp.com/terraform/${VERSION}/terraform_${VERSION}_linux_amd64.zip
unzip terraform_${VERSION}_linux_amd64.zip -d "$HOME/bin"
rm terraform_${VERSION}_linux_amd64.zip

# Verify installation
terraform version
```

## Cloud Account Integration - AWS Control Tower - AWS Integration for inventory and audit logging via CloudFormation

For AWS Organizations Using AWS Control Tower, cloudformation is recommended for the AWS Integration.

Docs: [AWS Integration - CloudFormation](https://docs.fortinet.com/document/forticnapp/latest/administration-guide/399671/aws-control-tower-integration-using-cloudformation)

1. Log into AWS control Tower management account and AWS region where Control Tower is deployed.

2. Get AWS Control Tower details:
- Log Archive Account ID: eg. 123456789012
- Log Archive Account Name: eg. aws-controltower-LogArchiveAccount
- Audit Account ID: eg. 123456789012
- Audit Account Name: eg. aws-controltower-AuditAccount
- CloudTrail Name: eg. aws-controltower-BaselineCloudTrail

3. Check if the CloudTrail S3 bucket (in Log Archive Account) has KMS encryption enabled
- If enabled, get the CloudTrail S3 KMS Key Identifier ARN (optional): eg. arn:aws:kms:us-west-2:123456789012:key/12345678-1234-1234-1234-123456789012

4. Check if the CloudTrail S3 bucket has SNS notifications enabled
- If enabled, get the SNS Topic ARN: eg. arn:aws:sns:us-west-2:123456789012:Lacework-Control-Tower-Integration-SNS
- If not enabled, create a new SNS topic and get the ARN: eg. arn:aws:sns:us-west-2:123456789012:Lacework-Control-Tower-Integration-SNS

5. Launch stack:
https://console.aws.amazon.com/cloudformation/home?#/stacks/create/review?templateURL=https://lacework-alliances.s3.us-west-2.amazonaws.com/lacework-control-tower-cfn/templates/control-tower-integration.template.yaml

Include the following parameters:
- Stack Name (e.g. "Lacework-Control-Tower-Integration")
- FortiCNAPP URL (eg. 1234567890)
- FortiCNAPP Access Key ID and Secret Key that you copied from your API keys file
- Capability Type: CloudTrail+Config (default)
- Monitor Existing Accounts: Yes (default)
- Existing AWS Control Tower CloudTrail Name
- If your CloudTrail S3 logs are encrypted, specify the KMS Key Identifier ARN
- Update the Control Tower Log Account Name and Audit Account Name if necessary

5. Create stack and wait for it to complete.

6. (optional) Update the KMS Key Policy for Cross-Account Role Access
This step is only required if your CloudTrail S3 logs are encrypted

**Find the role ARN:**

**Option A: Check CloudFormation stack outputs**
- In AWS Console, navigate to CloudFormation > Your Stack > Outputs
- Look for the role ARN in the stack outputs

**Option B: Look up the role in AWS IAM**
```bash
# In the log archive account, list IAM roles:
aws iam list-roles --query "Roles[?contains(RoleName, 'laceworkcwssarole')].Arn" --output text
```

**Update KMS Key Policy:**
```json
{
  "Sid": "Allow Lacework to decrypt logs",
  "Effect": "Allow",
  "Principal": {
    "AWS": [
      "arn:aws:iam::<log-archive-account-id>:role/<lacework-account-name>-laceworkcwssarole"
    ]
  },
  "Action": [
    "kms:Decrypt"
  ],
  "Resource": "*"
}
```

## Cloud Account Integration - AWS - Agentless Workload Scanning via Terraform

Docs:
- [Prerequisites](https://docs.fortinet.com/document/forticnapp/latest/administration-guide/122712/prerequisites)
- [Integrating agentless workload scanning with AWS using Terraform](https://docs.fortinet.com/document/forticnapp/latest/administration-guide/744245/terraform)

Steps:
- [Integrating agentless workload scanning for AWS organization account with Terraform](https://docs.fortinet.com/document/forticnapp/latest/administration-guide/864699/integrating-agentless-workload-scanning-for-aws-organization-account-with-terraform)

1. Generate Terraform configuration
```bash
lacework generate cloud-account aws agentless-workload-scanning
```

2. Deploy Terraform configuration
```bash
cd /home/cloudshell-user/lacework/aws
terraform init
terraform plan
terraform apply
### GCP Integration - Automated Configuration

Docs: [Google Cloud Integration - Automated Configuration](https://docs.fortinet.com/document/forticnapp/latest/administration-guide/756723/google-cloud-integration-automated-configuration)

1. Get a temporary access token for the Google Cloud Shell
```bash
gcloud auth print-access-token
```

2. Run Automated Configuration from FortiCNAPP Console


### Azure Integration

Docs: [Azure Integration - Automated Configuration](https://docs.fortinet.com/document/forticnapp/latest/administration-guide/948086/azure-integration-automated-configuration)


## Validating Integrations

```bash
lacework cloud-account list
```

Or verify in Console: **Settings > Integrations > Cloud Accounts**
