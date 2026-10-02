# Snowpipe + AWS S3 — Hands-on Lab

This README is a practical companion to Snowflake's **Getting Started with Snowpipe** guide.

**Official Snowflake guide:** https://www.snowflake.com/en/developers/guides/getting-started-with-snowpipe/

The lab builds an automated ingestion pipeline:

```text
AWS S3 Bucket
     |
     | ObjectCreated event
     v
Amazon SQS
     |
     | notification
     v
Snowpipe (auto_ingest = TRUE)
     |
     | COPY INTO
     v
Snowflake Table
```

## What you will learn

- Create an AWS S3 bucket and prefix/folder
- Create an AWS IAM policy and role for Snowflake
- Create a Snowflake storage integration
- Configure the IAM trust relationship
- Create a Snowflake database, table, external stage, and pipe
- Enable Snowpipe auto-ingest
- Configure an S3 event notification to Snowflake's SQS queue
- Check pipe status and verify ingestion
- Pause and remove a Snowpipe

---

## Prerequisites

### AWS

- AWS account
- Permission to create:
  - S3 buckets
  - IAM policies
  - IAM roles
  - S3 event notifications

### Snowflake

- Snowflake account
- `ACCOUNTADMIN` access for the setup
- An AWS-hosted Snowflake account is expected for this lab configuration
- AWS and Snowflake should be configured in a compatible region

> **Important:** Never commit AWS access keys, secret keys, passwords, or other credentials to this repository.

---

# 1. Create the S3 bucket

1. Sign in to AWS.
2. Open **S3**.
3. Select **Create bucket**.
4. Give the bucket a globally unique name.

Example:

```text
suma-snowpipe-demo-2026
```

5. Select an AWS region.
6. Leave the remaining settings at their defaults unless your lab requires otherwise.
7. Create the bucket.

Record:

```text
Bucket name: <YOUR_BUCKET>
AWS region:  <YOUR_REGION>
```

---

# 2. Create the S3 prefix/folder

Open the bucket and select **Create folder**.

Example:

```text
snowpipe/
```

Your S3 location will therefore look like:

```text
s3://<YOUR_BUCKET>/snowpipe/
```

Record this value because it is used by both AWS IAM and Snowflake.

---

# 3. Create the AWS IAM policy

Snowflake needs permission to read files from the S3 location.

Create an IAM policy named:

```text
snowflake_access
```

