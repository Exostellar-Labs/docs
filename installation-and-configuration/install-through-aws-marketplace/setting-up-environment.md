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

We understand that cloud control and security are essential to you. In order to install X-Infrastructure Optimizer and start saving right away, we need your help to set up the right environment for X-Infrastructure Optimizer. Once X-Infrastructure Optimizer is installed, it will manage and optimize your computing instances for you.

To install X-Infrastructure Optimizer and start cost savings, we need to set up the correct cloud infrastructures.

### Step 1: Create a VPC and a Subnet

This is where your X-Infrastructure Optimizer Controller and Worker reside.

After logging into your AWS Console:\
Go to **Virtual Private Cloud (VPC)** > **Your VPCs** > **Create VPC**, please specify a CIDR other than 192.168.137.0/24, since X-Infrastructure Optimizer creates an overlay.\
Go to **VPC** > **Subnets** > **Create subnet**, within the VPC you just created.\
Go to **VPC** > **Internet gateways** > **Create internet gateway**.\
Go to **VPC** > **Network Address Translation (NAT) gateways** > **Create NAT gateway**.

Please allow access to the following addresses and ports:

| Endpoint                                     | Address                                              | Port |
| -------------------------------------------- | ---------------------------------------------------- | ---- |
| AWS EC2 API                                  | [http://ec2.amazonaws.com](http://ec2.amazonaws.com) | 443  |
| X-Infrastructure Optimizer Advisor Service (optional) | 3.135.60.35                                          | 443  |

### Step 2: Create a Security Group

Go to **EC2** > **Security Groups** > **Create security group**.\
Add the following **Inbound rules**:

| Type        | Protocol | Port range | Source        |
| ----------- | -------- | ---------- | ------------- |
| All traffic | All      | All        | Your VPC CIDR |
| SSH         | TCP      | 22         | 0.0.0.0/0     |

{% hint style="info" %}
X-Infrastructure Optimizer Controller and Workers can be in different security groups as long as the internal traffic are allowed between them.
{% endhint %}

### Step 3: Create an IAM role with least-privilege permissions <a href="#step-3-create-an-iam-role-with-least-privilege-permissions" id="step-3-create-an-iam-role-with-least-privilege-permissions"></a>

Go to **Identity and Access Management (IAM)** > **Policies** > **Create policy**.

<details>

<summary>Compute-Optimizer-ControllerRole-rc.1.3.0.0</summary>

```
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

</details>

<details>

<summary>Compute-Optimizer-ControllerRole-rc.1.3.0.6</summary>

Statement blocks with an Sid starting with "Optional" can be removed if not required.

The following variables need to be replaced in the policy document before attaching it to a role:

* \<REGION>
* \<WORKER\_SUBNET\_ID>
* \<CONTROLLER\_IAM\_ROLE\_INSTANCE\_PROFILE\_ARN>
* \<WORKER\_IAM\_ROLE\_INSTANCE\_PROFILE\_ARN> - this resource definition can be removed if no IAM role is used for the worker
* \<CONTROLLER\_IAM\_ROLE\_ARN>
* (optional) \<WORKER\_IAM\_ROLE\_ARN>
* (optional) \<WATCHDOG\_TOPIC\_ARN>
* (optional) \<SSM\_PARAMETER\_STORE\_ARN+ROOT\_PATH>

```
{
  "Version": "2012-10-17",
  "Statement": [
      {
          "Sid": "xspotFleetCreation",
          "Effect": "Allow",
          "Action": "ec2:CreateFleet",
          "Resource": "arn:aws:ec2:<REGION>:*:launch-template/*"
      },
      {
          "Sid": "xspotFleetCreationTagging",
          "Effect": "Allow",
          "Action": "ec2:CreateFleet",
          "Resource": [
              "arn:aws:ec2:<REGION>:*:instance/*",
              "arn:aws:ec2:<REGION>:*:fleet/*"
          ],
          "Condition": {
              "StringLike": {
                  "aws:RequestTag/exo-fleet": "*"
              }
          }
      },
      {
          "Sid": "xspotInstanceCreation",
          "Effect": "Allow",
          "Action": "ec2:RunInstances",
          "Resource": [
              "arn:aws:ec2:<REGION>:*:subnet/<WORKER_SUBNET_ID>",
              "arn:aws:ec2:<REGION>:*:image/ami-*",
              "arn:aws:ec2:<REGION>:*:key-pair/*",
              "arn:aws:ec2:<REGION>:*:security-group/*",
              "arn:aws:ec2:<REGION>:*:launch-template/*"
          ]
      },
      {
          "Sid": "xspotInstanceCreationTagging",
          "Effect": "Allow",
          "Action": "ec2:RunInstances",
          "Resource": [
              "arn:aws:ec2:<REGION>:*:instance/*",
              "arn:aws:ec2:<REGION>:*:volume/*",
              "arn:aws:ec2:<REGION>:*:network-interface/*"
          ],
          "Condition": {
              "StringLike": {
                  "aws:RequestTag/exo-fleet": "*"
              }
          }
      },
      {
          "Sid": "xspotResourceCreation",
          "Effect": "Allow",
          "Action": [
              "ec2:CreateLaunchTemplate",
              "ec2:CreateVolume"
          ],
          "Resource": [
              "arn:aws:ec2:<REGION>:*:launch-template/*",
              "arn:aws:ec2:<REGION>:*:volume/*"
          ],
          "Condition": {
              "StringLike": {
                  "aws:RequestTag/exo-fleet": "*"
              }
          }
      },
      {
          "Sid": "xspotSnapshotCreation",
          "Effect": "Allow",
          "Action": "ec2:CreateSnapshot",
          "Resource": "arn:aws:ec2:<REGION>:*:volume/*"
      },
      {
          "Sid": "xspotSnapshotCreationTagging",
          "Effect": "Allow",
          "Action": "ec2:CreateSnapshot",
          "Resource": "arn:aws:ec2:<REGION>::snapshot/*",
          "Condition": {
              "StringLike": {
                  "aws:RequestTag/exo-fleet": "*"
              }
          }
      },
      {
          "Sid": "xspotTagsForResourceCreation",
          "Action": "ec2:CreateTags",
          "Effect": "Allow",
          "Resource": "arn:aws:ec2:<REGION>:*:*/*",
          "Condition": {
              "StringEquals": {
                  "ec2:CreateAction": [
                      "CreateLaunchTemplate",
                      "RunInstances",
                      "CreateVolume",
                      "CreateFleet",
                      "CreateSnapshot"
                  ]
              }
          }
      },
      {
          "Sid": "xspotResourceDescription",
          "Effect": "Allow",
          "Action": [
              "ec2:DescribeSpotPriceHistory",
              "ec2:DescribeInstances",
              "ec2:DescribeInstanceTypes",
              "ec2:DescribeTags",
              "ec2:DescribeSubnets",
              "ec2:DescribeSecurityGroups",
              "ec2:DescribeImages",
              "ec2:DescribeKeyPairs",
              "ec2:DescribeInstanceTypeOfferings",
              "ec2:DescribeVolumes",
              "ec2:DescribeSnapshots",
              "iam:ListInstanceProfiles"
          ],
          "Resource": "*"
      },
      {
          "Sid": "xspotResourceModification",
          "Effect": "Allow",
          "Action": [
              "ec2:AssignPrivateIpAddresses",
              "ec2:UnassignPrivateIpAddresses",
              "ec2:ModifyInstanceAttribute",
              "ec2:AttachVolume",
              "ec2:DetachVolume",
              "ec2:StopInstances",
              "ec2:RebootInstances"
          ],
          "Resource": [
              "arn:aws:ec2:<REGION>:*:network-interface/*",
              "arn:aws:ec2:<REGION>:*:volume/*",
              "arn:aws:ec2:<REGION>:*:instance/*"
          ],
          "Condition": {
              "StringLike": {
                  "aws:ResourceTag/exo-fleet": "*"
              }
          }
      },
      {
          "Sid": "xspotResourceDeletion",
          "Effect": "Allow",
          "Action": [
              "ec2:DeleteLaunchTemplate",
              "ec2:TerminateInstances",
              "ec2:DeleteVolume"
          ],
          "Resource": [
              "arn:aws:ec2:<REGION>:*:instance/*",
              "arn:aws:ec2:<REGION>:*:volume/*",
              "arn:aws:ec2:<REGION>:*:launch-template/*"
          ],
          "Condition": {
              "StringLike": {
                  "aws:ResourceTag/exo-fleet": "*"
              }
          }
      },
      {
          "Sid": "xspotInstanceProfile",
          "Effect": "Allow",
          "Action": "iam:GetInstanceProfile",
          "Resource": [
              "<CONTROLLER_IAM_ROLE_INSTANCE_PROFILE_ARN>",
              "<WORKER_IAM_ROLE_INSTANCE_PROFILE_ARN>"
          ]
      },
      {
          "Sid": "xspotPolicySimulation",
          "Effect": "Allow",
          "Action": "iam:SimulatePrincipalPolicy",
          "Resource": "<CONTROLLER_IAM_ROLE_ARN>"
      },
      {
          "Sid": "OptionalXspotWorkerIamRole",
          "Effect": "Allow",
          "Action": "iam:PassRole",
          "Resource": "<WORKER_IAM_ROLE_ARN>"
      },
      {
          "Sid": "OptionalXspotWatchDogSNS",
          "Effect": "Allow",
          "Action": "sns:Publish",
          "Resource": "<WATCHDOG_TOPIC_ARN>"
      },
      {
          "Sid": "OptionalXspotParameterStore",
          "Effect": "Allow",
          "Action": [
              "ssm:GetParametersByPath",
              "ssm:GetParameters"
          ],
          "Resource": "<SSM_PARAMETER_STORE_ARN+ROOT_PATH>/*"
      },
      {
          "Sid": "OptionalDebuggingHelp",
          "Action": "sts:DecodeAuthorizationMessage",
          "Effect": "Allow",
          "Resource": "*"
      }
  ]
}
```

</details>

Go to **Identity and Access Management (IAM)** > **Roles** > **Create role** and add the permissions policies for EC2.

{% hint style="info" %}
Please attach this role to EVERY X-Infrastructure Optimizer Controller.
{% endhint %}
