# AWS EC2 + NGINX Web Application Deployment

## 1. Project Overview

This project demonstrates how to prepare an AWS environment for deploying a basic web application on an EC2 instance and serving it using NGINX.

We will first verify AWS access, permissions, and networking. After the prerequisites are confirmed, the project can proceed to EC2 provisioning, NGINX installation, website verification, and troubleshooting.

This README documents the preparation completed so far. It does not claim that the web application has been deployed.

## 2. Project Objectives

By completing this project, we aim to learn how to:

- Configure the AWS CLI using a named profile.
- Verify the identity used to access AWS.
- Configure an IAM user and group with limited permissions.
- Understand EC2 discovery permissions.
- Inspect the default VPC and its subnets.
- Prepare the environment for an EC2 instance.
- Deploy and verify a basic NGINX web server in a later stage.
- Document commands, expected results, and troubleshooting steps.

## 3. Environment and Configuration

| Item | Current configuration |
|---|---|
| Local environment | WSL Ubuntu |
| Project directory | `~/projects/aws-ec2-nginx-deployment` |
| AWS Region | `us-west-2` |
| Region name | US West (Oregon) |
| AWS CLI profile | `ec2-nginx-lab` |
| IAM user | `ec2-nginx-lab-user` |
| IAM group | `EC2-NGINX-Lab-Group` |
| IAM discovery policy | `EC2-NGINX-Lab-Discovery` |
| VPC | Default VPC |
| EC2 instance | Not launched |
| NGINX deployment | Not started |

**Security note:** AWS access keys and secret access keys must remain private. Never paste their actual values into GitHub, a README, screenshots, or public messages.

---

## 4. Prerequisites

Before proceeding with the deployment, we need:

1. An AWS account.
2. AWS CLI installed in WSL Ubuntu.
3. An IAM identity with the required permissions.
4. AWS CLI credentials configured locally.
5. A verified AWS Region.
6. A suitable VPC and subnet.
7. An EC2 key pair or another supported secure access method.
8. An appropriately configured security group.
9. Permission to create and manage the resources required by the lab.

At this stage, we have verified AWS CLI authentication and selected EC2 discovery permissions. We have not yet verified all permissions needed to launch an EC2 instance.

## 5. IAM User and Group Setup

### 5.1 Create an IAM group

The IAM group created for this lab is:

`EC2-NGINX-Lab-Group`

An IAM group allows permissions to be managed centrally and assigned to its member users.

### 5.2 Create an IAM user

The IAM user created for this lab is:

`ec2-nginx-lab-user`

Console access was disabled for this user because the current workflow uses the AWS CLI from WSL Ubuntu.

### 5.3 Add the user to the group

The user `ec2-nginx-lab-user` was added to `EC2-NGINX-Lab-Group`.

This allows the user to receive permissions through the group policy.

### 5.4 Create the discovery policy

An inline IAM policy named `EC2-NGINX-Lab-Discovery` was attached to the group.

The policy allows selected EC2 read-only discovery operations. It does not grant permission to launch, stop, or terminate EC2 instances.

