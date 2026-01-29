# AWS CloudFormation - VPC with Public EC2 Instance

## Overview

This CloudFormation template creates a complete VPC infrastructure with a publicly accessible EC2 instance. It's designed as a quick-start template for development and testing environments.

## Architecture

The template creates the following AWS resources:

- **VPC** with CIDR block 10.0.0.0/16
- **Internet Gateway** attached to the VPC
- **Public Subnet** (10.0.1.0/24) with auto-assign public IP enabled
- **Route Table** with internet route (0.0.0.0/0 → Internet Gateway)
- **Security Group** allowing SSH (port 22) and HTTP (port 80) from anywhere
- **EC2 Instance** in the public subnet with a public IP address

## Network Diagram

```
                    Internet
                       │
                       │
                       ▼
              ┌─────────────────┐
              │ Internet Gateway│
              └─────────────────┘
                       │
                       │ [VPC Gateway Attachment]
                       │
              ┌────────▼───────────────────────┐
              │   VPC (10.0.0.0/16)            │
              │   - EnableDnsSupport: true     │
              │   - EnableDnsHostnames: true   │
              │                                │
              │  ┌──────────────────────────┐  │
              │  │   Route Table            │  │
              │  │                          │  │
              │  │  Routes:                 │  │
              │  │  • 10.0.0.0/16 → local   │◄─┼─── [Implicit Local Route]
              │  │  • 0.0.0.0/0 → IGW       │  │
              │  └────────┬─────────────────┘  │
              │           │                    │
              │           │ [Route Table       │
              │           │  Association]      │
              │           │                    │
              │  ┌────────▼─────────────────┐  │
              │  │  Public Subnet           │  │
              │  │  (10.0.1.0/24)           │◄─┼─── [Subnet Association with VPC]
              │  │  - AZ: us-east-1a        │  │
              │  │  - MapPublicIpOnLaunch   │  │
              │  │                          │  │
              │  │  ┌────────────────────┐  │  │
              │  │  │  EC2 Instance      │  │  │
              │  │  │                    │  │  │
              │  │  │  Security Group:   │  │  │
              │  │  │  • SSH (22)        │  │  │
              │  │  │  • HTTP (80)       │  │  │
              │  │  │                    │  │  │
              │  │  │  Public IP: x.x.x.x│  │  │
              │  │  └────────────────────┘  │  │
              │  └──────────────────────────┘  │
              └────────────────────────────────┘

Component Relationships:
├─ VPC Gateway Attachment: Links Internet Gateway to VPC
├─ Route: Individual routing rule (0.0.0.0/0 → IGW for internet traffic)
├─ Route Table Association: Connects Route Table to Public Subnet
└─ Subnet Association: Subnet exists within and belongs to the VPC
```

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `InstanceType` | String | `t2.micro` | EC2 instance type (t2.micro, t3.micro, or t3.small) |
| `KeyName` | String | *Required* | Name of an existing EC2 KeyPair for SSH access |
| `ImageName` | String | `ubuntu-22.04` | Operating system (ubuntu-22.04, amazon-linux-2, or amazon-linux-2023) |

## Prerequisites

Before deploying this template, ensure you have:

1. An AWS account with appropriate permissions
2. An existing EC2 Key Pair in the target region (for SSH access)
3. AWS CLI installed (for CLI deployment) or access to AWS Console

## Deployment

### Using AWS Console

1. Navigate to CloudFormation in the AWS Console
2. Click **Create Stack** → **With new resources**
3. Upload this template file
4. Fill in the required parameters:
   - **Stack name**: e.g., `my-vpc-stack`
   - **KeyName**: Select your existing SSH key pair
   - **InstanceType**: Choose instance type (default: t2.micro)
   - **ImageName**: Choose OS (default: ubuntu-22.04)
5. Click **Next** → **Next** → **Create Stack**

### Using AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name my-vpc-stack \
  --template-body file://template.yaml \
  --parameters \
    ParameterKey=KeyName,ParameterValue=your-key-name \
    ParameterKey=InstanceType,ParameterValue=t2.micro \
    ParameterKey=ImageName,ParameterValue=ubuntu-22.04
