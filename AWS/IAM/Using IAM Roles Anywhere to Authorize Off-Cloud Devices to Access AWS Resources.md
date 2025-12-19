---
title: Using IAM Roles Anywhere to Authorize Off-Cloud Devices to Access AWS Resources
slug: iam-roles-anywhere-authorize-offCloudDevices
categories:
  - IAM
tags:
  - AWS
halo:
  site: https://blog.silentmo.cn
  name: 
  publish: true
---

# IAM Role Anywhere
## Concepts

IAM Roles Anywhere uses the CreateSession API to provide temporary credentials. Off-cloud devices request this API by passing parameters such as trust anchor, profile, assumed role, certificate, and a SigV4 signature generated using the certificate's private key.

When IAM Roles Anywhere receives the request, it first verifies the signature using the certificate's public key, then validates that the certificate was issued by a trust anchor previously configured in the account. After both verifications succeed, the application is authenticated, and IAM Roles Anywhere creates a new role session for the role specified in the request by calling AWS Security Token Service (AWS STS) to obtain temporary security credentials.

CreateSession is essentially an x509 wrapper around the AssumeRole API and is not currently included in any SDK. For detailed authentication information, please refer to [IAM Roles Anywhere](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/authentication.html) authentication.

Therefore, in this example, we need to use a credential helper tool provided by IAM Roles Anywhere, which is compatible with the credential_process feature in various programming language SDKs. Currently, this tool supports three platforms: Linux, Windows, and Darwin (macOS).

Basic terminology:
- Trust anchors: A trust anchor represents a trust relationship between IAM Roles Anywhere and one of your CA certificates. Devices use x509 certificates issued by the CA and trust anchors to authenticate and obtain temporary IAM credentials.
- Roles: IAM roles with specific permissions that must trust the IAM Roles Anywhere service. Trust anchors are bound to IAM roles through the `aws:SourceArn` condition key.
- Profiles: Used to specify roles that IAM Roles Anywhere can assume to obtain temporary credentials. We can also use session policies in profiles to restrict the permissions of temporary sessions.

## Use Cases

- IoT devices accessing cloud services, such as uploading application logs to Amazon S3
- Data center equipment backing up data to S3
- Data center machines accessing cloud resources in hybrid cloud deployments
- Some unreasonable requirements that don't follow best practices. For example: startups where developers want local access to S3, and the boss is in a hurry and agrees

