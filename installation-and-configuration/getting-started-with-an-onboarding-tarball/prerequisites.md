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
3. Create or choose a VPC and a subnet that you want to put your X-Spot controller and workers in.
   * X-Spot creates an internal overlay network with CIDR 192.168.137.0/24 by default.
   * We need to ensure the overlay network does not conflict the CIDR in your VPC.
   * If there is any overlapping, we will modify[ X-Spot configurations](pre-configuration.md) as described later.
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
   * (optional) X-Spot advisor service endpoint:
     * [https://3.135.60.35](https://3.135.60.35/)
6. Create a security group for X-Spot controller and workers.
   * In addition to the inbound rules that are required by your application (such as the SSH port, or licensing and license-vendor-daemon ports), it should also allow all internal traffic within your subnet.
   * For example, if your subnet uses 172.31.0.0/20, your security group should allow "All traffic" from 172.31.0.0/20.
   * You can put the controller and workers in different security groups as long as both of them allow all internal traffic.
7.  Create an IAM role for the X-Spot controller

    * X-Spot makes requests to Amazon Web Services.
    * To support that, you need to create and attach an IAM role to the instance running the X-Spot controller.
    * The included policy [iam-policy.json](https://github.com/Exostellar-Labs/docs/blob/main/iam-policy.json) needs to be attached to the role
    * ```
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

    > **IMPORTANT:**
    >
    > This role will need to be attached to **every X-Spot controller**.

    7.1. Changelog: 2023-08-11 : added ec2:DescribeInstanceTypes into the IAM policy.
8. Make note of linux and AWS services that are likely to be required, as well as any pertinent version information. Some filesystem or directory services may require additional configuration and validation to ensure compatability. The services listed below fall into two general categories: authentication or directory services (ldap, sssd) and remote-filesystems (Lustre, NetApp ONTAP, autofs). If a required service is not listed below but is required, please let support know about your needs and use-case.
   * Amazon FSx for Lustre
   * Amazon FSx for NetApp ONTAP
   * autofs
   * sssd
   * ldap
9. X-Spot currently supports 64-bit applications. 32-bit applications are not yet supported.
10. There are two limiting factors in terms of how many workers or containers can be started on one X-Spot controller:

    > **EQUATION:** Num of containers + num of workers \* 2 <= 245.

    * This is due to how X-Spot manages its internal metadata connections. Based on this equation, the max number of containers that you can start depends on how you configure X-Spot. Typically we will start one worker for each container, so that effectively limits the number of containers (or workers) to be 81. But if you configure X-Spot to pack multiple containers on one worker, then the number of containers can be higher.
    * Root file system usage for all the containers cannot exceed the total disk space available to docker for storage. For example, if we are expecting that each container will generate 30GB data on the root file system, and the docker thin pool device has a 300GB SSD, then we may run into issues if 10 or more containers are concurrently using 30G of storage to run their jobs, with some failing due to storage errors, "No space left on device."
    * The real limit of how many containers we can start on a single controller will be the smaller number of the two mentioned above. For the controller, CPU usage is typically not a concern, since the disk space for docker images is the bottleneck. But we recommend 4 CPUs (2 cores) and 8GB memory and a local SSD for the controller. On AWS this would be a c5d.xlarge or m5d.xlarge VM, and can go with other instance types depending on the storage requirements.
11. VMs or EC2 instances required for this guide:
    *   HPC Cluster Environment



        <table><thead><tr><th width="161">VM Reference Name</th><th width="120">Required?</th><th width="162" align="center">Recommended Instance Type</th><th align="right">Comment</th></tr></thead><tbody><tr><td>X-Spot controller</td><td>Yes</td><td align="center">m5d.xlarge or an instance with "d" before the period</td><td align="right">Most steps take place on this node. Ideally it is a fully functional and validated compute node in the cluster.</td></tr><tr><td>X-Spot worker AMI builder</td><td>Yes</td><td align="center">m5.xlarge or c5.xlarge</td><td align="right">This VM will only exist for 15 - 30 minutes and can be terminated thereafter.</td></tr></tbody></table>

        > **IMPORTANT:**
        >
        > Amazon Linux 2023 instances are not supported. See [Amazon Documentation](https://docs.aws.amazon.com/linux/al2023/ug/compare-with-al2.html#epel) for more information.

        > **TAKE-AWAY:**
        >
        > Minimum requirements are workflow and environment dependent. It's recommended to start with m5d.xlarge instance for X-Spot controllers, which has a dedicated 150GB SSD for docker storage. m5d.2xlarge has a 300GB SSD, so could run roughly double the number of jobs. See [AWS Instance Descriptsion](https://aws.amazon.com/ec2/instance-types/) for more information.