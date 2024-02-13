# Overview of stackset and template yml

### deploy-params.yml

:warning: **Deployment Parameters:** This file is the first set of parameters to update for the environment. These will be passed to the stack deployment and overwrite any default values originally set in the stack deployment. If there is a need to change anything within the `cloudformation/stackset.yml`, there should be a basic knowledge of Cloudformation and how stacks are creating AWS Services.

```yml
template-file-path: cloudformation/stackset.yml
parameters:
  VersionParam: 1.0.0
  BackupVaultName: backupVault # Name of the backup vault
  BackupPlanName: backupPlan # Name of the backup plan
  BackupPlanRuleName: DailyBackup # Name of Backup rule name
  BackupRunTime: cron(0 9 * * ? *) # Daily at 04:00 AM EST (09:00 PM UTC)
  BackupStartWindow: 60 # A value in minutes the backup must start by.
  BackupCompletionWindow: 120 # number of minutes to complete backup
  BackupRetention: 7 # retain for number of days
  BackupTagName: "cit:backup-scheme" # Key value of Tag
  BackupTagValue: "default" # Value of Tag set for instance with value to be backed up
tags: {}
```

#### Default Parameter Values

| **Parameter** | **Default Value** | **Notes** |
|:-:|:-:|:-:|
| VersionParam | 1.0.0 | |
| `BackupVaultName` | `backupVault` | Name of the backup vault |
| `BackupPlanName` | `backupPlan` | Name of the backup plan |
| `BackupPlanRuleName` | `DailyBackup` | Name of Backup rule name |
| `BackupRunTime` | `cron(0 9 * * ? *)` | Daily at 04:00 AM EST (09:00 PM UTC) |
| `BackupStartWindow` | `60` | A value in minutes the backup must start by. |
| `BackupCompletionWindow` | `120` | Number of minutes to complete backup |
| `BackupRetention` | `7` | Retain for number of days |
| `BackupTagName` | `"cit:backup-scheme"` | Key value of Tag |
| `BackupTagValue` | `"default"` | Value of Tag set for instance with value to be backed up |

---

### stackset.yml

#### Mappings

```yml
Mappings: 
  RegionMap:
    us-east-1:
      AMI: ami-0c7217cdde317cfec # Ubuntu Server 22.04 LTS (HVM),EBS General Purpose (SSD) Volume Type
```

This is used to target a specific AMI being used when deploying the test EC2 instance and what AMI to use for deployment.

---

#### EC2

```yml
  MyEC2Instance1:
    Type: AWS::EC2::Instance
    DeletionPolicy: Delete
    Properties:
      InstanceType: t2.micro
      ImageId: !FindInMap [RegionMap, !Ref "AWS::Region", AMI]
      Tags:
        - Key: "Name"
          Value: "ec2backupTestInstance"
        - Key: !Ref BackupTagName
          Value: !Ref BackupTagValue
        - Key: Environment
          Value: !Ref EnvironmentParam
        - Key: Description
          Value: "Ubuntu EC2 Instance for Testing"
        - Key: Documentation
          Value: !Ref DocumentationURLParam
        - Key: cit:contact-email
          Value: !Ref ContactEmailParam
        - Key: cit:version
          Value: !Ref VersionParam
        - Key: cit:source
          Value: !Ref SourceURLParam
      BlockDeviceMappings:
        - DeviceName: "/dev/sdh"
          Ebs:
            VolumeSize: 10
            DeleteOnTermination: true
            VolumeType: gp2
```

This code is originally commented out so that the initial deployment does not create a new EC2 instance. This section of the code can be uncommented and will allow for a small EC2 instance to be deployed with the `cit:backup-scheme`=`default` tag.

:warning: ***<ins>ATTENTION:</ins>*** This block is commented out and will not be deployed unless added to the used configuration by uncommenting this block. This is defaulted to not deploy due to this being a small test instance. 
> **To uncomment out this block remove all `# ` before each line in the block of configurations shown above for `MyEC2Instance1` inside of `stackset.yml`.**

