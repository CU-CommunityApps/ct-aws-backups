# Overview of stackset and template yml

### deploy-params.yml

`template-file-path: cloudformation/stackset.yml`: This points to the stackset.yml which is used for the resource deployments

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
          Value: "ct-gitsync-awsbackup1"
        - Key: "cit:backup-scheme"
          Value: "default"
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


| **Attribute** | **Value** | **Notes** |
|:-:|:-:|:-:|
| Type | AWS::EC2::Instance | The AWS resource type. |
| `DeletionPolicy` | `Delete` | Specifies the instance is deleted on removal. |
| `InstanceType` | `t2.micro` | Specifies the instance type. |
| `ImageId` | `!FindInMap [RegionMap, !Ref "AWS::Region", AMI]` | Maps the AMI based on the region. |
| Tags - `Name` | `ct-gitsync-awsbackup1` | The name tag for the instance. |
| Tags - `cit:backup-scheme`| `default` | Backup scheme tag value. |
| - | - | - |
| BlockDeviceMappings - `DeviceName` | `"/dev/sdh"` | Specifies the device name for block device mapping. |
| BlockDeviceMappings - `VolumeSize` | `10` | Specifies the volume size (in GB) for the EBS volume. |
| BlockDeviceMappings - `DeleteOnTermination` | `true` | Indicates the EBS volume will be deleted on instance termination. |
| BlockDeviceMappings - `VolumeType` | `gp2` | Specifies the volume type of the EBS volume. |

---

#### AWS Backup Vault

```yml
  citVault:
    Type: AWS::Backup::BackupVault
    UpdateReplacePolicy: Retain
    DeletionPolicy: Retain
    Properties:
      BackupVaultName: citVault
```

This section creates a vault within AWS Backup service. The [UpdateReplacePolicy attribute](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-updatereplacepolicy.html) = `retain` to verify if the vault is deleted that any backups are not lost accidentally. Normally any vault would not allow for deletion if the vault contains *any* restore points but this is configured as a percaution. 

:warning: ***<ins>Attention:</ins>*** **If a vault is to be deleted, make sure to manually clean up the vaults and restore points left by this deployment.**

| **Attribute** |  **Value** | **Available Values** | **Notes** |
|:-:|:-:|:-:|:-:|
| `Type` | `AWS::Backup::BackupVault` | The AWS resource type for creating a backup vault. |
| `UpdateReplacePolicy:` | `Retain` | `Retain`, `Delete`, `Snapshot` | [AWS Documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-updatereplacepolicy.html) Determines the policy to apply when the resource is replaced. |
| `DeletionPolicy:` | `Retain` | `Retain`, `Delete`, `Snapshot`, `RetainExceptOnCreate` | [AWS Documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html) Determines the policy to apply when the resource is deleted. |
| `BackupVaultName:` | `citVault` | This is the name used when creating the Vault in AWS Backup Service. |

---

#### AWS Backup Plan

```yml
  citBackupPlan:
    Type: AWS::Backup::BackupPlan
    Properties:
      BackupPlan:
        BackupPlanName: citBackupPlan
        BackupPlanRule:
          - RuleName: DailyBackup
            TargetBackupVault: !Ref citVault
            ScheduleExpression: cron(0 9 * * ? *)  # Daily at 04:00 AM EST (09:00 PM UTC)
            StartWindowMinutes: 60 
            CompletionWindowMinutes: 120
            Lifecycle:
              DeleteAfterDays: 7  # Retain backup for 7 days
```
This sets the scheduling time, time for backup start, time allowed to complete backup and how long to keep the backups for. It is important to note

| **Attribute** | **Value** | **Notes** |
|:-:|:-:|:-:|
| `Type` | `AWS::Backup::BackupPlan` | The AWS resource type for creating a backup plan. |
| `BackupPlanName` | `citBackupPlan` | The name of the backup plan. |
| `RuleName` | `DailyBackup` | The name given to the backup rule within the plan. |
| `TargetBackupVault` | `!Ref citVault` | Reference to the backup vault where backups are stored. |
| `ScheduleExpression` | `cron(0 9 * * ? *)` | Defines the backup schedule using cron expression. Daily at 09:00 UTC (04:00 AM EST). |
| `StartWindowMinutes` | `60` | The backup operation must start within 60 minutes of the scheduled time. |
| `CompletionWindowMinutes` | `120` | The backup operation must complete within 120 minutes after the start window begins. |
| Lifecycle - `DeleteAfterDays`| `7` | Specifies the number of days after which the backup is to be deleted. |

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
      BackupPlanId: !Ref citBackupPlan
      BackupSelection:
        SelectionName: AllInstancesSelection
        IamRoleArn: !GetAtt citIAMrole.Arn
        ListOfTags:
          - ConditionType: STRINGEQUALS
            ConditionKey: "cit:backup-scheme"
            ConditionValue: "default"
```
| **Attribute** | **Value** | **Notes** |
|:-:|:-:|:-:|
| `Type` | `AWS::Backup::BackupSelection` | The AWS resource type for creating a backup selection. |
| `BackupPlanId` | `!Ref citBackupPlan` | Reference to the backup plan ID this selection is associated with. |
| `IamRoleArn` | `!GetAtt citIAMrole.Arn` | The ARN of the IAM role that AWS Backup uses to authenticate when backing up the selected resources. |
| `ConditionType` | `STRINGEQUALS` | The type of condition applied to the AWS resource tags for selection. |
| `ConditionKey` | `"cit:backup-scheme"` | The key of the tag for which the condition is applied. |
| `ConditionValue` | `"default"` | The value of the tag that must be met for the condition to apply. |


This configuration tells the backup service what resources to target for the backup. Using this Cloudformation Gitsync process, the target can be ARNs or Tags.

---
