# AWS EC2 + NGINX — Complete Manual Commands When No Existing VPC Is Available

**Repository:** `aws-ec2-nginx-deployment`  
**File:** `NETWORKING-MANUAL-COMMANDS-NO-EXISTING-VPC.md`  
**AWS Region:** `us-west-2` (Oregon)  
**AWS CLI Profile:** `ec2-nginx-lab`  
**Instance Type:** `t3.micro`  
**AMI:** Latest available Amazon Linux 2023 x86-64

## Purpose

This guide is for someone starting with an AWS account who does not have an existing VPC and subnet to reuse.

We will prepare AWS access, configure permissions, discover the required AWS resources, create networking from scratch, launch an EC2 instance, install NGINX, verify the website, and clean up the resources.

### Resources we will create

1. IAM user and group, with the required permissions.
2. VPC.
3. Public subnet.
4. Internet Gateway.
5. Internet Gateway attachment.
6. Route table.
7. Default internet route.
8. Route table association.
9. Security group.
10. EC2 instance running NGINX.

**Important:** IAM setup requires an administrator or another identity with IAM management permissions. Do not use the new lab user's credentials to create its own IAM user, group, or policies.

---

# PART 1 — Prepare the AWS environment

## Step 1 — Open the terminal

Open Ubuntu in WSL.

Check the operating system:

```bash
uname -a
```

Check AWS CLI:

```bash
aws --version
```

Check whether the AWS CLI is installed.

If it is not installed, follow the official AWS CLI installation instructions:

https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

Check whether `curl` is available:

```bash
curl --version
```

We will use `curl` later to retrieve the public IP address and test the website.

## Step 2 — Understand the IAM setup

We will use the following names:

| Item | Name |
|---|---|
| IAM user | `ec2-nginx-lab-user` |
| IAM group | `EC2-NGINX-Lab-Group` |
| Discovery policy | `EC2-NGINX-Lab-Discovery` |
| Deployment policy | `EC2-NGINX-Lab-Deployment` |
| AWS CLI profile | `ec2-nginx-lab` |

We use two policies because they have different purposes.

- Discovery policy: allows us to inspect AWS resources.
- Deployment policy: allows us to create and manage resources for this lab.

## Step 3 — Create the IAM group

Sign in to the AWS Management Console using an administrator account.

Navigate to:

**IAM → User groups → Create group**

Enter:

```text
EC2-NGINX-Lab-Group
```

We will attach the policies to this group so that users added to the group receive the required permissions.

## Step 4 — Create the discovery policy

Navigate to:

**IAM → Policies → Create policy → JSON**

Enter the following JSON.

Policy name:

`EC2-NGINX-Lab-Discovery`

Policy JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeImages",
        "ec2:DescribeInstanceTypeOfferings",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeInstances",
        "ec2:DescribeInternetGateways",
        "ec2:DescribeKeyPairs",
        "ec2:DescribeRegions",
        "ec2:DescribeRouteTables",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSubnets",
        "ec2:DescribeVpcs"
      ],
      "Resource": "*"
    }
  ]
}
```

This policy allows us to inspect:

- Availability Zones.
- AMIs.
- Instance types and their availability.
- Existing EC2 instances.
- Internet Gateways.
- Key pairs.
- Route tables.
- Security groups.
- Subnets.
- VPCs.

It does not grant permission to create or modify these resources.

Create the policy.

## Step 5 — Create the deployment policy

Navigate to:

**IAM → Policies → Create policy → JSON**

Policy name:

`EC2-NGINX-Lab-Deployment`

Use the following lab-oriented policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVpc",
        "ec2:ModifyVpcAttribute",
        "ec2:CreateSubnet",
        "ec2:ModifySubnetAttribute",
        "ec2:CreateInternetGateway",
        "ec2:AttachInternetGateway",
        "ec2:DetachInternetGateway",
        "ec2:DeleteInternetGateway",
        "ec2:CreateRouteTable",
        "ec2:CreateRoute",
        "ec2:DeleteRoute",
        "ec2:AssociateRouteTable",
        "ec2:DisassociateRouteTable",
        "ec2:DeleteRouteTable",
        "ec2:DeleteSubnet",
        "ec2:DeleteVpc",
        "ec2:CreateSecurityGroup",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:RevokeSecurityGroupIngress",
        "ec2:DeleteSecurityGroup",
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "ec2:CreateTags",
        "ec2:DeleteTags",
        "ec2:Describe*"
      ],
      "Resource": "*"
    }
  ]
}
```