| **Attribute** | **Value** | **Notes** |
|:-:|:-:|:-:|
| Type | AWS::EC2::Instance | The AWS resource type. |
| `DeletionPolicy` | `Delete` | Specifies the instance is deleted on removal. |
| `InstanceType` | `t2.micro` | Specifies the instance type. |
| `ImageId` | `!FindInMap [RegionMap, !Ref "AWS::Region", AMI]` | Maps the AMI based on the region. |
| Tags - `Name` | `ec2backupTestInstance` | The name tag for the instance. |
| Tags - `!Ref BackupTagName` | `!Ref BackupTagValue` | Backup scheme tag value. (Parameterized Default Values: `cit:backup-scheme`=`default`) |
| - | - | - |
| BlockDeviceMappings - `DeviceName` | `"/dev/sdh"` | Specifies the device name for block device mapping. |
| BlockDeviceMappings - `VolumeSize` | `10` | Specifies the volume size (in GB) for the EBS volume. |
| BlockDeviceMappings - `DeleteOnTermination` | `true` | Indicates the EBS volume will be deleted on instance termination. |
| BlockDeviceMappings - `VolumeType` | `gp2` | Specifies the volume type of the EBS volume. |

---

#### AWS Backup Vault

```yml
  Vault:
    Type: AWS::Backup::BackupVault
    UpdateReplacePolicy: Retain
    DeletionPolicy: Retain
    Properties:
      BackupVaultName: !Ref BackupVaultName
      # You can also set encryption settings and access policies here
```

This section creates a vault within AWS Backup service. The [UpdateReplacePolicy attribute](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-updatereplacepolicy.html) = `retain` to verify if the vault is deleted that any backups are not lost accidentally. Normally any vault would not allow for deletion if the vault contains *any* restore points but this is configured as a percaution. 

:warning: ***<ins>Attention:</ins>*** **If a vault is to be deleted, make sure to manually clean up the vaults and restore points left by this deployment.**

| **Attribute** |  **Value** | **Available Values** | **Notes** |
|:-:|:-:|:-:|:-:|
| `Type` | `AWS::Backup::BackupVault` | The AWS resource type for creating a backup vault. |
| `UpdateReplacePolicy:` | `Retain` | `Retain`, `Delete`, `Snapshot` | [AWS Documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-updatereplacepolicy.html) Determines the policy to apply when the resource is replaced. |
| `DeletionPolicy:` | `Retain` | `Retain`, `Delete`, `Snapshot`, `RetainExceptOnCreate` | [AWS Documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html) Determines the policy to apply when the resource is deleted. |
| `BackupVaultName:` | `!Ref BackupVaultName` | This is the name used when creating the Vault in AWS Backup Service. (Parameterized Default Name: `backupVault`) |

---

#### AWS Backup Plan

```yml
  BackupPlan:
    Type: AWS::Backup::BackupPlan
    Properties:
      BackupPlan:
        BackupPlanName: !Ref BackupPlanName
        BackupPlanRule:
          - RuleName: !Ref BackupPlanRuleName
            TargetBackupVault: !Ref Vault
            ScheduleExpression: !Ref BackupRunTime
            StartWindowMinutes: !Ref BackupStartWindow
            CompletionWindowMinutes: !Ref BackupCompletionWindow
            Lifecycle:
              DeleteAfterDays: !Ref BackupRetention
```

This sets the scheduling time, time for backup start, time allowed to complete backup and how long to keep the backups for. It is important to note

