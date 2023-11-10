# Compute Optimizer Worker AMI Preparation

> **NOTE:**
>
> A disposable or scratch [AWS Nitro-enabled](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-types.html#ec2-nitro-instances) VM will be required. The required VM will be discarded after use. For this short duration-task, we **recommend a c5.xlarge or m5.xlarge.** Using t2.micro, for example, will result in errors.
>
> This step can be done in parallel with the following section, if you'd like to save time.
>
> For example, during [Containerization step](containerization.md) after issuing the `make base` command might be a good time come back to this step while that command runs.

1.  Create the worker AMI.

    * Create a new CentOS-7 VM in the AWS Console.
    * **Note:** The AWS Console provides information for connecting the the new VM but this information contains an error:
      * If creating an CentOS VM, the user who needs to connect is `centos` not `ec2-user` as reported via the AWS Console.

    > **Note:**
    >
    > You should be :

    | Who? | On which system?          | In which directory? |
    | ---- | ------------------------- | ------------------- |
    | root | Compute Optimizer worker AMI Builder | /tmp                |

    * Ensure the onboarding folder is available on this new VM:
      * Copy the onboarding assets to `/tmp`
      * Mount the `/nfs-apps` folder containing the onboarding assets on the new VM and navigate to it.
    * If Security Enhanced Linux (SELinux) is enabled on this VM/EC2 Instance, an error will be encountered.
      * Amazon Linux 2 instances generally have SELinux disabled by default.
      * Disable selinux via the following commands if necessary:
      * ```
        setenforce 0
        ```
      * ```
        sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
        ```
    *   Run the following:

        ```
        make install-xspot-worker
        ```

        * **Note:** Running `make install-xspot-worker` can take about 5 or 10 minutes to run to completion.
        * This installs everything required for xspot-workers.
    *   Reboot the newly created CentOS-7 VM and issue the following command:

        ```
        xl info
        ```

        * That's XL INFO all in lowercase and many lines of output should be generated.
        * If instead an error message is thrown, [contact support](mailto:support@exostellar.io) for assistance:
    * Create a new AMI from the VM.
      * To create a new AMI, go to the AWS Console.
      * Navigate to EC2 Services and select the Instance ID of the disposable VM.
      * In the upper right corner, select 'Actions' > 'images and templates' from the drop down.
      *   Click on 'Create image'

          * A meaningful name will be required.

          > **NOTE:**
          >
          > It is recommended to tag this Compute Optimizer Release Version as found in the `onboarding/conf/env.cfg` file so that compatability can be easily tracked and compared in the future.
      * Upon completion, make note of the AMI ID, which will be required in the [Compute Optimizer Integration step](compute-optimizer-integration.md).
      * **Note:** It can take a few minutes for the AMI ID to display as `available` in the AWS Console.
