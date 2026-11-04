# AWS Networking – Manual Deployment Commands

**Repository:** `aws-ec2-nginx-deployment`  
**Local path:** `~/projects/aws-ec2-nginx-deployment`  
**AWS Region:** `us-west-2` (US West – Oregon)  
**AWS CLI Profile:** `ec2-nginx-lab`

This document covers the networking discovery and security group configuration required before launching the EC2 instance.

We are reusing an existing default VPC, subnet, route table and Internet Gateway. We are not creating a new VPC or Internet Gateway.

## 1. Understand the networking components

| Component | Purpose |
|---|---|
| VPC | Our private network inside AWS |
| Subnet | A smaller IP address range inside the VPC |
| Availability Zone | The physical location within an AWS Region where the subnet exists |
| Route table | Controls where network traffic is directed |
| Internet Gateway | Connects the VPC to the internet |
| Security Group | Controls allowed inbound and outbound traffic |
| EC2 | The virtual server that will host NGINX |

Our network flow:

`VPC → Subnet → EC2 → NGINX`

The Internet Gateway and route table provide the network path. The security group controls permitted traffic.

## 2. Open the manual project and configure variables

```bash
cd ~/projects/aws-ec2-nginx-deployment
```

Set the AWS profile and region:

```bash
export PROFILE=ec2-nginx-lab
export REGION=us-west-2
```

These variables are reused in the AWS CLI commands below.

## 3. Discover the existing VPC

Retrieve the default VPC ID and store it in `VPC_ID`.

```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --profile "$PROFILE" \
  --region "$REGION" \
  --query 'Vpcs[0].VpcId' \
  --output text)
```

Verify:

```bash
echo "Default VPC discovered."
```

Inspect the VPC:

```bash
aws ec2 describe-vpcs \
  --vpc-ids "$VPC_ID" \
  --profile "$PROFILE" \
  --region "$REGION" \
  --query 'Vpcs[].{State:State,CIDR:CidrBlock,IsDefault:IsDefault}' \
  --output table
```

**Variable created:** `VPC_ID`

**What it stores:** The ID of the existing default VPC.

**Where it is reused:** Subnet discovery, route table discovery, Internet Gateway verification and security group creation.

## 4. Discover the subnets

List the subnets belonging to the discovered VPC.

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --profile "$PROFILE" \
  --region "$REGION" \
  --query 'Subnets[].{SubnetId:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,MapPublicIP:MapPublicIpOnLaunch,AvailableIPs:AvailableIpAddressCount}' \
  --output table
```

Review the subnet ID, Availability Zone, CIDR, automatic public IP assignment and available IP addresses.

For this project, we use the existing subnet in `us-west-2a`.

Store its ID:

```bash
export SUBNET_ID=subnet-02ecdd68edfbf7326
```

**Variable created:** `SUBNET_ID`

**What it stores:** The ID of the selected existing subnet.

**Where it is reused:** When launching the manual EC2 instance.

## 5. Inspect the route table

First, check whether the selected subnet has an explicitly associated route table.

```bash
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=$SUBNET_ID" \
  --profile "$PROFILE" \
  --region "$REGION" \
  --query 'RouteTables[].{RouteTableId:RouteTableId,Routes:Routes[?DestinationCidrBlock==`0.0.0.0/0`].{Destination:DestinationCidrBlock,Target:GatewayId,State:State}}' \
  --output table
```

Look for:

- Destination: `0.0.0.0/0`
- Target: `igw-xxxxxxxx`
- State: `active`

### If no route table is returned

Check the VPC's main route table.

```bash
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=association.main,Values=true" \
  --profile "$PROFILE" \
  --region "$REGION" \
  --query 'RouteTables[].{RouteTableId:RouteTableId,Routes:Routes[?DestinationCidrBlock==`0.0.0.0/0`].{Destination:DestinationCidrBlock,Target:GatewayId,State:State}}' \
  --output table
```

A subnet without an explicit route table association uses the VPC's main route table.

**Important:** Do not assume the subnet has internet access unless you verify an active default route pointing to an Internet Gateway.

## 6. Verify the Internet Gateway

Check whether an Internet Gateway is attached to the selected VPC.

```bash
aws ec2 describe-internet-gateways \
  --filters "Name=attachment.vpc-id,Values=$VPC_ID" \
  --profile "$PROFILE" \
  --region "$REGION" \
  --query 'InternetGateways[].{State:Attachments[0].State,GatewayId:InternetGatewayId}' \
  --output table
