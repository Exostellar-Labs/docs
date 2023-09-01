# [Onboarding Guide](#onboarding-guide)

> Overview / At-a-glance
> | Basic Phase | Critical Concepts or Components | Additional Information | Steps in this guide |
> | :--- | :--- | :--- | ---: |
> | Before you begin | Setup | | [Prerequisites](#prerequisites) |
> | Download installation assets | Obtain and unpack onboarding tarball | On X-Spot Controller, remote filesystem mounted on instance, intended to permanently house X-Spot assets, available to all X-Spot Controller instances | [Onboarding Assets](#onboarding-assets) |
> | Site-specific default settings | file: `onboarding/conf/env.cfg` | On X-Spot Controller | [Pre-Configuration](#pre-configuration) |
> | AMI creation | Temporary instance will be discarded immediately | On X-Spot Worker AMI instance | [X-Spot Worker AMI Preparation](#x-spot-worker-ami-preparation) |
> | Environment snapshot | Fully validated compute resources used to run all workflows | On X-Spot Controller -- this resource will be dedicated to X-Spot use | [Containerization](#containerization) |
> | X-Spot installation | Significant modification required during installation | On X-Spot Controller, considerable testing and validation will be required | [X-Spot Integration](#x-spot-integration) [X-Spot Validation](#x-spot-validation) |
> | Deploy to production | Package validated assets for production | On X-Spot Controller, make available to every X-Spot Controller | [Script Deployment](#script-deployment) |
> | Replicate X-Spot functionality on multiple targets | Install and activate X-Spot and production assets on fleet of X-Spot Controllers | Can be done in parallel | [Cluster Deployment](#cluster-deployment) |
> | Full integration system-wide | Scheduler-specific integration | How end-user pragmatically leverages X-Spot solution | [Running Workloads with X-Spot](#running-workloads-with-x-spot) |
> | Upgrade for new release | Test and re-validate | On one X-Spot Controller and then replicate on others | [Upgrading X-Spot Controllers](#upgrading-x-spot-controllers) |

___

1. [Prerequisites](#prerequisites)
1. [Onboarding Assets](#onboarding-assets)
1. [Pre-Configuration](#pre-configuration)
1. [X-Spot Worker AMI Preparation](#x-spot-worker-ami-preparation)
1. [Containerization](#containerization)
1. [X-Spot Integration](#x-spot-integration)
   * [X-Spot Configuration](#x-spot-configuration)
   * [exorun Configuration](#exorun-configuration)
   * [X-Spot Installation](#x-spot-installation)
1. [X-Spot Validation](#x-spot-validation)
   * [As root](#x-spot-validation-as-root)
   * [As User](#x-spot-validation-as-user)
   * [When Satisfied](#x-spot-validation-when-satisfied)
1. [Script Deployment](#script-deployment)
1. [Cluster Deployment](#cluster-deployment)
1. [Running Workloads with X-Spot](#running-workloads-with-x-spot)
1. [Upgrading X-Spot Controllers](#upgrading-x-spot-controllers)
   * [Backup or Archive](#backup-or-archive)
   * [Fresh Upgrades](#fresh-upgrades)
   * [Upgrading Configuration](#upgrading-configuration)
   * [Upgrading Cluster Deployments](#upgrading-cluster-deployments)

[Appendix](#appendix)
1. [AGE and qsub](#age-qsub.md)
1. [LSF and bsub](#lsf-bsub.md)
1. [SLURM and sbatch](#slurm-sbatch.md)
1. [Cost Calculation](#cost-calculation)
1. [S3 Sync](#s3-sync)

___

## Prerequisites

  1. All commands will be run as the root user, unless otherwise noted.

  1. Onboarding activities require an existing HPC Environment (e.g. a cluster) or at a minimum one VM.
     * If starting from scratch, create a VM with CentOS-7. We recommend EC2 instances with [Enhanced Networking](https://aws.amazon.com/premiumsupport/knowledge-center/enable-configure-enhanced-networking/) and EBS SSD volumes.
       > **IMPORTANT:**
       >
       > Amazon Linux 2023 instances are not supported. See [Amazon Documentation](https://docs.aws.amazon.com/linux/al2023/ug/compare-with-al2.html#epel) for more information.
     * If proceeding minimally with one VM only, install the targeted workflow's application requirements and validate proper execution of the workflow for end-users as required prior to typical go-live or release-to-production.
     * If proceeding with an existing HPC Cluster, all workflows, authentication systems, and licensing requirements should be fully validated and operational.

  1. Create or choose a VPC and a subnet that you want to put your X-Spot controller and workers in.
     * X-Spot creates an internal overlay network with CIDR 192.168.137.0/24 by default.
     * We need to ensure the overlay network does not conflict the CIDR in your VPC.
     * If there is any overlapping, we will [modify X-Spot configurations](#3-configuration) as described later.

  1. Add a NAT gateway to your subnet.

  1. Allow access to the following websites and URLs:
     * AWS EC2 API Endpoint:
       - [https://ec2.amazonaws.com/](https://ec2.amazonaws.com/)
     * Exostellar RPM repos:
       - [https://rpm.exostellar.dev/](https://rpm.exostellar.dev/)
       - [https://rpm.exostellar.io/](https://rpm.exostellar.io/)
     * Entitlement API server:
       - [https://13.66.175.108](https://13.66.175.108)
     * Microsoft eventhub API endpoint:
       - TBD
     * (optional) X-Spot advisor service endpoint:
       - [https://3.135.60.35](https://3.135.60.35)

  1. Create a security group for X-Spot controller and workers.
     * In addition to the inbound rules that are required by your application (such as the SSH port, or licensing and license-vendor-daemon ports), it should also allow all internal traffic within your subnet.
     * For example, if your subnet uses 172.31.0.0/20, your security group should allow "All traffic" from 172.31.0.0/20.
     * You can put the controller and workers in different security groups as long as both of them allow all internal traffic.

  1. Create an IAM role for the X-Spot controller
     * X-Spot makes requests to Amazon Web Services.
     * To support that, you need to create and attach an IAM role to the instance running the X-Spot controller.
     * The included policy [iam-policy.json](iam-policy.json) needs to be attached to the role
     
     > **IMPORTANT:**
     >
     > This role will need to be attached to **every X-Spot controller**.
    
      7.1. Changelog:
        2023-08-11 : added ec2:DescribeInstanceTypes into the IAM policy.

  1. Make note of linux and AWS services that are likely to be required, as well as any pertinent version information. Some filesystem or directory services may require additional configuration and validation to ensure compatability. The services listed below fall into two general categories: authentication or directory services (ldap, sssd) and remote-filesystems (Lustre, NetApp ONTAP, autofs). If a required service is not listed below but is required, please let support know about your needs and use-case.
     * Amazon FSx for Lustre
     * Amazon FSx for NetApp ONTAP
     * autofs
     * sssd
     * ldap

  1. X-Spot currently supports 64-bit applications. 32-bit applications are not yet supported.

  1. There are two limiting factors in terms of how many workers or containers can be started on one X-Spot controller:
     > **EQUATION:** Num of containers + num of workers * 2 <= 245.
     * This is due to how X-Spot manages its internal metadata connections. Based on this equation, the max number of containers that you can start depends on how you configure X-Spot. Typically we will start one worker for each container, so that effectively limits the number of containers (or workers) to be 81. But if you configure X-Spot to pack multiple containers on one worker, then the number of containers can be higher.
     * Root file system usage for all the containers cannot exceed the total disk space available to docker for storage. For example, if we are expecting that each container will generate 30GB data on the root file system, and the docker thin pool device has a 300GB SSD, then we may run into issues if 10 or more containers are concurrently using 30G of storage to run their jobs, with some failing due to storage errors, "No space left on device."
     * The real limit of how many containers we can start on a single controller will be the smaller number of the two mentioned above. For the controller, CPU usage is typically not a concern, since the disk space for docker images is the bottleneck. But we recommend 4 CPUs (2 cores) and 8GB memory and a local SSD for the controller. On AWS this would be a c5d.xlarge or m5d.xlarge VM, and can go with other instance types depending on the storage requirements.

  1. VMs or EC2 instances required for this guide:
     * HPC Cluster Environment
       * | VM Reference Name | Required? | Recommended Instance Type | Comment |
         | :--- | :--- | :---: | ---: |
         | X-Spot controller | Yes | m5d.xlarge or an instance with "d" before the period | Most steps take place on this node. Ideally it is a fully functional and validated compute node in the cluster. |
         | X-Spot worker AMI builder | Yes | m5.xlarge or c5.xlarge | This VM will only exist for 15 - 30 minutes and can be terminated thereafter. |
       > **IMPORTANT:**
       >
       > Amazon Linux 2023 instances are not supported. See [Amazon Documentation](https://docs.aws.amazon.com/linux/al2023/ug/compare-with-al2.html#epel) for more information.

       > **TAKE-AWAY:**
       >
       > Minimum requirements are workflow and environment dependent. It's recommended to start with m5d.xlarge instance for X-Spot controllers, which has a dedicated 150GB SSD for docker storage. m5d.2xlarge has a 300GB SSD, so could run roughly double the number of jobs. See [AWS Instance Descriptsion](https://aws.amazon.com/ec2/instance-types/) for more information.

[Back to Top](#onboarding-guide)

_______

## Onboarding Assets
> **Note:**
>
> You should be :
> | Who? | On which system? | In which directory? |
> | :--- | :--- | :--- |
> | root | X-Spot controller | /tmp |
  1. Identify a compute node from the HPC Cluster and prevent jobs from running on it by marking it down, offline, drain, etc.
     * Alternatively, you may proceed with only a single VM but ensure that no workflows are running on it.
  1. Download the tarball of the latest release of X-Spot via the following command:
     ```bash
     wget https://rpm.exostellar.io/download/onboarding.tar
     ```
  1. This tarball will need to be unpacked into a location that is available to all the nodes in the cluster. The assumption is that this will be a remotely mounted directory. For example, many systems locate software available cluster-wide in the `/apps` or `/opt` directory.
     * The following commands **should not be copy-pasted** because `/nfs-apps` is a generic term or stand-in for your remote directory/mount.
     * To keep things contained, let's make a directory for these onboarding assets under `/nfs-apps`:
       ```bash
       mkdir /nfs-apps/exostellar
       ```
     * We'll unpack the tarball via the `tar` command wherever the `onboarding.tar` file was downloaded:
       ```bash
       tar xvf onboarding.tar -C /nfs-apps/exostellar
       ```
     * Next, we'll `cd` to the `onboarding` directory that resulted from the previous command:
       ```bash
       cd /nfs-apps/exostellar/onboarding
       ```
     * To track changes that are anticipated, we will use `git`:
       - Initialize an empty local repository:
         ```bash
         git init .
         ```
       - Add everything in the directory to the `git` version-control system:
         ```bash
         git add .
         ```
       - "Commit" the current state of the files in and under this directory so we can track changes.
         ```bash
         git commit -m $( cat version )
         ```
       - We will see more `git` commands later on in the onboarding process, particularly when upgrading for a latest release of X-Spot.
     * We are now ready to begin configuration and installation steps.

[Back to Top](#onboarding-guide)

_______

## Pre-Configuration
> **Note:**
>
> You should be :
> | Who? | On which system? | In which directory? |
> | :--- | :--- | :--- |
> | root | X-Spot controller | /nfs-apps/exostellar/onboarding |
>
> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder
  1. Edit the `conf/env.cfg` file and set required variables.
     * The `conf/env.cfg` contains useful explanations, as shown below:
       - ```bash
         #COMMENTS allowed on their own lines or after declarations
         #TRAILING slashes in dir_names are not allowed and will be stripped
         #BLANK lines are fine and will be ignored

         #REMOTE_DIR         shared storage mounted remotely, will house production assets
         REMOTE_DIR=/efs/deploy

         #CONTAINER_FS_SIZE  max size the container will consume on disk
         CONTAINER_FS_SIZE=50G

         #IMAGE_DEPLOY_DIR   where production images will reside
         IMAGE_DEPLOY_DIR=${REMOTE_DIR}/images

         #SCRIPT_DEPLOY_DIR  where production tools will reside
         SCRIPT_DEPLOY_DIR=${REMOTE_DIR}/scripts

         #LUSTRE_NEEDED=1    #uncomment to activate LUSTRE preparation,  default is LUSTRE_NEEDED=0
         #AUTOFS_NEEDED=1    #uncomment to activate AUTOFS preparation,  default is AUTOFS_NEEDED=0
         #LDAPS_NEEDED=1     #uncomment to activate LDAPS preparation,   default is LDAPS_NEEDED=0
         #SSSD_NEEDED=1      #uncomment to activate SSSD preparation,    default is SSSD_NEEDED=0

         #ONTAP_NEEDED=1     #uncomment to activate ONTAP preparation,   default is ONTAP_NEEDED=0

         #NetApp ONTAP folders required in the container must be known during Pre-Configuration
         #   give the full path, e.g. ONTAP_FOLDERS=( "/ontap/folder1" "/ontap/folder2" )
         #ONTAP_FOLDERS=( "/ontap/1" "/ontap/2" )

         #### Modifications of the below settings are likely only needed when requested by Support.
         #### Typical installations would not generally modify the values below.

         #list of image names, space separated
         #e.g.: IMAGE_NAME_LIST=("docker-img1" "docker-img2:tag")
         IMAGE_NAME_LIST=("docker-base")

         #XSPOT VERSIONING
         XSPOT_RPM_VERSION=xspot-1.3.0.1-rc_5cb0ffb0.el7

         XSPOT_PLATFORM=xcontainer-4.12.0-v23.08.07+abaaf51.el7
         XSPOT_KERNEL=kernel-4.19.49_xcontainer-v23.08.07+abaaf51
         XSPOT_GUEST_KERNEL=xspot-guest-kernel-4.19.49-v23.08.07-abaaf5112.el7


         CLI_WRAPPER_VERSION=XSpotCLIWrapper-v2.0.0-0192f37.el7

         #IS_SYSTEMD=no      #IS_SYSTEMD=no is the only way to get non-systemd.template assets

         ```
     * Ensure `REMOTE_DIR` is configured properly.
       - The `REMOTE_DIR` is required to be on a remote filesystem and available to all nodes in the cluster.
     * `CONTAINER_FS_SIZE` will act as a hard limit on storage utilization.
       - If a target workflow typicaly makes significant use of scratch space on the local filesystem, this limit should be carefully considered.
       - If more data than this limit is written to the local filesystem during job run, an error will occur and the job will error terminate.
     * `REMOTE_DIR`, `IMAGE_DEPLOY_DIR`, and `SCRIPT_DEPLOY_DIR` can be set according to your preferences, with the following caveats:
       - `REMOTE_DIR` is expected to be an NFS, LUSTRE, AUTOFS, or ONTAP mount.
       - `IMAGE_DEPLOY_DIR` and `SCRIPT_DEPLOY_DIR` should be available to any X-Spot controller.
     * Certain AWS or linux services that may be in use in your environment should be toggled "on" now in the config, for example:
       - Amazon FSx for Lustre
       - Amazon FSx for NetApp ONTAP
       - autofs
       - sssd
       - ldap
       - For example, remove the `#` at the front of the line that starts with `#AUTOFS_NEEDED=1` if `autofs` is part of the environment.
       > **NOTE:**
       >
       > Later on during [validations](#x-spot-validation-as-root) specific information unique to the site will need to be tested and validated. For example, mounting of remote filesystems may be accomplished in several ways, so a final decision will be made during validation and the configuration will be finalized at that time.
     * Amazon FSx for NetApp ONTAP handling
       > **IMPORTANT:**
       >
       > NetApp ONTAP storage needs some special attention.
       >
       > If your ONTAP storage is mounted by `autofs`, make sure that you toggle both `ONTAP_NEEDED` and `AUTOFS_NEEDED` on in `env.cfg`, and also configure `ONTAP_FOLDERS` correctly. Incorrect configuration of `ONTAP_FOLDERS` will cause failures in worker creation.
       >
       > If your ONTAP storage is not mounted by `autofs`, then append the following section to the end of the `scripts/templates/customize_worker_ami.sh` file:
       >```bash
       >    mkdir -p /fsx
       >    mount -t nfs ONTAP_SERVER_IP:/volume1 /fsx
       >```
       >
       > Be sure to customize the ip ( `ONTAP_SERVER_IP` ) and the volume name ( `/volume1` ) and the mount point ( `/fsx` ).
       >
       > If more volumes are required, these lines should be repeated for each volume.
       >
       > The purpose of these steps is to make sure that ONTAP stroage is mounted correctly on the X-Spot worker node. In the step of [X-Spot Validation](#x-spot-validation), you can login to the worker node with ssh or Session Manager, and check if all ONTAP storage folders are mounted correctly.
       >
       > When there are multiple NetAPP ONTAP Servers in play, the minimum requirement is one volume per server should be specified in the `customize_worker_ami.sh` script as shown in the example above. Make sure all the mount points are listed from `conf/env.cfg`'s ONTAP_FOLDERS.

  1. Once edited, the following command will report if any errors are found in conf/env.cfg
     ```bash
     make check-config
     ```
     * The above step is optional.
     * The configuration will be automatically checked at each step just in case any unexpected modifications take place and when errors are found, they will be reported and the installation will be interrupted as a matter of safe-guarding.

[Back to Top](#onboarding-guide)

_______

## X-Spot Worker AMI Preparation

> **NOTE:**
>
> A disposable or scratch [AWS Nitro-enabled](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-types.html#ec2-nitro-instances) VM will be required. The required VM will be discarded after use. For this short duration-task, we **recommend a c5.xlarge or m5.xlarge.** Using t2.micro, for example, will result in errors.
>
> This step can be done in parallel with the following section, if you'd like to save time.
>
> For example, during [Containerization step](#containerization) after issuing the `make base` command might be a good time come back to this step while that command runs.

  1. Create the worker AMI.
     * Create a new CentOS-7 VM in the AWS Console.
     * **Note:** The AWS Console provides information for connecting the the new VM but this information contains an error:
       - If creating an CentOS VM, the user who needs to connect is `centos` not `ec2-user` as reported via the AWS Console.
     > **Note:**
     >
     > You should be :
     > | Who? | On which system? | In which directory? |
     > | :--- | :--- | :--- |
     > | root | X-Spot worker AMI Builder | /tmp |

     * Ensure the onboarding folder is available on this new VM:
       - Copy the onboarding assets to `/tmp`
       - Mount the `/nfs-apps` folder containing the onboarding assets on the new VM and navigate to it.
     * If Security Enhanced Linux (SELinux) is enabled on this VM/EC2 Instance, an error will be encountered.
       - Amazon Linux 2 instances generally have SELinux disabled by default.
       - Disable selinux via the following commands if necessary:
       - ```bash
         setenforce 0
         ```
       - ```bash
         sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
         ```
     * Run the following:
       ```bash
       make install-xspot-worker
       ```
       - **Note:** Running `make install-xspot-worker` can take about 5 or 10 minutes to run to completion.
       - This installs everything required for xspot-workers.
     * Reboot the newly created CentOS-7 VM and issue the following command:
       ```bash
       xl info
       ```
       - That's XL INFO all in lowercase and many lines of output should be generated.
       - If instead an error message is thrown, [contact support](mailto:support@exostellar.io) for assistance:
     * Create a new AMI from the VM.
       - To create a new AMI, go to the AWS Console.
       - Navigate to EC2 Services and select the Instance ID of the disposable VM.
       - In the upper right corner, select 'Actions' > 'images and templates' from the drop down.
       - Click on 'Create image'
         * A meaningful name will be required.
         > **NOTE:**
         >
         >   It is recommended to tag this X-Spot Release Version as found in the `onboarding/conf/env.cfg` file so that compatability can be easily tracked and compared in the future.
       - Upon completion, make note of the AMI ID, which will be required in the [X-Spot Integration step](#x-spot-integration).
       - **Note:** It can take a few minutes for the AMI ID to display as `available` in the AWS Console.

[Back to Top](#onboarding-guide)

_______

## Containerization

  > **NOTE:**
  >
  >   Subsequent steps require a fully functional and validated compute node from the cluster or a single VM if no HPC Cluster: the goal is to replicate the perfect environment.

  > **NOTE:**
  >
  >   If the node already has docker installed, pre-existing docker images will be removed. Therefore, docker images that are on the critical path and not backed up or housed anywhere but on this system's disk could be lost. Please make sure you understand this and will not lose unique data before proceeding.

  > **NOTE:**
  >
  >   No jobs should be running on the compute node during the following steps: the cluster admin can help ensure the node is drained, offline, or otherwise prevented from participating in cluster-wide scheduling for a short period of time while the environment is replicated.

> **Note:**
>
> You should be :
> | Who? | On which system? | In which directory? |
> | :--- | :--- | :--- |
> | root | X-Spot controller | /nfs-apps/exostellar/onboarding |
>
> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder

  1. Snapshot the perfect environment's filesystem.
     ```bash
     make base
     ```
     * **Note:** It could take 15 minutes for this command to complete, depending on your environment.
     * If at some later time, a new base snapshot is required, you can replace this asset with the following command:
       - ```bash
         make new-base
         ```
       - This would be a rare occurence in most environments.

  1. Generate or build docker images.
     ```bash
     make build-image
     ```
     * **Note:** It will probably take about 5 to 10 minutes for this command to complete, depending on your environment.
     * The `Dockerfile` and other build assets of the docker image will be available in `./images/docker-base/docker_build_path`

[Back to Top](#onboarding-guide)

_______

## X-Spot Integration

### X-Spot Configuration
> **Note:**
>
> You should be :
> | Who? | On which system? | In which directory? |
> | :--- | :--- | :--- |
> | root | X-Spot controller | /nfs-apps/exostellar/onboarding |
>
> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder
  1. Update `./scripts/controller/integrate-xspot/config/config.toml`
     * The following section of the file can accept more preferences
       ```bash
       [aws.instance_tags]
       Name = xspot-worker
       #i_key2 = "i_value2"
       ```
       - If you would like to introduce more AWS Tagging, copy as necessary the line starting with `#i_key2` and set key-value pairs.
     * Main configs to tune:
       - | Expression | Explanation |
         | :--- | :--- |
         | `ami =` | The worker AMI ID from [X-Spot Worker AMI Preparation step](#x-spot-worker-ami-preparation). . |
         | `overlay_prefix =` | This class C network definition cannot overlap with X-Spot controller's subnet. |
         | `on_demand_types =` | List of instance type and families for on-demand instances. |
         | `spot_fleet_types =` | List of instance type and families for spot instances. |
         | `hyperthreading_disabled =` | Setting to true disables hyperthreading on worker instances. |

  1. The following sections of the file may need updates for your site:
     * Licensing updates to `./scripts/controller/integrate-xspot/config/config.toml`:
       ```bash
       [license]

       #Specify the floating key for the license system. For a mechanism other than floating license, use one of the following values:
       key = "x-spot-metering"
       ```
  1. (optional) Log files by default reside in `/var/log/xspot` on each X-Spot controller.
     * To modify the default behavior, edit the log file configuration:
       ```bash
       ./scripts/controller/integrate-xspot/config/log_upload.env
       ```
     * Look for and reset the `LOG_PATH=` directive in the config file.
     * We recommend a remote filesystem to centralize the X-Spot controller logs if changing the default location.

### exorun Configuration
> **Note:**
>
> You should be :
> | Who? | On which system? | In which directory? |
> | :--- | :--- | :--- |
> | root | X-Spot controller | /nfs-apps/exostellar/onboarding |
>
> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder
There are several scripts and a config file used to configure the X-Spot CLI Wrapper known as `exorun`:
  * `./scripts/controller/integrate-xspot/config/dockerWrapper.toml`:
    - used by system administrators to customize the CLI wrapper installation.
    - Any changes to this file will need a config reload to the cli wrapper to take effect (`exorun config`)
        - `exorun config` is only available to superusers (UID == 0)
      * `container_image`: default container image to use when starting a new container with the CLI wrapper
        - The default is `docker-base`.
      * `approved_container_images`: list of approved container images to be used with the CLI wrapper
      * `log_level`: set how detailed the logs are
      * `env_var_filter`: the list of environment variables in the user's environment to NOT pass into the container
      * `container_start_timeout`: the amount of time (in minutes) to retry starting containers that previously failed to start for a job
        - The default is 30 minutes
      * `rand_backoff_ration`: this ratio is applied to the calculation before calculating the exponential back off time
        - The default is 0.5
        - The ration can be in the range: 0 <= X < 1
      * `container_startup`: location of `startup.sh`
      * `job_startup_*` and `job_cleanup_*`: the location of the startup and cleanup scripts
      * `runtime`: the runtime to use when starting a new container. `runxc` to run inside of X-Spot, `runc` to use the default docker runtime
  * `./scripts/controller/integrate-xspot/scripts/startup.sh`:
    - configures necessary resources inside of the container for the CLI wrapper, creates the user account and groups, sets up hostname of the container, etc.
  * The following scripts are used to by the customer to make configurations on how the docker containers startup/stop
    - `./scripts/controller/integrate-xspot/scripts/job_startup_user.sh`:
      *  anything that needs to happen in the container as the non-root user before the job starts
    - `./scripts/controller/integrate-xspot/scripts/job_startup_root.sh`:
      * anything that needs to happen in the container as root before the job starts
    - `./scripts/controller/integrate-xspot/scripts/job_cleanup_user.sh`:
      * anything that needs to happen after the job finishes as the non-root user
    - `./scripts/controller/integrate-xspot/scripts/job_cleanup_root.sh`:
      * anything that needs to happen after the job finishes as the root user

### X-Spot Installation

Install X-Spot components.

> **NOTE:**
>
>   It is recommended to use an instance that has an additional solid state drive, such as **m5d.xlarge** which has an SSD that docker can leverage for significant performance gains, while m5.xlarge does not have the extra SSD.

> **NOTE:**
>
>   If the node already has docker installed, pre-existing docker images will be removed. Therefore, docker images that are on the critical path and not backed up or housed anywhere but on this system's disk could be lost. Please make sure you understand this and will not lose unique data before proceeding.

> **Note:**
>
> You should be :
> | Who? | On which system? | In which directory? |
> | :--- | :--- | :--- |
> | root | X-Spot controller | /nfs-apps/exostellar/onboarding |
>
> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder
  1. Run the following command:
     ```bash
     make controller
     ```
     * **Note:** It will probably take about 5 to 10 minutes for this command to complete, depending on your environment.

[Back to Top](#onboarding-guide)

_______

## X-Spot Validation

### X-Spot Validation as root
> **Note:**
>
> You should be :
> | Who? | On which system? | In which directory? |
> | :--- | :--- | :--- |
> | root | X-Spot controller | /nfs-apps/exostellar/onboarding |
>
> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder

Validate X-Spot installation and docker images.
  1. It's often preferred to leverage a terminal multiplexer such as GNU's `screen` or `tmux` or simply to open multiple shells on the node. Some windows can be used to monitor progress while others can be used to run commands associated with validation steps.
  1. Run the following command:
     ```bash
     xspot check -f
     ```
     - This command should run with no errors.
  1. To faciliate testing on the X-Spot controller, toggle the X-Spot scheduler off via:
     ```bash
     xspot scheduler off
     ```
  1. Further faciliate testing by requesting an ondemand instance from the X-Spot controller via:
     ```bash
     xspot add -c 4 -m 8
     ```
     * **Note:** It will probably take 2-4 minutes for this command to return.
     > **NOTE:**
     >
     >   If 4 CPU cores and 8GB of memory seem insufficient for your validation workflow, please modify the `xspot add` request for more CPU cores `-c` and for more memory `-m`.

  1. You can monitor progress of the above command by waiting or by opening another shell/terminal-multiplexer-window and issuing the following command:
     ```bash
     xspot ps -t
     ```
  1. When the Worker's Status is listed as "Ready" (last line of output), proceed to the next step.
     > **NetAPP ONTAP Mounts:**
     >
     > If you are using NetAPP ONTAP storages, login to the worker VM created in this step with ssh or Session Manager, and verify that all ONTAP folders are mounted correctly. If the controller's root account has ssh key generated under `/root/.ssh`, the key will be propagated to all worker VMs. You can ssh directly into worker VMs with the root account without the need of a password.
     >
     > To find a worker's ip address, issue the `xspot ps` command which will show similar output to below:
     >```bash
     >====License====
     >SpotEnabled: true
     >ContainersEnabled: true
     >====Config====
     >SchedulerEnabled : false
     >OnDemandTypes : [r5 r5n r5b r5d]
     >SpotFleetTypes : [r5 r5n r5b r5d]
     >DefaultAMI : ami-0222b6150e1c29556
     >BootingTimeout : 5
     >DefaultContainerCpu : 4
     >DefaultContainerMem : 4096
     >PausedWorkerTypes: [r5.24xlarge]
     >
     >
     >====Sandboxes====
     >SANDBOX ID	WORKER ID	TARGET WORKER ID	STATUS	CPU	MEMORY	EC2 IP	CREATED	TAGS
     >
     >====Workers====
     >WORKER ID	STATUS	INSTANCE ID        	INSTANCE TYPE	REMAINING MEM	REMAINING CPU	EC2 IP      	UP TIME  	DEADLINE  	SPOT	PRICE
     >4465ce68 	Ready 	i-0222d5115daa6ef2e	r5b.xlarge   	29694        	4            	172.31.27.26	268h3m43s	-267h8m41s	true	0.088
     >```
     >
     > For the example output above, as root on xspot-controller, you can `ssh 172.31.27.26` and you can inspect your expected mount points with `df` or the `mount` utilities on the worker.
  1. In the original shell or a new one, run the following command to start a container with X-Spot and connect to its console:
     * If `conf/env.cfg` was configured for an image name other than the default, `docker-base`, then the below command may need modification based on your image name.
     ```bash
     exorun run -c 4 -m 8 -i docker-base -- /bin/bash
     ```

  1. Because each site will likely require customization, a working session with support is recommended initially.
  1. Various site-specific services and customizations need to be configured and/or validated.
     - E.g.: authentication ( ldaps, ssd ), mounts ( nfs, autofs ), etc.
     - If `LUSTRE_NEEDED`, `ONTAP_NEEDED`, `AUTOFS_NEEDED`, `LDAPS_NEEDED`, or `SSSD_NEEDED` were toggled "on" during [Configuration](#configuration) above, then these requirements will need to be addressed in the next step.
     - The general approach here is to enter the sandbox started manually with the `exorun` command above and look for the services required. As the `root` user, it should be possible to determine the exact commands that are needed to bring up any required services. These commands and remediations should be tracked for the next step.
  1. Modifications to the container or sandbox environment are expected, which will require editing the `startup.sh` script located at `/etc/xspot/scripts/`.
     - Also possibly the `Dockerfile` and/or `/etc/xspot/config/dockerWrapper.toml` may need changes.

### X-Spot Validation as User
> **Note:**
>
> You should be :
> | Who? | On which system? | In which directory? |
> | :--- | :--- | :--- |
> | an end-user | X-Spot controller | any valid job-submit directory for the cluster |
>
> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder

Test with a valid or real workload.
  1. Once the system administrator's tunings as root are validated, proceed to testing with a real user account.
  1. Run the following command to start a container with X-Spot and connect to its console:
     ```bash
     exorun run -c 4 -m 8 -i docker-base -- /bin/bash
     ```
   
  1. Modify the previous command to run a real workflow.
     * If the workflow is a simple command and its arguments, it will look like the following:
       - ```bash
         exorun run -c 4 -m 8 -i docker-base -- /full/path/to/command param1 param2 [more options or parameters...]
         ```
         > **Reminder:**
         >
         > `stty sane` or `reset` is needed when finished.
     * Or if the command is on your PATH, then the following will work:
       - ```bash
         exorun run -c 4 -m 8 -i docker-base -- command param1 param2 [more options or parameters...]
         ```
         > **Reminder:**
         >
         > `stty sane` or `reset` is needed when finished.
     * If the workflow is leverages a script, `job.sh`, and it is located in the current working directory, then use the following:
       - ```bash
         exorun run -c 4 -m 8 -i docker-base -- ./job.sh [param1 param2 ...]
         ```
         > **Reminder:**
         >
         > `stty sane` or `reset` is needed when finished.

  1. When testing concludes, reenable the X-Spot scheduler via:
     ```bash
     xspot scheduler on
     ```
     - This will clean up the ondemand instance added above via `xspot add -c 4 -m 8`, but if you prefer, you can remove it immediately by issuing:
       ```bash
       xspot rm <worker-id-hash>
       ```

### X-Spot Validation When Satisfied
> **Note:**
>
> You should be :
> | Who? | On which system? | In which directory? |
> | :--- | :--- | :--- |
> | root | X-Spot controller | /nfs-apps/exostellar/onboarding |
>
> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder

  1. Anytime changes are made to the image's `Dockerfile`, the image will need to be updated via the following command:
     ```bash
     make rebuild-image
     ```

[Back to Top](#onboarding-guide)

_______

## Script Deployment
> **Note:**
>
> You should be :
> | Who? | On which system? | In which directory? |
> | :--- | :--- | :--- |
> | root | X-Spot controller | /nfs-apps/exostellar/onboarding |
>
> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder
  1. Before deploying to production, check that `/etc/xspot` configuration assets are synced with the source onboarding directory:
     ```bash
     make sync-config-preview
     ```
     * If any important config files are showing discrepancies, the `diff` will be displayed.
  1. To synchronize assets in the source onboarding directory based on the working configurations in `/etc/xspot`, run the following command:
     ```bash
     make sync-config
     ```
     * You can now proceed to releasing the assets to production in the next step.
  1. Take a snapshot of the `onboarding` folder in its current state as a safeguard and finalize the recently validated image for use running jobs with X-Spot:
     ```bash
     make deploy
     ```
  1. As a final validation, restart the X-Spot suite of services on the X-Spot controller.
     * To faciliate, make use of X-Spot's `restart` command:
       ```bash
       xspot restart
       ```
  1. (Optional) Rerun a quick validation workload or retest X-Spot interactively:
     ```bash
     exorun run -c 4 -m 8 -i docker-base -- /bin/bash
     ```


[Back to Top](#onboarding-guide)

_______

## Cluster Deployment
> **Note:**
>
> You should be :
> | Who? | On which system? | In which directory? |
> | :--- | :--- | :--- |
> | root | Cluster Head or Master node | Depends on HPC Scheduler |
>

  1. Create a new "xspot" queue in the cluster's scheduler configuration. This new queue or partition is meant to be comprised soley of X-Spot controllers.
  1. There are a few ways of bringing up X-Spot controllers automatically:
     * You may prefer to create an AMI from a fully validated X-Spot controller, and use that for new controller VMs.
       - Create a new AMI from the VM.
         * To create a new AMI, go to the AWS Console.
         * Navigate to EC2 Services and select the Instance ID of an X-Spot controller.
         * In the upper right corner, select 'Actions' > 'images and templates' from the drop down.
         * Click on 'Create image'
         * A meaningful name will be required.
         * Upon completion, make note of the AMI ID, which can expedite deployments of future X-Spot controllers for the site.
     * You can run these commands by hand on an X-Spot controller that will persist beyond reboots.
       > **Note:**
       >
       > You should be :
       > | Who? | On which system? | In which directory? |
       > | :--- | :--- | :--- |
       > | root | X-Spot controller | ${REMOTE_DIR}/scripts/latest/ also known as ${SCRIPT_DEPLOY_DIR}/latest |
       >
       > **Reminder:** `REMOTE_DIR` and `SCRIPT_DEPLOY_DIR` are set in the `onboarding/conf/env.cfg` file.
       - ```bash
         cd ${SCRIPT_DEPLOY_DIR}/latest
         make controller
         systemctl start post-docker &
         ```
         > **Note:**
         >
         > Running `systemctl start post-docker` can easily take 15 minutes depending on your environment and the number of images configured.
       - You can monitor the progress of starting the `post-docker` service with the following command:
         ```bash
         journalctl -u post-docker -f
         ```
       - You will know the `post-docker` service has successfully completed when you this following message in the output of the previous command:
         ```bash
         ... post-docker.sh: Building of conatiner docker-base: SUCCESS!
         ... post-docker.sh: Finished.
         ... systemd: Started Post Docker Configuration.
         ```
     * Similar to the previous option, you can add this logic to X-Spot controller bootstrapping in auto-scaling environments.
       - Run the following script when the VM boots:
         ```bash
         cd ${SCRIPT_DEPLOY_DIR}/latest
         make controller
         systemctl start post-docker
         ```
         * **Note:** Running `systemctl start post-docker` can easily take 15 minutes depending on your environment and the number of images configured.
  > **IMPORTANT:**
  >
  > The IAM role from the [prerequisites step](#prerequisites) will need to be attached to **every X-Spot controller**.

[Back to Top](#onboarding-guide)

_______

## Running workloads with X-Spot
In most production environments, X-Spot will be integrated with the scheduler for frictionless adoption, but it would still be possilbe leverage `exorun` manually since it is will be on on every users' PATH on X-Spot controllers by default.
> **Note:**
>
> You should be :
> | Who? | On which system? | In which directory? |
> | :--- | :--- | :--- |
> | an end-user | any valid submit host or login node | a valid job-submit directory |

  1. X-Spot integration for job submission via native `bsub`, `qsub`, and `sbatch` commands:
     * Once the `bsub`, `qsub`, `sbatch` integration is setup, X-Spot jobs (routed to the xspot queue) will automatically interoperate with the scheduler. E.g:
       ```bash
       bsub -q xspot -n 2 ./job.sh
       ```
  1. Refer to the documents on scheduler integration for more details.
     * [bsub](.lsf-bsub.md)
     * [qsub](.age-qsub.md)
     * [sbatch](.slurm-sbatch.md)
  1. To add `exorun` to your current shell's path, you can run a few commands to simplify.
  1. Validate the `exorun` executable's location and permissions:
     ```bash
     ls -la /usr/bin/exorun
     ```
  1. If it's in that place as exepcted, ensure its mode is correct:
     ```bash
     chmod 755 /usr/bin/exorun
     ```
  1. If `/usr/bin` isn't already on your path, add that directory to your path:
     ```bash
     export PATH=${PATH}:/usr/bin
     ```
  1. Validate your shell knows where to find `exorun`:
     ```bash
     which exorun
     ```
  1. Jobs can be started in X-Spot containers by using the job wrapper:
     ```bash
     exorun run -c <cpu_num> -m <mem_size> -- command param1 param2 ...
     ```

  1. Run the following command to get more usage information:
     ```bash
     exorun -h
     ```
  1. To get more information on running jobs with `exorun`:
     ```bash
     exorun run -h
     ```

[Back to Top](#onboarding-guide)

_______

## Upgrading X-Spot Controllers
> **Note:**
>
> You should be :
> | Who? | On which system? | In which directory? |
> | :--- | :--- | :--- |
> | root | X-Spot controller | /nfs-apps/exostellar/onboarding |
>
> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder

To update X-Spot with a new or updated onboarding tarball, we recommend starting the onboarding process from scratch. It is, however, also possible to upgrade "in place." Either way, we'll start with a backup step and move to the upgrade procedures once that's accomplished.

### Backup or Archive
  * We recommend relocating, renaming, or archiving the previous `onboarding` folder to keep the work area tidy, but it is perfectly safe to destroy the previous `onboarding` directory entirely. This is because during the previous installation, the `make deploy` step did take a point-in-time snapshot of the directory.
    1. Start in the `onboarding` directory.
       ```bash
       cd /nfs-apps/exostellar/onboarding
       ```
    1. Create `git` "diff" file so we can easily track any customizations that currently exist.
       ```bash
       git diff > ../custom.diff
       ```
    1. Make an `archive` directory and relocate the `onboarding` folder there.
       ```bash
       mkdir ../archive
       cd ..
       mv onboarding/ archive/onboarding.$( date +%s )
       ```
    1. Download the latest tarball via the following command:
       ```bash
       pushd /tmp
       wget https://rpm.exostellar.io/download/onboarding.tar
       popd /tmp
       ```
    1. Unpack the tarball.
       ```bash
       tar xvf /tmp/onboarding.tar
       ```
    1. We should now have a fresh `onboarding` directory and an archive as follows:
       ```bash
       [root@x-spot-controller]# ls -la
       drwxr-xr-x   3 root  root    96 Jun 29 13:04 archive
       -rw-r--r--   1 root  root  1172 Jun 29 13:04 custom.diff
       drwxr-xr-x   9 root  root   288 Jun 29 13:04 onboarding

       [root@x-spot-controller]# ls -la archive/
       total 0
       drwxr-xr-x  3 root  root   96 Jun 29 13:04 .
       drwxr-xr-x  5 root  root  160 Jun 29 13:21 ..
       drwxr-xr-x  9 root  root  288 Jun 29 13:02 onboarding.1688058267
       ```
    1. Navigate to the new `onboarding` directory.
       ```bash
       cd onboarding
       ```
    1. We will initialize the `git` version-control system again as before:
       - Initialize an empty local repository:
         ```bash
         git init .
         ```
       - Add everything in the directory to the `git` version-control system:
         ```bash
         git add .
         ```
       - "Commit" the current state of the files in and under this directory so we can track changes.
         ```bash
         git commit $( cat version )
         ```
    1. Now, we can bring forward out customizations from the previous working configurations:
       ```bash
       git apply --reject ../custom.diff
       ```
    1. You can inspect the applied changes and verify your customizatioins with another `git diff` now:
       ```bash
       git diff
       ```
    1. You can inspect the applied changes and verify your customizatioins with another `git diff` now:

### Fresh Upgrades
Starting from scratch basically means following the onboarding procedure which you are already familiar with. For this reason, it may appear like the path of least resistance. It is the recommended path for upgrades because it is a well-known path. Provision a new VM to be an X-Spot controller and start from the [X-Spot Worker AMI Preparation step](#x-spot-worker-ami-preparation).

### Upgrading Configuration
It is possible to leverage assets from a previous X-Spot installation to avoid repeating steps. What follows are the steps required to save a little time as compared to the Fresh Upgrades option above.
> **NOTE:**
>
>   This process should not take place if the X-Spot controller is in use. The node should be devoid of any jobs and prevented from accepting jobs while the upgrade takes place. HPC Admins can drain, offline, or otherwise keep the node idle depending on the specifics of the environment.
> **Note:**
>
> You should be :
> | Who? | On which system? | In which directory? |
> | :--- | :--- | :--- |
> | root | X-Spot controller | /nfs-apps/exostellar/onboarding |
>
> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder
  1. Navigate to the updated onboarding directory:
     * The following commands **should not be copy-pasted** because `/nfs-apps` is a generic term or stand-in for your remote directory/mount.
     ```bash
     cd /nfs-apps/exostellar/onboarding
     ```
  1. Establish a new folder for images:
     ```bash
     make image-folder
     ```
  1. Copy existing image files from the current controller:
     ```bash
     make base
     ```
     > **NOTE:**
     >
     >   The upgrade process will necessarily remove docker images from this node. Not only is it assumed the jobs are not running on this node, but also that docker images reside some place safe and that their removal from this node's local disks does not represent the destruction of the only valid copy of any given image.
  1. Build the docker image.
     ```bash
     make build-image
     ```
     - **Note:** If the Dockerfile is customized during onboarding stages for your site, those modifications need to applied to the new Dockerfile:
       * Edit `./images/docker-base/docker_build_path/Dockerfile`
       * ```bash
         make rebuild-image
         ```
  1. Upgrade X-Spot worker
     - Create a new scratch VM as in the [X-Spot Worker Preparation step aboeve](#x-spot-worker-preparation).
     - Update the ami ID in `./scripts/controller/integrate-xspot/config.toml`
  1. Upgrade X-Spot controller
     ```bash
     make upgrade
     ```
  1. From the [Script Deployment step above](#script-deployment), do:
     * ```bash
       make sync-config-preview
       ```
     * ```bash
       make sync-config
       ```
     * ```bash
       make deploy
       ```
  1. Restart the post-docker service.
     ```bash
     systemctl restart post-docker &
     ```
     > **Note:**
     >
     > Running `systemctl restart post-docker` can easily take 15 minutes depending on your environment and the number of images configured.
     * You can monitor the progress of starting the `post-docker` service with the following command:
     ```bash
     journalctl -u post-docker -f
     ```
     * You will know the `post-docker` service has successfully completed when you this following message in the output of the previous command:
     ```bash
     ... post-docker.sh: Building of conatiner docker-base: SUCCESS!
     ... post-docker.sh: Finished.
     ... systemd: Started Post Docker Configuration.
     ```
  1. From [Cluster Deployment step above](#cluster-deployment) you may prefer to create an AMI from the updated X-Spot controller.

### Upgrading Cluster Deployments
Rolling upgrades or swing migrations make a lot of sense in large environments where downtime is to be avoided. The approach is relatively simple, but we'll cover the basic ideas here for your consideration. This is applicable when an environment has multiple or many X-Spot controllers in production.
> **Note:**
>
> You should be :
> | Who? | On which system? | In which directory? |
> | :--- | :--- | :--- |
> | root | X-Spot controller | ${REMOTE_DIR}/scripts/latest/ also known as ${SCRIPT_DEPLOY_DIR}/latest |
>
> **Reminder:** `REMOTE_DIR` and `SCRIPT_DEPLOY_DIR` are set in the `onboarding/conf/env.cfg` file.
  * Because the cloud offers a theoretically unlimited number of instances or VMs, you can set up a new X-Spot controller without taking any out of production. The new X-Spot controller would only be released to production when it's fully validated. At that stage, a previous X-Spot controller could be removed from production.
    - In fact, all the replacement X-Spot controllers can be brought online and —once fully validated— pushed into production before any previous X-Spot controllers are removed.
    - See [Cluster Deployment step above](#cluster-deployment).
  * If it's not possible to provision a new X-Spot controller, then one of the production X-Spot controllers will need to be idled. Once all jobs have cleared off it, upgrading can begin.
    ```bash
    cd ${SCRIPT_DEPLOY_DIR}/latest
    make upgrade
    systemctl restart post-docker &
    ```
    > **Note:**
    >
    > Running `systemctl restart post-docker` can easily take 15 minutes depending on your environment and the number of images configured.
    - You can monitor the progress of starting the `post-docker` service with the following command:
    ```bash
    journalctl -u post-docker -f
    ```
    - You will know the `post-docker` service has successfully completed when you this following message in the output of the previous command:
    ```bash
    ... post-docker.sh: Building of conatiner docker-base: SUCCESS!
    ... post-docker.sh: Finished.
    ... systemd: Started Post Docker Configuration.
    ```
    - In a similar fashion, each subsequent X-Spot controller marked for upgrade can be removed from production, one at a time or in mulitples at a time, while still preserving a limited production capacity of X-Spot controllers while the others are upgraded.
  * Regardless of your approach, (serial replacements or parallel replacements), it would be considered best practice to move through the upgrade process on a single X-Spot controller and fully validate it before moving on to address upgrades in the rest of the fleet of X-Spot controllers.
  * As noted above, and depending on the scale of the upgrade, [creating an AMI](#cluster-deployment) may be an efficient way forward.

[Back to Top](#onboarding-guide)

_______

# AGE and qsub

- [X-Spot Integration for Altair Grid Engine](#age-qsub)
  - [1. Prerequisites](#1-prerequisites)
  - [2. AGE Configuration Modifications](#2-age-configuration-modifications)
  - [3. qsub Wrapper Configuration](#3-qsub-wrapper-configuration)
- [X-Spot Integration via JSV for Altair Grid Engine](#age-jsv)

___

## 1. Prerequisites

  1. A fully functional Altair Grid Engine installation is required.
  1. Sufficient privileges to modify the Grid Engine configuration will be required.

___

## 2. AGE Configuration Modifications

  1. Assumptions about AGE configuration as a key to steps below:
     - The Master Host is represented below as `head`.
     - A single Execution Host is referenced below as `n011`, and it is a stand-in for all X-Spot controllers that will be participate in the Altair Grid Engine configuration.
  1. Add queue `xspot`.
     - difference between deafult all.q and xspot:
       * qname
         - `xspot` instead of `all.q`
       * hostlist
         - `n011` instead of `@allhosts`
       * slots
         - `1,[n011=50]" instead of `1,[n011=<num_cpus>],[head=2]"
         > **NOTE:**
         >
         >   Some testing is recommended for your workflows. A single X-Spot controller should be limited to 80 slots. This is largely independent of CPU CORES and MEMORY.
  1. Validation via the following commands:
     ```bash
     cd ${SGE_ROOT}/${SGE_CELL}
     diff ./spool/qmaster/cqueues/{all.q,xspot}
     ```

[Back to Top](#age-qsub)

___

## 3. qsub Wrapper Configuration

As a proof of concept, a relatively simple wrapper can be placed alongside `qsub` in the `${SGE_ROOT}/bin/${ARCH}` directory. The wrapper will inspect job commands given to `qsub`, watching for requests for the `xspot` queue. If none are found, the wrapper gets out of the way and passes everything to the original `qsub` which could be renamed `qsub.orig`. If a request for the `xspot` queue is discovered by the wrapper, it will intercept any job parameters that need modification for a successful X-Spot job launch and make the required changes before passing the rest of the job parameters to the original `qsub`. The AGE Administrator is encouraged to adopt a more robust strategy, e.g. leveraging AGE's JSV framework. Exostellar can deliver the requirements for successful X-Spot job launch.

  1. The following global variables should be set on the top of the wrapper script:
     1. The wrapper script should be found here:
        ```bash
        ${SCRIPTS_DEPLOY_DIR}/scripts/latest/tools/sched-wrapper/qsub.py
        ```
	 1. Name of the docker container image running the workload.
        ```bash
        DOCKER_IMAGE = "<your_docker_image>"
        ```
     1. Full path of the `exorun` script used to schedule the jobs on an xspot controller.
        * The default location is `/usr/bin/exorun`.
        ```bash
        XSPOT_RUN = "/usr/bin/exorun"
        ```
     1. Full path of the original qsub command on the system.
        * The default location is `${SGE_ROOT}/bin/${ARCH}/qsub`.
        ```bash
        REAL_EXE = "/path/to/real/qsub"
        ```
     1. Default amount of memory (in GB) used by the container on an xspot worker node when not provided on the command line.
        ```bash
        DEFAULT_MEM = 1
        ```
        * Otherwise, this is specified on the `qsub` commandline with `-l`. For example:
          ```bash
          -l mem_free=1024M
          ```
     1. Default number of cpus used by the container on an xspot worker node when not provided on the command line.
        ```bash
        DEFAULT_CPU = 1
        ```
        * Otherwise, this is specified on the `qsub` commandline with `-pe`. For example:
        ```bash
        -pe mpi 1
        ```
  1. Set up the wrapper and default installation binaries for cooperation:
     1. Rename the original `qsub` binary:
        ```bash
        cd ${SGE_ROOT}/bin/${ARCH}
        mv qsub qsub.orig
        ```
     1. Link (or copy) the wrapper in the `${SGE_ROOT}/bin/${ARCH}` directory:
        ```bash
        ln -s ${SCRIPTS_DEPLOY_DIR}/scripts/latest/tools/sched-wrapper/qsub.py ./qsub
        ```
  1. Invocation examples:
	 - Example #1 - 1 CPU, 1G
       ```bash
       qsub -q xspot -l mem_free=1024M qsub_job.sh
       ```
       * Actual job submitted:
         ```bash
         qsub.real -q xspot -pe mpi 1 -l mem_free=1G /usr/bin/exorun -i <your_docker_image> -c 1 -m 1 -- qsub_job.sh
         ```
	 - Example #2 - 1 CPU, rounds 2.5G to 3G
       ```bash
       qsub -q xspot -l mem_free=2.5G qsub_job.sh
       ```
       * Actual job submitted:
         ```bash
         qsub.real -q xspot -pe mpi 1 -l mem_free=1G /usr/bin/exorun -i <your_docker_image> -c 1 -m 3 -- qsub_job.sh
         ```
	 - Example #3 - 2 CPUs, rounds 2.5G to 3G
       ```bash
       qsub -q xspot -l mem_free=2.5G -pe mpi 2 qsub_job.sh
       ```
       * Actual job submitted:
         ```bash
         qsub.real -q xspot -pe mpi 1 -l mem_free=1G /usr/bin/exorun -i <your_docker_image> -c 2 -m 3 -- qsub_job.sh
         ```

[Back to Top](#age-qsub)

# AGE JSV

## Configuring jsv_xspot for AGE

This document covers the configuration of the AGE job scheduler to rewrite submitted jobs to execute on a controller node running X-spot.
### 1. Replacing the qsub command line tool
In order to support jobs submitted from inside a container over `ssh`, we need to install `sub_wrapper.py` as a replacement for the `qsub` binary. 
Assuming that `sub_wrapper.py` is installed in `$AGE_TOP/bin/lx-amd64` :

```
mv $AGE_TOP/bin/lx-amd64/qsub $AGE_TOP/bin/lx-amd64/qsub.orig
ln -s $AGE_TOP/bin/lx-amd64/sub_wrapper.py $AGE_TOP/bin/lx-amd64/qsub
```

Modify the path to the original `qsub` binary in `$AGE_TOP/bin/lx-amd64/qsub`:
Example:

```
### BEGIN CONFIGURATION #######################################################

real_sub = "/fsx/ontap/exo-opt/age_0/bin/lx-amd64/qsub.orig"
```

### 2. Installing jsv_xspot
Copy `jsv_xspot.py` to `$AGE_TOP/util/resources/jsv` and modify the queue, container image and memory configuration as needed:

```
### BEGIN CONFIGURATION #######################################################

queue_cfg = { "xspot" : "/usr/bin/exorun run" }

DOCKER_IMAGE = "" # image name here

controller_mem = 50 # in MB
controller_cpu = 1
container_mem = 1024 # in MB
container_cpu = 1
```

### 3. Configuring AGE
To enable `jsv` and to allow it to rewrite the details of the job submitted, run `qconf -mconf` and look for `jsv_url` in the configuration.
Add the path to `$AGE_TOP/util/resources/jsv/jsv_xspot.py`
Example:

```
jsv_url                      /fsx/ontap/exo-opt/age_0/util/resources/jsv/jsv_xspot.py
```

Save the file and `qsub` will start validating jobs submitted using `jsv_xspot.py`. 
Based on the configuration above, only jobs submitted to the `xspot` queue will be rewritten to use X-spot.

### 4. Testing
To verify that all the steps above were successful, a simple job can be submitted:

```
qsub -q xspot -l mem_free=2.5 simple.sh
```

Sample output:

```
controller_ip = None
NO stdin
NO stdin(sub_cmd)
Your job 141 ("JOB-NAME") has been submitted
```

The output indicates if the job is going over `ssh` (`controller_ip != None`) and if the job was submitted via stdin.

Job details output using `qstat -j 141`:

```
...
hard_resource_list:         mem_free=50M
...
job_args:                   run,-c,1,-m,3,--,simple.sh
script_file:                /usr/bin/exorun
...
```

In the example above, our job was rewritten to pass the resources requested to `/usr/bin/exorun` and the job on the controller is using 50M.
 
[Back to Top](#onboarding-guide)

[Back to Appendix](#appendix)

_______

## SLURM and sbatch

## Configuring CLI plug-in for Slurm

This document covers the configuration of the Slurm job scheduler to rewrite submitted jobs to execute on a controller node running X-spot.
### 1. Replacing the sbatch command line tool
In order to support jobs submitted from inside a container over `ssh`, we need to install `sub_wrapper.py` as a replacement for the `sbatch binary`. 
Assuming that `sub_wrapper.py` is installed in `$SLURM_TOP/sub_wrapper.py` :

```
mv $SLURM_TOP/sbatch $SLURM_TOP/sbatch.orig
ln -s $SLURM_TOP/sub_wrapper.py $SLURM_TOP/sbatch
```
Modify the path to the original `sbatch` binary in `$SLURM_TOP/sub_wrapper.py`:
Example:

```
### BEGIN CONFIGURATION #######################################################

real_sub = "/fsx/ontap/exo-opt/slurm_0/sbatch.orig"

real_job = "/usr/bin/exorun run"
```
The `real_job` parameter is the path to the job CLI wrapper that will execute on the controller.
### 2. Installing the CLI plug-in
Copy `cli_filter_xspot.so` to `$SLURM_TOP/cli_filter_xspot.so` so the plug-in is visible from all nodes. 
Depending on your configuration, the Slurm libraries may be located elsewhere and you will need to create a symbolic link to the shared location in `$SLURM_TOP/cli_filter_xspot.so`.
Example:

```
ln -s $SLURM_TOP/cli_filter_xspot.so /usr/lib64/slurm/cli_filter_xspot.so
```
### 3. Configuring multiple queues
By default, the plug-in recognizes two partition names, namely `xspot` and `chipspot`. The controller node expects our container wrapper to be installed in `/usr/bin/exorun`.

### 4. Configuring Slurm to use the plug-in
To enable the plug-in, edit `/etc/slurm/slurm.conf` (location may vary) with the following:

```
CliFilterPlugins=cli_filter/xspot
```
### 5. Testing
To verify that all the steps above were successful, a simple job can be submitted. Assuming `simple.sh` contains this:

```
#!/bin/bash
#SBATCH -o %j.out
#SBATCH -p xspot

for i in {0..99}; do
    echo $i
done
```
Job submittal:

```
sbatch --ntasks=2 --mem=3G /fsx/ontap/work/user/simple.sh
```
Sample output:

```
controller_ip = None
NO stdin
('script args:', ['/fsx/ontap/work/user/simple.sh'])
('shared_dir=', '/home/user/.xspot/head')
pre_submit: pn_min_memory = 3072, ntasks = 2
modified: partition = xspot, mem = 3G, cpu = 2
Submitted batch job 15030
```
Under `script args`, we can see the script and its arguments.
Under `shared_dir`, the location of a temporary directory used to rewrite the script contents to use X-spot. Files appear in that directory but are deleted immediately after `sbatch` is called.
Under `modified`, we can see the resources requested to create the container (3G/2CPUs).

The output also indicates if the job is going over `ssh` (`controller_ip != None`) and if the job was submitted via `stdin`.

[Back to Top](#onboarding-guide)

[Back to Appendix](#appendix)
 
_______

# LSF and bsub

- [X-Spot Integration for IBM Load Sharing Facility](#lsf-bsub)
  - [1. Prerequisites](#1-prerequisites)
  - [2. LSF Configuration Modifications](#2-lsf-configuration-modifications)
  - [3. bsub Wrapper Configuration](#3-bsub-wrapper-configuration)
- [X-Spot Integration via ESUB for IBM Load Sharing Facility](#lsf-esub)

___

## 1. Prerequisites

  1. A fully functional IBM Load Sharing Facility installation is required.
  1. Sufficient privileges to modify the LSF configuration will be required.

___

## 2. LSF Configuration Modifications

  1. If the LSF installation is not already allowing job-submission from any host, then LSF configuration changes are required when jobs are expected to submit other jobs during runtime.
     - The following modification allows for the hosts that have access to the ${LSF_ENVDIR} and ${LSF_BINDIR} directories ( the LSF installation and configuration directories ) to submit jobs. If this is not set, any client hosts need to be statically pre-configured.
       * Open the config for editing from:
         ```bash
         ${LSF_ENVDIR}/lsf.cluster.<your_cluster_name>
         ```
       * Add the following new section:
         ```bash
         Begin Parameters
         LSF_HOST_ADDR_RANGE=*.*.*.*
         FLOAT_CLIENTS_ADDR_RANGE=*.*.*.*
         FLOAT_CLIENTS=10
         End Parameters
         ```
       * The range above indicates all hosts, but limiting to your subnet (e.g.: 172.31.*.*) should be valid as well.
       * `FLOAT_CLIENTS` likely needs to be augmented for as many hosts as needed:
         - This number could be in the thousands.
         - Licensing may be an issue here, but 100, 200, or 300 are sane defaults.
         - If licensing erros are encountered, dropping back down to 10 is recommended for troubleshooting.
       * Restarting fully all LSF subsystems is recommended for this global change (depicted above).
  1.  Additionally, host name resolution can be a complicating factor as most HPC systems rely on this type of resolution as a matter of routine. DNS configuration may need to account for this. AWS default DNS in a subnet generally allows for hostname resolution by default, but if you’ve implemented your own DNS, you may need to consider adding an `${LSF_ENVDIR}/hosts` file. The hostnames in AWS are generally predictable and in the format of "ip-192-168-0-1" for a host whose IP Address is 192.168.0.1. Pre-populating the `${LSF_ENVDIR}/hosts` file with all the IP Addresses and hostnames in the subnet will facilitate job submissions from X-Spot worker nodes that may attempt to submit "sub jobs" back to the scheduler.
NOTE: Please avoid defining the existing cluster nodes (X-Spot Controllers) with the generic "ip-192-168-0-1" hostname in this file which might conflict with the cluster configuration, or in other words, look out for duplicate IP Addresses in this file.
  1. A host "Rosetta Stone" for clarity:
      - | LSF terminology | typical functionality | generic terminology |
        | :--- | :---: | ---: |
        | Master Host | Scheduling and Job Submissions | Head Node |
        | Server Host | Job Execution | Compute Node |
        | Client Host | Job Submissions | Login Node |
      - Master hosts are typically configured as client hosts.
      - Server hosts are typically configured as client hosts.
  1. Assumptions about LSF configuration as a key to steps below:
     - The Master Host is represented below as `head`.
     - A single Server Host is referenced below as `n011`, and it is a stand-in for all X-Spot controllers that will be participate in the IBM Load Sharing Facility configuration.
  1. Add queue `xspot`.
     > **NOTE:**
     >
     >   Some testing is recommended for your workflows. A single X-Spot controller should be limited to 80 for its MaxJobs setting. This is largely independent of CPU CORES and MEMORY.

[Back to Top](#lsf-bsub)

___

## 3. bsub Wrapper Configuration

As a proof of concept, a relatively simple wrapper can be placed alongside `bsub` in the `${LSF_BINDIR}` directory. The wrapper will inspect job commands given to `bsub`, watching for requests for the `xspot` queue. If none are found, the wrapper gets out of the way and passes everything to the original `bsub` which could be renamed `bsub.orig`. If a request for the `xspot` queue is discovered by the wrapper, it will intercept any job parameters that need modification for a successful X-Spot job launch and make the required changes before passing the rest of the job parameters to the original `bsub`. The LSF Administrator is encouraged to adopt a more robust strategy, e.g. leveraging LSF's `esub` framework. Exostellar can deliver the requirements for successful X-Spot job launch.

  1. The following global variables should be set on the top of the wrapper script:
     1. The wrapper script should be found here:
        ```bash
        ${SCRIPTS_DEPLOY_DIR}/scripts/latest/tools/sched-wrapper/bsub.py
        ```
	 1. Name of the docker container image running the workload.
        ```bash
        DOCKER_IMAGE = "<your_docker_image>"
        ```
     1. Full path of the `exorun` script used to schedule the jobs on an xspot controller.
        * The default location is `/usr/bin/exorun`.
        ```bash
        XSPOT_RUN = "/usr/bin/exorun"
        ```
     1. Full path of the original bsub command on the system.
        * The default location is `${LSF_BINDIR}/bsub`.
        ```bash
        REAL_EXE = "/path/to/real/bsub"
        ```
     1. Default amount of memory (in GB) used by the container on an xspot worker node when not provided on the command line.
        ```bash
        DEFAULT_MEM = 1
         ```
        * Otherwise, this is specified on the `bsub` commandline with `-R`. For example:
          ```bash
          -R rusage[mem=2.5G]
          ```
     1. Default number of cpus used by the container on an xspot worker node when not provided on the command line.
        ```bash
        DEFAULT_CPU = 1
        ```
        * Otherwise, this is specified on the `bsub` commandline with `-n`. For example:
        ```bash
        -n 5
       ```
  1. Set up the wrapper and default installation binaries for cooperation:
     1. Rename the original `bsub` binary:
        ```bash
        cd ${LSF_BINDIR}
        mv bsub bsub.orig
        ```
     1. Link (or copy) the wrapper in the `${LSF_BINDIR}` directory.
        ```bash
        ln -s ${SCRIPTS_DEPLOY_DIR}/scripts/latest/tools/sched-wrapper/bsub.py ./bsub
        ```
  1. Invocation examples:
	 - Example #1 - 2 CPUs, Default to 1024 MB
       ```bash
       bsub -q xspot -n 2 lsf-job.sh
       ```
       * Actual job submitted:
         ```bash
         bsub.real -q xspot -n 1 -R rusage[mem=1024] /usr/bin/exorun -i <your_docker_image> -c 2 -m 1 -- lsf-job.sh
         ```
	 - Example #2 - 5 CPUs, rounds 2.5G to 3G
       ```bash
       bsub -q xspot -n 5 -R rusage[mem=2.5G] -R span[hosts=1] lsf-job.sh
       ```
       * Actual job submitted:
         ```bash
         bsub -q xspot -n 1 -R rusage[mem=1024] -R span[hosts=1] /usr/bin/exorun -i <your_docker_image> -c 5 -m 3 -- lsf-job.sh
         ```
	 - Example #3 - 5 CPUs, rounds 2049M to 3G
       ```bash
       bsub -q xspot -n 5 -R rusage[mem=2049] lsf-job.sh
       ```
       * Actual job submitted:
         ```bash
         bsub.real -q xspot -n 1 -R rusage[mem=1024] /usr/bin/exorun -i <your_docker_image> -c 5 -m 3 -- lsf-job.sh
         ```

[Back to Top](#lsf-bsub)


# LSF ESUB

## Configuring esub.xspot for LSF

This document covers the configuration of the LSF job scheduler to rewrite submitted jobs to execute on a controller node running X-spot.
### 1. Replacing the bsub command line tool
In order to support jobs submitted from inside a container over `ssh`, we need to install `sub_wrapper.py` as a replacement for the `bsub` binary. 

Assuming that `sub_wrapper.py` is installed in `$LSF_TOP/10.1/linux2.6-glibc2.3-x86_64/bin` :


```
mv $LSF_TOP/10.1/linux2.6-glibc2.3-x86_64/bin/bsub $LSF_TOP/10.1/linux2.6-glibc2.3-x86_64/bin/bsub.orig
ln -s $LSF_TOP/10.1/linux2.6-glibc2.3-x86_64/bin/sub_wrapper.py $LSF_TOP/10.1/linux2.6-glibc2.3-x86_64/bin/bsub
```
Modify the path to the original `bsub` binary in `$LSF_TOP/10.1/linux2.6-glibc2.3-x86_64/bin/bsub`:

Example:

```
### BEGIN CONFIGURATION #######################################################

real_sub = "/fsx/ontap/exo-opt/lsf_0/10.1/linux2.6-glibc2.3-x86_64/bin/bsub.orig"
```
### 2. Installing esub.xspot
Copy `esub.xspot` to `$LSF_TOP/10.1/linux2.6-glibc2.3-x86_64/etc/esub.xspot` and modify the queue, container image and memory configuration as needed:

```
### BEGIN CONFIGURATION #######################################################

queue_cfg = { "xspot" : "/usr/bin/exorun run" }

DOCKER_IMAGE = "" # image name here

controller_mem = 50 # in MB
controller_cpu = 1
container_mem = 1024 # in MB
container_cpu = 1
```

### 3. Configuring an existing esub wrapper (Optional)
In some cases, LSF will be already configured to validate jobs with an `esub` wrapper.
Assuming we are using `$LSF_TOP/10.1/linux2.6-glibc2.3-x86_64/etc/esub.Synopsys`, we need to chain our `esub.xspot` wrapper.
Example:

```
#!/usr/bin/perl
#

print "Inside Synopsys esub...\n";

# ...
# Synopsys code
# ...
#

my $ret = system("/fsx/ontap/exo-opt/lsf_0/10.1/linux2.6-glibc2.3-x86_64/etc/esub.xspot")

# Exostellar modifications to LSB_SUB_MODIFY_FILE (in esub.xspot)

# check 'esub.xspot' ret
```
### 4. Configuring LSF
To enable `esub` and to allow it to rewrite the details of the job submitted, edit `$LSF_TOP/conf/lsf.conf` with the following:

```
LSB_ESUB_METHOD=Synopsys
LSB_SUB_COMMANDNAME=Y
```
Note that the example above reflects an optional `esub.Synopsys`. To use `esub.xspot` directly, edit `$LSF_TOP/conf/lsf.conf` with the following:

```
LSB_ESUB_METHOD=xspot
LSB_SUB_COMMANDNAME=Y
```
### 5. Testing
To verify that all the steps above were successful, a simple job can be submitted:

```
bsub -q xspot -n 2 -R rusage[mem=2.5GB] -o %J.out -e %J.err sleep 60
```
Sample output:

```
controller_ip = None
NO stdin
NO stdin(sub_cmd)
XSPOT_CONTROLLER_IP : None
modified >>>  {'LSB_SUB_QUEUE': '"xspot"', 'LSB_SUB_MEM_USAGE': 50, 'LSB_SUB_COMMANDNAME': '"/usr/bin/exorun.py"', 'LSB_SUB_RES_REQ': '"rusage[mem=50M]"', 'LSB_SUB_OUT_FILE': '"%J.out"', 'LSB_SUB_ERR_FILE': '"%J.err"', 'LSB_SUB_MAX_NUM_PROCESSORS': 1, 'LSB_SUB_NUM_TASKS': 1, 'LSB_SUB_NUM_PROCESSORS': 1, 'LSB_SUB_MAX_NUM_TASKS': 1, 'LSB_SUB_COMMAND_LINE': '"/usr/bin/exorun.py -c 2 -m 3 -- \'"sleep 60"\'"'}
Job <300> is submitted to queue <xspot>.
```
In the modified section, we can see that the job on the controller is using 50M/1CPU but the actual resources requested are used to create the container (3G/2CPUs).

The output also indicates if the job is going over `ssh` (`controller_ip != None`) and if the job was submitted via stdin.
Sample output using `ssh`:

```
controller_ip = 10.0.142.38
*** BEGIN ssh_stdin
...
***  END  ssh_stdin
NO stdin(ssh)
Warning: Permanently added '10.0.142.38' (ECDSA) to the list of known hosts.
modified >>>  {'LSB_SUB_QUEUE': '"xspot"', 'LSB_SUB_MEM_USAGE': 50, 'LSB_SUB_COMMANDNAME': '"/usr/bin/exorun.py"', 'LSB_SUB_RES_REQ': '"rusage[mem=50M]"', 'LSB_SUB_OUT_FILE': '"%J.out"', 'LSB_SUB_ERR_FILE': '"%J.err"', 'LSB_SUB_MAX_NUM_PROCESSORS': 1, 'LSB_SUB_NUM_TASKS': 1, 'LSB_SUB_NUM_PROCESSORS': 1, 'LSB_SUB_MAX_NUM_TASKS': 1, 'LSB_SUB_COMMAND_LINE': '"/usr/bin/exorun.py -c 4 -m 5 -- \'"/home/eric/simple.sh"\'"'}
Job <302> is submitted to queue <xspot>.
```

[Back to Top](#onboarding-guide)

[Back to Appendix](#appendix)


_______

# Cost Calculation
## Python script: cost_calculation.py

This is a python script which will generate the excel or json output with the Summary and Cost data as well as breaking down the worker/container/controller data.

It is a command line tool which parses the generated log files to calculate the actual cost and estimate cost savings based on on-demand pricing.

The script parses the data and collects information on the controller, workers and containers, then creates an output file. There are three types of output formats it generates, depending on what modules the environment has loaded or is willing to load.  There is an excel worksheet format, an html format and a json format.

1.	Python set up:

	The following python modules are necessary to run this script.

	Mandatory:

	- python3:
		- yum install python3
	- dateutil : module needed for handling timezones properly
		- python3 –m pip install python-dateutil

	Optional depending on output types:

	- To generate excel worksheet output
		- Xlsxwriter: 	pip3 install xlsxwriter
		- pandas: 	pip3 install pandas
	- To generate html format output
		- Json2html: 	pip3 install json2html

	python 3 or python 3.8 may be used

2.	The usage is as follows:

	Run a script on log files to calculate cost and estimate savings

    Options:

	**-l, --logs          	Location of log files to parse: must be directory   ex: logs/  (mandatory)**

	-s, --start        	 Start day  (in date format: 2022-03-17), (default to 30 days ago)

	-e, --end           	End day  (in date format: 2022-03-17), (default end of current day)

	-n, --num_days      	Number of days: start with today or number of days since, and end with end of
				day today (overrides any -s or -e entry) (default is this parameter is not used)

	-d, --discount      	Percent discount of on-demand instances to use for analysis, (default = 0%)

	-o, --output        	Enter type of output format: json, html, xlsx  (default is all 3 if modules are 							installed)

	-c, --container     	Enter container ID - will display only information associated with this container 							ID, (default = '', full output generated)

	-f, --containerLife 	Enter container ID - will display only information associated with this container ID in a 							'Lifetime' format, (default = '', full output generated)

   	-j, --jobid	 	Enter job ID - will display only information associated with this job ID in a 									'Lifetime' format, (default = '', full output generated)


Examples:

The parameter –l (--logs) is mandatory:  the log file path

python3.8 \<path\>cost_calculation.py -l \<log file path\>
```
python3.8 /var/log/cost_calculation.py -l /var/log/xspot/latest
```

Optional: to specify start date:
```
python3.8 <path>cost_calculation.py -l <log file path> -s 2023-03-17
```

Optional: to specify on xlsx output:
```
python3 cost_calculation.py -l <log file path> -s 2023-03-31 –o xlsx
```


## Cost Calculation Script Error Reference

| Installation errors:  | *specific packages are not available* |
| ----------- | ----------- |
| Python version must be 3 or greater   |   |
| No *\<module\>* found  | Timezone from datetime  |
|   | Json2html |
|   | Xlsxwrite |
|   | Pandas |

| Usage errors:  |  |
| ----------- | ----------- |
| *option -x not recognized* <br>   *\<Usage text should follow\>* | An incorrect option flag was entered |
| *Entry error:  -l: location of log files parameter is mandatory* | The –l (--logs) option is missing <br> This is a mandatory option |
| *Incorrect entry: --logs must be a directory* | The logs path must be a directory |
| *Incorrect entry: Start time (in 2022-03-17 format)* | The incorrect date format was used <br> Ex: -s 2023-04-01 <br> Default: the last 30 days are used |
| *Incorrect entry: End time (in 2022-03-17 format)* | The incorrect date format was used <br> Ex: -e 2023-04-01 <br> Default: end time is end of current day |
| *Incorrect Entry: --number of days must be integer only* | The value for the day range must be an integer <br>  Ex:  -n 10 <br> Date range will be the last 10 days <br> Default : it is not used  |
| *Incorrect entry: output file must be json, or html, or xlsx* | The output file type is entered incorrectly <br> Ex: -o xlsx <br> Default: all files are generated |
| *Incorrect Entry: --discount must be integer only* | The value for the discount must be an integer and is the percent discount (30% => -d 30) <br> Default is 0% |

| Log file parsing (time) errors:  |  |
| ----------- | ----------- |
| *No valid log files specified:* | Time stamp not found in first 5 and last 5 lines of file |
|   | Corrupt log file  |
|   | No write access to location of xspot log file(s)  |


| General errors:  |  |
| ----------- | ----------- |
| *Invalid instance type:* | ‘FinalizeWorkerMetadata’ not found – resulting in no instance type found |
| *json not matched:* | Terminating Worker Instance :  <br> misformed line in log file – no json data |
| *Error in file: No EndTime found for Controller endtime (or lastcount)* | Error while searching for controller end time – which is the date stamp in a line at the end of the file |
| *exception2:  list index out of range* | Controller data incorrect : endtime |
| *no instance type for:  \<worker number\>* | The instance type to be used for cost was unable to be found from worker log data |
| *Pricing file not found:<br>/etc/xspot/aws-pricing/.json* | The DefaultRegion was not found :  <br> which is in the Scheduler configurations line |
| *Pricing file not found:  /etc/xspot/aws-pricing/\<region\>.json* | The pricing files are not accessible or were not created |
| *Invalid instance type: \<instance type\>* | The pricing file was not found, or does not have the matching instance type |


| Running for one container only data (-c option):  |  |
| ----------- | ----------- |
| *The Container ID entered is not unique.* | More characters in the container ID string are needed |
| *The Container ID entered was not found.* | The entered Container ID did not match any in the log file |

[Back to Top](#onboarding-guide)

[Back to Appendix](#appendix)

_______

# s3sync-tool

## Syncing a Log Directory to an S3 Bucket using AWS CLI v2 and Cron

In this guide, we will walk through the process of setting up a script to periodically sync a local log directory to an S3 bucket using the AWS CLI v2 and a cron job.

### Prerequisites
- AWS CLI v2 installed on your system. You can find the installation instructions [here](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- A S3 bucket already created in the target AWS account.
- The source folder on your local system that you want to sync with the S3 bucket.
- The instance running the script should have the IAM permissions in its IAM role: `s3:ListBucket` and `s3:PutObject`.

### Setting up the Script
Open the `env.cfg` file and edit the following environment variables to match your setup:
- `S3_BUCKET`: the name of the destination S3 bucket.
- `SOURCE_FOLDER`: the path of the log folder on your local system that you want to sync with the S3 bucket.
- `CRON_FREQUENCY`: the frequency in minutes at which you want to run the sync job. The default is 60 minutes.

### Running the Script
Run the script with the following command:
```bash
./set_cron_job.sh
```
The script will add a cron job to run the sync periodically based on the frequency specified in the environment variable.


### Monitoring the Script
The job is not added by `crontab` hence it will not show up in the output of `crontab -l` command. To check the status of the cron job, you can use the command `ls -l /etc/cron.d/` to verify that the file is present and has the correct permissions.

You can also check the cron log to make sure `sync_log.sh` script is executed with the specified frequency.

[Back to Top](#onboarding-guide)

[Back to Appendix](#appendix)
