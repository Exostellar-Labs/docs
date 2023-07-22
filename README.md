# X-Spot Installation Guide

## **This guide is mostly superseded by the onboarding guide:** [**https://app.gitbook.com/o/epuhHXSGeL5CskpoqeGn/s/0Z1rTntWzJtAVt22hdpc/**](http://127.0.0.1:5000/o/epuhHXSGeL5CskpoqeGn/s/0Z1rTntWzJtAVt22hdpc/)





* [X-Spot Installation Guide](./#x-spot-installation-guide)
  * [1. Install X-Spot](./#1-install-x-spot)
    * [1.1 Setup X-Spot Controller](./#11-setup-x-spot-controller)
      * [1.1.1 Prerequisites](./#111-prerequisites)
      * [1.1.2 Installation](./#112-installation)
      * [1.1.3 Connect Docker to X-Spot](./#113-connect-docker-to-x-spot)
    * [1.2 Package an AMI for X-Spot worker nodes](./#12-package-an-ami-for-x-spot-worker-nodes)
      * [1.2.1 Customizing worker node installation](./#121-customizing-worker-node-installation)
      * [1.2.2 Manually installing the worker node](./#122-manually-installing-the-worker-node)
  * [2. Integrate with AWS Batch/ECS](./#2-integrate-with-aws-batchecs)
* [Onboarding Guide Extra/Troubleshooting](./#onboarding-troubleshooting)

This guide provides a step-by-step tutorial for installing and configuring X-Spot, including instructions for connecting X-Spot to AWS Batch/ECS.

**The commands below must be executed as `root` unless specified otherwise.**



## <mark style="color:yellow;">1. Install X-Spot</mark>

In this step you will: optionally create a VPC for your xspot deployment, create a controller VM, create worker AMIs and deploy workers using the xspot controller.

If you face errors, please check the [Troubleshoot section](./#1.1.6-troubleshoot). Also, please add problems you faced and how you solved it there.



## 1.1 Setup X-Spot Controller

X-Spot provides an OCI compatible runtime that connects to the Docker engine. When a Docker host has X-Spot installed, it becomes an _X-Spot controller_, and it will be able to run containers remotely in worker nodes. You can have as many X-Spot controllers as you want in your cluster. Each controller manages its own set of worker nodes independently.

### 1.1.1 Prerequisites and Instructions

*   A VPC and a subnet that you want to put your X-Spot controllers and workers.

    1. If you are installing X-spot for development, you don't **have** to create a VPC, simply use the default one. Skip to the next step.&#x20;
    2. Go to the the VPC AWS service, Create new VPC. Select `VPC and more`and choose a name for it in the Name tag.
    3. In `IPv4 CIDR`, use a subnet that is not the `10.0.0.0/24` default, e.g. `12.0.0.0/24,` since X-Spot creates an internal overlay network with CIDR `10.0.0.0/24`
    4. If workers should have access to internet or external network, add at least one AZ in `NAT gateway`. IPv6 is required but not enabled by default. To fix this, go to the main VPC page, Route tables and find the route table that was automatically created, which is based on the VPC name, and will likely end with `-rtb-public`. Select it, go to the Routes tab, Edit routes, Add route. Set destination to ::/0, in Target, select Internet Gateway, then the created internet gateway. Save changes. Next, make sure the VPC is automatically assigning public IPs to the workers: go to VPC, Subnets, find the private subnets that were created (the name will be something like `<VPC name>`**`-subnet-publicX-us-east-Y`** and, one at a time, select it, click Actions, Edit subnet settings, check that `Enable auto-assign public IPv4 address` is checked.


* A VM for the controller.
  1. Create a VM with `CentOS-7` or `Amazon Linux 2` on `x86_64` platforms (these may not be on the AMI list, so you might have to search for them).&#x20;
  2. Choose the VPC we just created. If you need an internet gateway, enable `Auto-assign public IP` ([AWS documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-ip-addressing.html#subnet-public-ip)). This is necessary to access the AWS SDK.
  3. Create a security group for X-Spot controller and workers. In addition to the inbound rules that are required by your application (such as the SSH port), it should also allow all internal traffic within your subnet. Create a new rule with: All traffic, custom source type and the subnet of the VPC, e.g. `12.0.0.0/24` (make sure the subnet mask is correct). You can put the controller and workers in different security groups as long as both of them allow all internal traffic. You can either, after the security is created, add a rule to it that allows all traffic from the this security group.
  4. Note: if you are preparing for the integration with AWS ECS or Batch, you don't need to choose ECS-optimized images because we don't support the `ecs-init` service. The support for other Linux distributions is under development. The controller has very minimum requirements on CPU and memory. But since the default network and storage of containers are handled by the controller, providing good network and storage performance can help. We recommend EC2 instances with [Enhanced Networking](https://aws.amazon.com/premiumsupport/knowledge-center/enable-configure-enhanced-networking/) and EBS SSD volumes.
  5. By default, X-Spot uses [Device Mapper storage driver for Docker](https://docs.docker.com/storage/storagedriver/device-mapper-driver/). You can configure Docker to use a `loopback` device (we will show how to do it next). Optionally, you can attach an unused SSD volume or create a new partition with a size big enough for your containers and images. We will show you how to configure it as well. This is recommended for production use. Do not format it or create a file system on it at this time.
* Set IAM role of the VM.
  1. For a quicker deployment, use the existing role: `XspotMaster`. To create one from scratch, follow the instructions [below](./#1.1.2-optional-creating-iam-role-from-scratch.).

####

### 1.1.2 (optional) Creating IAM role from scratch.

* X-Spot makes requests to Amazon Web Services. To support that, you need to create and attach an IAM role to the instance running the X-Spot controller. The following policy needs to be attached to the role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
      {
          "Effect": "Allow",
          "Action": [
              "ec2:DescribeSpotPriceHistory",
              "ec2:DescribeInstances",
              "ec2:DescribeTags",
              "ec2:GetSpotPlacementScores",
              "ec2:CreateTags",
              "ec2:CreateFleet",
              "ec2:CreateLaunchTemplate",
              "ec2:DeleteLaunchTemplate",
              "ec2:TerminateInstances",
              "ec2:RequestSpotFleet",
              "ec2:ModifySpotFleetRequest",
              "ec2:CancelSpotFleetRequests",
              "ec2:DescribeSpotFleetRequests",
              "ec2:DescribeSpotFleetInstances",
              "ec2:DescribeSpotFleetRequestHistory",
              "ec2:AssignPrivateIpAddresses",
              "ec2:UnassignPrivateIpAddresses",
              "ec2:AttachNetworkInterface",
              "ec2:DetachNetworkInterface",
              "ec2:CreateNetworkInterface",
              "ec2:DeleteNetworkInterface",
              "ec2:ModifyNetworkInterfaceAttribute"
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
              "pricing:GetProducts"
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
              "iam:GetInstanceProfile",
              "iam:GetRole",
              "iam:SimulatePrincipalPolicy",
              "ssm:GetParameters"
          ],
          "Resource": "*"
      }
  ]
}
```

The last group of permissions in the above policy enables the use of the X-Spot check tool to verify the installation of the controller node. Note that the `ssm:GetParameters` permission is only required if the AWS SSM parameter store will be used to store X-Spot configuration parameters.

For easier troubleshooting, it is also recommended to add the following permission to allow decoding authorization failures:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
        "Action": [
            "sts:DecodeAuthorizationMessage"
        ],
        "Effect": "Allow",
        "Resource": "*"
    }
  ]
}
```

7. X-Spot will issue spot fleet requests, this requires the existence of three IAM roles: `AWSServiceRoleForEC2SpotFleet`, `AWSServiceRoleForEC2Spot`, and `aws-ec2-spot-fleet-tagging-role`. These roles will exist if a spot fleet request has been issued from the AWS console before. If this is not the case they can be created manually by following the [AWS instructions for spot fleet roles](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/work-with-spot-fleets.html#spot-fleet-prerequisites) or by issuing a dummy spot fleet request via the AWS console.
8. (Optional) If the X-Spot CLI is already installed, `xspot check` can be used to verify that the prerequisites are met.

&#x20;        &#x20;

### 1.1.3 `xspot`  Installation

1. Update all the packages installed on the controller VM:

```bash
yum update -y

yum install -y epel-release
```

2. Add the X-Spot YUM repository:

```bash
yum-config-manager --add-repo https://rpm.exostellar.dev/exostellar-dev.repo
```

Confirm that the `xspot-stable` repository is enabled and that the latest version of the `xspot`&#x20;

For the stable release, the remove the `-dev` suffix.

RPM is now available:

```bash
$ yum repolist enabled | grep exostellar-dev
exostellar-dev        exostellar Dev - ${basearch}   367

$ yum list -q xspot
Available Packages
xspot.x86_64        0.1.0-dev.el7         @exostellar-dev
```

Install X-Spot controller:

```bash
yum install -y xspot
```

3. Update the configuration file of X-Spot in `/etc/exotanium/config.toml`. Start by making a copy of the template:

```bash
cd /etc/exotanium/
cp config.toml.in config.toml
```

Edit the config.toml file to fit your needs. You will need to set a valid and working AMI image for the _worker,_ under _aws:ami_ config.  See [Section 1.2](./#1.2-package-an-ami-for-x-spot-worker-nodes) for more details.

**Important**_:_ Pay attention to the comments in `config.toml`, especially the compatibility of instance types that you choose for spot fleet requests. Incompatible instance types can cause migrations to fail and containers to crash.

**Important:** you need the correct policy, otherwise you will get a cryptic error about licenses. In the `license:type` config, change it to `x-spot-developer`.

When running containers with X-Spot, keep in mind that the resources (vCPU and memory) that you specify for a container (either in Docker or in ECS/Batch) only affect how many containers will be allocated to a single controller node, but they are not the resources that the actual container will get. For example, if your controller node has 4 vCPUs and 16GB memory, and you specify 1 vCPU and 4GB memory for each container in AWS Batch, each controller node will be able to run up to 4 containers. The actual resources that each container will get, on the other hand, is controlled in `config.toml` with the following parameters:

```bash
on_demand_types = [“m5.2xlarge”, “m5n.2xlarge”, “t3.2xlarge”]

spot_fleet_types = [“m5.2xlarge”, “m5n.2xlarge”, “t3.2xlarge”]

default_container_cpu = 4

default_container_mem = 4096
```



4.  **(Optional)** It is also possible to configure X-Spot using the AWS System Manager Parameter Store. You can follow the steps below if you prefer to manage your configuration in a centralized store. **Note: If a parameter is present both in the local configuration file and the Parameter Store, the Parameter Store's value will override the value in the local configuration file.**



&#x20;  4.1   Add the following permission to the IAM role that you created earlier in [Section 1.1.1 Prerequisites](./#111-prerequisites) for X-Spot controller.

```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
          "Action": [
              "ssm:GetParametersByPath"
          ],
          "Effect": "Allow",
          "Resource": "*"
      }
    ]
  }
```

&#x20;  4.2   Create X-Spot parameters in the AWS System Manager Parameter Store. You can define whatever hierarchy you want for the root path of X-Spot parameters, for example, `/`, `/xspot`, or `/path/to/xspot/parameters/`. For more information about how to work with parameter hierarchies, please check the documentation on [Amazon.com](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-paramstore-hierarchies.html).

&#x20;  4.3   Set `ssm_root` in `/etc/exotanium/config.toml` to the root path for X-Spot in Parameter Store to turn on this feature.

```json
  ssm_root = "/xspot"
```

&#x20;  4.4   When creating X-Spot parameters inside the Parameter Store, make sure to use the same hierarchy as the configuration file. For example, `scheduler_enabled` in `[scheduler]` section with root path `/xspot` corresponds to `/xspot/scheduler/scheduler_enabled`.

`[aws.instance_tags]` and `[aws.volume_tags]` in `[aws]` section should include `aws.` as the prefix in the name of the parameter. Examples: \* `/xspot/aws/aws.instance_tags` \* `/xspot/aws/aws.volume_tags`

&#x20;  4.5    Make sure to choose **Type** `String` with **DataType** `text` for X-Spot parameters. The **Value** for all parameters should follow the same format in the configuration file. For instance, `true` or `false` for `scheduler_enabled`, `24002` for `port`.

String values in the configuration file should include quotes in the value, for example: `"debug"` for `log_level`, `["m5.2xlarge", "m5n.2xlarge", "t3.2xlarge"]` for `spot_fleet_types`.

For `[aws.instance_tags]` and `[aws.volume_tags]`, **Value** should be a full list of key value mappings.

Following are examples of the configurations:

| Config             | SSM Parameter                       | SSM Value                                    |
| ------------------ | ----------------------------------- | -------------------------------------------- |
| scheduler\_enabled | /xspot/scheduler/scheduler\_enabled | true                                         |
| port               | /xspot/scheduler/port               | 24002                                        |
| log\_level         | /xspot/scheduler/log\_level         | "debug"                                      |
| spot\_fleet\_types | /xspot/worker/spot\_fleet\_types    | \["m5.2xlarge", "m5n.2xlarge", "t3.2xlarge"] |
| aws.instance\_tags | /xspot/aws/aws.instance\_tags       | <p>Key1 = "value1"<br>Key2 = "value2"</p>    |
| aws.volume\_tags   | /xspot/aws/aws.volume\_tags         | <p>Key1 = "value1"<br>Key2 = "value2"</p>    |





5. Add the following entry to `/etc/exports`(run as root):

```bash
cat << EOF >> /etc/exports
/run/kata-containers *(rw,no_root_squash,fsid=0,crossmnt)
EOF
```

6. If `/usr/local/bin` and `/usr/local/sbin` are not already in your PATH, add them to it. You can do this by putting the following command into your `~/.bashrc` file.

```bash
echo 'export PATH=$PATH:/usr/local/bin:/usr/local/sbin' >> ~/.bashrc
```

7. Start X-Spot scheduler. Make sure your AWS credentials have already been set before this step.

```bash
systemctl start exo-sched
systemctl enable exo-sched
```

####

### 1.1.4 Connect Docker to X-Spot

1. If you haven't, install Docker Engine by following the documents on [Docker's website](https://docs.docker.com/engine/install/) (or [this link to use yum](https://docs.docker.com/engine/install/centos/#install-using-the-repository)). Do not use the default `docker` package that comes with the `extras` repository in CentOS. The package name should be `docker-ce`.

**IMPORTANT: before following the following instructions, stop all your running docker containers and save your images. We will switch to a different storage driver for docker, and all your existing containers and images will be gone after these steps.**

2. Create `/etc/docker/daemon.json` as following:

```bash
cat << EOF > /etc/docker/daemon.json
{
     "mtu": 8900,
     "runtimes": {
         "runxc": {
                 "path": "/usr/local/bin/kata-runtime"
         }
     }
}
EOF
```

3. Configure `DeviceMapper` storage for Docker. To verify your current storage setup, run `docker info` and you should see the following output:

```console
$ docker info
Client:
 Debug Mode: false

Server:
 Containers: 1
  Running: 0
  Paused: 0
  Stopped: 1
 Images: 2
 Server Version: 19.03.6-ce
 Storage Driver: devicemapper
  Pool Name: docker-docker--pool
  ...
```

If you see an error that says `ERROR: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?`, start the docker service: `systemctl start docker`.

The "Storage Driver" in the command above indicates that `DeviceMapper` is in use. If not, follow these steps:

* If you don't have a spare partition or disk, you can just use a `loopback` device:

```bash
cat << EOF > /etc/docker/daemon.json
{
     "mtu": 8900,
     "runtimes": {
         "runxc": {
                 "path": "/usr/local/bin/kata-runtime"
         }
     },
     "storage-driver": "devicemapper"
}
EOF
```

* If you are using a dedicated block device (e.g., `/dev/nvme1n1`):

```bash
cat << EOF > /etc/docker/daemon.json
{
     "mtu": 8900,
     "runtimes": {
         "runxc": {
                 "path": "/usr/local/bin/kata-runtime"
         }
     },
     "storage-driver": "devicemapper",
     "storage-opts": [
        "dm.directlvm_device=/dev/nvme1n1",
        "dm.thinp_percent=95",
        "dm.thinp_metapercent=1",
        "dm.thinp_autoextend_threshold=80",
        "dm.thinp_autoextend_percent=20",
        "dm.directlvm_device_force=false",
        "dm.basesize=10G"
     ]
}
EOF
```

4. Reload the Docker daemon and restart Docker.

```bash
systemctl daemon-reload
systemctl restart docker
```

5. (Optional) Verify the installation with the `xspot check -f` command. This should be in the PATH of the user. To run as root user, add the directory where `xspot` is to the root PATH. You can check this by running `whereis xspot` as the user.

Now X-Spot is installed, you can refer to [X-Spot instructions](http://repo.exotanium.io/private/f2dd2dba0c79b1790f02a62a9f09e244f6594f71/instructions.html) for detailed usage information.

### 1.1.5 Final check

Run `xspot check -f` to make sure everything is correct.

### 1.1.6 Troubleshoot

**a)** Can't reach the metadata server

**Answer**: xspot must run with root permissions.

**b)** xspot `command not found` or `path does not contain`

**Answer**: xspot must run with root permissions. If you are using `sudo`, the problem is that `sudo` does not inherit `PATH` from the user. You can fix this by running `sudo visudo` and modifying the `secure_path` variable by adding `/usr/bin:/usr/local/bin` to the end of it. For example:

`Defaults secure_path = /sbin:/bin:/usr/sbin:/usr/bin:/usr/local/bin:/usr/local/sbin`

**c)** Error: `Verifying that docker is running: failure`

**Answer:** This error will appear when you start a fresh e xspot controller. Start the docker service: `sudo systemctl start docker`

**d)** `[ERROR] xspot is not running`

**Answer:** Run `xspot restart.`

## 1.2 Package an AMI for X-Spot worker nodes

### 1.2.1 Create the AMI

From your controller instance, you can start and configure a worker node using a base instance of CentOS-7 or Amazon Linux 2. (for base CentOS AMIs, [check this link](https://www.centos.org/download/aws-images/)).

_Note: When using a base CentOS-7 or Amazon Linux 2 AMI, the worker AMI creation time may take longer than the default timeout. Because of this we recommend temporarily increasing the timeout to 12 minutes (or longer if the image is out of date) using:_

```bash
xspot updateconfig booting_timeout:15
```

1. For example, update `config.toml` to use a base Amazon Linux 2 instance (Ohio region):

```bash
#default ami for creating worker instances
ami = "ami-0adc4758ea76442e2"
```

You can also use CI/CD images. You can find these on the AMI list with names like `centos7-worker-xspot-0.2.0.rpm.`

_**Note:** If you are trying to launch a VM that you want to stay alive, you must **disable the scheduler**, which is the second line of config.toml, otherwise the worker will set up and get terminated._

2. Restart the scheduler to reload your `config.toml` changes:

```bash
systemctl stop exo-d; rm -rf /run/exotanium/recovery; systemctl start exo-sched;
```

3. Launch a self-installing worker instance:

```bash
xspot add
```

You can check the progress by going to the AWS console, locate the VM being created, press Connect, EC2 serial console, Connect.

Once the instance is fully configured, it will connect to your controller and the status should be reported as 'Ready':

```console
# xspot ps
====Config====
SchedulerEnabled : false
DefaultContainerCpu : 4
DefaultContainerMem : 4096


====Containers====
CONTAINER ID	WORKER ID	TARGET WORKER ID	STATUS	CPU	MEMORY

====Workers====
WORKER ID	STATUS	INSTANCE ID        	INSTANCE TYPE	REMAINING MEM	EC2 IP      	UP TIME	DEADLINE	SPOT 	PRICE
8bb2b18a 	Ready 	i-0e11e2fc2966c4622	m5.xlarge    	12288        	172.31.22.91	1m10s  	53m49s  	false	0.192
```

To troubleshoot issues during installation, you can use the EC2 console for your worker instance or monitor the installation log: `/usr/share/exotanium/logs/<instance_id>/install.log`

4. SSH into the worker VM and make sure that Xen is running by checking the output of the following command:

```console
$ /usr/sbin/xl list
Name                                        ID   Mem VCPUs  State Time(s)
Domain-0                                     0  1024     4     r-----     122.6
```

5. Test if docker can launch containers in the worker:

`sudo docker run -it --rm --runtime runxc hello-world`

5. **(Recommended)** Package the VM into an AMI. While the self-installation of the worker is convenient, we recommend creating an AMI and updating `config.toml` to use that AMI to speed up booting of your worker instances. **When creating an image, enable "No reboot" or otherwise the worker instance will be terminated and the image creation will fail.** You can follow the steps listed in [Amazon AWS Documentation](https://docs.aws.amazon.com/toolkit-for-visual-studio/latest/user-guide/tkv-create-ami-from-instance.html).
6. Troubleshooting:
   1. Look through `/var/log/xspot/latest/xspot.log` and `/var/log/xspot/latest/workers/<worker-ip>/messages` to monitor the worker and docker container initialization process.
   2. Docker Error: CreateNewContainer failed to set license CPU
      1. In config.toml under \[LICENSE], set`type = "x-spot-developer"`. If this does not work, your x-spot version might be outdated.&#x20;

### 1.2.2 Customizing worker node installation

In addition to the default self-installation capabilities, it is possible to provide your own customization instructions when, for example, you need the worker instance to install additional packages or run other commands.

Whenever a pre-configured AMI is detected and launched, the contents of the script located at `/usr/share/exotanium/customize_worker_ami.sh` will be executed. We recommend testing those instructions on a running worker instance first to avoid issues when booting workers.



### **1.2.3** Manually installing the worker node

For further customization, it is possible to manually edit and run the worker installation script and connect to the controller by following these steps:

1. Select an Amazon Linux 2 instance. Choose the same security group used by the controller and launch your instance.
2. Create a directory to mount the X-Spot installation files:

```bash
mkdir -p /usr/share/exotanium
```

3. From the AWS console, get the private IP of the controller (MASTER\_IP) and use it to mount the installation directory:

```bash
mount -t nfs -o nfsvers=3,lock,proto=tcp,local_lock=all MASTER_IP:/usr/share/exotanium /usr/share/exotanium
```

4. Create the worker configuration file:

```bash
cat << EOF > /etc/exotanium/x-init.env
MASTER_IP=172.31.28.182
MASTER_IP_PORT=172.31.28.182:24002
WORKER_ID=man-01af44b8-7ac2-45bf-bfc7-439003423255
MNG_IP=10.0.0.232
MTU=8900
DEFAULT_IP=172.31.26.160
INSTANCE_ID=i-0198a3c8014da3329
EOF
```

| Parameter        | Description                                      |
| ---------------- | ------------------------------------------------ |
| MASTER\_IP       | Controller private IP                            |
| MASTER\_IP\_PORT | Controller private IP + port 24002               |
| WORKER\_ID       | User-assigned worker ID (Must start with "man-") |
| MNG\_IP          | Overlay network IP 10.0.0.x                      |
| MTU              | 8900 for AWS recommended                         |
| DEFAULT\_IP      | Worker private IP                                |
| INSTANCE\_ID     | AWS instance ID                                  |

5. Run the worker installation script:

```bash
/usr/share/exotanium/install_worker_base.sh
```

At the end of the installation, the worker should reboot and connect automatically to the controller.

\_\_\_

## <mark style="color:yellow;">2. Integrate with AWS Batch/ECS</mark>

1. Follow the above instructions and install an X-Spot controller. We will package the controller VM as an AMI that can be managed by AWS Batch.
2. Make `runxc` the default runtime of Docker by adding `"default-runtime": "runxc"` to `/etc/docker/daemon.json`. This is an example:

```bash
cat << EOF > /etc/docker/daemon.json
{
     "mtu": 8900,
     "default-runtime": "runxc",
     "runtimes": {
         "runxc": {
                 "path": "/usr/local/bin/kata-runtime"
         }
     },
     "storage-driver": "devicemapper",
     "storage-opts": [
        "dm.directlvm_device=/dev/nvme1n1",
        "dm.thinp_percent=95",
        "dm.thinp_metapercent=1",
        "dm.thinp_autoextend_threshold=80",
        "dm.thinp_autoextend_percent=20",
        "dm.directlvm_device_force=false",
        "dm.basesize=10G"
     ]
}
EOF
```

3. Enable X-Spot scheduler in `/etc/exotanium/config.toml`:

```
scheduler_enabled = true
```

4. Create the required **iptables** rules for the ECS agent:

```bash
echo 'net.ipv4.conf.all.route_localnet = 1' >> /etc/sysctl.conf
sysctl -p /etc/sysctl.conf

echo 'iptables -t nat -A PREROUTING -p tcp -d 169.254.170.2 --dport 80 -j DNAT --to-destination 127.0.0.1:51679' \
    >> /etc/exotanium/custom_init.sh
echo 'iptables -t nat -A OUTPUT -d 169.254.170.2 -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 51679' \
    >> /etc/exotanium/custom_init.sh
```

5. Create the required directories for the ECS agent.

```bash
mkdir -p /var/log/ecs /var/lib/ecs/data /etc/ecs
```

6. Create a config file for the ECS agent. Note that we skip `ECS_CLUSTER` here since it will be added by Batch automatically:

```bash
cat << EOF > /etc/ecs/ecs.config
ECS_DATADIR=/data
ECS_ENABLE_TASK_IAM_ROLE=true
ECS_ENABLE_TASK_IAM_ROLE_NETWORK_HOST=true
ECS_LOGFILE=/log/ecs-agent.log
ECS_AVAILABLE_LOGGING_DRIVERS=["json-file","awslogs"]
ECS_LOGLEVEL=info
EOF
```

7. Create a systemd unit file to start the ECS agent.

```bash
cat << EOF > /etc/systemd/system/docker-container@ecs-agent.service
[Unit]
Description=Docker Container %I
Requires=docker.service
After=cloud-final.service

[Service]
Restart=always
ExecStartPre=-/usr/bin/docker rm -f %i
ExecStart=/usr/bin/docker run --name %i --runtime runc --privileged --restart=on-failure:10 --volume=/var/run:/var/run --volume=/var/log/ecs/:/log:Z --volume=/var/lib/ecs/data:/data:Z --volume=/etc/ecs:/etc/ecs --net=host --env-file=/etc/ecs/ecs.config amazon/amazon-ecs-agent:latest
ExecStop=/usr/bin/docker stop %i

[Install]
WantedBy=default.target
EOF
```

8. Enable the Amazon ECS agent systemd unit.

```bash
systemctl enable docker-container@ecs-agent.service
```

9. Attach the following pre-defined policy to the role of your X-Spot controller VM:
   * `AmazonEC2ContainerServiceforEC2Role`
10. Connect to AWS ECS manually (optional)

At this point, your VM should be ready for packaging into an AMI. Since AWS Batch supports customized AMI, you can use this AMI for the compute environment. AWS ECS does not support AMI customization, but you can manually attach your VM to an ECS cluster. You can also use this step as a test to make sure that your setup is working before creating your AMI.

10.1 Open `/etc/ecs/ecs.config` and add the following line:

```
ECS_CLUSTER=your_ecs_cluster
```

10.2 Start X-Spot and the ECS agent:

```bash
systemctl start exo-sched
systemctl start docker-container@ecs-agent.service
```

Your VM should be able to join your ECS cluster and run ECS jobs.

11. Package an AMI

Before creating the AMI, clean up `/var/lib/ecs/data`, and remove the `ECS_CLUSTER` we added into `/etc/ecs/ecs.config`, if any:

```bash
rm -rf /var/lib/ecs/data/*

cat << EOF > /etc/ecs/ecs.config
ECS_DATADIR=/data
ECS_ENABLE_TASK_IAM_ROLE=true
ECS_ENABLE_TASK_IAM_ROLE_NETWORK_HOST=true
ECS_LOGFILE=/log/ecs-agent.log
ECS_AVAILABLE_LOGGING_DRIVERS=["json-file","awslogs"]
ECS_LOGLEVEL=info
EOF
```

12. Create a new compute environment in AWS batch with user-specified AMI. Remember the following important steps:

* You need to choose a security group for your compute environment that allows all internal traffic, as indicated in [Section 1.1.1 Prerequisites](./#111-prerequisites).
* Instead of using the default IAM role for the EC2 instances in your compute environment, you should use the role created specifically for your X-Spot controller.

## Onboarding Troubleshooting

{% embed url="https://exostellar-io.slack.com/archives/CRHEL13FD/p1687535512491909" %}
Some discussion on issues I found.
{% endembed %}

Container errors? e.g. Failed to verify docker image, pull access denied:\
&#x20;Follow the PAT setup in [https://github.com/Exostellar/xspot](https://github.com/Exostellar/xspot). Basically, go to [https://github.com/settings/tokens](https://github.com/settings/tokens), generate a classic token with `read:packages.`

On your terminal, run `echo <your_github_pat> | docker login -u <your_github_username> ghcr.io --password-stdin`

Follow  [https://github.com/hfingler/exo-udp-test](https://github.com/hfingler/exo-udp-test) for a migration test.

