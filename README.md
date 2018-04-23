# Serverless Janitor for EBS Snapshots
Many organizations use [Automated EBS snapshots](https://github.com/miztiik/serverless-backup) to create point-in-time recovery points to use in case of data loss or disaster. However, EBS snapshot costs can quickly get out of control if not closely controlled. Individual snapshots are not costly, but the cost can grow quickly when many are created.

Organizations can help get EBS snapshots back under control by using lambda functions to delete older snapshots based on tags or retention dates

![Fig : Valaxy-Serverless-Security-Group-Sentry](https://raw.githubusercontent.com/miztiik/serverless-janitor-ebs-snapshots/master/images/serverless-janitor-ebs-snapshots.png)

You can also follow this article in **[Youtube](https://youtu.be/eAVqOvlsztE)**

## Pre-Requisities
We will need the following pre-requisites to successfully complete this activity,
- Few `EBS Snapshots` with a Tag Key:`DeleteOn` and Value as `Date` in this format `YYYY-MM-DD`
- IAM Role - _i.e_ `Lambda Service Role` - _with_
  - `EC2FullAccess` _permissions_
  - _You may use an `Inline` policy with more restrictive permissions_

_The image above shows the execution order, that should not be confused with the numbering of steps given here_


## Step 1 - Configure Lambda Function- `Serverless Janitor`
The below script is written in `Python 2.7`. Remember to choose the same in AWS Lambda Functions.
### Customisations
_Change the global variables at the top of the script to suit your needs._
- `globalVars['findNeedle']` - My EBS Snapshots have tag `DeleteOn`, Set this to the value your EBS volumes have to have the filter work
- `globalVars['RetentionDays']` - Set the value you desire, by default it is set to 7 days

If you have a lot of EBS Snapshots, then consider increasing the lambda run time, the default is `3` seconds.

```py
import boto3
from botocore.exceptions import ClientError
import datetime

# Set the global variables
globalVars  = {}
globalVars['Owner']                 = "Miztiik"
globalVars['Environment']           = "Test"
globalVars['REGION_NAME']           = "ap-south-1"
globalVars['tagName']               = "Valaxy-Serverless-EBS-Penny-Pincher"
globalVars['findNeedle']            = "DeleteOn"
globalVars['RetentionDays']         = "7"
globalVars['tagsToExclude']         = "Do-Not-Delete"

ec2_client = boto3.client('ec2')

"""
This function looks at *all* snapshots that have a "DeleteOn" tag containing
the current day formatted as YYYY-MM-DD. This function should be run at least
daily.
"""

def janitor_for_snapshots():
    account_ids = list()
    account_ids.append( boto3.client('sts').get_caller_identity().get('Account') )

    snap_older_than_RetentionDays = ( datetime.date.today() - datetime.timedelta(days= int(globalVars['RetentionDays'])) ).strftime('%Y-%m-%d')
    delete_today = datetime.date.today().strftime('%Y-%m-%d')

    tag_key = 'tag:' + globalVars['findNeedle']
    filters = [
        {'Name': tag_key, 'Values': [delete_today]},
    ]
    # Get list of Snaps with Tag 'globalVars['findNeedle']'
    snaps_to_remove = ec2_client.describe_snapshots(OwnerIds=account_ids,Filters=filters)

    # Get the snaps that doesn't have the tag and are older than Retention days
    all_snaps = ec2_client.describe_snapshots(OwnerIds=account_ids)
    for snap in all_snaps['Snapshots']:
        if snap['StartTime'].strftime('%Y-%m-%d') <= snap_older_than_RetentionDays:
            snaps_to_remove['Snapshots'].append(snap)

    snapsDeleted = {'Snapshots': []}

    for snap in snaps_to_remove['Snapshots']:
        try:
            ec2_client.delete_snapshot(SnapshotId=snap['SnapshotId'])
            snapsDeleted['Snapshots'].append({'Description': snap['Description'], 'SnapshotId': snap['SnapshotId'], 'OwnerId': snap['OwnerId']})
        except ClientError as e:
            if "is currently in use by" in str(e):
                print("Snapshot {} is part of an AMI".format(snap.get('SnapshotId')))

    snapsDeleted['Status']='{} Snapshots were Deleted'.format( len(snaps_to_remove['Snapshots']))

    return snapsDeleted

def lambda_handler(event, context):
    return janitor_for_snapshots()

if __name__ == '__main__':
    lambda_handler(None, None)

```

`Save` the lambda function

## Step 2 - Configure Lambda Triggers
We are going to use Cloudwatch Scheduled Events to take backup everyday.
```
rate(1 minute)
or
rate(5 minutes)
or
rate(1 day)
# The below example creates a rule that is triggered every day at 12:00pm UTC.
cron(0 12 * * ? *)
```
_If you want to learn more about the above Scheduled expressions,_ Ref: [CloudWatch - Schedule Expressions for Rules](http://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html#RateExpressions)

## Step 3 - Testing the solution
Create few EBS Snapshots and add the Tag `DeleteOn` with Value as `<TODAYS-DATE-IN-YYYY-MM-DD-FORMAT>`

### Summary
We have demonstrated how you can automatically identify and delete old and unused EBS Snapshots.