# Configuration Process: Step by Step
## Step 0: Prerequisites
1. Device has [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed
2. Device has downloaded [Credential-helper](https://docs.aws.amazon.com/zh_cn/rolesanywhere/latest/userguide/credential-helper.html). Place the executable in the `~/.aws/` directory, or any other directory as long as you remember the path and the file has execute permissions.
    > Darwin is macOS
## Step 1: Root CA Configuration

First, you need to have a Root CA. If you already have a PKI infrastructure, you can skip this step. Here we use AWS Private Certificate Authority to manage the PKI infrastructure. More information: [What is AWS Private CA?](https://docs.aws.amazon.com/privateca/latest/userguide/PcaWelcome.html)

Of course, you can also maintain your own or integrate with existing PKI infrastructure, such as using OpenSSL or EasyRSA to generate a self-managed PKI infrastructure.

AWS IAM Roles Anywhere also supports external CA certificates.

Below is the AWS Private Certificate Authority configuration process:
1. Log in to AWS Private Certificate Authority
2. Click Create a private CA
3. On the creation page, select General-purpose for Mode options, Root for CA type options; fill in the corresponding Subject distinguished name options information, scroll down and click create
    ![create private ca](https://gallery-lsky.silentmo.cn/i_blog/2025/07/iam-role-anywhere-1.png)

## Step 2: Create Trust Anchors
1. Log in to IAM Console, click Roles, find Roles Anywhere
    ![roles-anywhere](https://gallery-lsky.silentmo.cn/i_blog/2025/07/iam-role-anywhere-2.png)
2. Click manage, then click Create a trust anchor
    ![roles-anywhere-manage](https://gallery-lsky.silentmo.cn/i_blog/2025/07/iam-role-anywhere-2.png)
    ![create-trust-anchor](https://gallery-lsky.silentmo.cn/i_blog/2025/07/iam-role-anywhere-3.png)
3. On the creation page, select the corresponding CA
    ![create-trust-anchor-page](https://gallery-lsky.silentmo.cn/i_blog/2025/07/iam-role-anywhere-4.png)

## Step 3: Create IAM Roles
1. Create a role in IAM Console using the Roles Anywhere use case
    ![create-iam-role](https://gallery-lsky.silentmo.cn/i_blog/2025/07/iam-role-anywhere-5.png)
2. On the Add permissions page, select the corresponding permissions. This example uses AmazonS3FullAccess
    ![iam-role-add-permissions](https://gallery-lsky.silentmo.cn/i_blog/2025/07/iam-role-anywhere-6.png)
3. Finally, enter the Role name and click Create role

## Step 4: Generate Device Certificate
1. Request a private certificate in AWS Certificate Manager
    ![request certificate](https://gallery-lsky.silentmo.cn/i_blog/2025/07/iam-role-anywhere-7.png)
    ![request certificate-1](https://gallery-lsky.silentmo.cn/i_blog/2025/07/iam-role-anywhere-8.png)
2. View the details of the previously generated certificate, click Export, export the certificate (requires setting a password)
    ![export certificate](https://gallery-lsky.silentmo.cn/i_blog/2025/07/iam-role-anywhere-9.png)
    ![export certificate-1](https://gallery-lsky.silentmo.cn/i_blog/2025/07/iam-role-anywhere-10.png)
3. After exporting the certificate, download the Certificate body as `device.crt` file, download the Certificate private key as `device-pass.key` file
    ![download certificate](https://gallery-lsky.silentmo.cn/i_blog/2025/07/iam-role-anywhere-11.png)
4. Decrypt the private key file in local terminal
    ```shell
    openssl rsa -in device-pass.key -out device.key
    ```

## Step 5: Configure AWS Config
1. Edit the configuration file `vi ~/.aws/config`, see **Config File Parameter Description** below for parameter details
    ```shell
    [profile test-role-anywhere]
    region = us-east-1
    credential_process = /home/mile/.aws/aws_signing_helper credential-process \
    --certificate /app/device.crt --private-key /app/device.key \
    --trust-anchor-arn arn:aws:rolesanywhere:<region>:<account>:trust-anchor/<TA_ID> \
    --profile-arn arn:aws:rolesanywhere:<region>:<account>:profile/<PROFILE_ID> \
    --role-arn arn:aws:iam::<account>:role/<role-name> 
    ```
2. Test
    ```shell
    aws s3 ls
    ```
    ![test-shell](https://gallery-lsky.silentmo.cn/i_blog/2025/07/iam-role-anywhere-14.png)

**Config File Parameter Description**

1. credential_process: Path to the credential-helper executable file
2. --certificate: The crt file from earlier
3. --private-key: The decrypted private key file
4. --trust-anchor-arn: ARN of the trust anchor
    ![trust-anchor-arn](https://gallery-lsky.silentmo.cn/i_blog/2025/07/iam-role-anywhere-12.png)
5. --profile-arn: ARN of Roles anywhere
    ![profile-arn](https://gallery-lsky.silentmo.cn/i_blog/2025/07/iam-role-anywhere-13.png)
6. --role-arn: ARN of the IAM Role


# References
[1] [AWS Blogs - Use Amazon Cloud Technology IAM Roles Anywhere to Authorize Off-Cloud Devices to Access AWS Resources](https://aws.amazon.com/cn/blogs/china/use-amazon-cloud-technology-iam-roles-anywhere-to-authorize-off-cloud-devices-to-access-aws-resources)
[2] [Get temporary security credentials from IAM Roles Anywhere](https://docs.aws.amazon.com/zh_cn/rolesanywhere/latest/userguide/credential-helper.html)
