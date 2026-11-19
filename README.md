# Project 8: Amazon DynamoDB – TTL, Streams and Lambda Integration

## Objective
To automatically delete expired gaming login sessions using DynamoDB TTL and track player data changes using DynamoDB Streams and AWS Lambda.

## Services Used
- Amazon DynamoDB
- AWS Lambda
- Amazon CloudWatch
- AWS CloudShell
- Python (Boto3)

## Step 1: Create DynamoDB Tables

1. Open **Amazon DynamoDB** in AWS Console.
2. Navigate to **Tables → Create table**.
3. Create the following two tables:

| Table Name | Partition Key | Key Type |
|---|---|---|
| sessions | session_id | String |
| players | player_id | String |

4. Keep other settings as default and create both tables.

## Step 2: Upload CSV Data Using Boto3

1. Open **AWS CloudShell**.
2. Upload the following CSV files:
   - `Activity13_DynamoDB_sessions.csv`
   - `Activity13_DynamoDB_players.csv`
3. Create a Python script named `upload.py`.
4. Use Boto3 to read the CSV files and insert records into their respective DynamoDB tables.
5. Convert `expiry_time` into an integer (Number type) before inserting session records.
6. Execute the script:

```bash
python3 upload.py
```

7. Verify the uploaded records using **DynamoDB → Explore table items**.

## Step 3: Enable TTL on Sessions Table

1. Open the `sessions` table.
2. Navigate to **Additional settings → Time to Live (TTL)**.
3. Click **Enable TTL**.
4. Enter `expiry_time` as the TTL attribute name.
5. Enable TTL and verify that its status changes to **Enabled**.

**Note:** DynamoDB deletes expired items asynchronously, usually within a few days. Immediate deletion is not guaranteed.

## Step 4: Enable DynamoDB Streams on Players Table

1. Open the `players` table.
2. Navigate to **Exports and streams**.
3. Find **DynamoDB stream details** and click **Enable**.
4. Select `NEW_AND_OLD_IMAGES` as the Stream view type.
5. Enable the stream.

## Step 5: Create a Lambda Function

1. Open **AWS Lambda**.
2. Click **Create function → Author from scratch**.
3. Enter the following details:

| Setting | Value |
|---|---|
| Function name | PlayerChangeLogger |
| Runtime | Python 3.12 |
| Architecture | x86_64 |
| Execution role | Create a new role with basic Lambda permissions |

4. Click **Create function**.

## Step 6: Add Lambda Code

1. Open the `PlayerChangeLogger` function.
2. Navigate to the **Code** tab.
3. Add Python code to process DynamoDB stream records.
4. Handle the following event types:
   - `INSERT` – Logs newly created player records.
   - `MODIFY` – Logs updated player records along with old and new values.
   - `REMOVE` – Logs deleted player records.
5. Click **Deploy**.

## Step 7: Connect DynamoDB Stream to Lambda

1. Open the Lambda function.
2. Click **Add trigger**.
3. Select **DynamoDB** as the trigger source.
4. Select the `players` table.
5. Configure the following settings:

| Setting | Value |
|---|---|
| Table | players |
| Batch size | 100 |
| Starting position | Latest |
| Enable trigger | Checked |

6. Click **Add**.
7. Ensure the Lambda execution role has the required DynamoDB Streams read permissions and CloudWatch Logs permissions.

## Step 8: Insert, Update and Delete Player Records

Open **DynamoDB → players → Explore table items**.

**A. Insert a Player**

Create a new item with the following attributes:

| Attribute | Type | Value |
|---|---|---|
| player_id | String | PL_TEST |
| name | String | TestPlayer |
| level | Number | 10 |
| coins | Number | 500 |
| country | String | IN |

Click **Create item**. This generates an `INSERT` event.

**B. Update a Player**

1. Select the `PL_TEST` item.
2. Click **Edit item**.
3. Change `level` from 10 to 20.
4. Change `coins` from 500 to 1000.
5. Click **Save changes**.

This generates a `MODIFY` event.

**C. Delete a Player**

1. Select the `PL_TEST` item.
2. Click **Delete item**.
3. Confirm deletion.

This generates a `REMOVE` event.

## Step 9: Verify CloudWatch Logs

1. Open **AWS Lambda → PlayerChangeLogger**.
2. Navigate to the **Monitor** tab.
3. Click **View CloudWatch logs**.
4. Open the latest log stream.
5. Verify that all three events are recorded:

```text
Player Data Change: INSERT
Player Data Change: MODIFY
Player Data Change: REMOVE
```

## Expected Output

| Task | Expected Result |
|---|---|
| Sessions Table | Successfully created |
| Players Table | Successfully created |
| CSV Data Upload | Records inserted successfully |
| TTL Configuration | Enabled on `expiry_time` |
| DynamoDB Streams | Enabled on `players` |
| Lambda Integration | Successfully connected |
| INSERT Event | Visible in CloudWatch Logs |
| MODIFY Event | Visible in CloudWatch Logs |
| REMOVE Event | Visible in CloudWatch Logs |

## Result

Successfully implemented DynamoDB TTL to manage expired login sessions and integrated DynamoDB Streams with AWS Lambda to capture and log player data changes in Amazon CloudWatch.
