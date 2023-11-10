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

# Integrating Compute Optimizer Controller and Worker

Please login as _root_ into the Compute Optimizer Controller EC2 and tune configs in `/etc/xspot/config.toml`:

| Section   | Config               | Explanation                                                  |
| --------- | -------------------- | ------------------------------------------------------------ |
| \[aws]    | _ami_                | The created Compute Optimizer Worker AMI ID.                            |
| \[worker] | _on\_demand\_types_  | List of instance types and families for on-demand instances. |
| \[worker] | _spot\_fleet\_types_ | List of instance types and families for spot instances.      |
