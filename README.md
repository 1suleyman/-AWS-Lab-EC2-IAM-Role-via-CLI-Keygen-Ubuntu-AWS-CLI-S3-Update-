# 🤖 AWS Lab – EC2 + IAM Role via CLI (Keygen → Ubuntu → AWS CLI → S3 Update)

In this lab, I automated an end-to-end EC2 workflow using the **AWS CLI** and an **IAM instance profile** (“IAM robot”):
I generated an SSH key, imported it to EC2, launched an **Ubuntu** instance, fixed SSH access, installed **AWS CLI v2**, attached an **IAM role** to the instance, retrieved **temporary credentials** from IMDS, and **pulled/edited/pushed** a file in an S3 bucket.

---

## 📋 Lab Overview

**Goal:**

* Work primarily from the terminal to provision and configure an EC2 instance
* Use an **imported SSH key pair** for access
* Install **AWS CLI v2** on Ubuntu the correct way
* Attach an **IAM instance profile** and read **role credentials** from IMDS
* Read & write an object in **Amazon S3** from the instance

**Learning Outcomes:**

* Generate & import SSH keys; launch EC2 with the custom key
* Understand Ubuntu default user (`ubuntu`) vs Amazon Linux (`ec2-user`)
* Install AWS CLI v2 using the official installer
* Use **IMDSv2** to fetch role name and temporary security credentials
* Use `aws s3 cp` to download/edit/upload an object

---

## 🛠 Step-by-Step Journey

### Step 1 – Set Region

All tasks run in **us-east-1**.

```bash
aws configure set region us-east-1
```

✅ Region pinned for all CLI commands.

---

### Step 2 – Generate SSH Key (on the lab terminal)

```bash
ssh-keygen -t rsa -f ~/.ssh/lab-application -N ""
# public key at: ~/.ssh/lab-application.pub
```

✅ Key pair created locally.

---

### Step 3 – Import the Public Key to EC2

Console path: **EC2 → Key pairs → Import key pair**

* **Name:** `lab-application`
* Paste contents of `~/.ssh/lab-application.pub`

✅ Key pair available for launches.

---

### Step 4 – Launch Ubuntu Instance (publicly accessible)

Console path: **EC2 → Launch instances**

* **Name:** `lab-application`
* **AMI:** Ubuntu (latest LTS)
* **Instance type:** `t2.small`
* **Key pair:** `lab-application`
* **Security group:** allow **SSH (22)** from `0.0.0.0/0`
* **Public IP:** **Enabled**

✅ Instance launched and reachable from the internet.

---

### Step 5 – SSH Into the Instance (Ubuntu user)

Fix private key permissions and connect:

```bash
chmod 400 ~/.ssh/lab-application
ssh -i ~/.ssh/lab-application ubuntu@<PUBLIC_IP>
```

> Note: Ubuntu images use the **`ubuntu`** user (not `ec2-user`).

✅ SSH access confirmed.

---

### Step 6 – Install AWS CLI v2 on Ubuntu

```bash
sudo apt update -y
sudo apt install -y unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

✅ AWS CLI v2 installed correctly.

---

### Step 7 – Attach IAM Role to the Instance

Console path: **EC2 → Instances → lab-application → Actions → Security → Modify IAM role**

* Choose **`EC2LabInstanceProfile`** (or the lab’s provided instance profile)
* **Update IAM role**

✅ Instance now has temporary credentials via the role.

---

### Step 8 – Retrieve Role & Temporary Credentials from IMDS (on the instance)

**Get IMDSv2 session token:**

```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
```

**Discover role name:**

```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
# e.g. EC2LabRole
```

**Fetch credentials JSON:**

```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/<ROLE_NAME>
```

✅ Verified that the instance has role-based temporary credentials.

---

### Step 9 – Find the Lab S3 Bucket & Download `index.html`

From the instance:

```bash
aws s3 ls | grep -oE 'kk-[^ ]+'
# e.g. kk-13452
aws s3 cp s3://kk-13452/index.html /home/ubuntu/index.html
cat /home/ubuntu/index.html
```

✅ File downloaded locally.

---

### Step 10 – Edit and Re-Upload the File

```bash
sudo vi /home/ubuntu/index.html
# (update content to: "codecloud ec2 course")
aws s3 cp /home/ubuntu/index.html s3://kk-13452/index.html
```

✅ S3 object updated successfully.

---

## 🏁 End of Lab

### ✅ Key Commands Summary

| Task               | Command                                                                                                                   |                      |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| Set region         | `aws configure set region us-east-1`                                                                                      |                      |
| Generate SSH key   | `ssh-keygen -t rsa -f ~/.ssh/lab-application -N ""`                                                                       |                      |
| Fix key perms      | `chmod 400 ~/.ssh/lab-application`                                                                                        |                      |
| SSH (Ubuntu)       | `ssh -i ~/.ssh/lab-application ubuntu@<PUBLIC_IP>`                                                                        |                      |
| Install unzip      | `sudo apt install -y unzip`                                                                                               |                      |
| Install AWS CLI v2 | `curl -o awscliv2.zip … && unzip awscliv2.zip && sudo ./aws/install`                                                      |                      |
| Check CLI version  | `aws --version`                                                                                                           |                      |
| IMDSv2 token       | `TOKEN=$(curl -X PUT … /latest/api/token ...)`                                                                            |                      |
| Role name          | `curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/`            |                      |
| Role creds         | `curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/<ROLE_NAME>` |                      |
| List S3 buckets    | `aws s3 ls`                                                                                                               |                      |
| Filter kk-*        | `aws s3 ls                                                                                                                | grep -oE 'kk-[^ ]+'` |
| Download object    | `aws s3 cp s3://kk-13452/index.html /home/ubuntu/index.html`                                                              |                      |
| Upload object      | `aws s3 cp /home/ubuntu/index.html s3://kk-13452/index.html`                                                              |                      |

---

### 💡 Notes / Tips

* **Username matters:** Ubuntu AMIs use `ubuntu`; Amazon Linux uses `ec2-user`.
* **Permissions:** SSH private key must be **read-only** to your user (`chmod 400`).
* **Security Group:** For labs, opening port 22 to `0.0.0.0/0` is convenient; in prod, restrict to trusted IPs.
* **AWS CLI v2:** Prefer the official installer over apt for the latest version.
* **IMDSv2:** Always use the token flow; many environments require it.
* **S3 paths:** Use `s3://bucket/key` form with `aws s3 cp` for uploads/downloads.

---

### ✅ References

* AWS CLI v2 Install (Linux)
* EC2 Instance Metadata Service (IMDSv2)
* Using Instance Profiles with EC2
* AWS S3 CLI Commands
