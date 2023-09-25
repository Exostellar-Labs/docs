# X-Spot Validation

### X-Spot Validation as Root <a href="#user-content-x-spot-validation-as-root" id="user-content-x-spot-validation-as-root"></a>

> **Note:**
>
> You should be :

| Who? | On which system?  | In which directory?             |
| ---- | ----------------- | ------------------------------- |
| root | X-Spot controller | /nfs-apps/exostellar/onboarding |

> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder

Validate X-Spot installation and docker images.

1. It's often preferred to leverage a terminal multiplexer such as GNU's `screen` or `tmux` or simply to open multiple shells on the node. Some windows can be used to monitor progress while others can be used to run commands associated with validation steps.
2.  Run the following command:

    ```
    xspot check -f
    ```

    * This command should run with no errors.
3.  To faciliate testing on the X-Spot controller, toggle the X-Spot scheduler off via:

    ```
    xspot scheduler off
    ```
4.  Further faciliate testing by requesting an ondemand instance from the X-Spot controller via:

    ```
    xspot add -c 4 -m 8
    ```

    * **Note:** It will probably take 2-4 minutes for this command to return.

    > **NOTE:**
    >
    > If 4 CPU cores and 8GB of memory seem insufficient for your validation workflow, please modify the `xspot add` request for more CPU cores `-c` and for more memory `-m`.
5.  You can monitor progress of the above command by waiting or by opening another shell/terminal-multiplexer-window and issuing the following command:

    ```
    xspot ps -t
    ```
6.  When the Worker's Status is listed as "Ready" (last line of output), proceed to the next step.

    > **NetAPP ONTAP Mounts:**
    >
    > If you are using NetAPP ONTAP storages, login to the worker VM created in this step with ssh or Session Manager, and verify that all ONTAP folders are mounted correctly. If the controller's root account has ssh key generated under `/root/.ssh`, the key will be propagated to all worker VMs. You can ssh directly into worker VMs with the root account without the need of a password.
    >
    > To find a worker's ip address, issue the `xspot ps` command which will show similar output to below:
    >
    > ```
    > ====License====
    > SpotEnabled: true
    > ContainersEnabled: true
    > ====Config====
    > SchedulerEnabled : false
    > OnDemandTypes : [r5 r5n r5b r5d]
    > SpotFleetTypes : [r5 r5n r5b r5d]
    > DefaultAMI : ami-0222b6150e1c29556
    > BootingTimeout : 5
    > DefaultContainerCpu : 4
    > DefaultContainerMem : 4096
    > PausedWorkerTypes: [r5.24xlarge]
    >
    >
    > ====Sandboxes====
    > SANDBOX ID	WORKER ID	TARGET WORKER ID	STATUS	CPU	MEMORY	EC2 IP	CREATED	TAGS
    >
    > ====Workers====
    > WORKER ID	STATUS	INSTANCE ID        	INSTANCE TYPE	REMAINING MEM	REMAINING CPU	EC2 IP      	UP TIME  	DEADLINE  	SPOT	PRICE
    > 4465ce68 	Ready 	i-0222d5115daa6ef2e	r5b.xlarge   	29694        	4            	172.31.27.26	268h3m43s	-267h8m41s	true	0.088
    > ```
    >
    > For the example output above, as root on xspot-controller, you can `ssh 172.31.27.26` and you can inspect your expected mount points with `df` or the `mount` utilities on the worker.
7.  In the original shell or a new one, run the following command to start a container with X-Spot and connect to its console:

    * If `conf/env.cfg` was configured for an image name other than the default, `docker-base`, then the below command may need modification based on your image name.

    ```
    exorun run -c 4 -m 8 -i docker-base -- /bin/bash
    ```
8. Because each site will likely require customization, a working session with support is recommended initially.
9. Various site-specific services and customizations need to be configured and/or validated.
   * E.g.: authentication ( ldaps, ssd ), mounts ( nfs, autofs ), etc.
   * If `LUSTRE_NEEDED`, `ONTAP_NEEDED`, `AUTOFS_NEEDED`, `LDAPS_NEEDED`, or `SSSD_NEEDED` were toggled "on" during [Configuration](https://github.com/Exostellar-Labs/docs#configuration) above, then these requirements will need to be addressed in the next step.
   * The general approach here is to enter the sandbox started manually with the `exorun` command above and look for the services required. As the `root` user, it should be possible to determine the exact commands that are needed to bring up any required services. These commands and remediations should be tracked for the next step.
10. Modifications to the container or sandbox environment are expected, which will require editing the `startup.sh` script located at `/etc/xspot/scripts/`.
    * Also possibly the `Dockerfile` and/or `/etc/xspot/config/dockerWrapper.toml` may need changes.

### X-Spot Validation as User <a href="#user-content-x-spot-validation-as-user" id="user-content-x-spot-validation-as-user"></a>

> **Note:**
>
> You should be :

| Who?        | On which system?  | In which directory?                            |
| ----------- | ----------------- | ---------------------------------------------- |
| an end-user | X-Spot controller | any valid job-submit directory for the cluster |

> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder

Test with a valid or real workload.

1. Once the system administrator's tunings as root are validated, proceed to testing with a real user account.
2.  Run the following command to start a container with X-Spot and connect to its console:

    ```
    exorun run -c 4 -m 8 -i docker-base -- /bin/bash
    ```
3. Modify the previous command to run a real workflow.
   * If the workflow is a simple command and its arguments, it will look like the following:
     *   ```
         exorun run -c 4 -m 8 -i docker-base -- /full/path/to/command param1 param2 [more options or parameters...]
         ```

         > **Reminder:**
         >
         > `stty sane` or `reset` is needed when finished.
   * Or if the command is on your PATH, then the following will work:
     *   ```
         exorun run -c 4 -m 8 -i docker-base -- command param1 param2 [more options or parameters...]
         ```

         > **Reminder:**
         >
         > `stty sane` or `reset` is needed when finished.
   * If the workflow is leverages a script, `job.sh`, and it is located in the current working directory, then use the following:
     *   ```
         exorun run -c 4 -m 8 -i docker-base -- ./job.sh [param1 param2 ...]
         ```

         > **Reminder:**
         >
         > `stty sane` or `reset` is needed when finished.
4.  When testing concludes, reenable the X-Spot scheduler via:

    ```
    xspot scheduler on
    ```

    *   This will clean up the ondemand instance added above via `xspot add -c 4 -m 8`, but if you prefer, you can remove it immediately by issuing:

        ```
        xspot rm <worker-id-hash>
        ```

### X-Spot Validation When Satisfied <a href="#user-content-x-spot-validation-when-satisfied" id="user-content-x-spot-validation-when-satisfied"></a>

> **Note:**
>
> You should be :

| Who? | On which system?  | In which directory?             |
| ---- | ----------------- | ------------------------------- |
| root | X-Spot controller | /nfs-apps/exostellar/onboarding |

> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder

1.  Anytime changes are made to the image's `Dockerfile`, the image will need to be updated via the following command:

    ```
    make rebuild-image
    ```

\