| **Attribute** | **Value** | **Default Value** | **Notes** |
|:-:|:-:|:-:|:-:|
| `Type` | `AWS::Backup::BackupPlan` | | The AWS resource type for creating a backup plan. |
| `BackupPlanName` | `!Ref BackupPlanName` | `backupPlan` | The name of the backup plan. |
| `RuleName` | `!Ref BackupPlanRuleName` | `DailyBackup` | The name given to the backup rule within the plan. |
| `TargetBackupVault` | `!Ref Vault` | `backupVault` | Reference to the backup vault where backups are stored. |
| `ScheduleExpression` | `!Ref BackupRunTime` | `cron(0 9 * * ? *)` | Defines the backup schedule using cron expression. Daily at 09:00 UTC (04:00 AM EST). |
| `StartWindowMinutes` | `!Ref BackupStartWindow` | `60` | The backup operation must start within 60 minutes of the scheduled time. |
| `CompletionWindowMinutes` | `!Ref BackupCompletionWindow` | `120` | The backup operation must complete within 120 minutes after the start window begins. |
| Lifecycle - `DeleteAfterDays`| `!Ref BackupRetention` | `7` | Specifies the number of days after which the backup is to be deleted. |

---

#### AWS Backup Role

```yml
  citIAMrole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: backup.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: BackupRolePolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - ec2:DescribeVolumes
                  - ec2:DescribeSnapshots
                  - ec2:CreateTags
                  - ec2:CreateSnapshot
                  - ec2:DeleteSnapshot
                  - ec2:DescribeTags
                  - ec2:DescribeImages
                  - ec2:DescribeInstances
                  - ec2:DescribeVolumes
                  - ec2:CreateImage
                  - ec2:DescribeInstanceCreditSpecifications
                  - ec2:DescribeInstanceAttribute
                  - ec2:DescribeSnapshotTierStatus
                  - ec2:RestoreSnapshotTier
                  - ec2:ModifySnapshotTier
                  - ec2:DescribeNetworkInterfaces
                  - tag:GetResources
                Resource: "*"
```

This is the role created and used by the AWS Backup service to perform the backup actions.

| **Attribute** | **Value** | **Notes** |
|:-:|:-:|:-:|
| `Type` | `AWS::IAM::Role` | The AWS resource type for creating an IAM role. |
| `Action` | `sts:AssumeRole` | Action allowed for the principal service. |
| `PolicyName` | `BackupRolePolicy` | Name of the policy attached to the role. |
| `Actions Allowed` | Various EC2 and tag actions | Includes actions like `ec2:DescribeVolumes`, `ec2:CreateSnapshot`, etc. |
| `Resource` | `"*"` | Policy applies to all resources. |

---

#### AWS Backup Selection

```yml
BackupSelection:
    Type: AWS::Backup::BackupSelection
    Properties:
      BackupPlanId: !Ref BackupPlan
      BackupSelection:
        SelectionName: AllInstancesSelection
        IamRoleArn: !GetAtt IAMrole.Arn
        ListOfTags:
          - ConditionType: STRINGEQUALS
            ConditionKey: !Ref BackupTagName
            ConditionValue: !Ref BackupTagValue
```

| **Attribute** | **Value** | **Default Value** | **Notes** |
|:-:|:-:|:-:|:-:|
| `Type` | `AWS::Backup::BackupSelection` | | The AWS resource type for creating a backup selection. |
| `BackupPlanId` | `!Ref BackupPlan` | `backupPlan` | Reference to the backup plan ID this selection is associated with. |
| `IamRoleArn` | `!GetAtt citIAMrole.Arn` | | The ARN of the IAM role that AWS Backup uses to authenticate when backing up the selected resources. |
| `ConditionType` | `STRINGEQUALS` | | The type of condition applied to the AWS resource tags for selection. |
| `ConditionKey` | `!Ref BackupTagName` | `"cit:backup-scheme"` | The key of the tag for which the condition is applied. |
| `ConditionValue` | `!Ref BackupTagValue` | `"default"` | The value of the tag that must be met for the condition to apply. |


This configuration tells the backup service what resources to target for the backup. Using this Cloudformation Gitsync process, the target can be ARNs or Tags.

---
