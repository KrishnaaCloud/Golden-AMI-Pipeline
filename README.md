# Enterprise Golden AMI Pipeline (DevSecOps)

This project contains the architectural overview and deployment automation for an enterprise-grade "Golden AMI" (Amazon Machine Image) pipeline. 

The goal of this pipeline is to enforce strict organizational security, standardize operating environments, and optimize EC2 autoscaling boot times across **9 separate AWS consumer accounts**.

## 🛠️ Tools & Infrastructure
* **AWS EC2 Image Builder:** Bakes AMIs with organizational dependencies (Elastic Agent, EFS utils, AWS CLI).
* **AWS Systems Manager (SSM) Parameter Store:** Distributes the resulting AMI IDs globally to consumer accounts.
* **AWS Lambda & EventBridge:** Post-build automation for cross-account AMI sharing and tagging.
* **AWS CDK (Python):** Deploys the Image Builder pipelines and consumes the AMIs in downstream application stacks.
* **Elastic Agent / Fleet:** Centralized host monitoring and logging for all instances.
* **Python (boto3) & Bash:** Ad-hoc cross-account IAM management and stale resource cleanup.

## 🚀 Key Workflows & Engineering Challenges

### 1. Global SSM Parameter Standardization
To streamline consumption across 9 different DevOps and Application teams, legacy SSM parameters were standardized to a global naming convention. Downstream AWS CDK stacks dynamically resolve these paths to ensure Autoscaling Groups always launch with the latest compliant image.
* **Path Pattern:** `/golden-ami/[os-type]/[architecture]/latest`
* **Automation:** Wrote Python (boto3) and Bash scripts to iterate through all AWS profiles, update IAM cross-account roles, and prune stale parameter records.

### 2. Automated Cross-Account Distribution
An EventBridge-triggered AWS Lambda function was engineered to automatically intercept successful EC2 Image Builder pipeline completions. 
* The Lambda handler validates tags (e.g., `DevOpsGoldenAmi: PipelineAMI`), explicitly modifies AMI launch permissions to include all 9 target AWS Account IDs, and updates the respective SSM parameters in each account.

### 3. Resolving Cloud-Init Hangs & Optimizing ECS Boot Times
During initial rollouts to the UAT cluster, newly baked instances were taking 150-225 seconds to register to the Amazon ECS cluster. 

**Root Cause:**
The `UserData` script was executing `sudo elastic-agent enroll --force --non-interactive`. Because the Elastic daemon was not yet running, the command would hang in a retry loop, severely stalling the EC2 `cloud-init` sequence.

**The Fix:**
Replaced the invalid flag with `--delay-enroll`. This correctly configures the agent but defers the actual connection attempt until the daemon is started on the subsequent line, allowing `cloud-init` to complete instantly.

```bash
# Enroll the pre-installed agent without hanging cloud-init
sudo /opt/Elastic/Agent/elastic-agent enroll \
  --url=https://[FLEET_SERVER_URL] \
  --enrollment-token="$ENROLL_TOKEN" \
  --force \
  --delay-enroll

# Start the service (enrollment completes automatically in the background)
sudo systemctl enable --now elastic-agent.service
```

## 📊 Final Metrics & Business Impact
By resolving the `cloud-init` bottlenecks, we accurately measured the ASG scale-out sequence using a custom Python daemon:

* **T0 (Pending):** 0 seconds
* **T1 (Running):** 4 seconds
* **T3 (ECS Ready):** 102 seconds
* **T2 (EC2 Status Checks Pass):** 175 seconds

**Impact:**
1. **33% Speedup in ASG Scale-out:** ECS cluster readiness dropped from 153+ seconds down to **exactly 102 seconds**. The cluster can now begin scheduling application containers long before AWS finishes its internal 2/2 status checks.
2. **100% Observability:** The Elastic Agent correctly and silently enrolls itself into the Fleet dashboard on every boot without crashing, ensuring all newly provisioned instances are fully observable immediately upon boot.
