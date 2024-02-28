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

# Containerization

> **NOTE:**
>
> Subsequent steps require a fully functional and validated compute node from the cluster or a single VM if no HPC Cluster: the goal is to replicate the perfect environment.

> **NOTE:**
>
> If the node already has docker installed, pre-existing docker images will be removed. Therefore, docker images that are on the critical path and not backed up or housed anywhere but on this system's disk could be lost. Please make sure you understand this and will not lose unique data before proceeding.

> **NOTE:**
>
> No jobs should be running on the compute node during the following steps: the cluster admin can help ensure the node is drained, offline, or otherwise prevented from participating in cluster-wide scheduling for a short period of time while the environment is replicated.

> **Note:**
>
> You should be :

| Who? | On which system?  | In which directory?             |
| ---- | ----------------- | ------------------------------- |
| root | X-Infrastructure Optimizer controller | /nfs-apps/exostellar/onboarding |

> **Reminder:** `/nfs-apps` is a stand-in for the path to your remote folder

1.  Snapshot the perfect environment's filesystem.

    ```
    make base
    ```

    * **Note:** It could take 15 minutes for this command to complete, depending on your environment.
    * If at some later time, a new base snapshot is required, you can replace this asset with the following command:
      * ```
        make new-base
        ```
      * This would be a rare occurence in most environments.
2.  Generate or build docker images.

    ```
    make build-image
    ```

    * **Note:** It will probably take about 5 to 10 minutes for this command to complete, depending on your environment.
    * The `Dockerfile` and other build assets of the docker image will be available in `./images/docker-base/docker_build_path`
