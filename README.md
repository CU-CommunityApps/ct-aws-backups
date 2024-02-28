# ct-aws-backups

The purpose of this process is to give a repo to a group and connect Cloudformation gitsync to this repository for an automated process for backing up EC2 instances with the Tag `cit:backup-scheme` = `default` for all instances.

---

#### Change Log:
- 2024.02.28: Created Public Access & Updated Full service offering documentation link
- 2024.02.14: Announced Service Offering
- 2024.02.08: Update Step-by-Step, Removed Test Instance
- 2024.02.07: Updating with process documentation for implementing via Cloudformation Gitsync

---

#### Resource Links:
- [Cornell AWS Service Information](https://confluence.cornell.edu/display/CLOUD/AWS+Backup+Service+via+Service+Catalog+or+Git+Sync): Full Documentation of service offering
- [Step-by-Step Guide for Deploying AWS Backup Service](step-by-step/README.md): How to guide
- [AWS Backup pricing](https://aws.amazon.com/backup/pricing/): Pay by storage used (*Warm vs Cold Storage & Restores*). This is a GB per month cost (*No AWS Backup Service costs*).
- [What is AWS Backup?](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html#features-by-resource): Additional Information about the capabilities of the AWS Service

---

#### Suggested Repo Use
*Due to the way Gitsync polls the repo branch for commit changes this is the suggested way to create and run this repo.*

- *Default Branch*: `deploy` This branch will be used to trigger the deployment.
- *Stable Branch*: `master` or `main` This should be used for stable deployments. Once a deployment succeeds this branch should have the latest successful deployment. This branch is considered stable so it can be merged into deploy if a deployment fails.

The [Step-by-Step Guide](step-by-step/README.md) has this process for creating the repo in this way and locks the repo down to require a Pull request being created from a branch off of `deploy` branch. This follows the gitflow process requiring 1 review of the Pull Request to merge and trigger the deployment.

---

#### Test Instance (_Commented Code_)

*:warning: Normally commented code is not left within a configuration but this code can be useful for testing.*

Commented code within [stackset.yml](cloudformation/stackset.yml) `MyEC2Instance1` was left for anyone wanting to fork this repo and create a test EC2 instance with the appropriate tag.
- Tag Name: `cit:backup-scheme`
- Tag Value: `default`

---
