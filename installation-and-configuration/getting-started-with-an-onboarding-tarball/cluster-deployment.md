# Cluster Deployment

> **Note:**
>
> You should be :

| Who? | On which system?            | In which directory?      |
| ---- | --------------------------- | ------------------------ |
| root | Cluster Head or Master node | Depends on HPC Scheduler |

1. Create a new "xspot" queue in the cluster's scheduler configuration. This new queue or partition is meant to be comprised soley of X-Spot controllers.
2. There are a few ways of bringing up X-Spot controllers automatically:
   * You may prefer to create an AMI from a fully validated X-Spot controller, and use that for new controller VMs.
     * Create a new AMI from the VM.
       * To create a new AMI, go to the AWS Console.
       * Navigate to EC2 Services and select the Instance ID of an X-Spot controller.
       * In the upper right corner, select 'Actions' > 'images and templates' from the drop down.
       * Click on 'Create image'
       * A meaningful name will be required.
       * Upon completion, make note of the AMI ID, which can expedite deployments of future X-Spot controllers for the site.
   *   You can run these commands by hand on an X-Spot controller that will persist beyond reboots.

       > **Note:**
       >
       > You should be :

       | Who? | On which system?  | In which directory?                                                        |
       | ---- | ----------------- | -------------------------------------------------------------------------- |
       | root | X-Spot controller | ${REMOTE\_DIR}/scripts/latest/ also known as ${SCRIPT\_DEPLOY\_DIR}/latest |

       > **Reminder:** `REMOTE_DIR` and `SCRIPT_DEPLOY_DIR` are set in the `onboarding/conf/env.cfg` file.

       *   ```
           cd ${SCRIPT_DEPLOY_DIR}/latest
           make controller
           systemctl start post-docker &
           ```

           > **Note:**
           >
           > Running `systemctl start post-docker` can easily take 15 minutes depending on your environment and the number of images configured.
       *   You can monitor the progress of starting the `post-docker` service with the following command:

           ```
           journalctl -u post-docker -f
           ```
       *   You will know the `post-docker` service has successfully completed when you this following message in the output of the previous command:

           ```
           ... post-docker.sh: Building of conatiner docker-base: SUCCESS!
           ... post-docker.sh: Finished.
           ... systemd: Started Post Docker Configuration.
           ```
   * Similar to the previous option, you can add this logic to X-Spot controller bootstrapping in auto-scaling environments.
     *   Run the following script when the VM boots:

         ```
         cd ${SCRIPT_DEPLOY_DIR}/latest
         make controller
         systemctl start post-docker
         ```

         * **Note:** Running `systemctl start post-docker` can easily take 15 minutes depending on your environment and the number of images configured.

> **IMPORTANT:**
>
> The IAM role from the [prerequisites step](https://github.com/Exostellar-Labs/docs#prerequisites) will need to be attached to **every X-Spot controller**.
