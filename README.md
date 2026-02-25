# DS5220 Data Project 1

## Overview

This project implements an **event-driven anomaly detection flow** on time series data. When a new batch of observations arrives (e.g., IoT sensor readings, server metrics, weather station data), the system runs an anomaly detection pass using scikit-learn’s **IsolationForest**, then writes back a scored version of the file where each row is annotated with an anomaly flag and a score.

The instance maintains a **running statistical baseline** in S3 (a small JSON file, `baseline.json`, tracking rolling mean and standard deviation per sensor/channel), which it updates with each new batch. Detection improves over time through adaptive statistics without retraining, simulating stateful stream-like processing over batch files.

Your developers have already created working code for this project (though it needs both error handling and logging implemented), so your primary task in this project is to define a resource stack in CloudFormation that will enable other users in your org to launch identical solutions in the future.

![Solution Diagram](https://s3.amazonaws.com/uvasds-systems/images/diagram.png)

---

## CloudFormation Template

You must create a **CloudFormation template** that provisions the entire solution. A single template must include the following:

### EC2 instance
- **AMI:** Ubuntu 24.04 LTS
- **Instance type:** `t3.micro`
- **Boot volume:** 16 GB
- **User data / bootstrap:** Install and configure the application so that:
  - The necessary Python libraries (from the repo’s `requirements.txt`) are installed within a virtual environment (see Notes below).
  - It pulls a copy of your forked `anomaly-detection` app into the instance.
  - The environment variable `BUCKET_NAME` is set. This should be done in two ways: as an `export` command in the bootstrapping (e.g. `export BUCKET_NAME='my-bucket'`) as well as set as a global environment variable to be set upon future reboots or logins (e.g., by setting `BUCKET_NAME='my-bucket'` in `/etc/environment`). The value of this variable should be the name of the S3 bucket created by your stack. See [this example](https://github.com/uvasds-systems/ds5220-cloud/blob/347a4dc436096826c172f1b258a541b90a389455/reference-iac/cloudformation/0-BASE-ec2-instance.yaml#L55) where the `!Sub` feature in the `UserData` allows you to call other stack variables with `!Ref SomeResourceName`.
  - A final command to run the FastAPI API (e.g., `fastapi run app.py` or equivalent) so the service starts on boot or via a process manager.

### Security group
- Allow **port 22** (SSH) from YOUR specific IP address, e.g. (`1.2.3.4/32`)
- Allow **port 8000** (API) from anywhere (`0.0.0.0/0`)
- Attach this security group to the EC2 instance

### Elastic IP
- Create an **Elastic IP** and **attach it to the EC2 instance** so the instance has a stable public IP for the SNS subscription endpoint.

### S3 bucket
- Create a **new S3 bucket** (name can be parameterized or generated). This is the bucket the application will use for raw uploads, processed output, state, and logs.

### IAM role and policy
- Create an **IAM instance profile** (role) attached to the EC2 instance.
- The role must have an **IAM policy** that grants **full access** (read/write/delete/list) to **that single S3 bucket only** (no other resources).

### SNS topic and subscription
- Create an **SNS topic** named **`ds5220-dp1`**.
- Create an **SNS subscription**:
  - **Protocol:** HTTP
  - **Endpoint:** The Elastic IP of the instance, port 8000, path `/notify` — e.g. `http://<Elastic-IP>:8000/notify`
  - The subscription must reference the instance’s Elastic IP (use CloudFormation refs/attributes so the endpoint is correct after stack creation).

### S3 event notification
- Configure an **S3 event notification** on the bucket that:
  - **Prefix:** `raw/`
  - **Suffix:** `*.csv` (or equivalent filter for CSV objects in `raw/`)
  - **Destination:** Publish each event to the **SNS topic** `ds5220-dp1`

When a CSV file is uploaded to the bucket under `raw/`, S3 notifies SNS, and SNS sends an HTTP request to `http://<Elastic-IP>:8000/notify`, which FastAPI uses to trigger processing.

You may want/need to build and destroy some instances along the way for testing purposes. This is normal.

Be sure to clear your bucket of all objects after testing, before you perform your full
60 minute run. This can be done in one command:

```
# show which files would be deleted
aws s3 rm s3://YOUR-BUCKET/ --recursive --dry-run

# then the actual deletion
aws s3 rm s3://YOUR-BUCKET/ --recursive

```
---

## Setup

1. **Fork the repository**  
   Fork [uvasds-systems/anomaly-detection](https://github.com/uvasds-systems/anomaly-detection) so you have your own copy of the python application to work with. Get your code in a working state with (error handling and logging) that is ready to deploy. [**Create a Fork**](https://github.com/uvasds-systems/anomaly-detection/fork).

2. **Launch your Resources**
   Create a stack with all the resources described above, connecting resources where required, i.e. associate the EIP with your instance, connect your S3 bucket's event triggers with the SNS topic, etc.

3. **Bootstrap the instance**  
   Within the template, bootstrap your instance with any required software and configuration. Ensure that `BUCKET_NAME` is set as a global environment variable (e.g., add `KEY="VALUE"` to `/etc/environment`) for subsequent logins and reboots, but also `export` is explicitly within bootstrapping (e.g. `export BUCKET_NAME='my-bucket'`). The application requires an S3 bucket and an IAM role with read/write access to that bucket; it will not run without them. Bootstrapping should also include pulling down a copy of your forked code.

4. **Python environment**  
   - Create and activate a virtual environment (`virtualenv`, `pipenv`, etc.) for your code to run properly.
   - Install dependencies from `requirements.txt`.
   - From within the directory containing `app.py`, run:
     ```bash
     # or provide full paths to both fastapi and app.py
     fastapi run app.py
     ```
   The API will be available at `http://YOUR-EC2-IP-ADDRESS:8000/`.

---

## Code Structure

These modules support the detection pipeline and are imported or invoked by the main API:

| File           | Role |
|----------------|------|
| `baseline.py`  | Maintains and updates per-channel rolling statistics (mean, std) in `baseline.json`; state is synced to `s3://BUCKET_NAME/state/baseline.json`. |
| `detector.py`  | Runs anomaly detection (IsolationForest) on incoming data and produces anomaly flags and scores. |
| `processor.py` | Orchestrates reading data, calling the detector, and writing scored output. |

Read through these files to understand the end-to-end flow.

---

## API Endpoints

The service is a **FastAPI** application in `app.py` with five endpoints:

| Method | Endpoint | Description |
|--------|----------|-------------|
| **POST** | `/notify` | Receives SNS messages; handles subscription confirmation and dispatches incoming S3 object keys to `process_file` as a background task. **Use this endpoint for your SNS subscription.** |
| **GET**  | `/anomalies/recent` | Scans the 10 most recent processed CSVs and returns rows where `anomaly == True`, with an optional `limit` query parameter. |
| **GET**  | `/anomalies/summary` | Aggregates `_summary.json` files for a high-level view: total rows scored, total anomalies, and overall anomaly rate across batches. |
| **GET**  | `/baseline/current` | Returns the current per-channel statistics (mean, std, observation count, baseline maturity). |
| **GET**  | `/health` | Liveness check to confirm the service started correctly. |
| **GET**  | `/docs` | Auto-generated documentation for your API that describes all the endpoints. |

---

## Error Handling

Implement error handling within the application as appropriate. Use `try`/`except` stanzas
and print errors to both the screen and your logfile.

Place error logic appropriately where requests or transactions are most likely to experience
failures or issues.

---

## Logging

- Implement **logging of your FastAPI to a local file**.
- **Sync a copy of that log file to your S3 bucket** whenever your application pushes `baseline.json`.
- Log important events, for example:
  - Arrival of a new file
  - New calculations
  - Baseline updates

This keeps a single application log file backed up from the EC2 instance to S3.

With error handling and logging in place, add/commit/push all changes back to your fork.

---

## Publish Test Data to Your Bucket

- A test script **`test_producer.py`** is provided. Run it on your laptop or from another SSH session on your instance (it has its own dependencies).
- It produces a CSV with 100 records every 60 seconds and uploads them to your S3 bucket under a `raw/` folder.
- A [sample data file](https://github.com/uvasds-systems/anomaly-detection/blob/main/sensors_20260224T001051.csv) is available in the repository for reference.

Once you have successfully tested your solution to a working state, let it run unattended for 60 minutes.

You should then stop the script and copy two files from your bucket into your forked repository

1. A copy of your full log file should be saved to the `submit/` directory of your fork.
2. A copy of your final baseline file (found in `s3://YOUR_BUCKET/state/baseline.json`) should be saved in the same location.

Add, commit, and push those files to your fork.

---

## Summary of Deliverables

### All Students

- **CloudFormation template** that builds the full solution (EC2, security group, Elastic IP, S3 bucket, IAM role, SNS topic `ds5220-dp1`, SNS HTTP subscription to `http://<Elastic-IP>:8000/notify`, S3 event notification on `raw/*.csv` → SNS). This file should be saved to the `submit/` folder of your forked repository.
- A copy of your `baseline.json` file should be saved in the same directory.
- A copy of your full log file should be saved to the same directory.
- Submit the URL to your fork of the `anomaly-detection` repo in Canvas.

### Graduate Students

In addition to the above requirements:

- Write a complete working template of this solution in **Terraform**. Add this to the `submit/` directory of your fork.
- You should test your solution to be sure it is in good working order, but you do not need to submit additional baseline or log files.
- Submit your **answers to the following questions** in a markdown or PDF file in the same folder of your forked repository.

**Questions**

1. **Technical Challenges** Describe the greatest challenge(s) you encountered in translating the template from CloudFormation to Terraform. (1-2 paragraphs)
2. **Access Permissions** What element (specify file and line #) grants the SNS subscription permission to send messages to your API? Locate and explain your answer.
3. **Event flow and reliability:** Trace the path of a single CSV file from the moment it is uploaded to `raw/` in S3 until the FastAPI app processes it. What happens if the EC2 instance is down or the `/notify` endpoint returns an error? How does SNS behave (e.g., retries, dead-letter behavior), and what would you change if this needed to be production-grade?
4. **IAM and least privilege:** The IAM policy for the EC2 instance grants full access to one S3 bucket. List the specific S3 operations the application actually performs (e.g., GetObject, PutObject, ListBucket). Could you replace the “full access” policy with a minimal set of permissions that still allows the app to work? What would that policy look like?
5. **Architecture and scaling:** This solution uses batch-file events (S3 + SNS) to drive processing, with a rolling statistical baseline in memory and in S3. How would the design change if you needed to handle 100x more CSV files per hour, or if multiple EC2 instances were processing files from the same bucket? Address consistency of the shared `baseline.json`, concurrent processing, and any tradeoffs.

---

## Cleanup

After running your stack and application, and after gathering the material for submission,
delete your stack using the appropriate method for CloudFormation or Terraform.

Note that your S3 bucket will not be deleted by either method since it contains objects.
Keep these for future reference.

---

## Reference 

- Full working code and files: [https://github.com/uvasds-systems/anomaly-detection](https://github.com/uvasds-systems/anomaly-detection).
- **CloudFormation** reference: [https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/introduction.html](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/introduction.html)
- **Terraform** reference: [https://developer.hashicorp.com/terraform/docs](https://developer.hashicorp.com/terraform/docs)
- **FastAPI** reference: [https://fastapi.tiangolo.com/](https://fastapi.tiangolo.com/)

## Notes

### Bootstrapping with Virtual Environments

I would recommend using either `venv` or `pipenv`. The snippet below could be a portion
of your bootstrapping:

```
#!/bin/bash
set -e

# Update and install Python + pip
apt-get update -y
apt-get install -y git python3 python3-pip python3-venv git

# clone and cd
# Create a virtualenv in a known location
python3 -m venv /opt/anomaly-detection/venv

# Activate and install deps
source /opt/anomaly-detection/venv/bin/activate
/opt/anomaly-detection/venv/bin/pip install -r /opt/anomaly-detection/requirements.txt

# The app.py FastAPI app can be run using full paths if necessary, even from outside the virtualenv:
/opt/anomaly-detection/venv/bin/fastapi run /opt/anomaly-detection/app.py --reload

```

### SNS Subscription Confirmation

Your API uses the `/notify` endpoint for SNS subscriptions. If you [look at the code](https://github.com/uvasds-systems/anomaly-detection/blob/c485eb56fdfb1ed652681daf597cfa5506fc9856/app.py#L25) in `app.py` you will see that `SubscriptionConfirmation` messages get confirmed automatically.

However, you should check in the web console or using the CLI to verify the subscription has been confirmed. If your API is up and running, go to your topic in the SNS service, and review the subscription. If necessary, select it using the radio button on the left and click "Request Confirmation" to try again.

Using the CLI you can verify a confirmed subscription. The `SubscriptionArn` attribute 
with either have a full ARN or a `Pending` status:

```
# Be sure to replace the ARN with your own:
$ aws sns list-subscriptions-by-topic --topic-arn "arn:aws:sns:us-east-1:440848399208:ds5220-dp1"

{
    "Subscriptions": [
        {
            "SubscriptionArn": "arn:aws:sns:us-east-1:440848399208:ds5220-dp1:045ce4c3-6431-430c-88fe-c2b66f21aa34",
            "Owner": "440848399208",
            "Protocol": "http",
            "Endpoint": "http://13.222.90.189:8000/notify",
            "TopicArn": "arn:aws:sns:us-east-1:440848399208:ds5220"
        },
        {
            "SubscriptionArn": "PendingConfirmation",
            "Owner": "440848399208",
            "Protocol": "http",
            "Endpoint": "http://34.233.126.83:8000/notify",
            "TopicArn": "arn:aws:sns:us-east-1:440848399208:ds5220"
        }
    ]
}
```

### Custom AMI

Thinking about building your own custom machine image with all dependencies and then using that to launch your stack?

**Go for it!** Please note that you are doing so with inline comments in your template.
Even with a custom AMI, be sure to bootstrap accordingly so that an "updated" FastAPI repository gets pulled into any new instance(s).


### BEWARE of Recursion!

Be sure that your S3 event notification is ONLY scoped to the `raw/` subfolder (the prefix)
and is triggered only on the arrival of `*.csv` files (the suffix). Otherwise, when the 
scored version of each CSV file is generated and pushed to the bucket, the notification is
triggered again and again. This could spawn an endless spiral of meaningless files in your bucket.

### Use of GenAI / LLMs / AI-assisted IDEs

Can I use an AI tool with this project? **Yes you may**.

GenAI is particularly good at tedious, repetitive production like code, templates,
configuration files, and so forth. Furthermore, both CloudFormation and Terraform
are well established templating frameworks used broadly across the community of
software developers, DevOps engineers, Data Scientists, and others, and so AI
output is likely to be more useful than not.

Therefore I encourage the use of these tools, but it is clearly up to you to give
guidance and to *thoroughly* test any results. I would suggest writing your own
stub of a template that (in comments) itemizes each resource and describes their features, and then working on individual blocks with Codex, Cursor, etc. Be prepared to test repeatedly for a complete, working solution.
