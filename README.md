
# Ansible with AWS SSM Connection

This repository demonstrates how to use **Ansible** with **AWS Systems Manager (SSM)** to securely manage EC2 instances across multiple AWS accounts **without SSH**.

---


## Youtube: https://www.youtube.com/watch?v=i8ZBZ9Z11w8


## References

- [Ansible AWS SSM Connection Plugin](https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ssm_connection.html#ansible-collections-amazon-aws-aws-ssm-connection)  
- [Ansible AWS Collection Guide](https://github.com/ansible-collections/amazon.aws/blob/main/docs/docsite/rst/aws_ec2_guide.rst)

---

## ⚙️ Installation

```bash
# Enable and install Ansible
sudo amazon-linux-extras enable ansible2
sudo yum install -y ansible

# Verify Ansible
ansible --version

# Install AWS SSM Session Manager plugin
sudo yum install -y https://s3.amazonaws.com/session-manager-downloads/plugin/latest/linux_64bit/session-manager-plugin.rpm

# Check if AWS Ansible collection is installed
ansible-galaxy collection list

# Install Python dependencies
sudo yum install -y python3-pip
pip install boto3 botocore

```

## Infrastructure Setup
1. S3 Bucket

Create an S3 bucket to store artifacts used by the SSM connection plugin.
Add a resource-based policy to allow access from the Ansible role in member accounts:
```
   {
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::xxxx:role/Ansible-member-role"
      },
      "Action": [
        "s3:GetBucketLocation",
        "s3:ListBucket",
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::ansible-ssm-bucket",
        "arn:aws:s3:::ansible-ssm-bucket/*"
      ]
    }
  ]
}
```

### 2. IAM Role – Ansible-control-role (in Master Account)

This role should be **attached to the Ansible Master Node EC2 instance**.

**Policies required:**

- `AmazonSSMManagedInstanceCore`  
- **Assume Role** → `Ansible-member-role` in all target AWS accounts  
- **S3 permissions** → `s3:GetObject`


### 3. IAM Role – Ansible-member-role (in Member Accounts)

This role is **trusted by the Ansible-control-role**.

**Permissions granted (EC2 + SSM + S3):**

- **EC2**:
  - `ec2:DescribeInstances`
  - `ec2:DescribeTags`
  - `ec2:DescribeRegions`

- **SSM**:
  - `ssm:SendCommand`
  - `ssm:StartSession`
  - `ssm:DescribeInstanceInformation`

- **S3**:
  - `s3:GetObject`
  - `s3:PutObject`



## 🖥️ Important Commands

```bash
# List inventory
ansible-inventory -i acc1_aws_ec2.yml --list
# (Note: file extension MUST be .yml, otherwise it won’t work)

# Assume cross-account role
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::xxxx:role/Ansible-member-role \
  --role-session-name AnsibleSSM)

export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r '.Credentials.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r '.Credentials.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r '.Credentials.SessionToken')
export AWS_DEFAULT_REGION=us-east-1


## ✅ Test Cross-Account Role Setup

```bash
# Verify role assumption
aws sts assume-role \
  --role-arn arn:aws:iam::629843008849:role/Ansible-member-role \
  --role-session-name "AnsibleSSM"

# Test SSM connection to EC2 instance
aws ssm start-session --target i-fdfdf --region us-east-1


