---
description: Demonstrating what Compute Optimizer can do for a high-performance application
cover: >-
  https://images.unsplash.com/photo-1621508727844-09cc245de555?crop=entropy&cs=srgb&fm=jpg&ixid=M3wxOTcwMjR8MHwxfHNlYXJjaHwxfHxzdGVsbGFyfGVufDB8fHx8MTY5NjI3ODUwN3ww&ixlib=rb-4.0.3&q=85
coverY: 0
---

# Astronomical increase in compute for same cloud spend in DOE demonstration

{% hint style="info" %}
**By Taylor Reo, Ashbocker, Nathan Lee, Woodruff, Christopher S Ritter, Originally posted on** [**osti.gov**](https://www.osti.gov/biblio/1860639-exotanium-doe-sbir-phase-results-summary)**.**

**Technical Report –** [https://inldigitallibrary.inl.gov/sites/sti/sti/Sort\_59631.pdf](https://inldigitallibrary.inl.gov/sites/sti/sti/Sort\_59631.pdf)
{% endhint %}

### Introduction

Exostellar demonstrated this technology to the Department of Energy (DOE) with the Idaho National Laboratory’s MASTODON application, a Multiphysics environment designed to run typical high-performance computing (HPC) simulations for structural dynamics, seismic analysis, and risk assessment. The MASTODON application was packaged into a container using Docker, Deployed on Amazon ECS, and managed through a custom Scale-Out Compute on AWS (SOCA) implementation.

Exostellar has developed novel technologies that dynamically allocate idle or over-sized application containers and virtual machines (VMs) to take advantage of deeply discounted server space. The technology addresses the cost-saving needs of its customer segments by leveraging spot market discounts ([Compute Optimizer](https://exostellar.io/solutions/#xspot)), and further, consolidating idle workloads and over-sized application containers and VMs (X-Consolidate).

The first product, Compute Optimizer, spawns application containers on discounted VMs and orchestrates containers between such instances based on availability and price. Spot instances can be reclaimed by the cloud provider at any time with very short notice; so traditionally, containers can only use these instances for workloads that tolerate interruption or failure. Compute Optimizer dramatically improves the spot-instance usability for critical applications, however, with comparable reliability and cost reductions upwards of 70% compared to regular on-demand, or native, instances.

### Application

Exostellar demonstrated this technology with the Idaho National Laboratory’s MASTODON application, a Multiphysics environment designed to run typical high-performance computing (HPC) simulations for structural dynamics, seismic analysis, and risk assessment. The MASTODON application was packaged into a container using Docker, Deployed on Amazon ECS, and managed through a custom Scale-Out Compute on AWS (SOCA) implementation.

### RESULT

The experimental setup utilized a master and worker node configuration on AWS with the m5.large and m5.8xlarge instance types, respectively. This worker node is configured with 32 cores and 128 GiB memory. Both Compute Optimizer and Compute Optimizer Migrate strategies were tested, where the Migrate strategy is distinct for its use of hybrid cloud environments. Additionally, this research has demonstrated the orchestration of containers between multiple cloud providers, at one time beginning a workload on the Google Cloud Platform and ending on AWS.

This experiment produced results in three separate trials, each involving a MASTODON workload of varying intensity. Results are determined by the difference between native and spot-instance runtime and pricing.

First, an Compute Optimizer trial involving a **1-hour workload** yielded a **70% cost reduction** while forfeiting only a **6% overhead**. For Compute Optimizer Migrate, the results were similar with a **69% reduction** at only a **7% overhead**.

<figure><img src="../../.gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>

Next, an Compute Optimizer trial involving a **4-hour workload** yielded a **73% cost reduction** with a **4.2% performance enhancement**. For Compute Optimizer Migrate, the results were similar with a **72% reduction** and a **3.5% performance enhancement**.

<figure><img src="../../.gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>

Finally, an Compute Optimizer trial involving a **16-hour workload** yielded a **73% cost reduction** with a **2.8% performance enhancement**. For Compute Optimizer Migrate, the results were similar with a **72% cost reduction** and a **2.7% performance enhancement**.

<figure><img src="../../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

Further, this research extended the Compute Optimizer platform to support intensive workloads like image processing and machine learning, by creating a peripheral component interconnect (PCI) pass-through to a graphics processing unit (GPU)-based instance. This support enables large, highly compute-intensive Department of Energy applications to reliably run uninterrupted while taking advantage of the spot market’s significant discounts.

Exostellar also developed X-Consolidate, which addresses idle resource waste in the cloud by packing idle containers onto a small number of VMs during the idle period, thereby minimizing the number of active machines and reducing the cost of keeping services online. When the workload increases, X-Consolidate relocates containers onto different VMs, without any service interruption. This enables the ability of stateful applications, like databases or streaming processors, to automatically scale with the workload, reducing the need to over-provision resources to accommodate their peak workloads.

<figure><img src="../../.gitbook/assets/Screenshot 2023-09-25 at 1.13.00 PM.png" alt=""><figcaption></figcaption></figure>

### NEXT STEPS

The next steps include the following. First, expanding the supported cloud environments to include the GovCloud. This will enable more National Laboratory workloads to run using Exostellar’s software products. Second, expand to Microsoft Azure, which will allow more public clouds to be used. Third, evaluating support of interactive workloads where optimizations can be significant, such as reducing the resource consumption and costs of servers that are idle. Lastly, evaluating the feasibility of supporting GPUs based migration that will allow accelerators to benefit from the cloud optimizations that Exostellar provides.
