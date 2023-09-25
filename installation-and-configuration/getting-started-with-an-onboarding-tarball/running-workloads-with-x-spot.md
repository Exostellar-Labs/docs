# Running workloads with X-Spot

In most production environments, X-Spot will be integrated with the scheduler for frictionless adoption, but it would still be possilbe leverage `exorun` manually since it is will be on on every users' PATH on X-Spot controllers by default.

> **Note:**
>
> You should be :

| Who?        | On which system?                    | In which directory?          |
| ----------- | ----------------------------------- | ---------------------------- |
| an end-user | any valid submit host or login node | a valid job-submit directory |

1. X-Spot integration for job submission via native `bsub`, `qsub`, and `sbatch` commands:
   *   Once the `bsub`, `qsub`, `sbatch` integration is setup, X-Spot jobs (routed to the xspot queue) will automatically interoperate with the scheduler. E.g:

       ```
       bsub -q xspot -n 2 ./job.sh
       ```
2. Refer to the documents on scheduler integration for more details.
   * [qsub](https://github.com/Exostellar-Labs/docs/blob/main/.age-qsub.md)
   * [sbatch](https://github.com/Exostellar-Labs/docs/blob/main/.slurm-sbatch.md)
3. To add `exorun` to your current shell's path, you can run a few commands to simplify.
4.  Validate the `exorun` executable's location and permissions:

    ```
    ls -la /usr/bin/exorun
    ```
5.  If it's in that place as exepcted, ensure its mode is correct:

    ```
    chmod 755 /usr/bin/exorun
    ```
6.  If `/usr/bin` isn't already on your path, add that directory to your path:

    ```
    export PATH=${PATH}:/usr/bin
    ```
7.  Validate your shell knows where to find `exorun`:

    ```
    which exorun
    ```
8.  Jobs can be started in X-Spot containers by using the job wrapper:

    ```
    exorun run -c <cpu_num> -m <mem_size> -- command param1 param2 ...
    ```
9.  Run the following command to get more usage information:

    ```
    exorun -h
    ```
10. To get more information on running jobs with `exorun`:

    ```
    exorun run -h
    ```
