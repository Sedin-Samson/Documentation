# Jenkins Ephemeral EC2 Agent Design

## 1. Objective

The goal of this design is to dynamically create, use, and destroy EC2-based Jenkins agents per pipeline run.

**This approach:**
- Eliminates long-running Jenkins agents
- Improves security (short-lived credentials & instances)
- Reduces AWS cost
- Supports cross-account CI/CD

---

## 2. High-Level Architecture

### Flow Overview

```
Jenkins Controller
   |
   | (Assume Role)
   v
Target AWS Account
   |
   |-- Launch EC2 (Ephemeral Agent)
   |-- Agent connects back via JNLP
   |-- Job executes
   |-- EC2 is terminated
```

### Key Design Principles

- **Ephemeral Infrastructure** – Agent exists only for the job
- **Cross-Account Access** – Jenkins does not live in the target account
- **Zero Manual Cleanup** – Guaranteed termination in `post { always }`
- **Reusable Shared Library** – One implementation, many pipelines

---

## 3. Pipeline Overview (Jenkinsfile)

### Pipeline Stages

| Stage | Purpose |
|-------|---------|
| Create EC2 Agent | Launch EC2 and register it as Jenkins agent |
| Run Job on EC2 Agent | Execute workload (Docker, builds, deploys) |
| Debug Window | Optional troubleshooting window |
| Post Cleanup | Terminate EC2 agent |

---

## 4. Environment Configuration

```groovy
environment {
  TARGET_ROLE_ARN = "arn:aws:iam::22222222:role/Jenkins-CICD-Role"
  AWS_REGION = "us-east-1"

  AMI_ID = "ami-0ecb62995f68bb549"
  INSTANCE_TYPE = "t3.micro"
  SECURITY_GROUP_ID = "<SG-ID>"
  KEY_NAME = "Testing-Key"

  AGENT_NAME = "ec2-ephemeral"
  JENKINS_URL = "http://<MASTER_IP>:8080"
  JENKINS_SECRET = "****"
  IAM_INSTANCE_PROFILE_ARN = "arn:aws:iam::222222222:instance-profile/EC2-Session-Role"
}
```

### Why these variables matter

- **TARGET_ROLE_ARN** → Enables cross-account access
- **AMI_ID** → Pre-approved OS baseline
- **IAM_INSTANCE_PROFILE** → No static AWS credentials on EC2
- **JENKINS_SECRET** → Secure agent authentication

---

## 5. Shared Library: createEc2Agent

### Purpose

This function:
- Assumes IAM role in target AWS account
- Launches EC2 with required tooling
- Auto-registers EC2 as Jenkins agent
- Returns EC2 instance ID cleanly

[Github Link for Shared Library creating](https://github.com/RF-Junior-Devops/jenkins-shared-library/blob/main/vars/createEc2Agent.groovy) 
### Step-by-Step Execution

#### 5.1 Assume Role into Target Account

```bash
aws sts assume-role \
  --role-arn "${config.TARGET_ROLE_ARN}" \
  --role-session-name jenkins-agent
```

✔ Ensures Jenkins never has direct AWS credentials

#### 5.2 Prepare EC2 User Data

```bash
apt-get install -y openjdk-17-jre docker.io curl jq
```

**Installs:**
- **Java** → Jenkins agent
- **Docker** → Container workloads
- **Curl/JQ** → Debug & API calls

#### 5.3 Auto-Register Jenkins Agent

```bash
java -jar agent.jar \
  -url ${config.JENKINS_URL} \
  -secret ${config.JENKINS_SECRET} \
  -name ${config.AGENT_NAME} \
  -webSocket
```

✔ Agent connects outbound (no inbound SSH needed)

#### 5.4 Launch EC2

```bash
aws ec2 run-instances \
  --image-id "${config.AMI_ID}" \
  --instance-type "${config.INSTANCE_TYPE}" \
  --iam-instance-profile Arn="${config.IAM_INSTANCE_PROFILE_ARN}"
```

✔ Instance auto-joins Jenkins after boot  
✔ Instance ID returned cleanly to pipeline

---

## 6. Job Execution on EC2 Agent

```groovy
stage('Run Job on EC2 Agent') {
  agent { label 'ec2-ephemeral' }
}
```

**At this point:**
- Jenkins waits until EC2 agent is online
- Workloads execute directly on EC2

### Example Workload

- Pull Docker image
- Run NGINX container
- Validate service locally

**This proves:**
- Docker works
- Network works
- Agent is production-ready

---

## 7. Debug Window (Optional)

```groovy
sleep 600
```

### Why this exists

- SSH into EC2
- Inspect logs
- Validate environment
- Troubleshoot failures

---

## 8. Shared Library: terminateEc2Agent

### Purpose

Ensures guaranteed cleanup, even if:
- Job fails
- Jenkins restarts
- Script errors occur
[Github Link for Shared Library Terminate](https://github.com/RF-Junior-Devops/jenkins-shared-library/blob/main/vars/terminateEc2Agent.groovy) 
### Cleanup Logic

- Re-assume target account role
- Terminate EC2 instance
- Wait for termination

```bash
aws ec2 terminate-instances --instance-ids "${INSTANCE_ID}"
aws ec2 wait instance-terminated
```

✔ Zero orphan instances  
✔ Zero cost leakage

---

## 9. Security Model

### IAM Design

| Component | Permissions |
|-----------|-------------|
| Jenkins | `sts:AssumeRole` only |
| EC2 Agent | Minimal EC2 + required AWS services |
| No Secrets on Disk | Temporary STS credentials only |

---

## 10. Cost Optimization Benefits

| Traditional Agent | Ephemeral Agent |
|------------------|-----------------|
| 24/7 EC2 running | Runs only during job |
| Idle costs | Zero idle cost |
| Manual cleanup | Automatic |
| Static credentials | Temporary STS |

---

## 11. Future Enhancements

- Auto-tag EC2 with JobName, BuildNumber
- Spot instances for cost reduction
- Email-Notification for a particular team

---