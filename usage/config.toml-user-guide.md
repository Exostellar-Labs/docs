# Config.toml User Guide

This document provides insight into the configuration file, `config.toml`, used by Compute Optimizer

Each key/value pair represents a field used by Compute Optimizer for some configurable option.

Each key has a description of its use, along with a default if not required to be set. Key with a default of "N/A" are required to be set\
Options where a key is marked 'optional' are not used unless set

### `[scheduler]` <a href="#scheduler" id="scheduler"></a>

This section contains parameters used by the Compute Optimizer scheduling algorithm

`scheduler_enabled` - Whether the automatic scheduling feature is on or not. The scheduling algorithm determines whether Compute Optimizer run workloads should be automatically migrated to a different workload based on Compute Optimizer's scheduling policy.\
If the scheduler is disabled, no automatic migration will be performed.

**Default**: false

`evacuation_threshold` - The minimum amount of time to keep a workload on a spot instance before evacuating instances of that type. There will be a test run as we approach this time limit if no spot instances of that type have\
started naturally.

**Default**: 35 minutes

`booting_timeout` - How long the controller should wait for a worker to complete booting before trying a new instance

**Default**: 5 minutes

`try_spot_period` - How long the conroller should try for a spot instance when creating a new worker for new containers.\
If the spot request doesn't get fulfilled within this period, try an on-demand instance instead.\
Setting this value to -1 means we disable spot instances for new containers.

**Default**: 15 seconds

`spot_delay_period` - (optional) Time period that starts after a spot worker connects to the controller and after which it is considered stable enough to use.\
It is intended to prevent problems with very fast spot cancellations which tend to happen within the first minute of a worker joining.\
For no spot delay set this to -1 rather than to 0.

**Default**: 60 seconds

Note: setting it to zero is effectively the same as setting it to the default value

`idle_time` - (optional) How long a worker can remain idle (no workload running on it) before being terminated. This does not apply to newly started workers which can idle for 30s max after entering a ready state.

**Default**: 2 minutes

`port` - The listening port for the Compute Optimizer scheduler grpc service

**Default**: 24002

`log_file` - Where Compute Optimizer's scheduler should write controller log messages. If set to stdout the logs will be picked up by the xspot log service.

Example: /var/log/xspot.log

**Default**: stdout

`log_level` - The level of logging

**Options**: debug, info, warn, error

**Default**: info

`advisor_service` - (optional) The advisor is a service that can provide additional insights for the scheduler\
Set to an advisor endpoint to enable, set to "off" to disable

