# Compute Optimizer Integration for Altair Grid Engine

### Prerequisites <a href="#user-content-1-prerequisites" id="user-content-1-prerequisites"></a>

1. A fully functional Altair Grid Engine installation is required.
2. Sufficient privileges to modify the Grid Engine configuration will be required.

### AGE Configuration Modifications <a href="#user-content-2-age-configuration-modifications" id="user-content-2-age-configuration-modifications"></a>

1. Assumptions about AGE configuration as a key to steps below:
   * The Master Host is represented below as `head`.
   * A single Execution Host is referenced below as `n011`, and it is a stand-in for all Compute Optimizer controllers that will be participate in the Altair Grid Engine configuration.
2. Add queue `xspot`.
   * difference between deafult all.q and xspot:
     * qname
       * `xspot` instead of `all.q`
     * hostlist
       * `n011` instead of `@allhosts`
     *   slots

         * `1,[n011=50]" instead of` 1,\[n011=\<num\_cpus>],\[head=2]"

         > **NOTE:**
         >
         > Some testing is recommended for your workflows. A single Compute Optimizer controller should be limited to 80 slots. This is largely independent of CPU CORES and MEMORY.
3.  Validation via the following commands:

    ```
    cd ${SGE_ROOT}/${SGE_CELL}
    diff ./spool/qmaster/cqueues/{all.q,xspot}
    ```

### qsub Wrapper Configuration <a href="#user-content-3-qsub-wrapper-configuration" id="user-content-3-qsub-wrapper-configuration"></a>

As a proof of concept, a relatively simple wrapper can be placed alongside `qsub` in the `${SGE_ROOT}/bin/${ARCH}` directory. The wrapper will inspect job commands given to `qsub`, watching for requests for the `xspot` queue. If none are found, the wrapper gets out of the way and passes everything to the original `qsub` which could be renamed `qsub.orig`. If a request for the `xspot` queue is discovered by the wrapper, it will intercept any job parameters that need modification for a successful Compute Optimizer job launch and make the required changes before passing the rest of the job parameters to the original `qsub`. The AGE Administrator is encouraged to adopt a more robust strategy, e.g. leveraging AGE's JSV framework. Exostellar can deliver the requirements for successful Compute Optimizer job launch.

1. The following global variables should be set on the top of the wrapper script:
   1.  The wrapper script should be found here:

       ```
       ${SCRIPTS_DEPLOY_DIR}/scripts/latest/tools/sched-wrapper/qsub.py
       ```
   2.  Name of the docker container image running the workload.

       ```
       DOCKER_IMAGE = "<your_docker_image>"
       ```
   3.  Full path of the `exorun` script used to schedule the jobs on an xspot controller.

       * The default location is `/usr/bin/exorun`.

       ```
       XSPOT_RUN = "/usr/bin/exorun"
       ```
   4.  Full path of the original qsub command on the system.

       * The default location is `${SGE_ROOT}/bin/${ARCH}/qsub`.

       ```
       REAL_EXE = "/path/to/real/qsub"
       ```
   5.  Default amount of memory (in GB) used by the container on an xspot worker node when not provided on the command line.

       ```
       DEFAULT_MEM = 1
       ```

       *   Otherwise, this is specified on the `qsub` commandline with `-l`. For example:

           ```
           -l mem_free=1024M
           ```
   6.  Default number of cpus used by the container on an xspot worker node when not provided on the command line.

       ```
       DEFAULT_CPU = 1
       ```

       * Otherwise, this is specified on the `qsub` commandline with `-pe`. For example:

       ```
       -pe mpi 1
       ```
2. Set up the wrapper and default installation binaries for cooperation:
   1.  Rename the original `qsub` binary:

       ```
       cd ${SGE_ROOT}/bin/${ARCH}
       mv qsub qsub.orig
       ```
   2.  Link (or copy) the wrapper in the `${SGE_ROOT}/bin/${ARCH}` directory:

       ```
       ln -s ${SCRIPTS_DEPLOY_DIR}/scripts/latest/tools/sched-wrapper/qsub.py ./qsub
       ```
3. Invocation examples:
   *   Example #1 - 1 CPU, 1G

       ```
       qsub -q xspot -l mem_free=1024M qsub_job.sh
       ```

       *   Actual job submitted:

           ```
           qsub.real -q xspot -pe mpi 1 -l mem_free=1G /usr/bin/exorun -i <your_docker_image> -c 1 -m 1 -- qsub_job.sh
           ```
   *   Example #2 - 1 CPU, rounds 2.5G to 3G

       ```
       qsub -q xspot -l mem_free=2.5G qsub_job.sh
       ```

       *   Actual job submitted:

           ```
           qsub.real -q xspot -pe mpi 1 -l mem_free=1G /usr/bin/exorun -i <your_docker_image> -c 1 -m 3 -- qsub_job.sh
           ```
   *   Example #3 - 2 CPUs, rounds 2.5G to 3G

       ```
       qsub -q xspot -l mem_free=2.5G -pe mpi 2 qsub_job.sh
       ```

       *   Actual job submitted:

           ```
           qsub.real -q xspot -pe mpi 1 -l mem_free=1G /usr/bin/exorun -i <your_docker_image> -c 2 -m 3 -- qsub_job.sh

           ```