This policy is intended for the learning lab and grants broad EC2 permissions on `Resource: "*"`. It is not a production least-privilege policy. For production, permissions should be restricted by resource, action, and conditions wherever supported.

Create the policy.

## Step 6 — Attach both policies to the IAM group

Navigate to:

**IAM → User groups → EC2-NGINX-Lab-Group → Permissions**

Choose **Add permissions → Attach policies**.

Attach:

1. `EC2-NGINX-Lab-Discovery`
2. `EC2-NGINX-Lab-Deployment`

Verify that both policies are attached.

## Step 7 — Create the IAM user

Navigate to:

**IAM → Users → Create user**

Enter:

```text
ec2-nginx-lab-user
```

For programmatic AWS CLI access, create an access key for the user.

Add the user to:

```text
EC2-NGINX-Lab-Group
```

Complete the user creation.

Create an access key for CLI use and securely save:

- Access key ID.
- Secret access key.

**Important:** The secret access key is shown only when it is created. Store it securely. Never commit it to GitHub or share it in screenshots or recordings.

---

# PART 2 — Configure AWS CLI

## Step 8 — Configure the AWS CLI profile

Return to Ubuntu.

Run:

```bash
aws configure --profile ec2-nginx-lab
```

Enter the values when prompted:

```text
AWS Access Key ID: <YOUR_ACCESS_KEY_ID>
AWS Secret Access Key: <YOUR_SECRET_ACCESS_KEY>
Default region name: us-west-2
Default output format: json
```

This creates a named AWS CLI profile.

We use a profile so that the lab's AWS credentials and configuration are kept separate from other profiles.

## Step 9 — Set the profile and region variables

```bash
export PROFILE=ec2-nginx-lab
export REGION=us-west-2
```

Verify:

```bash
echo "$PROFILE"
echo "$REGION"
```

Expected:

```text
ec2-nginx-lab
us-west-2
```

Variable explanation:

| Variable | Purpose |
|---|---|
| `PROFILE` | Selects the AWS CLI credentials profile. |
| `REGION` | Selects the AWS region for our resources. |

## Step 10 — Verify AWS CLI access

Run a read-only discovery command:

```bash
aws ec2 describe-regions \
  --profile "$PROFILE" \
  --region "$REGION" \
  --query 'Regions[].RegionName' \
  --output table
```

If the command succeeds, AWS CLI can communicate with the EC2 service using the configured profile.

If you receive `AccessDenied`, check that both policies are attached to the IAM group and that the user belongs to that group.

Do not continue until the required permissions are working.

---

# PART 3 — Discover the required AWS resources

Before creating networking, we will check the Availability Zone, instance type, AMI, and key pair.

## Step 11 — Discover Availability Zones

```bash
aws ec2 describe-availability-zones \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'AvailabilityZones[?State==`available`].[ZoneName,State]' \
  --output table
```

For this lab, we will use:

```bash
export AZ="${REGION}a"
```

Verify:

```bash
echo "$AZ"
```

Expected:

```text
us-west-2a
```

An Availability Zone is a separate location within an AWS Region. Our subnet and EC2 instance will be created in the selected Availability Zone.

## Step 12 — Check whether the instance type is available

We are using `t3.micro`.

Check its availability in the selected Availability Zone:

```bash
aws ec2 describe-instance-type-offerings \
  --location-type availability-zone \
  --filters \
    "Name=instance-type,Values=t3.micro" \
    "Name=location,Values=$AZ" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'InstanceTypeOfferings[].{InstanceType:InstanceType,Location:Location}' \
  --output table
```

If `t3.micro` is returned, it is offered in the selected Availability Zone.

If no result is returned, select an Availability Zone where the instance type is offered and update `AZ`.

