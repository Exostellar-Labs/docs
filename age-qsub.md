# AGE qsub

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
 
