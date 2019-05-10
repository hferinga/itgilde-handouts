# Handout Kickoff "Learning by doing" challenge 1; 16 may 2019 
This document is formatted for Gitlab markdown.

Publicly available on GitHub:

https://github.com/hferinga/itgilde-handouts/blob/master/handout-c1.md

and on GitLab:

https://gitlab.com/itgilde-hansferinga/learning-by-doing-ch1/blob/master/handout-c1.md


- Challenge: `Building a minimal serverless application`
- Region: `eu-west-1 (Ireland)`

## Some references
* https://aws.amazon.com/blogs/compute/using-amazon-api-gateway-as-a-proxy-for-dynamodb/
* (if you have an account on Linux Academy or A Cloud Guru, check out the relevant training labs)

Note: Recently Linux Academy released [Learn AWS by doing](https://linuxacademy.com/amazon-web-services/labs) hands-on labs


## Services used
- DynamoDB
- API Gateway
- S3 (static website)
- IAM (roles and policies)
- CloudWatch (for troubleshooting)
- (optionally: Route53, hosted zone)

# Create a non-root/non-admin user for this challenge
- create user itgilde
- create group workshop and add itgilde as member

## Create Access Key for programmatic access
Configure your aws cli to use this key
(you can skip this if you do not want to use programmatic access via CLI or SDK)

## Create some policies
### ITGildeIAMPassRole
Used for non-admin user to configure the API Gateway to use the ITGilde-DynamoDBFullAccess role
- Effect: `Allow`
- Action: `iam:PassRole`
- Resoure: `role ITGilde-DynamoDBFullAccess`

### FullAccessOnBucketITGildePOTD
Used for accessing S3 (limited actions)
- Effect: `Allow`
- Action: 
    * `s3:PutAccountPublicAccessBlock`
    * `s3:GetAccountPublicAccessBlock`
    * `s3:ListAllMyBuckets`
    * `s3:HeadBucket`
- Resource: `*`   

Used for accessing the S3 bucket and objects in it (all actions)
- Effect: `Allow`
- Action: `s3:*`
- Resource:
    * `arn:aws:s3:::<yourbucketname>`
    * `arn:aws:s3:::<yourbucketname>/*`

### ITGilde-WAF-Permissions
Used for deploying an API Gateway to a stage
- Effect: `Allow`
- Action:
    * `waf-regional:AssociateWebACL`
    * `waf-regional:ListWebACLs`
- Resoure: `*`


## Attach permissions(policies) for group workshop
AWS managed:
- AmazonDynamoDBFullAccess
- CloudWatchLogsFullAccess
- AmazonAPIGatewayAdministrator

Custom:
- ITGildeIAMPassRole
- FullAccessOnBucketITGildePOTD
- ITGilde-WAF-Permissions

*Alternatively you can give full access to S3, if you don't want to wrestle IAM too much*

# IAM Roles
## ITGilde-DynamoDBFullAccess
Attached policies:
- AmazoneAPIGatewayInvokeFullAccess
- AmazonDynamoDBFullAccess

Trust relationships add:
- identity provider: `apigateway.amazonaws.com`

## API-Gateway-Allow_log
Attached policies:
- AmazonAPIGatewayPushToCloudWatchLogs

# DynamoDB
## Set up DynamoDB tables and indexes
- itgilde_poll
- itgilde_vote

### For all tables:
- untick "Use default settings"
- untick "AutoScaling Read/Write Capacity"

Configure provisioned capacity:
  - rcu: 1
  - wcu: 1   
Leave rest default


### Table: itgilde_poll
Contains the poll statement and answer choices
- partition_key: pollId (N)
Indexes:
- none

### Table: itgilde_vote
Contains the actual votes and optional info
- partition key: pollId (N)
- sort key: uuid (S)
Indexes:
- partition key: pollId (N)
- sort key: answer (S)
- projected attributes: ALL

# API Gateway
Contains the resources:
- poll
- results
- vote

*Note: tick CORS enable*

For all Integration Types (Methods):
* Integration Type: AWS Service
* Region: eu-west-1
* Services: DynamoDB
* HTTP method: POST (always, this is the comms between API Gateway and the API of the DynamoDB)
* Execution Role: arn:aws:iam::<youraccountid>:role/ITGilde-DynamoDBFullAccess

Role ARN for CloudWatch logging (global setting):
- arn:aws:iam::<youraccoundid>:role/API-Gateway-Allow_log

## Resource poll
- Method: GET
- Integration Request:
  *  Action: Scan
  *  Mapping Templates (application/json): When there are no templates defined.

```json
{
    "TableName": "itgilde_poll",
    "AttributesToGet": [ "pollId","poll", "qa" ],
    "ScanFilter": { "active": { "AttributeValueList" : [ {"BOOL": true } ], "ComparisonOperator": "EQ" } }
}
```

Some references for mapping templates:
https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html  
http://velocity.apache.org/engine/devel/vtl-reference.html

## Resource results
- Method: GET
- Method Request:
   * URL Query String Parameters: `pollId, required`
   * Request Validator: `Validate query string parameters and headers`
- Integration Request:
  * Action: `Query`
  * Mapping Templates (application/json): `When there are no templates defined.`

```json
{
  "TableName" : "itgilde_vote",
  "IndexName": "pollId-uuid-index",
  "KeyConditionExpression" : "pollId = :v1 and #u > :v2",
  "ExpressionAttributeValues" : {
     ":v1"  : {
         "N" : "$input.params('pollId')"
         },
     ":v2" : {
         "S": "0"
        }
    },
  "ExpressionAttributeNames" : {
     "#u" : "uuid"
   },
   "ReturnConsumedCapacity": "TOTAL"
}
```

## Resource vote
- Method: POST
- Integration Request:
  * Action: `PutItem`
  * Mapping Templates (application/json): `When there are no templates defined.`

```json
#set($inputRoot = $input.path('$'))
#set($comment = $inputRoot.comment)
#set($origin = $inputRoot.origin)
#set($email = $inputRoot.email)
#set($theirUUID = $inputRoot.uuid)
#set($pollId = $inputRoot.pollId)
{
    "TableName": "itgilde_vote",
    "Item": {
        "uuid": {
            "S": "$context.requestId"
            },
        "answer": {
            "S": "$input.path('$.answer')"
            },
        #if("$!pollId" != "")
        "pollId": {
            "N": "$pollId"
        },
        #else
        "pollId": {
            "N": "1"
        },
        #end
        #if("$!theirUUID" != "")
        "theiruuid": {
            "S": "$theirUUID"
        },
        #end
        #if("$!origin" != "")
        "origin": {
            "S": "$origin"
        },
        #end
        #if("$!comment" != "")
        "comment": {
            "S": "$comment"
        },
        #end
        #if("$!email" != "")
        "email": {
            "S": "$email"
        },
        #end
        #if($!context.requestTime != "")
        "date": {
            "S": "$context.requestTime"
        },
        #end
        "ipaddress": {
            "S": "$context.identity.sourceIp"
        }
    }
}
```

# Test the API Gateway configuration
For each resource select the configured method and select the Test icon.  
Some Tests need a json body to be submitted, others a URL query string.

If these tests succeed, we can continue to the S3 part

# S3 static website
- Create bucket
- You put your files into a bucket configured for static website hosting.
- Your html files will need javascript to be able to communicate with
  the API Gateway, to at least convert to JSON when submitting a form.

## Create bucket and configure Static website hosting
**If you want to use it with Route53, make sure the name is a DNS compliant name**
- Allow public access
- Upload files (again grant public access)

## some example files can be downloaded from
[http://itgilde-potd.aws.linux-it-services.nl/itgilde-ch1.tgz](http://itgilde-potd.aws.linux-it-services.nl/itgilde-ch1.tgz)

Configure the urls.js.example file with your API Gateway endpoint URLs and 
upload the files to your S3 bucket.


# Appendices
## Curl test get poll info example
```bash
#!/bin/bash

curl -X GET https://<yourapigatewaystageurl>

```

## Curl test vote example
This will not check CORS setting though. So even if CORS is not configured correctly this will succeed,
unless other things fail.
```bash
#!/bin/bash

curl -X POST \
https://<yourapigatewaystageurl> \
 -H 'Content-Type: application/json' \
 -d '{
 "uuid": '"$(uuidgen)"',
 "answer": "No",
 "origin": "Mars",
 "comment": "They will go nuts for a mars!",
 "email": me@me.org,
}'
```

## json text to use in the API Gateway Test
Paste this in the JSON text field:
```json
{
 "uuid": "2e1514a8-7ae9-4ec6-a0ab-b993ab4bf52b",
 "pollId": "1",
 "answer": "no",
 "origin": "Mars",
 "comment": "They will go nuts for a mars!",
 "email": "me@me.org"
}
```