```

### Using AWS CLI with Parameter File

Create a `parameters.json` file:

```json
[
  {
    "ParameterKey": "KeyName",
    "ParameterValue": "your-key-name"
  },
  {
    "ParameterKey": "InstanceType",
    "ParameterValue": "t2.micro"
  },
  {
    "ParameterKey": "ImageName",
    "ParameterValue": "ubuntu-22.04"
  }
]
```

Deploy:

```bash
aws cloudformation create-stack \
  --stack-name my-vpc-stack \
  --template-body file://template.yaml \
  --parameters file://parameters.json
```

## Outputs

After successful deployment, the stack outputs:

| Output | Description |
|--------|-------------|
| `InstancePublicIP` | The public IP address of the EC2 instance |

### Retrieve Outputs

```bash
aws cloudformation describe-stacks \
  --stack-name my-vpc-stack \
  --query 'Stacks[0].Outputs'
```

## Accessing the EC2 Instance

Once deployed, SSH into your instance:

```bash
# For Ubuntu
ssh -i /path/to/your-key.pem ubuntu@<InstancePublicIP>

# For Amazon Linux
ssh -i /path/to/your-key.pem ec2-user@<InstancePublicIP>
```

Replace `<InstancePublicIP>` with the IP from the stack outputs.

## Security Considerations

⚠️ **Important Security Notes:**

- This template opens SSH (port 22) and HTTP (port 80) to the entire internet (0.0.0.0/0)
- **For production use**, restrict the Security Group to specific IP ranges:
  - Replace `0.0.0.0/0` with your IP address or corporate IP range
  - Consider using AWS Systems Manager Session Manager instead of SSH
- The instance has a public IP and is internet-facing
- Review and adjust security settings based on your requirements

## Cost Estimation

This stack will incur charges for:

- EC2 instance (t2.micro eligible for free tier)
- Data transfer out from EC2
- Elastic IP if associated (minimal charge if attached)

**Estimated monthly cost**: ~$8-10 USD (t2.micro, outside free tier)

## Cleanup

To delete all resources created by this stack:

### Using AWS Console

1. Go to CloudFormation console
2. Select your stack
3. Click **Delete**

### Using AWS CLI

```bash
aws cloudformation delete-stack --stack-name my-vpc-stack
```

Verify deletion:

```bash
aws cloudformation describe-stacks --stack-name my-vpc-stack
```

## Customization Ideas

- Add additional subnets for multi-AZ deployment
- Create a private subnet with NAT Gateway
- Add Application Load Balancer for web applications
- Include RDS database in private subnet
- Add CloudWatch alarms for monitoring
- Configure auto-scaling groups

## Troubleshooting

### Stack Creation Failed

1. Check CloudFormation Events tab for error details
2. Common issues:
   - Invalid KeyName (key pair doesn't exist)
   - Insufficient permissions
   - Region-specific AMI not found

### Cannot SSH to Instance

1. Verify Security Group allows your IP
2. Check instance state is "running"
3. Verify you're using correct username (ubuntu/ec2-user)
4. Ensure key pair permissions: `chmod 400 your-key.pem`

### Instance Not Getting Public IP

- Ensure `MapPublicIpOnLaunch: true` is set on subnet
- Check route table has internet gateway route
- Verify internet gateway is attached to VPC

## Resources Created

| Resource Type | Logical ID | Description |
|--------------|------------|-------------|
| VPC | `MyVPC` | Virtual Private Cloud (10.0.0.0/16) |
| Internet Gateway | `InternetGateway` | Allows internet access |
| VPC Gateway Attachment | `AttachGateway` | Attaches IGW to VPC |
| Subnet | `PublicSubnet` | Public subnet (10.0.1.0/24) |
| Route Table | `PublicRouteTable` | Routes for public subnet |
| Route | `PublicRoute` | Internet route (0.0.0.0/0 → IGW) |
| Subnet Association | `SubnetRouteTableAssociation` | Associates subnet with route table |
| Security Group | `EC2SecurityGroup` | Firewall rules for EC2 |
| EC2 Instance | `EC2Instance` | Virtual machine |

## Additional Resources

- [AWS CloudFormation Documentation](https://docs.aws.amazon.com/cloudformation/)
- [VPC User Guide](https://docs.aws.amazon.com/vpc/)
- [EC2 User Guide](https://docs.aws.amazon.com/ec2/)

## License

This template is provided as-is for educational and development purposes.

## Contributing

Feel free to submit issues or pull requests to improve this template.

---

**Last Updated**: January 2026
