# Project 8: Amazon DynamoDB – TTL, Streams and Lambda Integration

## Objective
To automatically delete expired gaming login sessions using DynamoDB TTL and track every change made to player data using DynamoDB Streams and AWS Lambda.

## AWS Services Used
- Amazon DynamoDB
- AWS Lambda
- Amazon CloudWatch
- AWS CloudShell
- Python (Boto3)

## Dataset Files
1. `Activity13_DynamoDB_sessions.csv`
2. `Activity13_DynamoDB_players.csv`

---

# Part A: Create DynamoDB Tables and Load Data

## Step 1: Open Amazon DynamoDB

1. Log in to the AWS Management Console.
2. Search for **DynamoDB**.
3. Open Amazon DynamoDB.
4. Select **Tables** from the left navigation panel.

## Step 2: Create Sessions Table

1. Click **Create table**.
2. Enter the following details:

| Setting | Value |
|---|---|
| Table name | sessions |
| Partition key | session_id |
| Key type | String |
| Sort key | Not required |

3. Keep other settings as default.
4. Click **Create table**.

## Step 3: Create Players Table

1. Click **Create table** again.
2. Enter the following details:

| Setting | Value |
|---|---|
| Table name | players |
| Partition key | player_id |
| Key type | String |
| Sort key | Not required |

3. Keep other settings as default.
4. Click **Create table**.

## Step 4: Open AWS CloudShell and Upload CSV Files

1. Open **AWS CloudShell** from the top navigation bar of AWS Console.
2. Click **Actions → Upload file**.
3. Upload both CSV files:
   - `Activity13_DynamoDB_sessions.csv`
   - `Activity13_DynamoDB_players.csv`
4. Verify that both files are available in the CloudShell directory.

## Step 5: Create Boto3 Python Script

Create a Python file named `upload.py`:

```bash
nano upload.py
```

Paste the following code:

```python
import boto3
import csv

dynamodb = boto3.resource('dynamodb')

def load_data(filename, table_name):
    table = dynamodb.Table(table_name)

    with open(filename, 'r') as file:
        reader = csv.DictReader(file)

        with table.batch_writer() as batch:
            for row in reader:

                if table_name == 'sessions':
                    row['login_time'] = int(row['login_time'])
                    row['expiry_time'] = int(row['expiry_time'])

                elif table_name == 'players':
                    row['level'] = int(row['level'])
                    row['coins'] = int(row['coins'])

                batch.put_item(Item=row)

    print(table_name, "data uploaded successfully")


load_data(
    'Activity13_DynamoDB_sessions.csv',
    'sessions'
)

load_data(
    'Activity13_DynamoDB_players.csv',
    'players'
)
```

Save the file:
- Press `Ctrl + O` and Enter.
- Press `Ctrl + X`.

## Step 6: Execute the Python Script

Run the following command:

```bash
python3 upload.py
```

Expected output:

```text
sessions data uploaded successfully
players data uploaded successfully
```

**Note:** The script uses Boto3 to connect to DynamoDB, reads both CSV files, converts numeric attributes into integers and inserts records using batch_writer().

CloudShell uses your current AWS Console credentials. Ensure your IAM identity has permission to write items to both DynamoDB tables.

## Step 7: Verify Uploaded Data

1. Open **DynamoDB → Tables → sessions**.
2. Select **Explore table items**.
3. Verify that session records are present.
4. Repeat the same process for the `players` table.
5. Check that `expiry_time`, `login_time`, `level` and `coins` are stored as Number attributes.

---

# Part B: Enable TTL on Sessions Table

## Step 8: Configure Time to Live (TTL)

1. Open **DynamoDB → Tables → sessions**.
2. Navigate to the **Additional settings / Time to Live (TTL)** section.
3. Click **Enable TTL**.
4. Enter the TTL attribute name:

```text
expiry_time
```

5. Click **Enable TTL** to confirm.
6. Verify that TTL status changes to **Enabled**.

**Note:** The `expiry_time` attribute contains Unix epoch timestamps in seconds. DynamoDB automatically deletes expired items in the background, usually within a few days. Immediate deletion is not guaranteed.

---

# Part C: Enable DynamoDB Streams and Integrate Lambda

## Step 9: Enable DynamoDB Streams on Players Table

1. Open **DynamoDB → Tables → players**.
2. Navigate to the **Exports and streams** tab.
3. Find the **DynamoDB stream details** section.
4. Click **Enable**.
5. Select the following Stream view type:

```text
NEW_AND_OLD_IMAGES
```

6. Click **Enable stream**.
7. Verify that the stream status is enabled.

**Why NEW_AND_OLD_IMAGES?**

It captures both the previous and updated versions of an item, allowing us to track changes in player records.

## Step 10: Create a Lambda Function

1. Search for **AWS Lambda** in AWS Console.
2. Click **Create function**.
3. Select **Author from scratch**.
4. Enter the following details:

| Setting | Value |
|---|---|
| Function name | PlayerChangeLogger |
| Runtime | Python 3.12 |
| Architecture | x86_64 |
| Execution role | Create a new role with basic Lambda permissions |

5. Click **Create function**.

## Step 11: Configure Lambda Permissions

1. Open the `PlayerChangeLogger` function.
2. Navigate to **Configuration → Permissions**.
3. Click the execution role name to open it in IAM.
4. Select **Add permissions → Attach policies**.
5. Search for and attach:

```text
AWSLambdaDynamoDBExecutionRole
```