**Default**: [https://spot-advisor.exostellar.io](https://spot-advisor.exostellar.io/)

`advisor_level` - (optional) Indicates the level of information to send with the advisor messages\
The current levels are default which sends all information the advisor might need, and minimal which sends the absolute minimum amount of information. Sending all information increases the likelihood that the scheduler will be able to leverage future improvements to the advisor service

**Options**: “default”, “minimal”

**Default**: "default"

`advisor_frequency` - (optional) The number of seconds between requests for advice on which spot instance types to pause from the advisor.\
The minimum value this can be set to is 30. Any lower value will be ignored.

**Default**: 300

`http_proxy` - (optional) Proxy to use for the connection with the advisor

**Default**: none

`discount_aws` - Discount used for cost calculations - this is the percent discount AWS provides

**Default**: 0%

`dashboard_frequency` - the number of minutes between log updates to the dashboard log file

**Default**: 1

`hyperthreading_disabled` - (optional) Whether to disable the use of hyperthreading

**Default**: "off"

### `[aws]` <a href="#aws" id="aws"></a>

#### Options specific to AWS <a href="#options-specific-to-aws" id="options-specific-to-aws"></a>

`ssm_root` - (optional) The root path and region for loading xspot configurations from AWS SSM parameter store\
Note that the parameter store parameters will override the local configuration if this value is set to a non-empty string.\
Parameters inside the store need to follow the same hiarachy as the configuration file, for example: ${ssm\_root}/scheduler/scheduler\_enabled, ${ssm\_root}/aws/ami, please refer to Compute Optimizer Installation Guide for more information.

example: "/", "/xspot", "/path/to/xspot/parameters/"

**Default**:

`ssm_root` - (optional) "/xspot"

**Default**:

`ssm_region` - (optional) = "us-east-1"

**Default**: Controller's region

`ami` - The ami for creating worker instances\
This can be one of:

* AMI ID: the ID given to the AMI upon creation
* AMI Name: the name of the AMI
* key/value: a tag on the desired AMI. The key/value pair is separated with a "/". If multiple AMIs have the same tag, the newest AMI will be selected

**Default**: N/A

`security_groups` - (optional) Default security groups for creating worker instances. Comment this out or set to an empty list to use the security groups of the controller

**Default**: defaults to the controller security groups

`key_name` - (optional) ssh key for creating worker instances. Comment this out or set an empty string "" to use the key of the controller

**Default**: defaults to the controller key

`subnet_id` - (optional) Default subnet to which you want to add worker instances. Comment this out or set an empty string "" to use the subnet of the controller

**Default**: defaults to the controller subnet

The following options specify tags that you want to add to the worker instances and volumes.\
inherit\_instance\_tags and inherit\_volume\_tags allow you to propagate tags from the controller node.\
You can also add or override the tags by manually specifying instance\_tags and volume\_tags

`inherit_instance_tags` - Default: false

`inherit_volume_tags` - Default: alse

### `[aws.instance_tags]` <a href="#aws.instance_tags" id="aws.instance_tags"></a>

#### Tags passed to worker instances <a href="#tags-passed-to-worker-instances" id="tags-passed-to-worker-instances"></a>

`i_key1` = "i\_value1"

`i_key2` = "i\_value2"

### `[aws.volume_tags]` <a href="#aws.volume_tags" id="aws.volume_tags"></a>

#### Tags passed to worker volumes <a href="#tags-passed-to-worker-volumes" id="tags-passed-to-worker-volumes"></a>

`v_key1` = "v\_value1"

`v_key2` = "v\_value2"

### `[worker]` <a href="#worker" id="worker"></a>

#### Fields that are used to configure Compute Optimizer workers <a href="#fields-that-are-used-to-configure-compute-optimizer-workers" id="fields-that-are-used-to-configure-compute-optimizer-workers"></a>

`on_demand_types` - A list of all instance types and/or instance type families to use for on demand workers\
For any instance size (cpu and mem) in on\_demand\_types there should be a worker of\
matching size in spot\_fleet\_types and vice versa.

Note: these instance types should use the same CPU vendor (Intel or AMD).

Example: \["r5", "r5d", "r5n"]

Example: \["r5.2xlarge", "r5.4xlarge", "r5.8xlarge"]

You can also include priorities associated with each instance type (in the range \[0,9999]). A bigger number indicates a higher priority.\
Any type not given a priority defaults to priority 0.

Example: \["r5:3", "r5d:2", "r5n:1"]

`spot_fleet_types` - A list of all instance types and/or instance type families to use for spot workers. Spot types can have priority (see on\_demand\_types)\
Example: \["m5:1", "m5d:0", "m5n:0", "m6i:2"]\
**Default**: \["m5.2xlarge", "m5n.2xlarge"]

(optional) The allocation strategies for on-demand and spot instance types. These strategies are used by AWS and determine\
from which capacity pools new instances are launched.\
Note - Only options containing "prioritized" will honor the priorities specified for the instance types.\
`spot_allocation_strategy`\
**Options**: lowest-price, diversified, capacity-optimized, capacity-optimized-prioritized, price-capacity-optimized\
**Default**: capacity-optimized\
`on_demand_allocation_strategy`\
**Options**: lowest-price, prioritized\
**Default**: prioritized

`dom0_memory` - Reserved memory size in MB for Domain-0 in worker nodes\
**Default**: 2048 MB

`default_container_cpu` - The default number of cpu for containers\
**Default**: 1

`default_container_mem` - The default memory size for containers in MB\
**Default**: 2048\
Note that this cannot be bigger than the total memory size of the worker instance minus reserved dom0 memory\
e.g.: if you choose m5.2xlarge as the instance type for worker nodes (32GB memory), and you set dom0\_memory=2048 (2GB),\
the maximum default\_container\_mem you can set is 32GB - 2GB = 30GB (30720 MB)

### `[network]` <a href="#network" id="network"></a>

#### Section used to configure the internal overlay network used by Compute Optimizer for a controller and its workers and containers <a href="#section-used-to-configure-the-internal-overlay-network-used-by-compute-optimizer-for-a-controller-an" id="section-used-to-configure-the-internal-overlay-network-used-by-compute-optimizer-for-a-controller-an"></a>

`overlay_prefix` - The prefix for the internal overlay network used by Compute Optimizer (24 bits)\
Example: a "10.0.0." prefix means a CIDR block: 10.0.0.0/24\
**Default**: 192.168.137.

`mtu` - The maximum transmission unit for the overlay network\
**Default**: 8900\
Note - The docker mtu is configured separately in /etc/docker/daemon.json

### `[exo]` <a href="#exo" id="exo"></a>

### Section to configure Compute Optimizer runtime options shared by controller and worker runtime <a href="#section-to-configure-compute-optimizer-runtime-options-shared-by-controller-and-worker-runtime" id="section-to-configure-compute-optimizer-runtime-options-shared-by-controller-and-worker-runtime"></a>

`worker_grpc_port` - The default worker grpc port, listened on by worker nodes, and dialed by the controller\
**Default**: 1235

`shared_mount_source` - mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport \<aws\_efs\_endpoint> /run/xspot/share/\
Source for a mount that is shared between controller and workers.\
**Default**: none\
Note - If set to localhost, we use the controller node as the nfs server, and exportfs /run/xspot/share. If set to none, the shared mount is disabled

`shared_mount_type` - Type of shared mount\
**Default**: nfs

`shared_mount_options` - Options for the shared mount

Example "nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport"\
**Default**: none\
Note - If `shared_mount_source` is localhost, this option is not used

`enable_balloon` - EXPERIMENTAL - Turns on self-ballooning which minimizes container memory throughout its lifetime\
**Default**: false

### `[exo.master]` <a href="#exo.master" id="exo.master"></a>

#### Runtime options unique to the controller <a href="#runtime-options-unique-to-the-controller" id="runtime-options-unique-to-the-controller"></a>

`log_file` - Where Compute Optimizer should write controller log messages\
If log\_file is not set to stdout, the full path would include the instanceid,\
e.g., /run/xspot/share/${controller\_instanceid}/${log\_file}\
**Default**: stdout

`log_level` - The level of logging

**Options**: info, debug\
**Default**: debug

`shared_mount_point` - Directory for the shared mount\
**Default**: /run/xspot/share\
Note - if `shared_mount_source` is locahost, not used

`crash_timeout` - Time a worker or container can be unresponsive before it is considered crashed and cleaned up.\
**Default**: 2 min

### `[exo.worker]` <a href="#exo.worker" id="exo.worker"></a>

#### Runtime options unique to the worker <a href="#runtime-options-unique-to-the-worker" id="runtime-options-unique-to-the-worker"></a>

`log_file` - Where Compute Optimizer should write worker log messages\
If log\_file is not set to stdout, the full path would include the instanceid,\
e.g., /run/xspot/share/${controller\_instanceid}/${instance\_id}/${log\_file}\
**Default**: stdout

`log_level` - The level of logging

**Options**: info, debug\
**Default**: debug

`shared_mount_point` - Directory for the shared mount\
**Default**: /run/xspot/share

`enable_debug` - Whether to enable debug messages in log files. Disabling will delete log files after terminating the container\
**Default**: false

`enable_vifroute` - Enable vif-native-route for host network connectivity\
**Default**: false

`virt_mode` - Virtualization mode for the worker

**Options**: pv, pvh, or auto\
**Default**: auto

\
pvh virtualization only works on bare metal instances. auto will detect the instance type; if bare metal, then it will select pvh virtualization, else it will use pv virtualization

`cpu_pin` - Enable vCPU pinning to physical cores. This feature takes into consideration hyperthreading information and NUMA node information.\
**Default**: false

`cpuid` - CPUID instruction configuration value in libxl format for the container\
Setting this to "host,htt=0" will present vCPUs as single-threaded cores to the container.\
Setting this to "host,htt=1" will present vCPUs as hyperthreaded cores to the container.\
Note: "htt=1" is valid only when the underlying physical CPUs are hyperthreaded.\
Setting this to "host,pse=0" will disable super pages for the container. This is useful if the kernel doesn't properly support super pages.\
**Default**: "host,htt=0"

(for advanced use only) cpuid mask in xend format, comment out or specify empty list to disable\
see below for example specification\
cpuid = \[\
"0x00000001:ecx=xxxxxxxxxx0xxxxxxxxxxxxxxxxxxxxx,edx=xxxx0xxxxxxxxxxxxxxx0xxxxxxxxxxx",\
"0x80000001:ecx=xxxxxxxxxxxxxxxxxxxxxxx000xxxx0x,edx=xxxxxx0xx0xxxxxxxxxxxxxxxxxxxxxx",\
"0x0000000d,1:eax=xxxxxxxxxxxxxxxxxxxxxxxxxxxxx00x",\
"0x00000007,0:ebx=0000xxx00xxx0000xxxx0x0xxxx0xxxx,ecx=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx,edx=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",\
"0x80000007:edx=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",\
"0x80000008:ebx=xxxxxxxxxxxxxxxxxxxxxxxxxxxxx0x0",\
"0x00000007,1:eax=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",\
]

`scratch_type`- Type of scratch storage to use. The size of the scratch storage in GB is set per container with the environment variable XSPOT\_SCRATCH when creating a container.\
If the environment variable is not set no scratch storage is added. The scratch storage is exposed to the container as a device.\
Note: to enable file based scratch storage the config parameters beginning with "shared\_mount" need to be set

**Options**: file, ebs\
**Default**: none

`scratch_caching` - Enables the use of local ssd storage, if available, to cache the scratch storage\
**Default**: false

### `[license]` <a href="#license" id="license"></a>

#### Section to set license related information <a href="#section-to-set-license-related-information" id="section-to-set-license-related-information"></a>

`key` - Specify the floating key for the license system.\
For a mechanism other than floating license, use one of the following values:

* marketplace: "x-spot-marketplace"
* metering: "x-spot-metering"

**Default**: If neither the designated 'marketplace' or 'metering' string are matched, Compute Optimizer will assume a floating key should be passed
