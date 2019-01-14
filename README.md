# 参考元

https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/with-on-demand-https-example.html

# 導入

- Lambda作成

```
$aws lambda create-function --function-name LambdaFunctionOverHttps --zip-file fileb://function.zip --handler index.handler --runtime nodejs8.10 --role arn:aws:iam::1111111111:role/lambda-apigateway-role
```
- Lambda test

```
$aws lambda invoke --function-name LambdaFunctionOverHttps --payload fileb://input.txt outputfile.txt
```

- APIGateway作成

```
$aws apigateway create-rest-api --name DynamoDBOperations
{
    "name": "DynamoDBOperations", 
    "endpointConfiguration": {
        "types": [
            "EDGE"
        ]
    }, 
    "id": "oua4zhbxif", 
    "createdDate": 1547177837
}

$API=oua4zhbxif
$echo $API
oua4zhbxif
```

- APIGateway確認

```
$ aws apigateway get-resources --rest-api-id $API
{
    "items": [
        {
            "path": "/", 
            "id": "dmzcue7vte"
        }
    ]
}
```

ルートリソースのみ

- リソースの追加

```
$ aws apigateway create-resource --rest-api-id $API --path-part DynamoDBManager --parent-id dmzcue7vte
{
    "path": "/DynamoDBManager", 
    "pathPart": "DynamoDBManager", 
    "id": "iv1z59", 
    "parentId": "dmzcue7vte"
}
```

parentIdはルートリソースを指定

- 追加したリソースにmethodの追加

RESOURCEは前述のリソースid

```
$RESOURCE=iv1z59
$ aws apigateway put-method --rest-api-id $API --resource-id $RESOURCE --http-method POST --authorization-type NONE
{
    "apiKeyRequired": false, 
    "httpMethod": "POST", 
    "authorizationType": "NONE"
}
```

- 統合リクエスト設定

```
$ REGION=ap-northeast-1
$ ACCOUNT=1111111111
$ aws apigateway put-integration --rest-api-id $API --resource-id $RESOURCE --http-method POST --type AWS --integration-http-method POST --uri arn:aws:apigateway:$REGION:lambda:path/2015-03-31/functions/arn:aws:lambda:$REGION:$ACCOUNT:function:LambdaFunctionOverHttps/invocations
{
    "passthroughBehavior": "WHEN_NO_MATCH", 
    "timeoutInMillis": 29000, 
    "uri": "arn:aws:apigateway:ap-northeast-1:lambda:path/2015-03-31/functions/arn:aws:lambda:ap-northeast-1:1111111111:function:LambdaFunctionOverHttps/invocations", 
    "httpMethod": "POST", 
    "cacheNamespace": "iv1z59", 
    "type": "AWS", 
    "cacheKeyParameters": []
}
```


- POST メソッドのレスポンスおよび統合レスポンスのcontent-typeをjsnに(API メソッドが返すレスポンスのタイプ)

```
$ aws apigateway put-method-response --rest-api-id $API --resource-id $RESOURCE --http-method POST --status-code 200 --response-models '{"application/json": "Empty"}'
{
    "responseModels": {
        "application/json": "Empty"
    }, 
    "statusCode": "200"
}
```

- POST メソッドの統合レスポンス(Lambda 関数が返すレスポンスのタイプ)

```
$ aws apigateway put-integration-response --rest-api-id $API --resource-id $RESOURCE --http-method POST --status-code 200 --response-templates '{"application/json": ""}'
{
    "statusCode": "200", 
    "responseTemplates": {
        "application/json": null
    }
}
```

- STAGEデプロイ

```
$ aws apigateway create-deployment --rest-api $API --stage-name prod
{
    "id": "ncuaey", 
    "createdDate": 1547185700
}
```

- APIGatewayにLambda呼び出しのアクセス許可を設定

このままだとテストで実行しても「Execution failed due to configuration error: Invalid permissions on Lambda function」。
PERMISSIONを与えていく