## Step 13 — Discover the latest Amazon Linux 2023 AMI

First, inspect the available images:

```bash
aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=al2023-ami-2023.*-x86_64" \
    "Name=state,Values=available" \
    "Name=architecture,Values=x86_64" \
    "Name=root-device-type,Values=ebs" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'Images | sort_by(@, &CreationDate)[-1].[ImageId,Name,CreationDate]' \
  --output table
```

Now store the latest AMI ID in a variable:

```bash
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=al2023-ami-2023.*-x86_64" \
    "Name=state,Values=available" \
    "Name=architecture,Values=x86_64" \
    "Name=root-device-type,Values=ebs" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'Images | sort_by(@, &CreationDate)[-1].ImageId' \
  --output text)

test -n "$AMI_ID" && test "$AMI_ID" != "None" \
  && echo "AMI discovery successful: $AMI_ID"
```

Variable explanation:

- `AMI_ID` stores the Amazon Machine Image ID.
- We will use it when launching EC2.
- The AMI is discovered; we are not creating a new AMI.

## Step 14 — Verify the EC2 key pair

Set the key pair name:

```bash
export KEY_NAME=ec2-nginx-lab-key
```

Check whether it exists in the selected region:

```bash
aws ec2 describe-key-pairs \
  --key-names "$KEY_NAME" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'KeyPairs[].{KeyName:KeyName,KeyType:KeyType,CreateTime:CreateTime}' \
  --output table
```

A key pair is used for SSH access to the EC2 instance.

You need the matching private key file on your local machine.

If the key pair does not exist, create one:

```bash
aws ec2 create-key-pair \
  --key-name "$KEY_NAME" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'KeyMaterial' \
  --output text > "${KEY_NAME}.pem"
```

Restrict its permissions:

```bash
chmod 400 "${KEY_NAME}.pem"
```

**Important:** Save the private key securely. AWS does not provide the same private key again after creation. Do not commit the PEM file to GitHub.

If you already have the key pair, do not create another one. Make sure its matching private key file is available.

---

# PART 4 — Create networking from scratch

We will create:

1. VPC.
2. Public subnet.
3. Internet Gateway.
4. Internet Gateway attachment.
5. Route table.
6. Default route.
7. Route table association.

## Step 15 — Create a VPC

A VPC is our logically isolated network in AWS.

We will use the CIDR range:

```text
10.0.0.0/16
```

Create the VPC:

```bash
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'Vpc.VpcId' \
  --output text)

echo "VPC_ID=$VPC_ID"
```

Variable explanation:

- `VPC_ID` stores the ID of the newly created VPC.

Enable DNS support:

```bash
aws ec2 modify-vpc-attribute \
  --vpc-id "$VPC_ID" \
  --enable-dns-support '{"Value":true}' \
  --region "$REGION" \
  --profile "$PROFILE"
```

Enable DNS hostnames:

```bash
aws ec2 modify-vpc-attribute \
  --vpc-id "$VPC_ID" \
  --enable-dns-hostnames '{"Value":true}' \
  --region "$REGION" \
  --profile "$PROFILE"
```

Verify:

```bash
aws ec2 describe-vpcs \
  --vpc-ids "$VPC_ID" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'Vpcs[0].[VpcId,CidrBlock,State]' \
  --output table
```

Expected: the VPC ID, CIDR `10.0.0.0/16`, and state `available`.

## Step 16 — Create a public subnet

A subnet is a smaller network range inside our VPC.

We will use:

```text
10.0.1.0/24
```

Create the subnet:

```bash
SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block 10.0.1.0/24 \
  --availability-zone "$AZ" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'Subnet.SubnetId' \
  --output text)

echo "SUBNET_ID=$SUBNET_ID"
```

Variable explanation:

- `SUBNET_ID` stores the newly created subnet ID.
- The subnet belongs to the VPC identified by `VPC_ID`.

Enable automatic public IPv4 address assignment:

```bash
aws ec2 modify-subnet-attribute \
  --subnet-id "$SUBNET_ID" \
  --map-public-ip-on-launch \
  --region "$REGION" \
  --profile "$PROFILE"
```

Verify:

