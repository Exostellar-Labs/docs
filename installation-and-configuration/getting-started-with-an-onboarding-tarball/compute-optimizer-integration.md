# Compute Optimizer Integration

### Compute Optimizer Configuration <a href="#user-content-compute-optimizer-configuration" id="user-content-compute-optimizer-configuration"></a>

> **Note:**
>
> You should be :

| Who? | On which system?             | In which directory?             |
| ---- | ---------------------------- | ------------------------------- |
| root | Compute Optimizer controller | /nfs-apps/exostellar/onboarding |

> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder

1. Update `./scripts/controller/integrate-xspot/config/config.toml`
   *   The following section of the file can accept more preferences

       ```
       [aws.instance_tags]
       Name = xspot-worker
       #i_key2 = "i_value2"
       ```

       * If you would like to introduce more AWS Tagging, copy as necessary the line starting with `#i_key2` and set key-value pairs.
   * Main configs to tune:
     * | Expression                  | Explanation                                                                                                                                                 |
       | --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
       | `ami =`                     | The worker AMI ID from [Compute Optimizer Worker AMI Preparation step](https://github.com/Exostellar-Labs/docs#compute-optimizer-worker-ami-preparation). . |
       | `overlay_prefix =`          | This class C network definition cannot overlap with Compute Optimizer controller's subnet.                                                                  |
       | `on_demand_types =`         | List of instance type and families for on-demand instances.                                                                                                 |
       | `spot_fleet_types =`        | List of instance type and families for spot instances.                                                                                                      |
       | `hyperthreading_disabled =` | Setting to true disables hyperthreading on worker instances.                                                                                                |
2. The following sections of the file may need updates for your site:
   *   Licensing updates to `./scripts/controller/integrate-xspot/config/config.toml`:

       ```
       [license]

       #Specify the floating key for the license system. For a mechanism other than floating license, use one of the following values:
       key = "x-spot-metering"
       ```
3. (optional) Log files by default reside in `/var/log/xspot` on each Compute Optimizer controller.
   *   To modify the default behavior, edit the log file configuration:

       ```
       ./scripts/controller/integrate-xspot/config/log_upload.env
       ```
   * Look for and reset the `LOG_PATH=` directive in the config file.
   * We recommend a remote filesystem to centralize the Compute Optimizer controller logs if changing the default location.

### exorun Configuration <a href="#user-content-exorun-configuration" id="user-content-exorun-configuration"></a>

> **Note:**
>
> You should be :

| Who? | On which system?             | In which directory?             |
| ---- | ---------------------------- | ------------------------------- |
| root | Compute Optimizer controller | /nfs-apps/exostellar/onboarding |

> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder There are several scripts and a config file used to configure the Compute Optimizer CLI Wrapper known as `exorun`:

* `./scripts/controller/integrate-xspot/config/dockerWrapper.toml`:
  * used by system administrators to customize the CLI wrapper installation.
  * Any changes to this file will need a config reload to the cli wrapper to take effect (`exorun config`)
    * `exorun config` is only available to superusers (UID == 0)
    * `container_image`: default container image to use when starting a new container with the CLI wrapper
      * The default is `docker-base`.
    * `approved_container_images`: list of approved container images to be used with the CLI wrapper
    * `log_level`: set how detailed the logs are
    * `env_var_filter`: the list of environment variables in the user's environment to NOT pass into the container
    * `container_start_timeout`: the amount of time (in minutes) to retry starting containers that previously failed to start for a job
      * The default is 30 minutes
    * `rand_backoff_ration`: this ratio is applied to the calculation before calculating the exponential back off time
      * The default is 0.5
      * The ration can be in the range: 0 <= X < 1
    * `container_startup`: location of `startup.sh`
    * `job_startup_*` and `job_cleanup_*`: the location of the startup and cleanup scripts
    * `runtime`: the runtime to use when starting a new container. `runxc` to run inside of Compute Optimizer, `runc` to use the default docker runtime
* `./scripts/controller/integrate-xspot/scripts/startup.sh`:
  * configures necessary resources inside of the container for the CLI wrapper, creates the user account and groups, sets up hostname of the container, etc.
* The following scripts are used to by the customer to make configurations on how the docker containers startup/stop
  * `./scripts/controller/integrate-xspot/scripts/job_startup_user.sh`:
    * anything that needs to happen in the container as the non-root user before the job starts
  * `./scripts/controller/integrate-xspot/scripts/job_startup_root.sh`:
    * anything that needs to happen in the container as root before the job starts
  * `./scripts/controller/integrate-xspot/scripts/job_cleanup_user.sh`:
    * anything that needs to happen after the job finishes as the non-root user
  * `./scripts/controller/integrate-xspot/scripts/job_cleanup_root.sh`:
    * anything that needs to happen after the job finishes as the root user

### Compute Optimizer Installation <a href="#user-content-compute-optimizer-installation" id="user-content-compute-optimizer-installation"></a>

Install Compute Optimizer components.

> **NOTE:**
>
> It is recommended to use an instance that has an additional solid state drive, such as **m5d.xlarge** which has an SSD that docker can leverage for significant performance gains, while m5.xlarge does not have the extra SSD.

> **NOTE:**
>
> If the node already has docker installed, pre-existing docker images will be removed. Therefore, docker images that are on the critical path and not backed up or housed anywhere but on this system's disk could be lost. Please make sure you understand this and will not lose unique data before proceeding.

> **Note:**
>
> You should be :

| Who? | On which system?             | In which directory?             |
| ---- | ---------------------------- | ------------------------------- |
| root | Compute Optimizer controller | /nfs-apps/exostellar/onboarding |

> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder

1.  Run the following command:

    ```
    make controller
    ```

    * **Note:** It will probably take about 5 to 10 minutes for this command to complete, depending on your environment.
