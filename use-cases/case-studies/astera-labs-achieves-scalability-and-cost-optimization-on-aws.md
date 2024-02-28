---
description: Exostellar enabled significant cost savings and quicker time to market
cover: >-
  https://images.unsplash.com/photo-1621508727844-09cc245de555?crop=entropy&cs=srgb&fm=jpg&ixid=M3wxOTcwMjR8MHwxfHNlYXJjaHwxfHxzdGVsbGFyfGVufDB8fHx8MTY5NTY2MDI4OHww&ixlib=rb-4.0.3&q=85
coverY: 0
---

# Astera Labs achieves scalability and cost-optimization on AWS

> **“Exostellar’s X-Infrastructure Optimizer allowed us to leverage the cost-effectiveness of spot instances with the reliability as-if they were on-demand instances, allowing Astera to run more simulation and physical design workloads resulting in reduced development time and quicker time to market.”**
>
> – Jitendra Mohan, _CEO of Astera Labs_

### Introduction

[Astera Labs](https://www.asteralabs.com/), a fabless semiconductor company, pioneered its Electronic Design Automation (EDA) workflow on Amazon Web Services (AWS) to help it scale compute requirements that can fluctuate by 10x during a project timeline. However, over the last several years, Astera Labs’ AWS spend had increased significantly. To tackle the increased spending, Astera Labs partnered with Exostellar to optimize its design and verification workflows on AWS. During nine months of production deployment, Exostellar’s [X-Infrastructure Optimizer](https://exostellar.io/solutions/#xspot) enabled Astera Labs to achieve significant cost reductions of up to 50% for on-demand compute resources while maintaining high-performance and reliability.

### Challenge

During its chip design and verification process (Physical Design Workflow), Astera Labs relies heavily on EDA tools for the tape-out process. Tape-out is the final stage in chip design where a design is thoroughly verified and prepared for manufacturing. The tape-out stage is missioncritical and demands substantial compute resources and reliability. Any errors or oversights during this phase can lead to costly delays and manufacturing defects. To enhance performance and scalability, Astera Labs made the decision to migrate its workloads to the AWS cloud platform.

**EDA on cloud: Fast, scalable, higher quality but expensive**

While running compute-intensive tape-outs with Synopsys and other tools using on-demand AWS instances, Astera Labs encountered the challenge of escalating compute costs. The unpredictable patterns of its workload made it difficult to optimize resource utilization effectively, resulting in inefficient spending and a significant increase in AWS expense.

### Solution

#### Running Physical Design Workflow using X-Infrastructure Optimizer on AWS

To address significant spending, Astera Labs ran its HPC workloads in AWS using Exostellar’s X-Infrastructure Optimizer, which is a state-of-the-art cloud optimization solution that allows users to seamlessly relocate stateful and compute-intensive EDA applications between spot instances and on-demand instances resulting in up to 5x increase in compute for the same dollar spent (80% cost reduction).

**EDA on cloud + X-Infrastructure Optimizer: Fast, scalable, higher quality, and cost-effective**

Astera Labs’ Synopsys workloads were executed using X-Infrastructure Optimizer, which offers a reliable and consistent virtual environment for spot instances. These workloads were seamlessly scheduled within the existing Portable Batch System (PBS) job scheduler with minimal modifications to the job launch process. Additionally, X-Infrastructure Optimizer facilitated the automatic live migration of workers between on-demand and spot instances, as depicted in Figure 1.

<figure><img src="../../.gitbook/assets/image (13).png" alt=""><figcaption><p><em>Figure 1, X-Infrastructure Optimizer integration flow with Synopsys workload</em></p></figcaption></figure>

Specifically, the on-demand instances utilized for the workloads were of type r6i, while the spot instances used were of type r6\*/r5\* (r6i, r6id/ r5, r5n, r5d, r5nd, others). This integration flow ensured optimal resource utilization and cost efficiency.

Importantly, the live migration process between different instance types occurred smoothly, without changing network addresses or network connections allowing the job to continue running without any pausing or other interruptions.

### Result

Astera Labs has now increased compute capacity by up to 2x for the same budget to run thousands of chip design workloads by using X-Infrastructure Optimizer.

<figure><img src="../../.gitbook/assets/image (14).png" alt=""><figcaption><p><em>Figure 2, Synopsys workload migration using X-Infrastructure Optimizer</em></p></figcaption></figure>

With X-Infrastructure Optimizer, Astera Labs was able to lower its escalating on-demand EC2 instances cost by 50% and achieve a scalable cost-optimized EDA infrastructure in full production. The significant cost savings were achieved by migrating mission-critical workloads 90,103 times between ondemand and spot instances reliably as shown in Figure 2 and Table 1.

<figure><img src="../../.gitbook/assets/image (15).png" alt=""><figcaption><p><em>Table 1, X-Infrastructure Optimizer Performance and Reliability</em></p></figcaption></figure>

### Summary

The utilization of X-Infrastructure Optimizer resulted in the successful running of nearly 20,000 jobs, with a total of 90,103 live migrations performed. Notably, 91% of the total job execution time was in the spot market. This data highlights the critical role of X-Infrastructure Optimizer in ensuring the high reliability of successful execution of a significant majority of jobs. In contrast, running these jobs without XSpot would have led to failure for more than half of them.

The adoption of Exostellar’s X-Infrastructure Optimizer solution by Astera Labs has proven to be a transformative move with significant business implications, resulting in enhanced cash flow and decreased capital expenditure (CapEx) requirements.

**1.** **Cost Reduction:** By optimizing resource utilization and leveraging spot instances, XSpot helps Astera Labs achieve significant cost reductions in its AWS spending. Exostellar achieved cost reductions of up to 50% for on-demand compute resources which directly translates into lower expenses for Astera Labs, allowing the company to allocate its financial resources more efficiently. This also allowed Astera Labs to run more workloads in a shorter time period for the same cost as before, thus accelerating the chip development process and reducing time to market.

**2. Operational Expenditure (OpEx) Model:** X-Infrastructure Optimizer operates on an OpEx model, which means that Astera Labs pays for the cloud resources it uses on a consumption basis. This model eliminates the need for upfront capital investments and long-term commitments associated with buying reserved instances (or compute savings plans) from public cloud providers (AWS/ Azure/ GCP). By shifting to an OpEx model, Astera Labs can reduce its CapEx requirements and free up cash flow that would have otherwise been tied to infrastructure investments.

**3. Scalability without Capital Investment:** Astera Labs can scale its compute resources without the need for additional capital investments. X-Infrastructure Optimizer’s ability to seamlessly migrate workloads between on-demand and spot instances enables Astera Labs to increase its compute capacity by up to 2x within its existing budget. This scalability allows Astera Labs to meet the fluctuating demands of its projects.
