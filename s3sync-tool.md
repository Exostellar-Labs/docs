# s3sync-tool

## Syncing a Log Directory to an S3 Bucket using AWS CLI v2 and Cron

In this guide, we will walk through the process of setting up a script to periodically sync a local log directory to an S3 bucket using the AWS CLI v2 and a cron job.

### Prerequisites
- AWS CLI v2 installed on your system. You can find the installation instructions [here](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- A S3 bucket already created in the target AWS account.
- The source folder on your local system that you want to sync with the S3 bucket.
- The instance running the script should have the IAM permissions in its IAM role: `s3:ListBucket` and `s3:PutObject`.

### Setting up the Script
Open the `env.cfg` file and edit the following environment variables to match your setup:
- `S3_BUCKET`: the name of the destination S3 bucket.
- `SOURCE_FOLDER`: the path of the log folder on your local system that you want to sync with the S3 bucket.
- `CRON_FREQUENCY`: the frequency in minutes at which you want to run the sync job. The default is 60 minutes.

### Running the Script
Run the script with the following command:
```bash
./set_cron_job.sh
```
The script will add a cron job to run the sync periodically based on the frequency specified in the environment variable.


### Monitoring the Script
The job is not added by `crontab` hence it will not show up in the output of `crontab -l` command. To check the status of the cron job, you can use the command `ls -l /etc/cron.d/` to verify that the file is present and has the correct permissions.

You can also check the cron log to make sure `sync_log.sh` script is executed with the specified frequency.

