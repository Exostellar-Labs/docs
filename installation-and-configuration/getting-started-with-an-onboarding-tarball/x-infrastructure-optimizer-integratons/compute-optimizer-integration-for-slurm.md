---
description: Configuring CLI plug-in for Slurm
---

# X-Infrastructure Optimizer Integration for SLURM

This document covers the configuration of the Slurm job scheduler to rewrite submitted jobs to execute on a controller node running X-spot.

### 1. Replacing the sbatch command line tool <a href="#user-content-1-replacing-the-sbatch-command-line-tool" id="user-content-1-replacing-the-sbatch-command-line-tool"></a>

In order to support jobs submitted from inside a container over `ssh`, we need to install `sub_wrapper.py` as a replacement for the `sbatch binary`. Assuming that `sub_wrapper.py` is installed in `$SLURM_TOP/sub_wrapper.py` :

```
mv $SLURM_TOP/sbatch $SLURM_TOP/sbatch.orig
ln -s $SLURM_TOP/sub_wrapper.py $SLURM_TOP/sbatch
```

Modify the path to the original `sbatch` binary in `$SLURM_TOP/sub_wrapper.py`: Example:

```
### BEGIN CONFIGURATION #######################################################

real_sub = "/fsx/ontap/exo-opt/slurm_0/sbatch.orig"

real_job = "/usr/bin/exorun run"
```

The `real_job` parameter is the path to the job CLI wrapper that will execute on the controller.

### 2. Installing the CLI plug-in <a href="#user-content-2-installing-the-cli-plug-in" id="user-content-2-installing-the-cli-plug-in"></a>

Copy `cli_filter_xspot.so` to `$SLURM_TOP/cli_filter_xspot.so` so the plug-in is visible from all nodes. Depending on your configuration, the Slurm libraries may be located elsewhere and you will need to create a symbolic link to the shared location in `$SLURM_TOP/cli_filter_xspot.so`. Example:

```
ln -s $SLURM_TOP/cli_filter_xspot.so /usr/lib64/slurm/cli_filter_xspot.so
```

### 3. Configuring multiple queues <a href="#user-content-3-configuring-multiple-queues" id="user-content-3-configuring-multiple-queues"></a>

By default, the plug-in recognizes two partition names, namely `xspot` and `chipspot`. The controller node expects our container wrapper to be installed in `/usr/bin/exorun`.

### 4. Configuring Slurm to use the plug-in <a href="#user-content-4-configuring-slurm-to-use-the-plug-in" id="user-content-4-configuring-slurm-to-use-the-plug-in"></a>

To enable the plug-in, edit `/etc/slurm/slurm.conf` (location may vary) with the following:

```
CliFilterPlugins=cli_filter/xspot
```

### 5. Testing <a href="#user-content-5-testing" id="user-content-5-testing"></a>

To verify that all the steps above were successful, a simple job can be submitted. Assuming `simple.sh` contains this:

```
#!/bin/bash
#SBATCH -o %j.out
#SBATCH -p xspot

for i in {0..99}; do
    echo $i
done
```

Job submittal:

```
sbatch --ntasks=2 --mem=3G /fsx/ontap/work/user/simple.sh
```

Sample output:

```
controller_ip = None
NO stdin
('script args:', ['/fsx/ontap/work/user/simple.sh'])
('shared_dir=', '/home/user/.xspot/head')
pre_submit: pn_min_memory = 3072, ntasks = 2
modified: partition = xspot, mem = 3G, cpu = 2
Submitted batch job 15030
```

Under `script args`, we can see the script and its arguments. Under `shared_dir`, the location of a temporary directory used to rewrite the script contents to use X-spot. Files appear in that directory but are deleted immediately after `sbatch` is called. Under `modified`, we can see the resources requested to create the container (3G/2CPUs).

The output also indicates if the job is going over `ssh` (`controller_ip != None`) and if the job was submitted via `stdin`.
