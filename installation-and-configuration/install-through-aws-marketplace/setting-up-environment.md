---
layout:
  title:
    visible: true
  description:
    visible: false
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
---

# Setting Up Environment

We understand that cloud control and security are essential to you. In order to install X-Spot and start saving right away, we need your help to set up the right environment for X-Spot. Once X-Spot is installed, it will manage and optimize your computing instances for you.

To install X-Spot and start cost savings, we need to set up the correct cloud infrastructures.

### Step 1: Create a VPC and a Subnet

This is where your X-Spot Controller and Worker reside.

After logging into your AWS Console:\
Go to **Virtual Private Cloud (VPC)** > **Your VPCs** > **Create VPC**, please specify a CIDR other than 192.168.137.0/24, since X-Spot creates an overlay.\
Go to  **VPC** > **Subnets** > **Create subnet**, within the VPC you just created.\
Go to  **VPC** > **Internet gateways** > **Create internet gateway**.\
Go to  **VPC** > **Network Address Translation (NAT) gateways** > **Create NAT gateway**.

Please allow access to the following addresses and ports:

| Endpoint                          | Address                                              | Port |
| --------------------------------- | ---------------------------------------------------- | ---- |
| AWS EC2 API                       | [http://ec2.amazonaws.com](http://ec2.amazonaws.com) | 443  |
| X-Spot Advisor Service (optional) | 3.135.60.35                                          | 443  |

### Step 2: Create a Security Group

Go to **EC2** > **Security Groups** > **Create security group**.\
Add the following **Inbound rules**:

| Type        | Protocol | Port range | Source        |
| ----------- | -------- | ---------- | ------------- |
| All traffic | All      | All        | Your VPC CIDR |
| SSH         | TCP      | 22         | 0.0.0.0/0     |

{% hint style="info" %}
X-Spot Controller and Workers can be in different security groups as long as the internal traffic are allowed between them.
{% endhint %}

### Step 3: Create an IAM role with least-privilege permissions <a href="#step-3-create-an-iam-role-with-least-privilege-permissions" id="step-3-create-an-iam-role-with-least-privilege-permissions"></a>

&#x20;Go to **Identity and Access Management (IAM)** > **Policies** > **Create policy**.

```
// X-SpotControllerRole-rc.1.3.0
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:RunInstances",
                "ec2:DescribeSpotPriceHistory",
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceTypes",
                "ec2:DescribeTags",
                "ec2:CreateTags",
                "ec2:CreateFleet",
                "ec2:CreateLaunchTemplate",
                "ec2:DeleteLaunchTemplate",
                "ec2:TerminateInstances",
                "ec2:AssignPrivateIpAddresses",
                "ec2:UnassignPrivateIpAddresses",
                "ec2:AttachNetworkInterface",
                "ec2:DetachNetworkInterface",
                "ec2:CreateNetworkInterface",
                "ec2:DeleteNetworkInterface",
                "ec2:ModifyNetworkInterfaceAttribute",
                "ec2:DescribeRegions"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:CreateServiceLinkedRole",
                "iam:ListRoles",
                "iam:ListInstanceProfiles",
                "iam:PassRole"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeSubnets",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeImages",
                "ec2:DescribeKeyPairs",
                "ec2:DescribeInstanceTypeOfferings",
                "iam:GetInstanceProfile",
                "iam:GetRole",
                "iam:SimulatePrincipalPolicy",
                "sns:Publish",
                "ssm:GetParameters"
            ],
            "Resource": "*"
        },
        {
            "Action": "sts:DecodeAuthorizationMessage",
            "Effect": "Allow",
            "Resource": "*"
        },
        {
            "Action": "ssm:GetParametersByPath",
            "Effect": "Allow",
            "Resource": "*"
        }
    ]
}
```

&#x20;Go to **Identity and Access Management (IAM)** > **Roles** > **Create role** and add the permissions policies for EC2.

{% hint style="info" %}
Please attach this role to EVERY X-Spot Controller.
{% endhint %}