The policy JSON used was:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2ReadOnlyDiscovery",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeImages",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeInstances",
        "ec2:DescribeKeyPairs",
        "ec2:DescribeRegions",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSubnets",
        "ec2:DescribeVpcs"
      ],
      "Resource": "*"
    }
  ]
}
```

#### Understanding the policy

- `Version`: Specifies the IAM policy language version.
- `Statement`: Contains the permission rules.
- `Sid`: A descriptive identifier for this statement.
- `Effect: Allow`: Allows the listed actions, subject to other applicable IAM policies and restrictions.
- `Action`: Lists the permitted EC2 discovery operations.
- `Resource: "*"`: Applies these describe permissions across resources supported by these actions.

**Important:** This policy is intended for discovery and inspection. It is not a complete deployment policy. Additional carefully scoped permissions will be required for later resource-creation tasks.

## 6. Configure AWS CLI Using a Named Profile

### 6.1 Start AWS CLI configuration

Run this command in WSL Ubuntu:

```bash
aws configure --profile ec2-nginx-lab
```

The terminal prompts for four values.

```text
AWS Access Key ID [None]:
AWS Secret Access Key [None]:
Default region name [None]: us-west-2
Default output format [None]: json
```

Enter the access key and secret access key created for `ec2-nginx-lab-user` directly into your terminal when prompted. Do not place actual credential values in this README.

The configured values are:

- Profile name: `ec2-nginx-lab`
- Region: `us-west-2`
- Output format: `json`
- Credential storage: AWS CLI shared credentials file

The CLI may display an informational tip about `aws login`. This does not mean the configuration failed. The subsequent identity verification is the important check.

### 6.2 Verify the AWS identity

Run:

```bash
aws sts get-caller-identity --profile ec2-nginx-lab
```

The command returned a successful response in this form:

```json
{
    "UserId": "*************",
    "Account": "*************",
    "Arn": "arn:aws:iam::*************:user/ec2-nginx-lab-user"
}
```

The account and user identifier values are masked here for privacy.

#### What does this command do?

`aws sts get-caller-identity` asks AWS Security Token Service to identify the IAM identity used for the request.

The `--profile ec2-nginx-lab` option ensures that the command uses the named profile instead of relying on the default profile.

The ARN confirms that the configured identity is `ec2-nginx-lab-user`.

**Result: Passed.** AWS authentication works with the named profile.

### 6.3 Verify the CLI configuration

Run:

```bash
aws configure list --profile ec2-nginx-lab
```

The observed output was:

```text
NAME       : VALUE                    : TYPE                     : LOCATION
profile    : ec2-nginx-lab            : manual                   : --profile
access_key : ****************         : shared-credentials-file  :
secret_key : ****************         : shared-credentials-file  :
region     : us-west-2                : config-file              : ~/.aws/config
```

The credential values are masked. The displayed secret-key suffix is included only to reflect the observed terminal output; never publish actual credential material.

#### What does this output tell us?

- `profile`: The command uses `ec2-nginx-lab`.
- `access_key`: The access key is loaded from the shared credentials file.
- `secret_key`: The secret access key is loaded from the shared credentials file.
- `region`: The configured Region is `us-west-2`.
- `~/.aws/config`: The AWS CLI configuration file stores the Region and other profile configuration.
- `~/.aws/credentials`: The shared credentials file normally stores the access key and secret access key.

**Result: Passed.** The expected profile and Region are configured.

---

## 7. Verify EC2 Discovery Permissions

Before creating an EC2 instance, verify that the IAM user can inspect the existing EC2 environment.

### 7.1 Check existing EC2 instances

Run:

```bash
aws ec2 describe-instances \
  --region us-west-2 \
  --profile ec2-nginx-lab \
  --output table
```

Observed output:

```text
-------------------
|DescribeInstances|
+-----------------+
```

#### Explanation

- `aws ec2 describe-instances`: Requests information about EC2 instances.
- `--region us-west-2`: Runs the request in Oregon.
- `--profile ec2-nginx-lab`: Uses the configured IAM user.
- `--output table`: Formats the result as a table.

The command succeeded, but no instances were returned in the displayed output.

This is a valid result when there are no matching instances visible to the IAM identity in that Region.

**Result: Passed.** The user can run `DescribeInstances`.

### 7.2 Check existing EC2 key pairs

Run:

```bash
aws ec2 describe-key-pairs \
  --region us-west-2 \
  --profile ec2-nginx-lab \
  --output table
```

Observed output:

```text
------------------
|DescribeKeyPairs|
+----------------+
```

#### Explanation

An EC2 key pair is commonly used to authenticate SSH access to a Linux EC2 instance.

The command lists key pairs visible in the selected Region.

The empty table indicates that no key pairs were returned.

**Result: Passed.** The user can run `DescribeKeyPairs`. We have not created a key pair yet.

---

## 8. Inspect the Default VPC

### 8.1 Why do we need a VPC?

A Virtual Private Cloud (VPC) is a logically isolated network in AWS. An EC2 instance must be associated with a VPC and a subnet.

For this beginner lab, we will inspect the existing default VPC before considering whether any network resources need to be created.

### 8.2 List VPCs in Oregon

Run:

```bash
aws ec2 describe-vpcs \
  --region us-west-2 \
  --profile ec2-nginx-lab \
  --query "Vpcs[].{VpcId:VpcId,State:State,IsDefault:IsDefault,CidrBlock:CidrBlock}" \
  --output table