```bash
aws ec2 describe-subnets \
  --subnet-ids "$SUBNET_ID" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'Subnets[0].[SubnetId,VpcId,CidrBlock,AvailabilityZone,MapPublicIpOnLaunch]' \
  --output table
```

The public IP assignment setting should be `True`.

## Step 17 — Create an Internet Gateway

An Internet Gateway allows communication between the VPC and the internet when the route table and other network settings permit it.

Create it:

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)

echo "IGW_ID=$IGW_ID"
```

Variable explanation:

- `IGW_ID` stores the Internet Gateway ID.

## Step 18 — Attach the Internet Gateway to the VPC

```bash
aws ec2 attach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID" \
  --region "$REGION" \
  --profile "$PROFILE"
```

Verify:

```bash
aws ec2 describe-internet-gateways \
  --internet-gateway-ids "$IGW_ID" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'InternetGateways[0].Attachments' \
  --output table
```

The attachment should show the VPC ID and state `available`.

## Step 19 — Create a route table

A route table contains rules that determine where network traffic is directed.

Create it:

```bash
RTB_ID=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'RouteTable.RouteTableId' \
  --output text)

echo "RTB_ID=$RTB_ID"
```

Variable explanation:

- `RTB_ID` stores the route table ID.
- The route table belongs to our new VPC.

## Step 20 — Add the default internet route

Add a route for all IPv4 destinations:

```bash
aws ec2 create-route \
  --route-table-id "$RTB_ID" \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id "$IGW_ID" \
  --region "$REGION" \
  --profile "$PROFILE"
```

This directs traffic destined for other IPv4 networks through the Internet Gateway.

## Step 21 — Associate the route table with the subnet

```bash
ASSOC_ID=$(aws ec2 associate-route-table \
  --route-table-id "$RTB_ID" \
  --subnet-id "$SUBNET_ID" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'AssociationId' \
  --output text)

echo "ASSOC_ID=$ASSOC_ID"
```

Variable explanation:

- `ASSOC_ID` stores the route table association ID.
- This connects our public subnet to the route table we created.

## Step 22 — Verify the route table

```bash
aws ec2 describe-route-tables \
  --route-table-ids "$RTB_ID" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,GatewayId,State]' \
  --output table
```

Expected route:

```text
Destination: 0.0.0.0/0
Gateway:     igw-xxxxxxxx
State:       active
```

At this point, the subnet has a route to the Internet Gateway and automatic public IPv4 assignment is enabled.

---

# PART 5 — Create the security group

## Step 23 — Create the security group

A security group acts as a virtual firewall attached to our EC2 instance.

We need two inbound rules:

| Protocol | Port | Source | Purpose |
|---|---:|---|---|
| TCP | 22 | Your public IP `/32` | SSH access |
| TCP | 80 | `0.0.0.0/0` | HTTP website access |

Create the security group:

```bash
SG_ID=$(aws ec2 create-security-group \
  --group-name ec2-nginx-lab-sg \
  --description "Security group for manual EC2 NGINX lab" \
  --vpc-id "$VPC_ID" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'GroupId' \
  --output text)

test -n "$SG_ID" && test "$SG_ID" != "None" \
  && echo "Security group created: $SG_ID"
