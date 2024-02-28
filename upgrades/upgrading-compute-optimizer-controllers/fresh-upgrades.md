# Fresh Upgrades

Starting from scratch basically means following the onboarding procedure which you are already familiar with. For this reason, it may appear like the path of least resistance. It is the recommended path for upgrades because it is a well-known path. Provision a new VM to be an X-Infrastructure Optimizer controller and start from the [X-Infrastructure Optimizer Worker AMI Preparation step](../../installation-and-configuration/getting-started-with-an-onboarding-tarball/x-infrastructure-optimizer-worker-ami-preparation.md).

### Upgrading Configuration <a href="#user-content-upgrading-configuration" id="user-content-upgrading-configuration"></a>

It is possible to leverage assets from a previous X-Infrastructure Optimizer installation to avoid repeating steps. What follows are the steps required to save a little time as compared to the Fresh Upgrades option above.

> **NOTE:**
>
> This process should not take place if the X-Infrastructure Optimizer controller is in use. The node should be devoid of any jobs and prevented from accepting jobs while the upgrade takes place. HPC Admins can drain, offline, or otherwise keep the node idle depending on the specifics of the environment. **Note:**
>
> You should be :

| Who? | On which system?  | In which directory?             |
| ---- | ----------------- | ------------------------------- |
| root | X-Infrastructure Optimizer controller | /nfs-apps/exostellar/onboarding |

> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder

1.  Navigate to the updated onboarding directory:

    * The following commands **should not be copy-pasted** because `/nfs-apps` is a generic term or stand-in for your remote directory/mount.

    ```
    cd /nfs-apps/exostellar/onboarding
    ```
2.  Establish a new folder for images:

    ```
    make image-folder
    ```
3.  Copy existing image files from the current controller:

    ```
    make base
    ```

    > **NOTE:**
    >
    > The upgrade process will necessarily remove docker images from this node. Not only is it assumed the jobs are not running on this node, but also that docker images reside some place safe and that their removal from this node's local disks does not represent the destruction of the only valid copy of any given image.
4.  Build the docker image.

    ```
    make build-image
    ```

    * **Note:** If the Dockerfile is customized during onboarding stages for your site, those modifications need to applied to the new Dockerfile:
      * Edit `./images/docker-base/docker_build_path/Dockerfile`
      * ```
        make rebuild-image
        ```
5. Upgrade X-Infrastructure Optimizer worker
   * Create a new scratch VM as in the [X-Infrastructure Optimizer Worker Preparation step above](../../installation-and-configuration/getting-started-with-an-onboarding-tarball/x-infrastructure-optimizer-worker-ami-preparation.md).
   * Update the ami ID in `./scripts/controller/integrate-xspot/config.toml`
6.  Upgrade X-Infrastructure Optimizer controller

    ```
    make upgrade
    ```
7. From the [Script Deployment step above](../../installation-and-configuration/getting-started-with-an-onboarding-tarball/script-deployment.md), do:
   * ```
     make sync-config-preview
     ```
   * ```
     make sync-config
     ```
   * ```
     make deploy
     ```
8.  Restart the post-docker service.

    ```
    systemctl restart post-docker &
    ```

    > **Note:**
    >
    > Running `systemctl restart post-docker` can easily take 15 minutes depending on your environment and the number of images configured.

    * You can monitor the progress of starting the `post-docker` service with the following command:

    ```
    journalctl -u post-docker -f
    ```

    * You will know the `post-docker` service has successfully completed when you this following message in the output of the previous command:

    ```
    ... post-docker.sh: Building of conatiner docker-base: SUCCESS!
    ... post-docker.sh: Finished.
    ... systemd: Started Post Docker Configuration.
    ```
9. From [Cluster Deployment step above](../../installation-and-configuration/getting-started-with-an-onboarding-tarball/cluster-deployment.md) you may prefer to create an AMI from the updated X-Infrastructure Optimizer controller.

### Upgrading Cluster Deployments <a href="#user-content-upgrading-cluster-deployments" id="user-content-upgrading-cluster-deployments"></a>

Rolling upgrades or swing migrations make a lot of sense in large environments where downtime is to be avoided. The approach is relatively simple, but we'll cover the basic ideas here for your consideration. This is applicable when an environment has multiple or many X-Infrastructure Optimizer controllers in production.

> **Note:**
>
> You should be :

| Who? | On which system?  | In which directory?                                                        |
| ---- | ----------------- | -------------------------------------------------------------------------- |
| root | X-Infrastructure Optimizer controller | ${REMOTE\_DIR}/scripts/latest/ also known as ${SCRIPT\_DEPLOY\_DIR}/latest |

> **Reminder:** `REMOTE_DIR` and `SCRIPT_DEPLOY_DIR` are set in the `onboarding/conf/env.cfg` file.

* Because the cloud offers a theoretically unlimited number of instances or VMs, you can set up a new X-Infrastructure Optimizer controller without taking any out of production. The new X-Infrastructure Optimizer controller would only be released to production when it's fully validated. At that stage, a previous X-Infrastructure Optimizer controller could be removed from production.
  * In fact, all the replacement X-Infrastructure Optimizer controllers can be brought online and —once fully validated— pushed into production before any previous X-Infrastructure Optimizer controllers are removed.
  * See [Cluster Deployment step above](../../installation-and-configuration/getting-started-with-an-onboarding-tarball/cluster-deployment.md).
*   If it's not possible to provision a new X-Infrastructure Optimizer controller, then one of the production X-Infrastructure Optimizer controllers will need to be idled. Once all jobs have cleared off it, upgrading can begin.

    ```
    cd ${SCRIPT_DEPLOY_DIR}/latest
    make upgrade
    systemctl restart post-docker &
    ```

    > **Note:**
    >
    > Running `systemctl restart post-docker` can easily take 15 minutes depending on your environment and the number of images configured.

    * You can monitor the progress of starting the `post-docker` service with the following command:

    ```
    journalctl -u post-docker -f
    ```

    * You will know the `post-docker` service has successfully completed when you this following message in the output of the previous command:

    ```
    ... post-docker.sh: Building of conatiner docker-base: SUCCESS!
    ... post-docker.sh: Finished.
    ... systemd: Started Post Docker Configuration.
    ```

    * In a similar fashion, each subsequent X-Infrastructure Optimizer controller marked for upgrade can be removed from production, one at a time or in mulitples at a time, while still preserving a limited production capacity of X-Infrastructure Optimizer controllers while the others are upgraded.
* Regardless of your approach, (serial replacements or parallel replacements), it would be considered best practice to move through the upgrade process on a single X-Infrastructure Optimizer controller and fully validate it before moving on to address upgrades in the rest of the fleet of X-Infrastructure Optimizer controllers.
* As noted above, and depending on the scale of the upgrade, creating an AMI may be an efficient way forward.
