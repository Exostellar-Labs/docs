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

# Prerequisites

1. All commands will be run as the root user, unless otherwise noted.
2. Onboarding activities require an existing HPC Environment (e.g. a cluster) or at a minimum one VM.
   *   If starting from scratch, create a VM with CentOS-7. We recommend EC2 instances with [Enhanced Networking](https://aws.amazon.com/premiumsupport/knowledge-center/enable-configure-enhanced-networking/) and EBS SSD volumes.

       > **IMPORTANT:**
       >
       > Amazon Linux 2023 instances are not supported. See [Amazon Documentation](https://docs.aws.amazon.com/linux/al2023/ug/compare-with-al2.html#epel) for more information.
   * If proceeding minimally with one VM only, install the targeted workflow's application requirements and validate proper execution of the workflow for end-users as required prior to typical go-live or release-to-production.
   * If proceeding with an existing HPC Cluster, all workflows, authentication systems, and licensing requirements should be fully validated and operational.
3. Create or choose a VPC and a subnet that you want to put your X-Infrastructure Optimizer controller and workers in.
   * X-Infrastructure Optimizer creates an internal overlay network with CIDR 192.168.137.0/24 by default.
   * We need to ensure the overlay network does not conflict the CIDR in your VPC.
   * If there is any overlapping, we will modify[ X-Infrastructure Optimizer configurations](pre-configuration.md) as described later.
4. Add a NAT gateway to your subnet.
5. Allow access to the following websites and URLs:
   * AWS EC2 API Endpoint:
     * [https://ec2.amazonaws.com/](https://ec2.amazonaws.com/)
   * Exostellar RPM repos:
     * [https://rpm.exostellar.dev/](https://rpm.exostellar.dev/)
     * [https://rpm.exostellar.io/](https://rpm.exostellar.io/)
   * Entitlement API server:
     * [https://13.66.175.108](https://13.66.175.108/)
   * Microsoft eventhub API endpoint:
     * TBD
   * (optional) X-Infrastructure Optimizer advisor service endpoint:
     * [https://3.135.60.35](https://3.135.60.35/)
6. Create a security group for X-Infrastructure Optimizer controller and workers.
   * In addition to the inbound rules that are required by your application (such as the SSH port, or licensing and license-vendor-daemon ports), it should also allow all internal traffic within your subnet.
   * For example, if your subnet uses 172.31.0.0/20, your security group should allow "All traffic" from 172.31.0.0/20.
   * You can put the controller and workers in different security groups as long as both of them allow all internal traffic.
7. Prepare a base AMI to use. It can be one of the following:
   1. Subscribe to [Exostellar's Computer Optimizer Worker](https://aws.amazon.com/marketplace/pp/prodview-joaaqojtokw3s?sr=0-2\&ref\_=beagle\&applicationId=AWSMPContessa)
   2. Subscribe to [CentOS 7 (x86\_64) base image](https://aws.amazon.com/marketplace/pp/prodview-foff247vr2zfw?sr=0-1\&ref\_=beagle\&applicationId=AWS-EC2-Console)
   3. Make your own CentOS7 (x86\_64) AMI&#x20;
8.  Create an IAM role for the X-Infrastructure Optimizer controller

    * X-Infrastructure Optimizer makes requests to Amazon Web Services.
    * To support that, you need to create and attach an IAM role to the instance running the X-Infrastructure Optimizer controller.
    * The included policy iam-policy.json needs to be modified and attached to the role. Statement blocks with an Sid starting with "Optional" can be removed if not required.
    * The following variables need to be replaced in the policy document before attaching it to a role:
      * \<REGION>
      * \<WORKER\_SUBNET\_ID>
      * \<CONTROLLER\_IAM\_ROLE\_INSTANCE\_PROFILE\_ARN>
      * \<WORKER\_IAM\_ROLE\_INSTANCE\_PROFILE\_ARN> - this resource definition can be removed if no IAM role is used for the worker
      * \<CONTROLLER\_IAM\_ROLE\_ARN>
      * (optional) \<WORKER\_IAM\_ROLE\_ARN>
      * (optional) \<WATCHDOG\_TOPIC\_ARN>
      * (optional) \<SSM\_PARAMETER\_STORE\_ARN+ROOT\_PATH>
    * ```
      // ComputeOptimizerControllerRole-rc.1.3.0.6
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
    * ```
      // ComputeOptimizerControllerRole-rc.1.3.0
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

    > **IMPORTANT:**
    >
    > This role will need to be attached to **every X-Infrastructure Optimizer controller**.

    7.1. Changelog: 2023-08-11 : added ec2:DescribeInstanceTypes into the IAM policy.\

9. Make note of linux and AWS services that are likely to be required, as well as any pertinent version information. Some filesystem or directory services may require additional configuration and validation to ensure compatability. The services listed below fall into two general categories: authentication or directory services (ldap, sssd) and remote-filesystems (Lustre, NetApp ONTAP, autofs). If a required service is not listed below but is required, please let support know about your needs and use-case.
   * Amazon FSx for Lustre
   * Amazon FSx for NetApp ONTAP
   * autofs
   * sssd
   * ldap
10. X-Infrastructure Optimizer currently supports 64-bit applications. 32-bit applications are not yet supported.
11. There are two limiting factors in terms of how many workers or containers can be started on one X-Infrastructure Optimizer controller:

    > **EQUATION:** Num of containers + num of workers \* 2 <= 245.

    * This is due to how X-Infrastructure Optimizer manages its internal metadata connections. Based on this equation, the max number of containers that you can start depends on how you configure X-Infrastructure Optimizer. Typically we will start one worker for each container, so that effectively limits the number of containers (or workers) to be 81. But if you configure X-Infrastructure Optimizer to pack multiple containers on one worker, then the number of containers can be higher.
    * Root file system usage for all the containers cannot exceed the total disk space available to docker for storage. For example, if we are expecting that each container will generate 30GB data on the root file system, and the docker thin pool device has a 300GB SSD, then we may run into issues if 10 or more containers are concurrently using 30G of storage to run their jobs, with some failing due to storage errors, "No space left on device."
    * The real limit of how many containers we can start on a single controller will be the smaller number of the two mentioned above. For the controller, CPU usage is typically not a concern, since the disk space for docker images is the bottleneck. But we recommend 4 CPUs (2 cores) and 8GB memory and a local SSD for the controller. On AWS this would be a c5d.xlarge or m5d.xlarge VM, and can go with other instance types depending on the storage requirements.
12. VMs or EC2 instances required for this guide:
    *   HPC Cluster Environment

        <table><thead><tr><th width="161">VM Reference Name</th><th width="120">Required?</th><th width="162" align="center">Recommended Instance Type</th><th align="right">Comment</th></tr></thead><tbody><tr><td>X-Infrastructure Optimizer controller</td><td>Yes</td><td align="center">m5d.xlarge or an instance with "d" before the period</td><td align="right">Most steps take place on this node. Ideally it is a fully functional and validated compute node in the cluster.</td></tr><tr><td>X-Infrastructure Optimizer worker AMI builder</td><td>Yes</td><td align="center">m5.xlarge or c5.xlarge</td><td align="right">This VM will only exist for 15 - 30 minutes and can be terminated thereafter.</td></tr></tbody></table>

        > **IMPORTANT:**
        >
        > Amazon Linux 2023 instances are not supported. See [Amazon Documentation](https://docs.aws.amazon.com/linux/al2023/ug/compare-with-al2.html#epel) for more information.

        > **TAKE-AWAY:**
        >
        > Minimum requirements are workflow and environment dependent. It's recommended to start with m5d.xlarge instance for X-Infrastructure Optimizer controllers, which has a dedicated 150GB SSD for docker storage. m5d.2xlarge has a 300GB SSD, so could run roughly double the number of jobs. See [AWS Instance Descriptsion](https://aws.amazon.com/ec2/instance-types/) for more information.