```
$ aws lambda add-permission --function-name LambdaFunctionOverHttps --statement-id apigateway-test-2 --action lambda:InvokeFunction --principal apigateway.amazonaws.com --source-arn "arn:aws:execute-api:$REGION:$ACCOUNT:$API/*/POST/DynamoDBManager"
{
    "Statement": "{\"Sid\":\"apigateway-test-2\",\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"apigateway.amazonaws.com\"},\"Action\":\"lambda:InvokeFunction\",\"Resource\":\"arn:aws:lambda:ap-northeast-1:1111111111:function:LambdaFunctionOverHttps\",\"Condition\":{\"ArnLike\":{\"AWS:SourceArn\":\"arn:aws:execute-api:ap-northeast-1:1111111111:oua4zhbxif/*/POST/DynamoDBManager\"}}}"
}
```

- 動作確認

curl

```
$ curl -X POST -d '{"operation":"create","tableName":"lambda-apigateway","payload":{"Item":{"id":"1","name":"Bob"}}}' https://$API.execute-api.$REGION.amazonaws.com/prod/DynamoDBManager
```

or

aws-cli
```
$ aws apigateway test-invoke-method --rest-api-id $API --resource-id $RESOURCE --http-method POST --path-with-query-string "" --body file://create-item.json
{
    "status": 200, 
    "body": "{}", 
    "log": "Execution log for request 09d4c16a-156c-11e9-ad5d-b743cd74e83d\nFri Jan 11 06:42:17 UTC 2019 : Starting execution for request: 09d4c16a-156c-11e9-ad5d-b743cd74e83d\nFri Jan 11 06:42:17 UTC 2019 : HTTP Method: POST, Resource Path: /DynamoDBManager\nFri Jan 11 06:42:17 UTC 2019 : Method request path: {}\nFri Jan 11 06:42:17 UTC 2019 : Method request query string: {}\nFri Jan 11 06:42:17 UTC 2019 : Method request headers: {}\nFri Jan 11 06:42:17 UTC 2019 : Method request body before transformations: {\n  \"operation\": \"create\",\n  \"tableName\": \"lambda-apigateway\",\n  \"payload\": {\n      \"Item\": {\n          \"id\": \"1234ABCD\",\n          \"number\": 5\n      }\n  }\n}\n\nFri Jan 11 06:42:17 UTC 2019 : Endpoint request URI: https://lambda.ap-northeast-1.amazonaws.com/2015-03-31/functions/arn:aws:lambda:ap-northeast-1:11111111111:function:LambdaFunctionOverHttps/invocations\nFri Jan 11 06:42:17 UTC 2019 : Endpoint request headers: {x-amzn-lambda-integration-tag=09d4c16a-156c-11e9-ad5d-b743cd74e83d, Authorization=*****************************************************************************************************************************************************************************************************************************************************************************************************************************b3fb44, X-Amz-Date=20190111T064217Z, x-amzn-apigateway-api-id=oua4zhbxif, X-Amz-Source-Arn=arn:aws:execute-api:ap-northeast-1:11111111111:oua4zhbxif/test-invoke-stage/POST/DynamoDBManager, Accept=application/json, User-Agent=AmazonAPIGateway_oua4zhbxif, X-Amz-Security-Token=FQoGZXIvYXdzEF8aDMjZEDuBR269u0RA1yLBAxXkI+XeSvJN1PA0Wfuyd9G+TSZeMjnXgFPknszvQc/MXztOfZce9frbGXz3Q1QkSrbJevONA/rjtNNBNFZFpqTrpiq+ifJPVLF4jN47wA6uXVQCsuN8EYKis+xcpMurODoIIWuOiyCW8Yoa3ngV3js2gK6SDePi2RSJ5qXwOcvcsIO1V5+MCrCsLEQNJaZCzhfEqdJEnSKMlqOGwrRzh0bdkblte5/DSHZYwq9YYccuovxR3v/q4ijexb6KjMHD4+3CrhDBUPGu0HuyGAlJfgqZVj8nvf [TRUNCATED]\nFri Jan 11 06:42:17 UTC 2019 : Endpoint request body after transformations: {\n  \"operation\": \"create\",\n  \"tableName\": \"lambda-apigateway\",\n  \"payload\": {\n      \"Item\": {\n          \"id\": \"1234ABCD\",\n          \"number\": 5\n      }\n  }\n}\n\nFri Jan 11 06:42:17 UTC 2019 : Sending request to https://lambda.ap-northeast-1.amazonaws.com/2015-03-31/functions/arn:aws:lambda:ap-northeast-1:11111111111:function:LambdaFunctionOverHttps/invocations\nFri Jan 11 06:42:19 UTC 2019 : Received response. Integration latency: 1508 ms\nFri Jan 11 06:42:19 UTC 2019 : Endpoint response body before transformations: {}\nFri Jan 11 06:42:19 UTC 2019 : Endpoint response headers: {Date=Fri, 11 Jan 2019 06:42:19 GMT, Content-Type=application/json, Content-Length=2, Connection=keep-alive, x-amzn-RequestId=09d64802-156c-11e9-b48f-fd5b0507376e, x-amzn-Remapped-Content-Length=0, X-Amz-Executed-Version=$LATEST, X-Amzn-Trace-Id=root=1-5c383ac9-32557f4fdcd8284fe3792ab7;sampled=0}\nFri Jan 11 06:42:19 UTC 2019 : Method response body after transformations: {}\nFri Jan 11 06:42:19 UTC 2019 : Method response headers: {X-Amzn-Trace-Id=Root=1-5c383ac9-32557f4fdcd8284fe3792ab7;Sampled=0, Content-Type=application/json}\nFri Jan 11 06:42:19 UTC 2019 : Successfully completed execution\nFri Jan 11 06:42:19 UTC 2019 : Method completed with status: 200\n", 
    "latency": 1516, 
    "headers": {
        "X-Amzn-Trace-Id": "Root=1-5c383ac9-32557f4fdcd8284fe3792ab7;Sampled=0", 
        "Content-Type": "application/json"
    }
}
```