Use the following policy, replacing `<YOUR_BUCKET>` and `<YOUR_PREFIX>`.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::<YOUR_BUCKET>/<YOUR_PREFIX>/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::<YOUR_BUCKET>",
      "Condition": {
        "StringLike": {
          "s3:prefix": [
            "<YOUR_PREFIX>/*"
          ]
        }
      }
    }
  ]
}
```

For example, if:

```text
Bucket = suma-snowpipe-demo-2026
Prefix = snowpipe
```

the object resource becomes:

```text
arn:aws:s3:::suma-snowpipe-demo-2026/snowpipe/*
```

---

# 4. Create the AWS IAM role

Create a role named:

```text
snowflake_role
```

Attach the:

```text
snowflake_access
```

policy.

For the initial trust configuration, follow the Snowflake lab's AWS-account/external-ID setup.

Record the role ARN:

```text
arn:aws:iam::<AWS_ACCOUNT_ID>:role/snowflake_role
```

You will need this ARN in Snowflake.

---

# 5. Create the Snowflake storage integration

Open a Snowflake worksheet and run:

```sql
CREATE OR REPLACE STORAGE INTEGRATION S3_role_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = S3
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::<AWS_ACCOUNT_ID>:role/snowflake_role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://<YOUR_BUCKET>/<YOUR_PREFIX>/');
```

Replace:

- `<AWS_ACCOUNT_ID>`
- `<YOUR_BUCKET>`
- `<YOUR_PREFIX>`

Then inspect the integration:

```sql
DESC INTEGRATION S3_role_integration;
```

You need to record these two values:

```text
STORAGE_AWS_IAM_USER_ARN
STORAGE_AWS_EXTERNAL_ID
```

---

# 6. Update the AWS IAM trust policy

Go back to:

**AWS Console → IAM → Roles → snowflake_role → Trust relationships**

Update the trust policy using the values returned by:

```sql
DESC INTEGRATION S3_role_integration;
```

Example structure:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "<STORAGE_AWS_IAM_USER_ARN>"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "<STORAGE_AWS_EXTERNAL_ID>"
        }
      }
    }
  ]
}
```

Replace the placeholders with the actual values from Snowflake.

---

# 7. Create the Snowflake database and table

Run:

```sql
CREATE OR REPLACE DATABASE S3_db;

CREATE OR REPLACE TABLE S3_db.public.S3_table (
    files STRING
);
```

Use the schema:

```sql
USE SCHEMA S3_db.public;
```

The lab table intentionally has a simple single-column structure:

```text
files STRING
```

Your input file should therefore contain data that can be loaded into this structure.

---

# 8. Create the external stage

Create a stage pointing to the S3 prefix:

```sql
CREATE OR REPLACE STAGE S3_stage
  URL = 's3://<YOUR_BUCKET>/<YOUR_PREFIX>/'
  STORAGE_INTEGRATION = S3_role_integration;
```

Verify the stage:

```sql
LIST @S3_stage;
```

If the S3 folder is empty, `LIST` may return no files. That is expected.

---

# 9. Create the Snowpipe

Create the pipe with automatic ingestion enabled:

```sql
CREATE OR REPLACE PIPE S3_db.public.S3_pipe
  AUTO_INGEST = TRUE
AS
  COPY INTO S3_db.public.S3_table
  FROM @S3_db.public.S3_stage;
```

The important setting is:

```sql
AUTO_INGEST = TRUE
```

This allows Snowpipe to react to cloud-storage notifications instead of requiring you to manually execute `COPY INTO` for every new file.

---

# 10. Configure Snowpipe permissions

The supplied class SQL creates a dedicated role:

```sql
USE ROLE ACCOUNTADMIN;

ALTER PIPE S3_db.public.S3_pipe
  SET PIPE_EXECUTION_PAUSED = TRUE;

GRANT OWNERSHIP
  ON PIPE S3_db.public.S3_pipe
  TO ROLE ACCOUNTADMIN;

CREATE OR REPLACE ROLE S3_role;

GRANT USAGE
  ON DATABASE S3_db
  TO ROLE S3_role;

GRANT USAGE
  ON SCHEMA S3_db.public
  TO ROLE S3_role;

GRANT INSERT, SELECT
  ON TABLE S3_db.public.S3_table
  TO ROLE S3_role;

GRANT USAGE
  ON STAGE S3_db.public.S3_stage
  TO ROLE S3_role;

GRANT OWNERSHIP
  ON PIPE S3_db.public.S3_pipe
  TO ROLE S3_role;
```

Then grant the role to your Snowflake user:

```sql
GRANT ROLE S3_role TO USER <YOUR_USERNAME>;

ALTER USER <YOUR_USERNAME>
  SET DEFAULT_ROLE = S3_role;
```

> Replace `<YOUR_USERNAME>` with your Snowflake username. Do not blindly use the username from someone else's class script.

---

# 11. Get the Snowpipe notification channel

Run:

```sql
SHOW PIPES;
```

Find:

```text
S3_PIPE
```

and copy the value from:

```text
notification_channel
```

This is the Snowflake-managed SQS queue ARN that AWS S3 needs to send notifications to.

Keep this value handy.

---

# 12. Create the S3 event notification

Go to:

**AWS Console → S3 → Your bucket → Properties → Event notifications**

Create an event notification.

Suggested configuration:

```text
Event name:
Auto-ingest Snowflake

Event types:
All object create events

Destination:
SQS Queue

SQS queue:
Use the ARN from SHOW PIPES
```

If your lab is intended to monitor only the Snowpipe folder, configure the object key/prefix filter for:

```text
<YOUR_PREFIX>/
```

Save the notification.

### Important

AWS does not allow conflicting/overlapping notification configurations for the same S3 path. If the bucket already has an event notification covering the target path, inspect it before creating another one.

---

# 13. Upload a test file

Create a small test file locally.

Example:

```text
hello
snowpipe
test
```

Upload it to:

```text
s3://<YOUR_BUCKET>/<YOUR_PREFIX>/
```

For example:

```text
s3://suma-snowpipe-demo-2026/snowpipe/test.txt
```

The expected flow is:

```text
test.txt
   |
   v
S3 ObjectCreated event
   |
   v
SQS notification
   |
   v
Snowpipe
   |
   v
COPY INTO S3_db.public.S3_table
   |
   v
S3_table
```

---

# 14. Check Snowpipe status

Run:

```sql
SELECT SYSTEM$PIPE_STATUS('S3_db.public.S3_pipe');
```

You can also inspect the pipe:

```sql
SHOW PIPES;
```

The status information helps determine whether Snowpipe is running and processing files.

---

# 15. Verify the loaded data

Query the target table:

```sql
SELECT *
FROM S3_db.public.S3_table;
```

You should see the contents of the file that was uploaded to S3.

You can also inspect the stage:

```sql
LIST @S3_db.public.S3_stage;
```

---

# 16. Useful Snowpipe commands

## Check all pipes

```sql
SHOW PIPES;
```

## Check a specific pipe

```sql
DESC PIPE S3_db.public.S3_pipe;
```

## Check pipe status

```sql
SELECT SYSTEM$PIPE_STATUS('S3_db.public.S3_pipe');
```

## Pause a pipe

```sql
ALTER PIPE S3_db.public.S3_pipe
SET PIPE_EXECUTION_PAUSED = TRUE;
```

## Resume a pipe

```sql
ALTER PIPE S3_db.public.S3_pipe
SET PIPE_EXECUTION_PAUSED = FALSE;
```

## Drop the pipe

```sql
DROP PIPE S3_db.public.S3_pipe;
```

---

# 17. Troubleshooting checklist

### Pipe exists but data is not loading

Check:

```sql
SHOW PIPES;
```

Then:

```sql
SELECT SYSTEM$PIPE_STATUS('S3_db.public.S3_pipe');
```

Check that:

- `AUTO_INGEST = TRUE`
- S3 event notification exists
- S3 event notification points to the correct SQS ARN
- The S3 prefix matches the stage location
- The file was uploaded after the notification was configured
- IAM permissions allow Snowflake to read the object
- IAM trust policy contains the correct Snowflake IAM user ARN
- IAM trust policy contains the correct external ID

### `LIST @S3_stage` returns nothing

Check:

```text
s3://<YOUR_BUCKET>/<YOUR_PREFIX>/
```

Make sure the uploaded file is actually under that prefix.

### Storage integration cannot access S3

Run:

```sql
DESC INTEGRATION S3_role_integration;
```

Verify:

```text
STORAGE_AWS_IAM_USER_ARN
STORAGE_AWS_EXTERNAL_ID
```

Then compare those values with the AWS role trust policy.

### Snowpipe is not receiving notifications

Run:

```sql
SHOW PIPES;
```

Copy the:

```text
notification_channel
```

and compare it with the SQS queue ARN configured in the S3 event notification.

---

# 18. Cleanup

If this is only a training environment, clean up resources when finished.

### Snowflake

```sql
DROP PIPE IF EXISTS S3_db.public.S3_pipe;
DROP STAGE IF EXISTS S3_db.public.S3_stage;
DROP TABLE IF EXISTS S3_db.public.S3_table;
DROP DATABASE IF EXISTS S3_db;
DROP INTEGRATION IF EXISTS S3_role_integration;
DROP ROLE IF EXISTS S3_role;
```

### AWS

Remove resources that were created specifically for this lab:

- S3 event notification
- IAM role
- IAM policy
- S3 bucket/prefix

Be careful not to delete shared or production resources.

---

# 19. Class SQL reference

The original class setup SQL is included alongside this README as:

```text
hol_setup.sql
```

The supplied script creates:

```text
S3_role_integration
S3_db
S3_table
S3_stage
S3_pipe
S3_role
```

It also contains commands for:

```text
DESC INTEGRATION
SHOW PIPES
SYSTEM$PIPE_STATUS
```

Before executing the class script, replace the empty placeholders for:

```sql
STORAGE_AWS_ROLE_ARN = ''
STORAGE_ALLOWED_LOCATIONS = ('')
URL = ('')
SELECT SYSTEM$PIPE_STATUS('')
```

with your actual AWS/Snowflake values.

---

# 20. Key concepts to remember

## Stage

A Snowflake stage is a location from which Snowflake can load data. In this lab, the stage points to an S3 location.

```text
S3 → Snowflake Stage
```

## Storage Integration

A storage integration provides a secure way for Snowflake to access external cloud storage without embedding AWS credentials directly in the stage definition.

```text
Snowflake
    |
    v
Storage Integration
    |
    v
AWS IAM Role
    |
    v
S3
```

## Pipe

A Snowpipe defines the loading operation.

```sql
CREATE PIPE ...
AS
COPY INTO ...
FROM @stage;
```

## Auto-ingest

```sql
AUTO_INGEST = TRUE
```

allows Snowpipe to be triggered by cloud-storage event notifications.

## S3 Event Notification

S3 generates an event when a new object is created. The notification is delivered through the Snowflake-managed SQS queue associated with the pipe.

```text
S3 ObjectCreated
      ↓
SQS
      ↓
Snowpipe
      ↓
COPY INTO
      ↓
Snowflake Table
```

---

## Reference

- Snowflake — Getting Started with Snowpipe:
  https://www.snowflake.com/en/developers/guides/getting-started-with-snowpipe/
- Snowflake — Automating Snowpipe for Amazon S3:
  https://docs.snowflake.com/en/user-guide/data-load-snowpipe-auto-s3
- Snowflake — Configure an S3 storage integration:
  https://docs.snowflake.com/en/user-guide/data-load-s3-config-storage-integration

> Snowflake's developer guide notes that the guide was updated December 20, 2025 and may not always reflect the latest Snowflake UI or product behavior. Use the current Snowflake documentation when a command or AWS configuration differs from the class material.