```

Expected attachment state:

`available`

We are only verifying the existing Internet Gateway. We are not creating a new one.

## 7. Create the manual security group

Create a security group inside the existing VPC.

```bash
SG_ID=$(aws ec2 create-security-group \
  --group-name ec2-nginx-lab-sg \
  --description "Security group for manual EC2 NGINX lab" \
  --vpc-id "$VPC_ID" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'GroupId' \
  --output text)
```

Verify:

```bash
test -n "$SG_ID" && test "$SG_ID" != "None" \
  && echo "Manual security group created."
```

**Variable created:** `SG_ID`

**What it stores:** The ID of the newly created manual security group.

**Where it is reused:** Inbound rule configuration, EC2 launch and cleanup.

## 8. Retrieve the current public IP

Retrieve the public IP of the machine from which we are connecting.

```bash
MY_IP=$(curl -4 -fsS https://checkip.amazonaws.com | tr -d '\n')
```

Check that the IP was retrieved without printing the actual address.

```bash
if [ -n "$MY_IP" ]; then
  echo "Public IP retrieved successfully."
else
  echo "Unable to retrieve public IP. Stop and check connectivity."
fi
```

**Variable created:** `MY_IP`

**What it stores:** The current public IPv4 address.

**Where it is reused:** To restrict SSH access to the current IP using `/32`.

## 9. Configure inbound security group rules

We need two inbound rules:

- Port 22: SSH access from our current public IP.
- Port 80: HTTP access from all IPv4 addresses.

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --ip-permissions "[
    {
      \"IpProtocol\": \"tcp\",
      \"FromPort\": 22,
      \"ToPort\": 22,
      \"IpRanges\": [
        {
          \"CidrIp\": \"${MY_IP}/32\",
          \"Description\": \"SSH from my current IP\"
        }
      ]
    },
    {
      \"IpProtocol\": \"tcp\",
      \"FromPort\": 80,
      \"ToPort\": 80,
      \"IpRanges\": [
        {
          \"CidrIp\": \"0.0.0.0/0\",
          \"Description\": \"Public HTTP access for the demo\"
        }
      ]
    }
  ]" \
  --region "$REGION" \
  --profile "$PROFILE"
```

## 10. Verify the security group rules

```bash
aws ec2 describe-security-groups \
  --group-ids "$SG_ID" \
  --region "$REGION" \
  --profile "$PROFILE" \
  --query 'SecurityGroups[0].IpPermissions[?IpProtocol==`tcp` && (FromPort==`22` || FromPort==`80`)][].[FromPort,ToPort,length(IpRanges[])]' \
  --output table
```

Confirm that both ports are configured.

This verification does not display the actual source IP address.

## 11. Variables reference

| Variable | Where it is assigned | What it stores | Used for |
|---|---|---|---|
| `PROFILE` | `export PROFILE=...` | AWS CLI profile name | AWS CLI authentication |
| `REGION` | `export REGION=...` | AWS Region | AWS resource operations |
| `VPC_ID` | `describe-vpcs` | Existing default VPC ID | Network discovery and security group |
| `SUBNET_ID` | `export SUBNET_ID=...` | Existing subnet ID | EC2 launch |
| `SG_ID` | `create-security-group` | New security group ID | Ingress rules, EC2 launch and cleanup |
| `MY_IP` | `curl` | Current public IP | SSH ingress rule |

**Important:** These are shell variables. They remain available in the same terminal session. If you close the terminal or open a new one, run the variable setup and discovery commands again before using them.

## 12. Network configuration completed

At this stage, we have:

- Discovered the existing default VPC.
- Inspected its CIDR and state.
- Discovered and selected an existing subnet.
- Inspected its route table.
- Verified the Internet Gateway.
- Created a dedicated manual security group.
- Configured SSH and HTTP inbound rules.
- Verified the security group.

The existing VPC, subnet, route table and Internet Gateway remain unchanged.

The next step is to use `VPC_ID`, `SUBNET_ID`, `SG_ID`, `AMI_ID` and `KEY_NAME` when launching the EC2 instance.
