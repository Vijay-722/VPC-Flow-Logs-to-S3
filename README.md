# Configure VPC Flow Logs and Store Logs in S3 Using IAM Role

## 📌 Project Overview
This project configures **AWS VPC Flow Logs** to capture network traffic metadata from a VPC and store it in an **Amazon S3 bucket** for long-term analysis and auditing. Proper access permissions are set using **IAM roles and S3 bucket policies**.  

The setup helps in:
- Monitoring traffic for security and compliance
- Auditing rejected/accepted connections
- Long-term archival of network logs

---

## 🎯 Objectives
- Enable VPC Flow Logs at the **VPC level**.
- Store logs in a dedicated **S3 bucket**.
- Configure **IAM role/bucket policy** for secure log delivery.
- Verify logs by generating traffic and analyzing captured entries.

---

## ⚙️ Setup Steps

### 1. **VPC & Subnet Setup**
- Created a **VPC** with CIDR block (e.g., `10.0.0.0/16`).
- Created a **public subnet** (`10.0.1.0/24`) and attached it to the VPC.
- Associated subnet with a **route table**.
- Added a route `0.0.0.0/0` → **Internet Gateway** for Internet access.

### 2. **EC2 Instance Setup**
- Launched an EC2 instance inside the subnet.
- Enabled **Auto-assign Public IP**.
- Configured Security Group to allow:
  - Inbound SSH (22)
  - Outbound Internet traffic

### 3. **S3 Bucket Setup**
- Created S3 bucket:  
- Name: `vpc-flow-logs-bucket-<unique-id>`
- Region: same region as your VPC.
- Block Public Access: keep enabled.
- Applied **bucket policy** to allow VPC Flow Logs service (`delivery.logs.amazonaws.com`) to put objects:

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "1",
			"Effect": "Allow",
			"Principal": {
				"AWS": "arn:aws:iam::340752812821:role/VPCFlowLogsToS3Role"
			},
			"Action": "s3:PutObject",
			"Resource": "arn:aws:s3:::vpc-flow-logs-bucket-340752812821/*"
		},
		{
			"Sid": "AWSLogDeliveryWrite1",
			"Effect": "Allow",
			"Principal": {
				"Service": "delivery.logs.amazonaws.com"
			},
			"Action": "s3:PutObject",
			"Resource": "arn:aws:s3:::vpc-flow-logs-bucket-340752812821/AWSLogs/340752812821/*",
			"Condition": {
				"StringEquals": {
					"aws:SourceAccount": "340752812821",
					"s3:x-amz-acl": "bucket-owner-full-control"
				},
				"ArnLike": {
					"aws:SourceArn": "arn:aws:logs:us-east-1:340752812821:*"
				}
			}
		},
		{
			"Sid": "AWSLogDeliveryAclCheck1",
			"Effect": "Allow",
			"Principal": {
				"Service": "delivery.logs.amazonaws.com"
			},
			"Action": "s3:GetBucketAcl",
			"Resource": "arn:aws:s3:::vpc-flow-logs-bucket-340752812821",
			"Condition": {
				"StringEquals": {
					"aws:SourceAccount": "340752812821"
				},
				"ArnLike": {
					"aws:SourceArn": "arn:aws:logs:us-east-1:340752812821:*"
				}
			}
		}
	]
}
```

### 4. IAM Role Setup

- Go to IAM → Roles → Create Role.
- Choose AWS Service → Select VPC Flow Logs.
- Add Custom Policy → allow s3 write.
- Applied **IAM Policy**

✅ IAM Policy(JSON)

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowPutToS3",
            "Effect": "Allow",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::vpc-flow-logs-bucket-340752812821/*"
        }
    ]
}
```

- Applied **IAM Trust Relatinoship**

✅ IAM Trust Relationship Policy(JSON)

```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "TrustVPCFlowLogs",
			"Effect": "Allow",
			"Principal": {
				"Service": "vpc-flow-logs.amazonaws.com"
			},
			"Action": "sts:AssumeRole",
			"Condition": {
				"StringEquals": {
					"aws:SourceAccount": "340752812821"
				}
			}
		}
	]
}
```

- Attach this role to VPC Flow Logs when creating them.

### 5. Enable VPC Flow Logs

- Go to VPC → Your VPCs → Select VPC → Flow Logs tab → Create Flow Log.
- Configured:
  - Filter (Traffic Type): All (captures accept & reject traffic).
  - Destination: Send to S3 bucket.
  - S3 bucket ARN: `arn:aws:s3:::vpc-flow-logs-bucket-340752812821`
  - IAM Role: Select the role created above.
  - Resource: VPC (recommended over subnet/ENI).
- Flow log state changed to Active.

### 6. Verification

- Connected to EC2 and generated traffic:
```
ping -c 4 8.8.8.8
curl https://aws.amazon.com
```
- After ~10 minutes, checked S3 bucket:
```
AWSLogs/<account-id>/vpcflowlogs/<region>/<year>/<month>/<day>/
```
- Found .gz log files.

### 7. Log Structure

```
2 123456789012 eni-0a1b2c3d4e5f6g7h 192.168.1.10 8.8.8.8 443 33456 6 10 840 1673602345 1673602405 ACCEPT OK
```

- 2 → Log format version

- 123456789012 → AWS account ID

- eni-0a1b2c3d4e5f6g7h → Network Interface ID

- 192.168.1.10 → Source IP (EC2 instance)

- 8.8.8.8 → Destination IP (Google DNS)

- 443 → Destination port (HTTPS)

- 33456 → Source port

- 6 → Protocol (6 = TCP)

- 10 → Packet count

- 840 → Byte count

- 1673602345–1673602405 → Start and end time (epoch)

- ACCEPT → Action taken (accepted/rejected)

- OK → Log status

---

## 🔑 Key Learnings

- VPC Flow Logs require subnets/ENIs to exist, otherwise no logs are generated.

- For S3 destinations, only a bucket policy is required.

- Logs are not immediate — allow 5–15 minutes before checking.

- Flow Logs capture metadata only (not packet contents).

---

## 📜 Deliverables

✅ IAM role JSON policy and trust relationship.

✅ S3 bucket policy JSON (for S3 destination).

✅ Screenshots: VPC Flow Log config, S3 bucket folder with logs.

✅ Sample log explained.

✅ This README.md
