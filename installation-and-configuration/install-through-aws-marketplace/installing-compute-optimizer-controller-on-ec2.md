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

# Installing X-Infrastructure Optimizer Controller on EC2

Go to **EC2 > Instances > Launch an instance**

### Step 1: Subscribe to X-Infrastructure Optimizer Controller <a href="#step-1-subscribe-to-x-infrastructure-optimizer-controller" id="step-1-subscribe-to-x-infrastructure-optimizer-controller"></a>

For **Application and OS Images (Amazon Machine Image)**, search for "x-infrastructure-optimizer controller".

Under the tab **AWS Marketplace AMIs**, select Exostellar's X-Infrastructure Optimizer Controller, subscribe, and continue.

<figure><img src="../../.gitbook/assets/Screenshot 2023-11-10 at 11.23.21 AM.png" alt=""><figcaption><p>Exostellar’s X-Infrastructure Optimizer Controller Via AWS Marketplace</p></figcaption></figure>

{% hint style="warning" %}
To configure the AMI for IMDSv2:

```
aws ec2 modify-image-attribute \
    --image-id ami-0123456789example \
    --imds-support v2.0
```
{% endhint %}

### Step 2: Launch the Instance Using the Created Resources <a href="#step-2-launch-the-instance-using-the-created-resources" id="step-2-launch-the-instance-using-the-created-resources"></a>

{% hint style="info" %}
The recommended instance type to start with is `m5d.xlarge`**.**
{% endhint %}

Please select your **Key pair (login)**, **Network**, and **Security group**.\
In **Advanced details**, please attached the created IAM Role to **IAM instance profile**.

All set! Please go ahead and **Launch instance**.
