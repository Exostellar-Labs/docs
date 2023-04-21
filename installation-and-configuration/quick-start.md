# Quick Start

## Prerequisites:

1. Login to [your AWS account](https://console.aws.amazon.com/console/home?nc2=h\_ct\&src=header-signin).
2. Go to [Identity and Access Management (IAM) > Policies](https://us-east-1.console.aws.amazon.com/iamv2/home?region=us-east-1#/policies) and click **Create policy**.\
   Copy and paste the following JSON code block to the **JSON tab**.

<details>

<summary>xspot iam.json</summary>

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:RunInstances",
                "ec2:DescribeSpotPriceHistory",
                "ec2:DescribeInstances",
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
                "sts:DecodeAuthorizationMessage",
                "ssm:GetParameters",
                "ssm:GetParametersByPath"
            ],
            "Resource": "*"
        }
    ]
}
```

</details>
3. Go to [Identity and Access Management (IAM) > Roles] (https://us-east-1.console.aws.amazon.com/iamv2/home#/roles) and click Create role.
