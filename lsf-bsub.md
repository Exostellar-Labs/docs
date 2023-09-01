# LSF bsub

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
