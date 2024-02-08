# ct-aws-backups
Repository used for creating aws backups.


 ## :warning: Attention :warning:
> :bulb: **P.O.C. Repo:**
>
> **This Repository is a Work in Progress and will be used a testing repo until the P.O.C. of the process has been completed.**

The purpose of this process is to give a repo to a group and connect Cloudformation gitsync to this repository for an automated process for backing up EC2 instances with the Tag `cit:backup-scheme` = `default` for all instances.

---

#### Change Log:
- 2024.02.07: Updating with process documentation for implementing via Cloudformation Gitsync

---

Resource Links:
- [Cornell AWS Service Information](https://confluence.cornell.edu/display/CLOUD/AWS+Backup+Service+Information): Documentation based on POC
- [Step-by-Step Guide for Deploying AWS Backup Service](step-by-step/README.md): How to guide
- [AWS Backup pricing](https://aws.amazon.com/backup/pricing/): Pay by storage used (*Warm vs Cold Storage & Restores*). This is a GB per month cost (*No AWS Backup Service costs*).
- [What is AWS Backup?](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html#features-by-resource): Additional Information about the capabilities of the AWS Service
