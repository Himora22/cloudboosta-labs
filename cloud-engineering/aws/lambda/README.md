# AWS Lambda — Functions Lab, EC2 Automation, EventBridge & S3 Thumbnails

**Programme:** Cloudboosta CBA Training Programme — Feb Cohort 1 (Cloud Engineering track)
**Topic:** AWS Lambda — creating functions via blueprint and from scratch
**Environment:** AWS Management Console, Visual Studio Code (Lambda remote editing)

## Objective

Build hands-on familiarity with AWS Lambda: creating functions both from a blueprint and from scratch, then using Lambda to automate EC2 instance lifecycle actions (directly and on an EventBridge schedule), and finally triggering a Lambda function from S3 uploads to generate image thumbnails.

## Skills demonstrated

- Creating a Lambda function from an AWS blueprint (Amazon Data Firehose stream processor)
- Creating a Lambda function from scratch with a chosen runtime and architecture
- Editing Lambda function code both in the browser editor and via the "Open in Visual Studio Code" remote connection
- Writing and deploying custom Python logic to a Lambda function
- Creating and invoking test events with custom JSON payloads
- Reading Lambda execution results and function logs to verify behavior
- Deleting Lambda functions after testing to avoid unnecessary cost
- Using the `boto3` AWS SDK inside a Lambda function to control EC2 instances (stop/start/terminate)
- Diagnosing and fixing an IAM permissions error (`UnauthorizedOperation`) by attaching a managed policy to a Lambda execution role
- Verifying EC2 state changes in the console after each Lambda-triggered action
- Provisioning a new EC2 instance from within a Lambda function using `ec2.run_instances()`, including tagging it for dynamic discovery
- Writing a single Lambda function that toggles an EC2 instance between stopped and running based on its current state
- Scheduling recurring Lambda invocations with Amazon EventBridge using cron expressions
- Troubleshooting a real AWS console UI change (EventBridge's Scheduled rules migrating out of Buses → Rules) by reading the console's own guidance
- Verifying scheduled automation end-to-end using Amazon CloudWatch Logs
- Triggering a Lambda function from an S3 PUT event to process uploaded files automatically
- Packaging third-party dependencies (Pillow/PIL) into a Lambda `.zip` deployment package, since they aren't included in the standard runtime
- Using environment variables to keep a destination bucket name out of hardcoded function code
- Diagnosing a silent Lambda timeout (function triggered but produced no output) and resolving it by increasing the timeout

## Task 1: Lambda Functions Lab — Blueprint vs. From Scratch

This lab covers two ways of creating an AWS Lambda function: starting from a pre-configured AWS blueprint, versus authoring one from a blank starting point with custom logic.

### Part A: Creating a function using a blueprint

I navigated to **Lambda → Create function** and selected **Use a blueprint**, choosing the **"Process records sent to an Amazon Data Firehose stream"** blueprint (Python 3.12). I named the function `cba-lamda-01` and kept the default (auto-created) IAM execution role.

![Blueprint selection and function configuration](screenshots/01-blueprint-create-function.png)

Clicking **Create function** confirmed success, and the Function overview panel showed the function's ARN and its auto-generated description (an Amazon Data Firehose stream processor).

![cba-lamda-01 successfully created — Function overview](screenshots/02-blueprint-function-created.png)

I opened the function in Visual Studio Code directly from the Lambda console (**Open in Visual Studio Code**) and replaced the default blueprint code with custom logic that reads `firstname`, `lastname`, and `school` fields from the event object:

```python
def lambda_handler(event, context):
    print('First Name = ' + event['firstname'])
    print('Last Name = ' + event['lastname'])
    print('School = ' + event['school'])
    return event['firstname']
```

![Modified Lambda function code in VS Code](screenshots/03-blueprint-vscode-code.png)

I created a test event named `CBA_Event` with a custom JSON payload matching those three fields, invocation type **Synchronous**, and visibility **Private**:

```json
{"firstname": "Seun", "lastname": "Okegbola", "school": "Cloudboosta"}
```

![Test event CBA_Event configuration](screenshots/04-blueprint-test-event.png)

Invoking the function with `CBA_Event` succeeded, returning `"Seun"` as the response, with the function logs confirming all three fields were read and printed correctly (duration 1.66 ms, billed 83 ms, memory 37 MB / 128 MB).

![Execution results for CBA_Event — Status: Succeeded, Response: "Seun"](screenshots/05-blueprint-execution-result.png)

### Part B: Creating a function from scratch

For the second method, I navigated to **Lambda → Create function** and selected **Author from scratch**, naming the function `cba-lamda-02` with the **Python 3.10** runtime, **x86_64** architecture, and the default execution role (CloudWatch Logs permissions only).

![Author from scratch — cba-lamda-02 configuration](screenshots/06-scratch-create-function.png)

I wrote custom Python logic directly in the Lambda editor: the function checks whether `name` in the event equals `'Cloudboosta'` and returns a matching message, or an interpolated message otherwise. My first pass at the condition used a single `=` instead of `==` — in Python, `=` is assignment, not comparison, so that line wasn't a valid conditional at all:

![Custom code deployed to cba-lamda-02 — Successfully updated](screenshots/07-scratch-code-deployed.png)

I caught this and corrected it to `==` before moving on to testing:

```python
import json

def lambda_handler(event, context):
    if event.get('name') == 'Cloudboosta':
        return "I am a Cloudboosta student!"
    else:
        return f"I am {event.get('name')} student!"
```

I created a test event named `cbaEvent2` with the payload `{"name": "Cloudboosta"}`, to exercise the first branch of the conditional.

![Creating test event cbaEvent2 with name = 'Cloudboosta'](screenshots/08-scratch-test-event-cbaEvent2.png)

Invoking the function with `cbaEvent2` matched the `'Cloudboosta'` condition and returned the expected string, `"I am a Cloudboosta student!"`.

![Execution result for cbaEvent2](screenshots/09-scratch-execution-cbaevent2.png)

To test the `else` branch, I created a second test event, `cbaEvent3`, with a deliberately different value — `'Cloud boosta'` (with a space) — so it would *not* match the exact string `'Cloudboosta'`.

![Creating test event cbaEvent3 with name = 'Cloud boosta'](screenshots/10-scratch-test-event-cbaEvent3.png)

Invoking with `cbaEvent3` fell through to the `else` branch as expected, returning `"I am Cloud boosta student!"` — confirming the dynamic string interpolation worked correctly for values that didn't match the special-cased name.

![Execution result for cbaEvent3](screenshots/11-scratch-execution-cbaevent3.png)

After confirming both functions worked as expected, I deleted `cba-lamda-01` and `cba-lamda-02`. The console confirmed both were deleted and the functions list was empty.

![Both Lambda functions successfully deleted — Functions list empty](screenshots/12-functions-deleted.png)

## Task 2: Lambda + EC2 — stop, start, and terminate an instance

This task uses a single Lambda function to control the lifecycle of an EC2 instance via the `boto3` SDK, editing and redeploying the function's code between each action.

I launched a `t3.micro` EC2 instance named `Lamda-inst` and confirmed it reached the **Running** state.

![Lamda-inst EC2 instance running](screenshots/13-ec2-instance-running.png)

I created a new Lambda function, **Author from scratch**, named `cba-lamda-Assign`, using the **Python 3.12** runtime and **x86_64** architecture, with the default (auto-created) execution role.

![Creating cba-lamda-Assign from scratch](screenshots/14-create-function-scratch.png)

I wrote code using `boto3` to stop the instance by ID, deployed it, and created a test event named `ec2-stop-event`:

```python
import boto3

instances = ['<instance-id>']
ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    ec2.stop_instances(InstanceIds=instances)
    print("stopped the running instance" + str(instances))
```

Invoking `ec2-stop-event` failed with an `UnauthorizedOperation` error — the function's IAM role didn't have permission to call EC2 actions.

![First invocation of ec2-stop-event — Status: Failed](screenshots/15-code-deployed-initial-error.png)

Checking the function's **Configuration → Permissions** tab confirmed the execution role only had the default CloudWatch Logs permissions, with no access to EC2 at all.

![Execution role permissions — CloudWatch Logs only](screenshots/16-permissions-cloudwatch-only.png)

I went to the role in the IAM console and attached a policy to grant EC2 access.

![IAM role — Add permissions / Attach policies](screenshots/17-iam-add-permissions.png)

After attaching the **AmazonEC2FullAccess** managed policy (alongside the existing `AWSLambdaBasicExecution` policy), the role showed two permissions policies attached.

![IAM role with AmazonEC2FullAccess and AWSLambdaBasicExecution attached](screenshots/18-iam-role-policies-attached.png)

Re-running `ec2-stop-event` succeeded this time, with the function log confirming the instance was stopped.

![ec2-stop-event succeeded after fixing IAM permissions](screenshots/19-stop-test-succeeded.png)

The EC2 console confirmed `Lamda-inst` had moved to the **Stopped** state.

![Lamda-inst instance state: Stopped](screenshots/20-ec2-stopped.png)

I then commented out the stop code and added a `start_instances` version below it, deploying the change and creating a new test event, `ec2-start-event`. Invoking it succeeded, with the log confirming the instance was started.

![ec2-start-event succeeded — code edited to call start_instances](screenshots/21-start-test-succeeded.png)

The EC2 console confirmed `Lamda-inst` was back in the **Running** state (with a new public IP assigned on restart).

![Lamda-inst instance state: Running again](screenshots/22-ec2-running-again.png)

Finally, I commented out the start code and added a `terminate_instances` version, creating a test event named `ec2-terminate-event`. Invoking it succeeded, with the log confirming the instance was terminated.

![ec2-terminate-event succeeded — code edited to call terminate_instances](screenshots/23-terminate-test-succeeded.png)

The EC2 console confirmed `Lamda-inst` reached the **Terminated** state.

![Lamda-inst instance state: Terminated](screenshots/24-ec2-terminated.png)

## Task 3: Automating EC2 Management with Scheduled Lambda Functions

This task builds a small automation pipeline: a Lambda function provisions an EC2 instance, a second Lambda function toggles that instance between running and stopped based on a tag, and Amazon EventBridge triggers the second function on a recurring schedule. CloudWatch Logs is then used to confirm the schedule actually fired.

### Pre-requisite: IAM role setup

Before any of the three Lambda-related tasks could work, I created an IAM role to give the functions permission to manage EC2 instances and write logs to CloudWatch — without it, every invocation would fail with an `UnauthorizedOperation` error.

In the IAM console, I chose **Create role**, selected **AWS service** as the trusted entity type, and **Lambda** as the use case, so the role could be assumed by Lambda functions.

![IAM Roles list, then trusted entity set to AWS service / Lambda](screenshots/25-iam-roles-list.png)
![Select trusted entity — AWS service, use case Lambda](screenshots/26-iam-trusted-entity-lambda.png)

I attached the **AmazonEC2FullAccess** managed policy (for launching, stopping, starting, and terminating instances) and **AWSLambdaBasicExecutionRole** (for CloudWatch Logs), then named the role `LambdaEC2ProvisionRole`.

![AmazonEC2FullAccess policy found and selected](screenshots/27-iam-attach-ec2fullaccess.png)
![Role review — name, trust policy, and both permissions policies before creation](screenshots/28-iam-role-review.png)

The role was created successfully with both policies attached.

![LambdaEC2ProvisionRole created — 2 permissions policies confirmed](screenshots/29-iam-role-created.png)

### 3a. Provisioning an EC2 instance using Lambda

I created a new Lambda function, `ProvisionEC2Lambda`, **Author from scratch**, Python 3.13, and attached `LambdaEC2ProvisionRole` as the execution role from the start (via **Use another role**).

![Create function — ProvisionEC2Lambda with Python 3.13 and LambdaEC2ProvisionRole](screenshots/30-provision-create-function.png)

The function uses `boto3`'s `ec2.run_instances()` to launch a `t3.micro` instance, using the correct Amazon Linux 2 AMI for `eu-west-2` (`ami-061e1ade216078a11`) and the key pair `cba-key-new`. The instance is tagged `AutoSchedule: True` so the second Lambda function can find it dynamically.

![Deployed code — ec2.run_instances() with the correct AMI ID and key pair](screenshots/32-provision-code-final-ami.png)

The default 3-second Lambda timeout wasn't enough for EC2 provisioning, so I increased it to 1 minute under **Configuration → General configuration → Edit**.

![General configuration before edit — default timeout 0 min 3 sec](screenshots/33-provision-timeout-before-edit.png)
![Edit basic settings — timeout updated to 1 min 0 sec](screenshots/34-provision-timeout-edit.png)

I created a test event, `TestProvisionEC2`, with an empty JSON payload, and invoked the function.

![Test event TestProvisionEC2 created with empty JSON payload](screenshots/35-provision-test-event-created.png)

The invocation succeeded, returning the new instance ID `i-0d414548022d52325`.

![Test result — Status: Succeeded, EC2 Instance i-0d414548022d52325 launched successfully](screenshots/36-provision-test-succeeded.png)

The EC2 console confirmed the new instance was **Running**, and its Tags tab confirmed the `AutoSchedule: True` tag had been applied.

![EC2 console — provisioned instance in Running state](screenshots/37-provision-ec2-running.png)
![EC2 Tags tab — AutoSchedule: True confirmed](screenshots/38-provision-tags-autoschedule.png)

### 3b. Stopping and starting the instance dynamically

With an instance running, I created a second function, `StopStartEC2Lambda` (Python 3.13), reusing `LambdaEC2ProvisionRole` since it already had the needed EC2 permissions.

![Create function — StopStartEC2Lambda with Python 3.13 and LambdaEC2ProvisionRole](screenshots/39-stopstart-create-function.png)

This function uses `ec2.describe_instances()` with a filter on the `AutoSchedule=True` tag to find the right instance(s), then checks each instance's current state: if `running`, it calls `ec2.stop_instances()`; if `stopped`, it calls `ec2.start_instances()`; anything else is skipped.

![StopStartEC2Lambda code — dynamic stop/start toggle logic](screenshots/40-stopstart-code-toggle-logic.png)

I increased this function's timeout to 1 minute as well, then invoked it using the `StartStop-EC2` test event. It succeeded, and the function logs confirmed `Started instance: i-0d414548022d52325` — the toggle correctly detected the instance was stopped and started it.

![Timeout increased to 1 minute for StopStartEC2Lambda](screenshots/41-stopstart-timeout-increased.png)
![Test result — Status: Succeeded, "Started instance" confirmed in function logs](screenshots/42-stopstart-test-succeeded.png)

The EC2 console confirmed the instance state changing in response, verifying the toggle logic was working.

![EC2 console — instance state transitioning](screenshots/43-stopstart-ec2-state-transition.png)

### 3c. Scheduling the Lambda function with EventBridge

With the toggle function working on demand, the next step was to automate it on a schedule using Amazon EventBridge.

![AWS console search — Amazon EventBridge and EventBridge Scheduler](screenshots/44-eventbridge-search.png)

**Challenge encountered:** the EventBridge console had changed since the assignment was originally written, and the expected schedule option wasn't where I first looked. A new **Visual rule builder** toggle was on by default under **Buses → Rules → Create rule**, replacing the classic step-by-step form; turning it off restored the familiar form. Even with the classic form, though, **Buses → Rules** no longer offered a "Schedule" rule type at all — Step 1 only led to an event-pattern builder, because AWS has fully migrated scheduled rules out of that section. The fix was reading the orange warning banner on the Rules page itself: *"We have moved Scheduled rules in your account to the Scheduler section, where you can view, create and edit a scheduled rule."* From there, **Scheduler → Scheduled rules (legacy)** in the left sidebar was the correct place to create a scheduled rule.

![EventBridge Rules page — orange banner confirming scheduled rules were moved to the Scheduler section](screenshots/45-eventbridge-scheduled-rules-banner.png)

Under **Scheduler → Scheduled rules (legacy)**, I created a rule named `EC2ScheduleRule` on the default event bus.

![Step 1 — Scheduled rule detail for EC2ScheduleRule](screenshots/46-rule1-detail-ec2schedulerule.png)

For the schedule pattern, I chose **A fine-grained schedule** and entered the cron expression `cron(0 8 * * ? *)`, which triggers daily at 8:00 AM UTC (9:00 AM local BST/GMT+1). The console displayed the next 10 trigger dates as confirmation.

![Step 2 — cron(0 8 * * ? *) schedule, triggers daily at 9:00 AM GMT+1](screenshots/47-rule1-cron-9am.png)

For the target, I set the type to **AWS service**, selected **Lambda function**, and chose `StopStartEC2Lambda`. EventBridge automatically created a new IAM role (`Amazon_EventBridge_Invoke_Lambda_1648311493`) to allow it to invoke the function.

![Step 3 — StopStartEC2Lambda selected as target, with a new auto-created execution role](screenshots/48-rule1-select-target.png)

After reviewing the full summary, I created the rule.

![Step 5 — Review and create: full summary of EC2ScheduleRule](screenshots/49-rule1-review-create.png)

I repeated the same process for a second rule, `EC2ScheduleRule2`, using the cron expression `cron(0 14 * * ? *)` — 2:00 PM UTC (3:00 PM GMT+1) — to provide a second daily automation window.

![Step 1 — Scheduled rule detail for EC2ScheduleRule2](screenshots/50-rule2-detail-ec2schedulerule2.png)
![Step 2 — cron(0 14 * * ? *), triggers daily at 3:00 PM GMT+1](screenshots/51-rule2-cron-3pm.png)
![Step 3 — StopStartEC2Lambda selected as target for EC2ScheduleRule2](screenshots/52-rule2-select-target.png)
![Step 5 — Review and create: full summary of EC2ScheduleRule2](screenshots/53-rule2-review-create.png)

`EC2ScheduleRule2` was created successfully.

![EC2ScheduleRule2 created — Status: Enabled, Type: Scheduled Standard](screenshots/54-rule2-created-successfully.png)

The Scheduled rules list confirmed both `EC2ScheduleRule` and `EC2ScheduleRule2` were enabled, of type Scheduled Standard, and attached to the default event bus.

![Scheduled rules list — both rules Enabled](screenshots/55-scheduled-rules-list-both-enabled.png)

### 3d. Verifying the automation with CloudWatch Logs

After the EventBridge rules went active, I checked `StopStartEC2Lambda`'s function overview and its CloudWatch log stream to confirm the scheduled triggers were actually firing.

![StopStartEC2Lambda function overview](screenshots/56-stopstart-function-overview.png)

The log stream showed two consecutive executions: the first confirmed the instance was started (`Started instance: i-0d414548022d52325`) and the second confirmed it was stopped (`Stopped instance: i-0d414548022d52325`), both completing within expected durations — confirming the automation cycle worked end-to-end.

![CloudWatch Logs — Started instance, then Stopped instance, both confirmed](screenshots/57-cloudwatch-logs-started-stopped.png)

The EC2 console showed the instance back in a **Stopped** state, matching the last scheduled action.

![EC2 console — instance stopped after the scheduled automation ran](screenshots/58-ec2-final-stopped.png)

## Task 4: Lambda + S3 — Generating Thumbnail Images on Upload

This task builds a small image-processing pipeline: uploading an image to one S3 bucket automatically triggers a Lambda function that generates a thumbnail and saves it to a second bucket, using the Pillow (PIL) library packaged alongside the function code.

### 4.1 Create the S3 buckets

I created two buckets in `eu-west-2` (Europe London): a source bucket to receive uploaded images, and a separate destination bucket to hold the generated thumbnails.

![Searching for S3 in the AWS console](screenshots/59-s3-search.png)
![Create bucket — source bucket: my-thumbnail-source-bucket-seun](screenshots/60-create-source-bucket.png)
![Create bucket — destination bucket: my-thumbnail-destination-bucket-seun](screenshots/61-create-destination-bucket.png)

Both buckets appeared in the S3 Buckets list once created.

![S3 Buckets list — both thumbnail buckets confirmed](screenshots/62-buckets-list-confirmed.png)

### 4.2 Create the IAM role

Before the Lambda function could read from the source bucket and write to the destination bucket, it needed a role with the right permissions. I created one with **AmazonS3FullAccess** (to read/write both buckets) and **AWSLambdaBasicExecutionRole** (to write logs to CloudWatch).

![Searching for IAM in the AWS console](screenshots/63-iam-search.png)
![Select trusted entity — AWS service, use case Lambda](screenshots/64-iam-trusted-entity-lambda.png)
![AmazonS3FullAccess policy found and selected](screenshots/65-iam-attach-s3fullaccess.png)
![AWSLambdaBasicExecutionRole selected — 2 of 2 policies attached](screenshots/66-iam-attach-lambdabasicexecution.png)

I named the role `LambdaS3ThumbnailRole-Seun`, reviewed both attached policies, and created it.

![Name, review and create — LambdaS3ThumbnailRole-Seun with both policies shown](screenshots/67-iam-role-review.png)
![IAM role created — both policies confirmed](screenshots/68-iam-role-created.png)

### 4.3 Create the Lambda function

With the buckets and role in place, I created the function itself — `GenerateThumbnail-Seun`, **Author from scratch**, Python 3.12 — and attached `LambdaS3ThumbnailRole-Seun` directly via **Use another role**.

![Create function — GenerateThumbnail-Seun, Python 3.12, LambdaS3ThumbnailRole-Seun](screenshots/69-create-function-generatethumbnail.png)

After creation, the function overview showed no triggers configured yet — that came next.

![Function overview — GenerateThumbnail-Seun created, no triggers yet](screenshots/70-function-overview-no-triggers.png)

### 4.4 Add the S3 trigger

I added a trigger so the function would fire automatically whenever a new object was uploaded to the source bucket. I selected **S3** as the source, `my-thumbnail-source-bucket-seun` as the bucket, and **PUT** as the event type. Since separate source and destination buckets were used, I acknowledged the recursive-invocation checkbox (which only matters if a function writes back into the same bucket that triggers it).

![Add trigger — S3 source bucket selected with PUT event type](screenshots/71-add-trigger-s3-put.png)

The trigger showed as active and linked to the source bucket.

![Triggers (1) — S3 trigger confirmed pointing to my-thumbnail-source-bucket-seun](screenshots/72-trigger-confirmed.png)

### 4.5 Upload the code package

The function needed the Pillow (PIL) library for image processing, which isn't included in the standard Lambda runtime. I packaged the function code together with Pillow and its dependencies into a `.zip` file (`lambda_package.zip`, 20.65 MB) and uploaded it via the console.

![Code source — empty editor before zip package upload](screenshots/73-code-editor-empty.png)
![Upload from dropdown — .zip file option selected](screenshots/74-upload-from-zip-dropdown.png)
![File picker — lambda_package.zip visible in Downloads folder](screenshots/75-file-picker-lambda-package.png)
![Upload dialog — lambda_package.zip (20.65 MB) selected and ready to save](screenshots/76-upload-dialog-confirm.png)

After the upload completed, the Code source explorer showed the full contents of the package — `boto3`, `botocore`, `PIL`, `urllib3`, `dateutil`, `jmespath`, and the other dependencies — confirming Pillow had been included successfully.

![Code source explorer — zip uploaded, all packages including PIL and boto3 visible](screenshots/77-packages-visible-after-upload.png)

### 4.6 Set the environment variable and review the code

Rather than hardcoding the destination bucket name into the function, I stored it as an environment variable, `DEST_BUCKET`, set to `my-thumbnail-destination-bucket-seun`.

![Environment variables — DEST_BUCKET set to my-thumbnail-destination-bucket-seun](screenshots/78-env-var-dest-bucket-edit.png)
![Environment variables confirmed — DEST_BUCKET saved](screenshots/79-env-var-confirmed.png)

I then reviewed `lambda_function.py`. It creates an S3 client with `boto3.client('s3')`, reads the destination bucket from the `DEST_BUCKET` environment variable, pulls the source bucket and object key out of the S3 event, filters for supported image types, and includes retry logic around `s3.get_object()` to handle S3's eventual consistency (`NoSuchKey`) on very recent uploads:

```python
import boto3
import os
import time
import io
from PIL import Image
import botocore.exceptions

s3 = boto3.client('s3')

# Destination bucket should be set as an environment variable
DEST_BUCKET = os.environ['DEST_BUCKET']

def lambda_handler(event, context):
    source_bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']

    # Filter supported image types
    if not key.lower().endswith(('.png', '.jpg', '.jpeg', '.webp')):
        print(f"File {key} is not a supported image format.")
        return

    # Retry logic in case of eventual consistency (NoSuchKey)
    for attempt in range(3):
        try:
            response = s3.get_object(Bucket=source_bucket, Key=key)
            break  # Success
        except botocore.exceptions.ClientError as e:
            if e.response['Error']['Code'] == 'NoSuchKey':
                print(f"[Retry {attempt}] ...")
                time.sleep(1)
```

![lambda_function.py — thumbnail generation code with PIL, retry logic and environment variable](screenshots/80-lambda-function-code.png)

### 4.7 First test upload — timeout failure

Before uploading, I confirmed the source bucket was empty.

![Source bucket my-thumbnail-source-bucket-seun — empty before first upload](screenshots/81-source-bucket-empty.png)

I uploaded a 9.0 MB test image (`wallhaven-zyvwyy_3200`, PNG), which completed successfully.

![S3 Upload screen — 9.0 MB PNG image selected for upload](screenshots/82-upload-test-image-9mb.png)
![Upload status — 1 file (9.0 MB) uploaded successfully, 0 failures](screenshots/83-upload-succeeded-9mb.png)

**Problem encountered:** after the upload, I checked the destination bucket for the generated thumbnail — nothing was there. The Lambda function had been triggered but failed to complete within the default 3-second timeout; a 9.0 MB image was too large to process in that window, so the function was timing out before it could finish generating and saving the thumbnail.

**Resolution:** I increased the function's timeout from the default 3 seconds to 1 minute, under **Configuration → General configuration → Edit**.

![Function after timeout fix — timeout updated to 1 min 0 sec, S3 trigger connected](screenshots/84-function-after-timeout-fix.png)

### 4.8 Re-test after the timeout fix

I deleted the previously uploaded file and uploaded two fresh images to the source bucket to trigger the S3 PUT event again with the corrected timeout in place: `wallhaven-0joerm_2800x1752.png` (692.8 KB) and `wallhaven-nzm3do_2800x1752.png` (6.4 MB).

![Source bucket — 2 new images uploaded successfully after the timeout was increased](screenshots/85-source-bucket-2-images-uploaded.png)

This time, a `thumbnails/` folder appeared automatically in the destination bucket, confirming the function had executed successfully and written output.

![Destination bucket — thumbnails/ folder created automatically by the function](screenshots/86-destination-bucket-thumbnails-folder.png)

Inside the folder were two PNG thumbnails, one per uploaded image, both dramatically smaller than their originals: `wallhaven-0joerm_2800x1752.png` reduced to 5.9 KB, and `wallhaven-nzm3do_2800x1752.png` reduced to 18.0 KB. Both were generated within seconds of the uploads, confirming the pipeline worked end-to-end.

![thumbnails/ folder — 2 thumbnail PNG files generated (5.9 KB and 18.0 KB)](screenshots/87-thumbnails-folder-2-files.png)

## Key concepts

| Concept | Description |
|---|---|
| Blueprint vs. from scratch | A blueprint gives a working, pre-configured function (here, a Firehose stream processor) that can be customized; "from scratch" starts from a blank Hello World template with full control over runtime, architecture, and logic from the first line. |
| Runtime and architecture choice | Each function independently chooses its Python runtime version (3.12 for the blueprint, 3.10 from scratch) and CPU architecture (x86_64 here; arm64 was also available) — these aren't fixed per account. |
| Execution role | By default, Lambda creates a new IAM role per function scoped to writing CloudWatch Logs; this can be swapped for an existing role if the function needs to touch other AWS services. |
| Test events | Saved, reusable JSON payloads (synchronous or asynchronous, private or shareable) that let you invoke a function repeatedly with the same input without wiring up a real trigger. |
| Editing in VS Code vs. the browser editor | The Lambda console's "Open in Visual Studio Code" option gives a full editor experience against the same deployed function code as the lightweight in-browser editor. |
| Cleaning up functions | Lambda functions don't cost anything while idle (billing is per-invocation), but deleting unused functions after a lab keeps the account tidy and avoids any confusion in future exercises. |
| `=` vs. `==` | `=` assigns a value; `==` compares two values. Using `=` inside an `if` condition is a Python syntax error, not a silent bug — it won't run at all until corrected to `==`. |
| Lambda execution role permissions | A Lambda function can only call the AWS APIs its execution role explicitly allows. The default role only permits writing to CloudWatch Logs — any other service (like EC2) needs a policy attached before the function can use it. |
| `UnauthorizedOperation` errors | This IAM-level error means the caller (here, the Lambda function's role) doesn't have permission for the action being attempted — the fix is to grant permission via the role, not to change the function's code. |
| `boto3.client('ec2')` actions | `stop_instances`, `start_instances`, and `terminate_instances` all take an `InstanceIds` list and are asynchronous — they trigger the state change but don't wait for it to complete before the function returns. |
| Tag-based instance discovery | `ec2.describe_instances()` with a `Filters` argument lets a function find instances dynamically (e.g. everything tagged `AutoSchedule=True`) instead of hardcoding instance IDs, so the same function keeps working even if instances are replaced. |
| EventBridge scheduled rules | A cron-based EventBridge rule can invoke a Lambda function on a recurring schedule with no polling or always-on compute required; EventBridge auto-creates a scoped IAM role to invoke the target. |
| Reading console banners before troubleshooting | When an AWS console's UI has changed since a lab was written, the console itself often explains where things moved (e.g. an orange banner pointing from Buses → Rules to Scheduler) — worth reading before assuming something's broken. |
| CloudWatch Logs as a verification tool | For scheduled or event-driven automation with no visible UI trigger, CloudWatch Logs is the way to confirm the automation actually ran, and when. |
| S3 event triggers | An S3 PUT trigger invokes a Lambda function automatically on every new object upload, passing the bucket name and object key in the event payload — no polling required. |
| Packaging dependencies for Lambda | Libraries not included in the base runtime (like Pillow) must be packaged into the deployment `.zip` alongside the function code, since Lambda can't `pip install` at runtime. |
| Separate source/destination buckets | Using two buckets (rather than writing output back into the trigger bucket) avoids the risk of a recursive invocation loop. |
| Silent timeouts | A Lambda function can be successfully triggered and still produce no output if it runs out of time mid-execution — this can look identical to "nothing happened" unless you check the timeout setting specifically. |

## What I learned

Working through both methods back to back made the trade-off clear: the blueprint got me to a working, realistic piece of code (a Firehose processor) almost immediately, but customizing it meant understanding someone else's code structure first. Writing `cba-lamda-02` from scratch took a little longer to set up but meant every line of logic was mine, which made it much faster to reason about and extend — for example, adding the second test event (`cbaEvent3`) to specifically exercise the `else` branch was an easy next step once I understood exactly what the function was checking for. The "Cloud boosta" (with a space) test case was a good reminder that Python's `==` does exact string matching — a value that's visually close but not identical will always fall through to the `else` branch. The `=`/`==` slip in Part B was a useful, if basic, reminder to read my own conditionals carefully before deploying — the kind of typo that's easy to make and just as easy to miss on a quick glance.

Task 2 was a good introduction to how Lambda's permissions model actually works in practice: the `UnauthorizedOperation` error on my first invocation wasn't a code bug at all, it was a missing permission on the function's role. Attaching `AmazonEC2FullAccess` fixed it immediately, which made the separation between "what the code does" and "what the code is allowed to do" very concrete. Reusing the same function across all three lifecycle actions (stop, start, terminate) by commenting out the previous block and writing the next one below it was a quick way to iterate, though for anything beyond a lab I'd want three separate, purpose-built functions rather than one function with commented-out history sitting in it.

Task 3 tied everything together into something closer to a real automation pipeline: one function provisions infrastructure, a second function manages its lifecycle by reading a tag rather than a hardcoded ID, and EventBridge removes the need for me to trigger anything manually. The most useful moment wasn't technical so much as procedural — hitting a UI that didn't match the assignment's screenshots and having to actually read the console's own banner to find where AWS had moved the feature, rather than assuming the lab instructions were wrong. It's a good reminder that AWS consoles change faster than static documentation, and that the console usually tells you where things went if you read it carefully. Setting up two schedules (9:00 AM and 3:00 PM GMT+1) and then confirming both fired correctly via CloudWatch Logs — rather than just trusting the rule was "Enabled" — was the right way to close the loop on something with no visible on-screen confirmation of its own.

Task 4 was the first time in this set of labs that a Lambda function needed a real third-party dependency, and packaging Pillow into the deployment zip made the runtime limitations concrete in a way reading about them hadn't. The most instructive moment was the first test upload: everything looked correctly configured, the trigger fired, and yet nothing appeared in the destination bucket — with no error message anywhere to point at. It took checking the timeout setting specifically to realise the function was running out of time partway through processing a 9 MB image, rather than failing outright. That's a different category of bug from the earlier IAM permission errors, which at least surface as a clear `UnauthorizedOperation`; a silent timeout produces no output and no obvious failure signal, so it's worth checking as a first suspect whenever a triggered function seems to do nothing at all.
