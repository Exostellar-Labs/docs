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

# Onboarding Assets

> **Note:**
>
> You should be :

| Who? | On which system?  | In which directory? |
| ---- | ----------------- | ------------------- |
| root | X-Infrastructure Optimizer controller | /tmp                |

1. Identify a compute node from the HPC Cluster and prevent jobs from running on it by marking it down, offline, drain, etc.
   * Alternatively, you may proceed with only a single VM but ensure that no workflows are running on it.
2.  Download the tarball of the latest release of X-Infrastructure Optimizer via the following command:

    ```
    wget https://rpm.exostellar.io/download/onboarding.tar
    ```
3. This tarball will need to be unpacked into a location that is available to all the nodes in the cluster. The assumption is that this will be a remotely mounted directory. For example, many systems locate software available cluster-wide in the `/apps` or `/opt` directory.
   * The following commands **should not be copy-pasted** because `/nfs-apps` is a generic term or stand-in for your remote directory/mount.
   *   To keep things contained, let's make a directory for these onboarding assets under `/nfs-apps`:

       ```
       mkdir /nfs-apps/exostellar
       ```
   *   We'll unpack the tarball via the `tar` command wherever the `onboarding.tar` file was downloaded:

       ```
       tar xvf onboarding.tar -C /nfs-apps/exostellar
       ```
   *   Next, we'll `cd` to the `onboarding` directory that resulted from the previous command:

       ```
       cd /nfs-apps/exostellar/onboarding
       ```
   * To track changes that are anticipated, we will use `git`:
     *   Initialize an empty local repository:

         ```
         git init .
         ```
     *   Add everything in the directory to the `git` version-control system:

         ```
         git add .
         ```
     *   "Commit" the current state of the files in and under this directory so we can track changes.

         ```
         git commit -m $( cat version )
         ```
     * We will see more `git` commands later on in the onboarding process, particularly when upgrading for a latest release of X-Infrastructure Optimizer.
   * We are now ready to begin configuration and installation steps.