```

Observed output:

```text
----------------------------------------------------------------------
|                            DescribeVpcs                            |
+----------------+------------+------------+-------------------------+
|    CidrBlock   | IsDefault  |   State    |          VpcId          |
+----------------+------------+------------+-------------------------+
|  172.31.0.0/16 |  True      |  available |  vpc-0307ce67a0df86851  |
+----------------+------------+------------+-------------------------+
```

#### Understanding the command

- `describe-vpcs`: Lists VPCs available to the current identity in the selected Region.
- `--region us-west-2`: Limits the request to Oregon.
- `--profile ec2-nginx-lab`: Uses the named IAM profile.
- `--query`: Selects the fields we want to display.
- `VpcId`: The VPC identifier.
- `State`: The VPC state.
- `IsDefault`: Indicates whether AWS identifies the VPC as the default VPC.
- `CidrBlock`: The VPC's IPv4 address range.
- `--output table`: Displays the selected fields in a readable table.

#### Results

| Field | Observed value | Meaning |
|---|---|---|
| VPC ID | `vpc-0307ce67a0df86851` | Identifier of the discovered VPC |
| State | `available` | The VPC is available |
| IsDefault | `True` | AWS identifies it as the default VPC |
| CIDR block | `172.31.0.0/16` | The VPC's IPv4 address range |

The `172.31.0.0/16` range contains private IPv4 addresses. The CIDR block is the overall network range, not the public IP address of an EC2 instance.

**Result: Passed.** A default VPC is available in Oregon.

No VPC was created or modified.

---

## 9. Inspect Subnets in the Default VPC

### 9.1 What is a subnet?

A subnet is a smaller IP address range within a VPC. An EC2 instance is launched into a subnet, which is associated with an Availability Zone.

Subnets, route tables, internet gateways, and security groups all contribute to how an instance communicates with other resources and the internet.

### 9.2 List subnets belonging to the discovered VPC

Run:

```bash
aws ec2 describe-subnets \
  --region us-west-2 \
  --profile ec2-nginx-lab \
  --filters "Name=vpc-id,Values=vpc-0307ce67a0df86851" \
  --query "Subnets[].{SubnetId:SubnetId,AZ:AvailabilityZone,State:State,DefaultForAZ:DefaultForAz,PublicIP:MapPublicIpOnLaunch,CIDR:CidrBlock}" \
  --output table
