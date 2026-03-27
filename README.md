# AWS Cloud Cost Optimization – Identifying & Deleting Stale EBS Snapshots

## 📌 Overview

This project helps reduce **AWS storage costs** by automatically identifying and deleting **stale EBS snapshots** that are no longer associated with any active EC2 instance.

Over time, unused snapshots accumulate and increase your AWS bill. This Lambda-based solution scans your account and removes unnecessary snapshots safely.

---

## 🎯 Problem Statement

EBS snapshots continue to incur **storage costs** even when:

* The original EC2 instance is terminated
* The EBS volume is deleted
* Snapshot is no longer required
* Snapshot is orphaned (not attached to any instance)

Manually finding these snapshots is difficult at scale. This automation solves that.

---

## ✅ Solution

This AWS Lambda function:

1. Fetches all **EBS snapshots** owned by the account
2. Retrieves all **active EC2 instances** (running + stopped)
3. Maps snapshots → volumes → instances
4. Identifies **stale snapshots**
5. Deletes unused snapshots automatically
6. Logs deleted snapshot IDs

---

## 🏗 Architecture

EventBridge (Cron Schedule)
↓
AWS Lambda Function
↓
EC2 API (Describe Instances)
↓
EBS API (Describe Snapshots)
↓
Delete Stale Snapshots

---

## ⚙️ Prerequisites

* AWS Account
* AWS Lambda (Python 3.9+)
* IAM Role with EC2 permissions
* EventBridge (optional for scheduling)

---

## 🔐 IAM Role Policy

Attach this policy to Lambda execution role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "ec2:DescribeSnapshots",
        "ec2:DeleteSnapshot",
        "ec2:DescribeInstances",
        "ec2:DescribeVolumes"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

---

## 🧠 Lambda Function Logic

* Get all snapshots owned by account
* Get all active EC2 instances
* Extract attached volume IDs
* Compare snapshot volume IDs
* Delete snapshots not linked to active instances

---

## 🧾 Lambda Code (Python)

```python
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')

    # Get all EBS snapshots
    response = ec2.describe_snapshots(OwnerIds=['self'])

    # Get all active EC2 instance IDs
    instances_response = ec2.describe_instances(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
    active_instance_ids = set()

    for reservation in instances_response['Reservations']:
        for instance in reservation['Instances']:
            active_instance_ids.add(instance['InstanceId'])

    # Iterate through each snapshot and delete if it's not attached to any volume or the volume is not attached to a running instance
    for snapshot in response['Snapshots']:
        snapshot_id = snapshot['SnapshotId']
        volume_id = snapshot.get('VolumeId')

        if not volume_id:
            # Delete the snapshot if it's not attached to any volume
            ec2.delete_snapshot(SnapshotId=snapshot_id)
            print(f"Deleted EBS snapshot {snapshot_id} as it was not attached to any volume.")
        else:
            # Check if the volume still exists
            try:
                volume_response = ec2.describe_volumes(VolumeIds=[volume_id])
                if not volume_response['Volumes'][0]['Attachments']:
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as it was taken from a volume not attached to any running instance.")
            except ec2.exceptions.ClientError as e:
                if e.response['Error']['Code'] == 'InvalidVolume.NotFound':
                    # The volume associated with the snapshot is not found (it might have been deleted)
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as its associated volume was not found.")
```

---

## ⏰ Schedule Using EventBridge (Optional)

Run Lambda daily to automatically clean stale snapshots.

### Rate Expression

```
rate(1 day)
```

### Cron Expression (2 AM UTC daily)

```
cron(0 2 * * ? *)
```

---

## 💰 Cost Savings

Unused snapshots silently increase AWS bill.

Example:

| Snapshots     | Size         | Monthly Cost |
| ------------- | ------------ | ------------ |
| 50            | 100 GB each  | ~$250/month  |
| After Cleanup | 10 snapshots | ~$50/month   |

Estimated Savings: **$200/month** 🚀

---

## 📊 Logging

Lambda logs will show:

* Snapshots scanned
* Snapshots deleted
* Errors (if any)

Example:

```
Deleting stale snapshot: snap-123abc
Deleting stale snapshot: snap-456def
```

---

## ⚠️ Important Notes

* Only deletes snapshots NOT linked to active instances
* Does NOT delete snapshots of running instances
* Test in dev account first
* Add tag filtering for production safety

---

## 🛡 Recommended Safety Enhancement

Delete only snapshots with tag:

```
AutoDelete = true
```

This prevents accidental deletion.

---

## 🚀 Future Improvements

* Slack notification
* Email alerts using SNS
* Tag-based filtering
* Dry-run mode
* Cost savings report
* Multi-region support
* Multi-account support

---

## 🧪 Testing Steps

1. Create EC2 instance
2. Create snapshot
3. Terminate EC2 instance
4. Run Lambda
5. Verify snapshot deleted

---

## 📁 Project Structure

```
aws-stale-snapshot-cleaner/
│
├── lambda_function.py
├── README.md
└── iam-policy.json
```

---

## 👨‍💻 Author

**Nikhil Gupta**
AWS Cloud & DevOps Engineer

AWS Cost Optimization • Lambda Automation • Cloud Infrastructure

---

## ⭐ Support

If this project helped you, consider giving it a ⭐ on GitHub.