- 一覧もこれでみれた

```
$ aws lambda invoke --function-name LambdaFunctionOverHttps --payload file://get-items.json outputfile.txt
{
    "ExecutedVersion": "$LATEST", 
    "StatusCode": 200
}
$ cat outputfile.txt | jq .
{
  "Items": [
    {
      "id": "1",
      "name": "Bob"
    },
    {
      "id": "1234ABCD",
      "number": 5
    }
  ],
  "Count": 2,
  "ScannedCount": 2
}
```

# Try

- GET追加してみる

GETメソッドを同じリソース下に作成

```
$ aws apigateway put-method --rest-api-id $API --resource-id $RESOURCE --http-method GET --authorization-type NONE
{
    "apiKeyRequired": false, 
    "httpMethod": "GET", 
    "authorizationType": "NONE"
}
```

統合リクエストにLambdaへのつなぎを設定

```
$ aws apigateway put-integration --rest-api-id $API --resource-id $RESOURCE --http-method GET --type AWS --integration-http-method GET --uri arn:aws:apigateway:$REGION:lambda:path/2015-03-31/functions/arn:aws:lambda:$REGION:$ACCOUNT:function:LambdaFunctionOverHttps/invocations
{
    "passthroughBehavior": "WHEN_NO_MATCH", 
    "timeoutInMillis": 29000, 
    "uri": "arn:aws:apigateway:ap-northeast-1:lambda:path/2015-03-31/functions/arn:aws:lambda:ap-northeast-1:111111111111:function:LambdaFunctionOverHttps/invocations", 
    "httpMethod": "GET", 
    "cacheNamespace": "iv1z59", 
    "type": "AWS", 
    "cacheKeyParameters": []
}
```
