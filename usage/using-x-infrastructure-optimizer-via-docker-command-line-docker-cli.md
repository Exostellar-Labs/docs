# Using X-Infrastructure Optimizer via Docker Command Line (Docker CLI)

**Start a container**\
By default, Docker uses `runc` as the runtime. To use X-Infrastructure Optimizer, add the following parameter to the Docker command line:

```
docker run --runtime runxc ...
```

**Specify the CPU**\
To specify the CPU of containers:

```
docker run ... -e XSPOT_CPU=4 ...
```

**Specify the memory**\
To specify the memory of containers:

```
docker run ... -e XSPOT_MEM=4096 ...
```

{% hint style="info" %}
The unit of XSPOT\_MEM is in MB.
{% endhint %}

**Configure network**\
For network, in order to get better performance, we recommend you to use host networking:

```
docker run ... --network host
```

**Add additional capabilities**\
Also for better performance, we recommend mounting external storage within the container. To add capabilities to the container:

```
docker run ... --cap-add ALL ...
```

Within the container, you can mount the file system as the root user.

**Sample docker command**

```
docker run --runtime runxc -d -it -e XSPOT_CPU=4 -e XSPOT_MEM=4096 busybox top # start a detached busyboxy container
```

**Remove a container**\
To remove a `runxc` container:

```
docker stop <cid>
docker rm <cid>
```