6. Verify that the role has the required permissions for:
   - Reading DynamoDB Streams.
   - Writing logs to CloudWatch.

This managed policy includes the required DynamoDB Streams read permissions and basic CloudWatch Logs permissions.

## Step 12: Add Python Code to Lambda

1. Open the `PlayerChangeLogger` function.
2. Navigate to the **Code** tab.
3. Replace the existing code with:

```python
def lambda_handler(event, context):

    for record in event['Records']:

        event_name = record['eventName']

        print("Player Data Change:", event_name)

        if event_name == 'INSERT':

            print(
                "New Player:",
                record['dynamodb'].get('NewImage')
            )

        elif event_name == 'MODIFY':

            print(
                "Updated Player:",
                record['dynamodb'].get('NewImage')
            )

            print(
                "Old Data:",
                record['dynamodb'].get('OldImage')
            )

        elif event_name == 'REMOVE':

            print(
                "Deleted Player:",
                record['dynamodb'].get('OldImage')
            )

    return {
        'statusCode': 200,
        'body': 'Player changes logged successfully'
    }
```

4. Click **Deploy**.

## Step 13: Connect DynamoDB Stream to Lambda

1. Open the `PlayerChangeLogger` Lambda function.
2. Click **Add trigger**.
3. Select **DynamoDB** as the trigger source.
4. Configure the following details:

| Setting | Value |
|---|---|
| Table | players |
| Batch size | 100 |
| Starting position | Latest |
| Enable trigger | Checked |

5. Click **Add**.
6. Verify that the DynamoDB trigger appears in the Lambda function overview.

**Note:** The Lambda function processes records from the DynamoDB Stream whenever player data changes. Starting position `Latest` means it processes new stream records generated after the event source mapping starts.

---

# Part D: Insert, Update and Delete Player Records

## Step 14: Insert a Player Record

1. Open **DynamoDB → Tables → players**.
2. Select **Explore table items**.
3. Click **Create item**.
4. Add the following attributes:

| Attribute | Type | Value |
|---|---|---|
| player_id | String | PL_TEST |
| name | String | TestPlayer |
| level | Number | 10 |
| coins | Number | 500 |
| country | String | IN |

5. Click **Create item**.

This generates an `INSERT` event in DynamoDB Streams.

## Step 15: Update the Player Record

1. Select the `PL_TEST` item.
2. Click **Edit item**.
3. Change the following values:

| Attribute | Old Value | New Value |
|---|---|---|
| level | 10 | 20 |
| coins | 500 | 1000 |

4. Click **Save changes**.

This generates a `MODIFY` event.

## Step 16: Delete the Player Record

1. Select the `PL_TEST` item.
2. Click **Delete item**.
3. Confirm the deletion.

This generates a `REMOVE` event.

---

# Part E: Verify Lambda Logs in CloudWatch

## Step 17: Check CloudWatch Logs

1. Open **AWS Lambda → PlayerChangeLogger**.
2. Navigate to the **Monitor** tab.
3. Click **View CloudWatch logs**.
4. Open the latest log stream.
5. Verify that all three event types are displayed.

Expected output:

```text
Player Data Change: INSERT

Player Data Change: MODIFY

Player Data Change: REMOVE
```

The logs also contain the new player data, updated player data and deleted player data.

**Note:** Stream processing is asynchronous. Wait for the Lambda function to process the events before checking CloudWatch Logs.

---

# Expected Verification

| Requirement | Expected Result |
|---|---|
| Sessions Table | Created successfully |
| Players Table | Created successfully |
| CSV Data Upload | Records inserted successfully |
| TTL | Enabled on expiry_time |
| DynamoDB Streams | Enabled on players |
| Stream View Type | NEW_AND_OLD_IMAGES |
| Lambda Function | PlayerChangeLogger created |
| Lambda Trigger | Connected to players table |
| INSERT Event | Visible in CloudWatch Logs |
| MODIFY Event | Visible in CloudWatch Logs |
| REMOVE Event | Visible in CloudWatch Logs |

# Result

Successfully implemented Amazon DynamoDB TTL to automatically manage expired gaming login sessions and integrated DynamoDB Streams with AWS Lambda to capture INSERT, MODIFY and REMOVE events. Verified all three events through Amazon CloudWatch Logs.

# Workflow

```text
CSV Files
    |
    v
Boto3 Python Script
    |
    v
DynamoDB Tables
    |
    |-----------------------------|
    |                             |
    v                             v
Sessions Table               Players Table
    |                             |
    v                             v
Enable TTL                 Enable Streams
    |                             |
    v                             v
Auto-delete Expired         Lambda Function
Sessions                         |
                                 v
                            CloudWatch Logs
                                 |
                                 v
                       INSERT / MODIFY / REMOVE
```

# Troubleshooting

| Problem | Solution |
|---|---|
| AccessDeniedException | Check IAM permissions for DynamoDB |
| CSV file not found | Verify that both files are uploaded to CloudShell |
| Table not found | Check table names and AWS Region |
| Lambda not triggered | Verify stream status and Lambda trigger configuration |
| No CloudWatch logs | Check Lambda execution role and wait for stream processing |
| TTL not enabled | Verify the TTL attribute name is exactly `expiry_time` |

# Cleanup (Optional)

To avoid unnecessary AWS charges after completing the experiment:

1. Remove the Lambda trigger.
2. Delete the `PlayerChangeLogger` Lambda function.
3. Delete the `sessions` table.
4. Delete the `players` table.

**Important:** Delete resources only after completing the experiment and capturing all required screenshots.