```

Observed output:

```text
-------------------------------------------------------------------------------------------------------
|                                           DescribeSubnets                                           |
+------------+-----------------+---------------+-----------+------------+-----------------------------+
|     AZ     |      CIDR       | DefaultForAZ  | PublicIP  |   State    |          SubnetId           |
+------------+-----------------+---------------+-----------+------------+-----------------------------+
|  us-west-2a|  172.31.32.0/20 |  True         |  True     |  available |  subnet-02ecdd68edfbf7326   |
|  us-west-2c|  172.31.0.0/20  |  True         |  True     |  available |  subnet-003413e14c2375f32   |
|  us-west-2d|  172.31.48.0/20 |  True         |  True     |  available |  subnet-0ce894c858eab9c2e   |
|  us-west-2b|  172.31.16.0/20 |  True         |  True     |  available |  subnet-006fde08c5f4e902c  |
+------------+-----------------+---------------+-----------+------------+-----------------------------+
```

### 9.3 Understanding the subnet command

- `describe-subnets`: Requests information about subnets.
- `--filters "Name=vpc-id,Values=..."`: Returns only subnets associated with the specified VPC.
- `SubnetId`: The identifier of each subnet.
- `AvailabilityZone`: The Availability Zone containing the subnet.
- `State`: The current subnet state.
- `DefaultForAz`: Indicates whether the subnet is the default subnet for its Availability Zone.
- `MapPublicIpOnLaunch`: Indicates whether instances launched into the subnet are configured to receive a public IPv4 address by default, subject to launch configuration and other conditions.
- `CidrBlock`: The subnet's IPv4 address range.

### 9.4 Results

| Availability Zone | Subnet ID | CIDR block | Default subnet | Auto-assign public IPv4 | State |
|---|---|---|---|---|---|
| `us-west-2a` | `subnet-02ecdd68edfbf7326` | `172.31.32.0/20` | Yes | Yes | `available` |
| `us-west-2c` | `subnet-003413e14c2375f32` | `172.31.0.0/20` | Yes | Yes | `available` |
| `us-west-2d` | `subnet-0ce894c858eab9c2e` | `172.31.48.0/20` | Yes | Yes | `available` |
| `us-west-2b` | `subnet-006fde08c5f4e902c` | `172.31.16.0/20` | Yes | Yes | `available` |

We found four available default subnets across four Availability Zones.

The `PublicIP` column in the command's output corresponds to `MapPublicIpOnLaunch`. A value of `True` indicates the subnet's default public-IP assignment setting; it does not by itself prove that an instance will be reachable from the internet. Internet reachability also depends on routing, security-group rules, network ACLs, and other configuration.

**Result: Passed.** We have identified available subnets in the default VPC.

No subnet was created or modified.

---

## 10. Progress Checklist

The following work has been completed and verified.

- [x] AWS account is available.
- [x] IAM group `EC2-NGINX-Lab-Group` created.
- [x] IAM user `ec2-nginx-lab-user` created.
- [x] Console access disabled for the lab user.
- [x] Lab user added to the IAM group.
- [x] Discovery policy `EC2-NGINX-Lab-Discovery` attached to the group.
- [x] AWS CLI profile `ec2-nginx-lab` configured.
- [x] AWS identity verified using STS.
- [x] CLI profile and Region verified.
- [x] EC2 instance discovery command succeeded.
- [x] EC2 key-pair discovery command succeeded.
- [x] Default VPC identified.
- [x] Four available subnets identified.

## 11. What Has Not Been Done Yet?

The following steps remain pending.

- [ ] Confirm the existing security groups.
- [ ] Inspect the VPC's route table and internet gateway configuration.
- [ ] Decide on a key-pair and SSH access approach.
- [ ] Design and attach the additional IAM permissions needed for the lab.
- [ ] Confirm the EC2 instance type and operating system image.
- [ ] Review potential AWS charges and free-tier eligibility before provisioning.
- [ ] Launch the EC2 instance.
- [ ] Verify instance status and connectivity.
- [ ] Install NGINX.
- [ ] Deploy or configure the sample web page.
- [ ] Verify the website from a browser.
- [ ] Document troubleshooting and cleanup commands.

## 12. Security and Cost Considerations

- Keep AWS access keys out of source control.
- Use a dedicated IAM identity for the lab rather than unrestricted administrative credentials.
- Grant only the permissions required for each stage.
- Do not open SSH (port 22) to the entire internet unless there is a justified, temporary need.
- For a web server, allow only the required inbound HTTP/HTTPS traffic.
- Review the instance type, storage, public IPv4 usage, and other potential charges before launching resources.
- Stop or terminate lab resources when they are no longer required, according to the lab's cleanup plan.

## 13. Current Project Status

**Current stage: AWS access and network discovery completed.**

The AWS CLI is authenticated through the `ec2-nginx-lab` profile in `us-west-2`. The selected IAM user can successfully inspect EC2 instances, key pairs, VPCs, and subnets using the tested commands.

The default VPC and its four available subnets have been identified. The environment is not yet ready to launch an instance until the remaining network, security, access, and IAM prerequisites are reviewed.

**No EC2 instance or NGINX web application has been deployed at this point.**

The next step is to inspect existing security groups and network routing, continuing one verification task at a time.
