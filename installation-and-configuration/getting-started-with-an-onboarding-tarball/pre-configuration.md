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

# Pre-Configuration

> **Note:**
>
> You should be :

| Who? | On which system?  | In which directory?             |
| ---- | ----------------- | ------------------------------- |
| root | X-Spot controller | /nfs-apps/exostellar/onboarding |

> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder

1. Edit the `conf/env.cfg` file and set required variables.
   * The `conf/env.cfg` contains useful explanations, as shown below:
     * ```
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
     * The `REMOTE_DIR` is required to be on a remote filesystem and available to all nodes in the cluster.
   * `CONTAINER_FS_SIZE` will act as a hard limit on storage utilization.
     * If a target workflow typicaly makes significant use of scratch space on the local filesystem, this limit should be carefully considered.
     * If more data than this limit is written to the local filesystem during job run, an error will occur and the job will error terminate.
   * `REMOTE_DIR`, `IMAGE_DEPLOY_DIR`, and `SCRIPT_DEPLOY_DIR` can be set according to your preferences, with the following caveats:
     * `REMOTE_DIR` is expected to be an NFS, LUSTRE, AUTOFS, or ONTAP mount.
     * `IMAGE_DEPLOY_DIR` and `SCRIPT_DEPLOY_DIR` should be available to any X-Spot controller.
   *   Certain AWS or linux services that may be in use in your environment should be toggled "on" now in the config, for example:

       * Amazon FSx for Lustre
       * Amazon FSx for NetApp ONTAP
       * autofs
       * sssd
       * ldap
       * For example, remove the `#` at the front of the line that starts with `#AUTOFS_NEEDED=1` if `autofs` is part of the environment.

       > **NOTE:**
       >
       > Later on during [validations](https://github.com/Exostellar-Labs/docs#x-spot-validation-as-root) specific information unique to the site will need to be tested and validated. For example, mounting of remote filesystems may be accomplished in several ways, so a final decision will be made during validation and the configuration will be finalized at that time.
   *   Amazon FSx for NetApp ONTAP handling

       > **IMPORTANT:**
       >
       > NetApp ONTAP storage needs some special attention.
       >
       > If your ONTAP storage is mounted by `autofs`, make sure that you toggle both `ONTAP_NEEDED` and `AUTOFS_NEEDED` on in `env.cfg`, and also configure `ONTAP_FOLDERS` correctly. Incorrect configuration of `ONTAP_FOLDERS` will cause failures in worker creation.
       >
       > If your ONTAP storage is not mounted by `autofs`, then append the following section to the end of the `scripts/templates/customize_worker_ami.sh` file:
       >
       > ```
       >    mkdir -p /fsx
       >    mount -t nfs ONTAP_SERVER_IP:/volume1 /fsx
       > ```
       >
       > Be sure to customize the ip ( `ONTAP_SERVER_IP` ) and the volume name ( `/volume1` ) and the mount point ( `/fsx` ).
       >
       > If more volumes are required, these lines should be repeated for each volume.
       >
       > The purpose of these steps is to make sure that ONTAP stroage is mounted correctly on the X-Spot worker node. In the step of [X-Spot Validation](https://github.com/Exostellar-Labs/docs#x-spot-validation), you can login to the worker node with ssh or Session Manager, and check if all ONTAP storage folders are mounted correctly.
       >
       > When there are multiple NetAPP ONTAP Servers in play, the minimum requirement is one volume per server should be specified in the `customize_worker_ami.sh` script as shown in the example above. Make sure all the mount points are listed from `conf/env.cfg`'s ONTAP\_FOLDERS.
2.  Once edited, the following command will report if any errors are found in conf/env.cfg

    ```
    make check-config
    ```

    * The above step is optional.
    * The configuration will be automatically checked at each step just in case any unexpected modifications take place and when errors are found, they will be reported and the installation will be interrupted as a matter of safe-guarding.