```

Variable explanation:

- `SG_ID` stores the new security group ID.
- The security group is created inside our new VPC.

## Step 24 — Retrieve your current public IP

```bash
MY_IP=$(curl -4 -fsS https://checkip.amazonaws.com | tr -d '\n')

if [ -n "$MY_IP" ]; then
  echo "Public IP retrieved successfully."
else
  echo "Unable to retrieve public IP. Stop and check connectivity."
fi
```

Variable explanation:

- `MY_IP` stores your current public IPv4 address.
- We use it to restrict SSH access.

## Step 25 — Add the SSH inbound rule

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 22 \
  --cidr "${MY_IP}/32" \
  --region "$REGION" \
  --profile "$PROFILE"
```

The `/32` means that only the specified IPv4 address is allowed by this rule.

## Step 26 — Add the HTTP inbound rule

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --region "$REGION" \
  --profile "$PROFILE"
```

This allows users to access the NGINX website over HTTP.

## Step 27 — Verify security group rules

```bash
aws ec2 describe-security-groups \
  --group-ids "$SG_ID" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'SecurityGroups[0].{GroupId:GroupId,GroupName:GroupName,VpcId:VpcId,InboundRules:IpPermissions[*].{Protocol:IpProtocol,FromPort:FromPort,ToPort:ToPort}}' \
  --output json
```

Check that:

- Port 22 is configured for SSH.
- Port 80 is configured for HTTP.
- The security group belongs to the new VPC.

---

# PART 6 — Launch the EC2 instance

## Step 28 — Prepare the NGINX User Data script

User Data allows us to execute commands automatically when the EC2 instance first starts.

Create the script:

```bash
cat > user-data.sh <<'EOF'
#!/bin/bash

dnf update -y
dnf install -y nginx

systemctl enable nginx
systemctl start nginx

cat > /usr/share/nginx/html/index.html <<'HTML'
<h1>My AWS EC2 Nginx Lab</h1>
<p>Nginx is running successfully!</p>
HTML
EOF
```

Verify the file:

```bash
cat user-data.sh
```

The script:

1. Updates the operating system packages.
2. Installs NGINX.
3. Enables NGINX at startup.
4. Starts NGINX.
5. Creates a custom HTML page.

## Step 29 — Launch the EC2 instance

We will use:

- AMI: `$AMI_ID`
- Instance type: `t3.micro`
- Key pair: `$KEY_NAME`
- Subnet: `$SUBNET_ID`
- Security group: `$SG_ID`

Run:

```bash
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t3.micro \
  --key-name "$KEY_NAME" \
  --security-group-ids "$SG_ID" \
  --subnet-id "$SUBNET_ID" \
  --associate-public-ip-address \
  --user-data file://user-data.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=manual-ec2-nginx},{Key=Project,Value=EC2-NGINX-Lab}]' \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "INSTANCE_ID=$INSTANCE_ID"
```

Variable explanation:

- `INSTANCE_ID` stores the EC2 instance ID.
- We use the AMI, instance type, key pair, subnet, and security group to define the instance.
- User Data installs and configures NGINX automatically.

## Step 30 — Verify the EC2 instance

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'Reservations[0].Instances[0].{InstanceId:InstanceId,State:State.Name,Type:InstanceType,AZ:Placement.AvailabilityZone}' \
  --output table
```

Wait until the instance status checks pass:

```bash
aws ec2 wait instance-status-ok \
  --instance-ids "$INSTANCE_ID" \
  --region "$REGION" \
  --profile "$PROFILE"

echo "EC2 instance status checks passed."
```

Remember: successful EC2 status checks do not guarantee that User Data has finished installing NGINX. We will verify the website separately.

## Step 31 — Retrieve the public IP address

```bash
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "PUBLIC_IP=$PUBLIC_IP"
```

Variable explanation:

- `PUBLIC_IP` stores the public IPv4 address of the EC2 instance.
- We use it to test the NGINX website.

## Step 32 — Verify the website using HTTP

```bash
curl -sS -o /dev/null -w 'HTTP status: %{http_code}\n' \
  "http://${PUBLIC_IP}/"
```

Expected:

```text
HTTP status: 200
```

HTTP status `200` indicates that the server responded successfully.

If the first request fails because User Data is still executing, wait briefly and retry.

## Step 33 — Display the website content

```bash
curl -sS "http://${PUBLIC_IP}/"
```

Expected HTML:

```html
<h1>My AWS EC2 Nginx Lab</h1>
<p>Nginx is running successfully!</p>
```

You can also open the website in your browser:

```text
http://<EC2-PUBLIC-IP>
```

Replace the placeholder with the value stored in `PUBLIC_IP`.

At this point, the manual EC2 and NGINX deployment is verified.

---

# PART 7 — Variables used throughout this lab

The following variables were used to avoid manually copying resource IDs between commands.

| Variable | What it stores |
|---|---|
| `PROFILE` | AWS CLI profile |
| `REGION` | AWS region |
| `AZ` | Selected Availability Zone |
| `AMI_ID` | Amazon Linux 2023 AMI ID |
| `KEY_NAME` | EC2 key pair name |
| `VPC_ID` | VPC ID |
| `SUBNET_ID` | Public subnet ID |
| `IGW_ID` | Internet Gateway ID |
| `RTB_ID` | Route table ID |
| `ASSOC_ID` | Route table association ID |
| `SG_ID` | Security group ID |
| `MY_IP` | Your public IPv4 address |
| `INSTANCE_ID` | EC2 instance ID |
| `PUBLIC_IP` | EC2 public IPv4 address |

**Important:** Shell variables are available only in the current terminal session. If you close the terminal, they are lost. You can retrieve resource IDs again using the corresponding `describe` commands.

---

# PART 8 — Cleanup

AWS resources should be removed after the lab is completed to avoid unnecessary charges.

Before deleting resources, make sure you are working with the IDs created for this lab.

## Step 34 — Terminate the EC2 instance

```bash
aws ec2 terminate-instances \
  --instance-ids "$INSTANCE_ID" \
  --region "$REGION" \
  --profile "$PROFILE"
```

Wait for termination:

```bash
aws ec2 wait instance-terminated \
  --instance-ids "$INSTANCE_ID" \
  --region "$REGION" \
  --profile "$PROFILE"

echo "EC2 instance terminated."
```

## Step 35 — Delete the security group

```bash
aws ec2 delete-security-group \
  --group-id "$SG_ID" \
  --region "$REGION" \
  --profile "$PROFILE"
```

## Step 36 — Disassociate the route table

```bash
aws ec2 disassociate-route-table \
  --association-id "$ASSOC_ID" \
  --region "$REGION" \
  --profile "$PROFILE"
```

## Step 37 — Delete the route table

```bash
aws ec2 delete-route-table \
  --route-table-id "$RTB_ID" \
  --region "$REGION" \
  --profile "$PROFILE"
```

## Step 38 — Detach the Internet Gateway

```bash
aws ec2 detach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID" \
  --region "$REGION" \
  --profile "$PROFILE"
```

## Step 39 — Delete the Internet Gateway

```bash
aws ec2 delete-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --region "$REGION" \
  --profile "$PROFILE"
```

## Step 40 — Delete the subnet

```bash
aws ec2 delete-subnet \
  --subnet-id "$SUBNET_ID" \
  --region "$REGION" \
  --profile "$PROFILE"
```

## Step 41 — Delete the VPC

```bash
aws ec2 delete-vpc \
  --vpc-id "$VPC_ID" \
  --region "$REGION" \
  --profile "$PROFILE"
```

## Step 42 — Verify cleanup

Check whether the VPC still exists:

```bash
aws ec2 describe-vpcs \
  --vpc-ids "$VPC_ID" \
  --region "$REGION" \
  --profile "$PROFILE"
```

After successful deletion, AWS should return an error indicating that the VPC ID does not exist.

You can also verify the instance:

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'Reservations[].Instances[].{InstanceId:InstanceId,State:State.Name}' \
  --output table
```

The instance should show `terminated`.

**Cleanup note:** The commands above remove the EC2 instance and networking resources created by this guide. They do not delete the IAM user, IAM group, IAM policies, access key, local private key file, or local User Data script. Review those separately and remove credentials or IAM resources only if they are no longer required. Do not delete a key pair or IAM user shared with another project.

---

# Final recap

In this lab, we:

1. Prepared the AWS CLI environment.
2. Created an IAM user and group.
3. Attached discovery and deployment permissions.
4. Configured an AWS CLI profile.
5. Discovered Availability Zones, instance types, and an Amazon Linux 2023 AMI.
6. Verified the EC2 key pair.
7. Created a VPC and public subnet.
8. Created and attached an Internet Gateway.
9. Created a route table and internet route.
10. Associated the route table with the subnet.
11. Created a security group with SSH and HTTP rules.
12. Launched an EC2 instance.
13. Installed NGINX using User Data.
14. Verified the website using HTTP status 200.
15. Cleaned up the resources created for the lab.

**The key learning:** We started without existing networking and built the required AWS network infrastructure manually, step by step, before deploying our EC2 NGINX web server.
