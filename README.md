# Infrastructure as Code CI/CD with AWS CodePipeline & CloudOne Conformity

This project demonstrates a CI/CD pipeline that provisions AWS resources using **CloudFormation**, integrates **CodeBuild** for template scanning, and runs compliance checks using **Trend Micro CloudOne Conformity** before deploying infrastructure.

---

## 📂 Project Structure

```

.
├── buildspec.yml                 # Build instructions for CodeBuild
├── template.json                 # CloudFormation template (provisions resources, e.g., S3 bucket)
└── .template-security/
└── config.json                   # Conformity template scanner configuration
└── README.md                     

````

---

## ⚙️ Pipeline Overview

The pipeline is built with **AWS CodePipeline** and has the following stages:

1. **Source**
   - Connected to GitHub (via CodeConnections).
   - Triggers pipeline on new commits to the `main` branch.

2. **Build (CodeBuild)**
   - Runs `buildspec.yml`.
   - Verifies Conformity API key exists.
   - Submits `template.json` to CloudOne Template Scanner for compliance checks.
   - Fails the build if violations meet or exceed the configured severity (`MEDIUM` in `config.json`).

3. **Deploy (CloudFormation)**
   - Deploys resources defined in `template.json`.
   - Uses CloudFormation to create/update a stack (`TestStack` or a chosen name).
   - Provides outputs such as the bucket name.

> ⚠️  **NOTE:**  
> - **CloudOne Conformity** must be integrated to GitHub and AWS account in your Account **(Integrations)**.
> - GitHub app must be added to your AWS environment for synchronization.

---

## 🎨 Architecture Diagram
![diagram](Security-In-IaC-Pipeline.png)

## 📝 Files

### `template.json`

A CloudFormation template that defines AWS resources (currently an S3 bucket):

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "Test S3 bucket for feature branch CI/CD",
  "Resources": {
    "FeatureTestBucket": {
      "Type": "AWS::S3::Bucket",
      "Properties": {
        "BucketName": "test-codepipeline-<unique-id>",
        "AccessControl": "Private",
        "VersioningConfiguration": { "Status": "Disabled" },
        "PublicAccessBlockConfiguration": {
          "BlockPublicAcls": true,
          "BlockPublicPolicy": true,
          "IgnorePublicAcls": true,
          "RestrictPublicBuckets": true
        }
      }
    }
  },
  "Outputs": {
    "BucketName": {
      "Description": "Bucket name",
      "Value": { "Ref": "FeatureTestBucket" }
    }
  }
}
````

> ⚠️ Bucket names must be **globally unique**. Update `BucketName` before deploying.

---

### `buildspec.yml`

Defines build instructions for CodeBuild:

```yaml
version: 0.2

phases:
  install:
    commands:
      - echo "Dependencies already available"
  build:
    commands:
      - echo "Checking if API key is injected..."
      - if [ -z "$CONFORMITY_API_KEY" ]; then echo "API key not found"; exit 1; else echo "API key is available"; fi
      - echo "Running Conformity Template Scanner..."
      - |
        curl -X POST \
          -H "Authorization: ApiKey $CONFORMITY_API_KEY" \
          -H "Content-Type: application/json" \
          -d @template.json \
          https://conformity.trendmicro.com/api/template-scanner/scan \
          | jq .
```

---

### `.template-security/config.json`

Configuration for Conformity template scanner:

```json
{
  "failOnSeverity": "MEDIUM",
  "scan": {
    "enabled": true,
    "outputFormat": "json"
  },
  "report": {
    "includePassedChecks": true,
    "includeSeveritySummary": true
  },
  "frameworks": {
    "terraform": { "templateFilesPattern": "**/*(*.tf|*.tfvars)" },
    "cloudformation": { "templateFilesPattern": "**/*(*.yml|*.yaml|*.json)" }
  },
  "settings": {
    "failOnSeverity": "MEDIUM"
  }
}
```

---

## 🚀 Deployment Flow

1. Commit and push changes to the **main branch**.
2. CodePipeline triggers:

   * Source → Build → Deploy.
3. CodeBuild runs compliance scan via Conformity.
4. CloudFormation deploys or updates the defined stack.
5. Outputs (like bucket name) are visible in the CloudFormation stack outputs.

---

## 🔑 Requirements

* AWS account with:

  * CodePipeline
  * CodeBuild
  * CloudFormation
  * IAM roles with proper permissions (`s3:ListBucket`, `s3:GetObject`, `cloudformation:*`, etc.)
* Trend Micro CloudOne account & API key (`CONFORMITY_API_KEY`).
* GitHub repo connected via **CodeConnections**.

---

## 🛑 Troubleshooting

* **`AlreadyExists` errors**: Change the `BucketName` in `template.json`.
* **`ROLLBACK_COMPLETE`**: Delete the stack in CloudFormation and redeploy.
* **`AccessDenied (s3:ListBucket)`**: Ensure the CodePipeline service role has `s3:ListBucket` permission on the artifact bucket.
* **Template not found**: Ensure `template.json` is in the root of the repo and referenced correctly as `SourceArtifact::template.json`.

---

## 📌 Next Steps

* Expand `template.json` to define more AWS resources (EC2, Lambda, VPC, etc.).
* Add tests in CodeBuild before deployment.
* Parameterize stack names for feature branch deployments.


