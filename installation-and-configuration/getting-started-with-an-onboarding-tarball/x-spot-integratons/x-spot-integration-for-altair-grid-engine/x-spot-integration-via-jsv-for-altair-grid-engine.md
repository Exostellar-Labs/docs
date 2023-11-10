---
description: Configuring jsv_xspot for AGE
---

# Compute Optimizer Integration via JSV for Altair Grid Engine

This document covers the configuration of the AGE job scheduler to rewrite submitted jobs to execute on a controller node running X-spot.

### &#x20;Replacing the qsub command line tool <a href="#user-content-1-replacing-the-qsub-command-line-tool" id="user-content-1-replacing-the-qsub-command-line-tool"></a>

In order to support jobs submitted from inside a container over `ssh`, we need to install `sub_wrapper.py` as a replacement for the `qsub` binary. Assuming that `sub_wrapper.py` is installed in `$AGE_TOP/bin/lx-amd64` :

```
mv $AGE_TOP/bin/lx-amd64/qsub $AGE_TOP/bin/lx-amd64/qsub.orig
ln -s $AGE_TOP/bin/lx-amd64/sub_wrapper.py $AGE_TOP/bin/lx-amd64/qsub
```

Modify the path to the original `qsub` binary in `$AGE_TOP/bin/lx-amd64/qsub`: Example:

```
### BEGIN CONFIGURATION #######################################################

real_sub = "/fsx/ontap/exo-opt/age_0/bin/lx-amd64/qsub.orig"
```

### Installing jsv\_xspot <a href="#user-content-2-installing-jsv_xspot" id="user-content-2-installing-jsv_xspot"></a>

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

### Configuring AGE <a href="#user-content-3-configuring-age" id="user-content-3-configuring-age"></a>

To enable `jsv` and to allow it to rewrite the details of the job submitted, run `qconf -mconf` and look for `jsv_url` in the configuration. Add the path to `$AGE_TOP/util/resources/jsv/jsv_xspot.py` Example:

```
jsv_url                      /fsx/ontap/exo-opt/age_0/util/resources/jsv/jsv_xspot.py
```

Save the file and `qsub` will start validating jobs submitted using `jsv_xspot.py`. Based on the configuration above, only jobs submitted to the `xspot` queue will be rewritten to use X-spot.

### Testing <a href="#user-content-4-testing" id="user-content-4-testing"></a>

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
