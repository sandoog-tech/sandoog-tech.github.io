# AWS Dynamodb stream trigger to Lambda function
> This tutorial has been created with reference to AWS tutorial [Tutorial: Process New Items with DynamoDB Streams and Lambda](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.Lambda.Tutorial.html#Streams.Lambda.Tutorial.CreateTrigger)

## Install Lambda tools for .net core
> dotnet tool install -g Amazon.Lambda.Tools

## Deploy function to zip file
> dotnet-lambda deploy-function

## Solve Issue error nu1605 when publish lambda function
```csharp=1
// add to your csproj file
 <PackageReference Include="Microsoft.NETCore.Targets" Version="3.0.0" PrivateAssets="all" /> 
```
> ### Run Deploy function command and it should work.

## Use LocalStack AWS CLI
> To facilitate working with aws cli install this command as a shorthand version.
> [AWSLocal cli](https://github.com/localstack/awscli-local)

---

## Create Table in Dynamodb with Stream enabled
```json
awslocal dynamodb create-table \
    --table-name adminOrders \
    --attribute-definitions \
            AttributeName=id,AttributeType=N \
            AttributeName=createdDate,AttributeType=S \
    --key-schema \
            AttributeName=id,KeyType=HASH \
            AttributeName=createdDate,KeyType=RANGE \
    --provisioned-throughput \
            ReadCapacityUnits=10,WriteCapacityUnits=5 \
    --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
```

## Create IAM role for invoking the function
```json=
awslocal iam create-role \
    --role-name SearchLambdaRole \
    --assume-role-policy-document "file://C:\AhmedMousa\AWS Lambda\service-role\trust-relationship.json"
```



## Create Role-policy which describe the granted permissions
```json=
awslocal iam put-role-policy \
    --role-name SearchLambdaRole \
    --policy-name SearchLambdaRolePolicy \
    --policy-document "file://C:\AhmedMousa\AWS Lambda\service-role\role-policy.json"
```


## Get Role ARN to insert it into the function definition
```json=
awslocal iam get-role --role-name SearchLambdaRole
```


## Create function
```json=
awslocal lambda create-function \
    --region eu-west-1 \
    --function-name SearchPopulater \
    --zip-file "fileb://C:\AhmedMousa\Sandoog\Sandoog.SearchPopulater.Lambda.zip" \
    --role arn:aws:iam::000000000000:role/SearchLambdaRole \
    --handler Sandoog.SearchPopulater.Lambda::Sandoog.SearchPopulater.Lambda.Function::FunctionHandler\
    --timeout 5 \
    --runtime dotnetcore3.1
```

## Test Invoke the function
```json=
awslocal lambda invoke \
    --function-name SearchPopulater \
    --cli-binary-format raw-in-base64-out \
    --payload file://"C:\AhmedMousa\AWS Lambda\service-role\payload.json" output.txt
```


> you shoud get result similar to this:
```json
{
    "StatusCode": 200,
    "LogResult": "",
    "ExecutedVersion": "$LATEST"
}
```

## Get table LatestStreamArn
```json=
awslocal dynamodb describe-table --table-name adminOrders
```

> **search for LatestStreamArn which will be used to define event source arn**
```json=
 "LatestStreamArn": "arn:aws:dynamodb:eu-west-1:000000000000:table/adminOrders/stream/2021-02-10T16:19:00.930"
```


## Create event-source mapping 
```json=
awslocal lambda create-event-source-mapping \
    --region eu-west-1 \
    --function-name SearchPopulater \
    --event-source-arn arn:aws:dynamodb:eu-west-1:000000000000:table/adminOrders/stream/2021-02-11T14:53:27.233 \
    --batch-size 1 \
    --starting-position TRIM_HORIZON
```



## List event-source mapping
```json=
awslocal lambda list-event-source-mappings \
    --function-name SearchPopulater
```


## Test the event stream trigger by addind data to dynamodb table
```json=
awslocal dynamodb put-item \
    --table-name adminOrders\
    --item id={N="4566891001"},createdDate={S="2021-02-10"}
```
> **you should get result simialr to this**
```json=
{
    "ConsumedCapacity": {
        "TableName": "adminOrders",
        "CapacityUnits": 1.0
    }
}
```
