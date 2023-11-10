# Learning Compute Optimizer Command Line Interface (Compute Optimizer CLI)

```
Usage:
   xspot {flags}
   xspot <command> {flags}
```

| Flag            |                                                                             |
| --------------- | --------------------------------------------------------------------------- |
| `-v, --version` | Displays version number (default: false)                                    |
| `-h, --help`    | Displays usage information of the application or a command (default: false) |

| Command              | Explanation                                                                                                                       |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `add`                | Add one more worker into the cluster                                                                                              |
| `check`              | Verify Compute Optimizer prerequisites and installation (default=prereqs only)                                                               |
| `console`            | Show console for a sandbox                                                                                                        |
| `create-vm`          | Creates a VM                                                                                                                      |
| `help`               | Displays usage information                                                                                                        |
| `license`            | Print license status                                                                                                              |
| `migrate`            | Migrate a container                                                                                                               |
| `migrate-cancel`     | Cancel on-going migration of a container                                                                                          |
| `ps`                 | Print out the current status of the system                                                                                        |
| `regenerate-pricing` | Regenerate a pricing file                                                                                                         |
| `restart`            | Restart the Compute Optimizer service                                                                                                        |
| `rm`                 | Remove a worker in the cluster                                                                                                    |
| `rmc`                | Force remove a container                                                                                                          |
| `rms`                | Deletes a sandbox                                                                                                                 |
| `scheduler`          | Turn on/off the scheduler (default=off)                                                                                           |
| `size`               | Get/Set default container size (CPU and memory)                                                                                   |
| `top`                | Show resource utilization of a worker                                                                                             |
| `updateconfig`       | Update configuration parameters: scheduler\_enabled, ami, booting\_timeout, on\_demand\_types, spot\_fleet\_types, reload\_config |
| `version`            | Displays version number                                                                                                           |
