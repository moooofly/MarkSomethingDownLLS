# AWS Serverless Workshop - build serverless application with AWS Lambda

## 只言片语

- s3 是 serverless 的；
- sam 是在 CloudFormation 的基础上的简化；
- sam 用于在本地模拟 lambda 运行环境
- lambda 背后是用 container 来运行的；
- lambda 是一种计算资源；
- lambda 是 stateless 的，若要持久化存储，则需要使用外部存储；
- CloudFormation 等价于 terraform ；
- 不需要关心 Scalability、Utilization、服务器管理；
- 在 vpc 内创建 lambda 的主要目的在于，要访问内部的一些资源（不建议使用，因此在 code start 过程中多了两个步骤）


## AWS Lambda 实验配置

| 名称 | 配置 |
| -- | -- |
| User name | serverless-workshop |
| Password | 8#@ATn[BWdp)+p-)i}lptfo)34$wVPO4&][RTarJ |
| Access key ID | AKIAO7SWN73EOHZRA5YQ |
| Secret access key | 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 |
| Console login link | https://330252173168.signin.amazonaws.cn/console |
| s3 bucket name | serverless-workshop |
| stack | workshop-fei |


### sam 使用

#### 本地运行（sam local commands）

> 对应 https://github.com/lazydragon/asap/tree/master/sam

##### echo example

- write a `template.yaml` config of your serverless application
- please check here for property details: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-function.html
- **sync invoke** a lambda called HelloWorld 

```
➜  sam git:(master) echo '{}' | sam local invoke "HelloWorld"
2018/04/10 11:29:35 Successfully parsed template.yaml
2018/04/10 11:29:35 Connected to Docker 1.37
2018/04/10 11:29:35 Runtime image missing, will pull....
2018/04/10 11:29:35 Fetching lambci/lambda:python3.6 image for python3.6 runtime...
python3.6: Pulling from lambci/lambda
5be106c3813f: Pull complete
e240967675e1: Pull complete
e1dddd3e665c: Pull complete
5898d3e42308: Pull complete
b61aa9f36223: Pull complete
Digest: sha256:02f773a97b4e1663e828d97a7f14b8f661453bbbe829e24d64618870aa662e74
Status: Downloaded newer image for lambci/lambda:python3.6
2018/04/10 11:32:02 Reading invoke payload from stdin (you can also pass it from file with --event)
2018/04/10 11:32:02 Invoking sync.lambda_handler (python3.6)
2018/04/10 11:32:07 WARNING: No AWS credentials found. Missing credentials may lead to slow startup times as detailed in https://github.com/awslabs/aws-sam-local/issues/134
2018/04/10 11:32:07 Mounting /Users/sunfei/workspace/Sandbox/asap/sam/python as /var/task:ro inside runtime container
START RequestId: 631fbbe8-985d-422f-acdd-5a4b125df622 Version: $LATEST
END RequestId: 631fbbe8-985d-422f-acdd-5a4b125df622
REPORT RequestId: 631fbbe8-985d-422f-acdd-5a4b125df622 Duration: 9 ms Billed Duration: 100 ms Memory Size: 128 MB Max Memory Used: 19 MB

"Hello World"


➜  sam git:(master)
```

##### go example

- build go code

```
go get github.com/aws/aws-lambda-go/lambda
GOOS=linux GOARCH=amd64 go build -o main main.go
```

- invoke lambda

```
➜  asap git:(master) go get github.com/aws/aws-lambda-go/lambda
➜  asap git:(master) cd sam/go/
➜  go git:(master) GOOS=linux GOARCH=amd64 go build -o main main.go
➜  go git:(master) ✗ cd ..
➜  sam git:(master) ✗ echo '{}' | sam local invoke "HelloWorldGo"
2018/04/10 13:07:42 Successfully parsed template.yaml
2018/04/10 13:07:42 Connected to Docker 1.37
2018/04/10 13:07:42 Runtime image missing, will pull....
2018/04/10 13:07:42 Fetching lambci/lambda:go1.x image for go1.x runtime...
go1.x: Pulling from lambci/lambda
5be106c3813f: Already exists
e240967675e1: Already exists
d537dac75d99: Pull complete
5e61f8bb93d6: Pull complete
Digest: sha256:0c3d4ad28d7226c5b916a04051173ba3c21e2b7d213d63281d67e3aeac4a901b
Status: Downloaded newer image for lambci/lambda:go1.x
2018/04/10 13:07:46 Reading invoke payload from stdin (you can also pass it from file with --event)
2018/04/10 13:07:46 Invoking main (go1.x)
2018/04/10 13:07:51 WARNING: No AWS credentials found. Missing credentials may lead to slow startup times as detailed in https://github.com/awslabs/aws-sam-local/issues/134
2018/04/10 13:07:51 Mounting /Users/sunfei/workspace/Sandbox/asap/sam/go as /var/task:ro inside runtime container
START RequestId: 46be9759-cb7f-1738-bd43-0389e751fa27 Version: $LATEST
END RequestId: 46be9759-cb7f-1738-bd43-0389e751fa27
REPORT RequestId: 46be9759-cb7f-1738-bd43-0389e751fa27	Duration: 1.40 ms	Billed Duration: 100 ms	Memory Size: 128 MB	Max Memory Used: 5 MB
"Hello ƛ!"


➜  sam git:(master) ✗
```

### 部署到远端（sam deployment commands）

需要创建 `/Users/sunfei/.aws` 目录，并将 config 配置放在下面，config 内容如下

```
[default]
aws_access_key_id = AKIAO7SWN73EOHZRA5YQ
aws_secret_access_key = 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
region = cn-north-1
```

之后就可以

- **package** and **upload** source code to s3

```
➜  sam git:(master) ✗ sam package --template-file template.yaml --s3-bucket serverless-workshop --output-template-file output.yaml
Uploading to 1df966252fa98952589fd07f39e865f3  175 / 175.0  (100.00%)100.00%)
Successfully packaged artifacts and wrote output template to file output.yaml.
Execute the following command to deploy the packaged template
aws cloudformation deploy --template-file /Users/sunfei/workspace/Sandbox/asap/sam/output.yaml --stack-name <YOUR STACK NAME>
➜  sam git:(master) ✗
```

- **deploy** the serverless application

会上传

```
➜  sam git:(master) ✗ sam deploy --template-file output.yaml --stack-name workshop-fei --capabilities CAPABILITY_IAM

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - workshop-fei
➜  sam git:(master) ✗
```

- **describe** output of the function

```
➜  sam git:(master) ✗ aws cloudformation describe-stacks --stack-name workshop-fei
{
    "Stacks": [
        {
            "StackId": "arn:aws-cn:cloudformation:cn-north-1:330252173168:stack/workshop-fei/aea928a0-3c72-11e8-b953-50fa18a2d236",
            "Description": "serverless hello world",
            "Tags": [],
            "Outputs": [
                {
                    "Description": "Go Lambda function name generated",
                    "OutputKey": "GoLambdaName",
                    "OutputValue": "workshop-fei-HelloWorldGo-ILLS7Y2H41EP"
                },
                {
                    "Description": "Lambda function name generated",
                    "OutputKey": "LambdaName",
                    "OutputValue": "workshop-fei-HelloWorld-SZSGC6WWWHDX"
                }
            ],
            "EnableTerminationProtection": false,
            "CreationTime": "2018-04-10T03:53:08.927Z",
            "Capabilities": [
                "CAPABILITY_IAM"
            ],
            "StackName": "workshop-fei",
            "NotificationARNs": [],
            "StackStatus": "CREATE_COMPLETE",
            "DisableRollback": false,
            "RollbackConfiguration": {},
            "ChangeSetId": "arn:aws-cn:cloudformation:cn-north-1:330252173168:changeSet/awscli-cloudformation-package-deploy-1523332388/c16c43e7-0a47-4ce5-baba-e9fe95a501f9",
            "LastUpdatedTime": "2018-04-10T03:53:14.414Z"
        }
    ]
}
➜  sam git:(master) ✗
```

- test **invoke** of lambda

对应远程 invoke

```
➜  sam git:(master) ✗ aws lambda invoke --function-name workshop-fei-HelloWorld-SZSGC6WWWHDX workshop-fei-HelloWorld-SZSGC6WWWHDX.output
{
    "ExecutedVersion": "$LATEST",
    "StatusCode": 200
}
➜  sam git:(master) ✗

➜  sam git:(master) ✗ aws lambda invoke --function-name workshop-fei-HelloWorldGo-ILLS7Y2H41EP workshop-fei-HelloWorldGo-ILLS7Y2H41EP.output
{
    "ExecutedVersion": "$LATEST",
    "StatusCode": 200
}
➜  sam git:(master) ✗
```

- **delete** stack

```
aws cloudformation delete-stack --stack-name <YOUR STACK NAME>
```


### code start 测试

> 对应 https://github.com/lazydragon/asap/tree/master/code_start

- **create** s3 bucket and **deploy**

```
➜  code_start git:(master) ✗ ./deploy.sh serverless-workshop workshop-fei
Uploading to 0cad4709cde641840f0d371bbeb20ea1  374 / 374.0  (100.00%)
Successfully packaged artifacts and wrote output template to file output.yaml.
Execute the following command to deploy the packaged template
aws cloudformation deploy --template-file /Users/sunfei/workspace/Sandbox/asap/code_start/output.yaml --stack-name <YOUR STACK NAME>

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - workshop-fei
{
    "Stacks": [
        {
            "StackId": "arn:aws-cn:cloudformation:cn-north-1:330252173168:stack/workshop-fei/aea928a0-3c72-11e8-b953-50fa18a2d236",
            "Description": "A function to experiment cold start time when code starts",
            "Tags": [],
            "Outputs": [
                {
                    "Description": "Lambda function name generated",
                    "OutputKey": "LambdaName",
                    "OutputValue": "workshop-fei-CodeStart-BNP2UW5HUAXB"
                }
            ],
            "EnableTerminationProtection": false,
            "CreationTime": "2018-04-10T03:53:08.927Z",
            "Capabilities": [
                "CAPABILITY_IAM"
            ],
            "StackName": "workshop-fei",
            "NotificationARNs": [],
            "StackStatus": "UPDATE_COMPLETE",
            "DisableRollback": false,
            "RollbackConfiguration": {},
            "ChangeSetId": "arn:aws-cn:cloudformation:cn-north-1:330252173168:changeSet/awscli-cloudformation-package-deploy-1523337864/32c0a2fe-07ca-4a6a-969a-93506e36fc33",
            "LastUpdatedTime": "2018-04-10T05:24:30.538Z"
        }
    ]
}
➜  code_start git:(master) ✗
```

- 测试

> 代码未调整前

```
➜  code_start git:(master) ✗ python invoke.py -mw 1 -it 30 -l workshop-fei-CodeStart-BNP2UW5HUAXB
BHSTMG: This is the 25 time this container is used

BHSTMG: This is the 26 time this container is used

BHSTMG: This is the 27 time this container is used

BHSTMG: This is the 28 time this container is used

BHSTMG: This is the 29 time this container is used

BHSTMG: This is the 30 time this container is used

3UFMV7: This is the 16 time this container is used

3UFMV7: This is the 17 time this container is used

3UFMV7: This is the 18 time this container is used

3UFMV7: This is the 19 time this container is used

3UFMV7: This is the 20 time this container is used

3UFMV7: This is the 21 time this container is used

3UFMV7: This is the 22 time this container is used

BHSTMG: This is the 31 time this container is used

BHSTMG: This is the 32 time this container is used

BHSTMG: This is the 33 time this container is used

BHSTMG: This is the 34 time this container is used

BHSTMG: This is the 35 time this container is used

BHSTMG: This is the 36 time this container is used

BHSTMG: This is the 37 time this container is used

BHSTMG: This is the 38 time this container is used

BHSTMG: This is the 39 time this container is used

BHSTMG: This is the 40 time this container is used

3UFMV7: This is the 23 time this container is used

3UFMV7: This is the 24 time this container is used

3UFMV7: This is the 25 time this container is used

3UFMV7: This is the 26 time this container is used

BHSTMG: This is the 41 time this container is used

BHSTMG: This is the 42 time this container is used

BHSTMG: This is the 43 time this container is used

➜  code_start git:(master) ✗
➜  code_start git:(master) ✗ python invoke.py -mw 10 -it 30 -l workshop-fei-CodeStart-BNP2UW5HUAXB | awk '{print $1}' | sort | uniq -c
  30
   5 3UFMV7:
   1 A5I0MD:
   1 ACD1AC:
   9 BHSTMG:
   1 D0DBUB:
   1 I7L44J:
   1 ST0DQE:
   1 T49RQV:
   1 VB79IM:
   1 XEVO3B:
   8 YXGXVZ:
➜  code_start git:(master) ✗ python invoke.py -mw 10 -it 30 -l workshop-fei-CodeStart-BNP2UW5HUAXB | awk '{print $1}' | sort | uniq -c
  30
   3 A5I0MD:
   4 ACD1AC:
   2 BHSTMG:
   3 D0DBUB:
   4 I7L44J:
   4 ST0DQE:
   4 T49RQV:
   2 VB79IM:
   3 XEVO3B:
   1 YXGXVZ:
➜  code_start git:(master) ✗
➜  code_start git:(master) ✗ python invoke.py -mw 10 -it 100 -l workshop-fei-CodeStart-BNP2UW5HUAXB | awk '{print $1}' | sort | uniq -c
 100
  10 A5I0MD:
  12 ACD1AC:
  11 BHSTMG:
  11 D0DBUB:
   2 I7L44J:
   9 ST0DQE:
  10 T49RQV:
  12 VB79IM:
  12 XEVO3B:
  11 YXGXVZ:
➜  code_start git:(master) ✗ python invoke.py -mw 10 -it 100 -l workshop-fei-CodeStart-BNP2UW5HUAXB | awk '{print $1}' | sort | uniq -c
 100
   1 28EK2L:
   7 3UFMV7:
  10 A5I0MD:
   9 ACD1AC:
  10 BHSTMG:
   9 D0DBUB:
   5 I7L44J:
   9 ST0DQE:
   9 T49RQV:
  11 VB79IM:
  10 XEVO3B:
  10 YXGXVZ:
➜  code_start git:(master) ✗
```

> 调整代码后（只需要调整 src/lambda.py）

> ```
>  18 def lambda_handler(event, context):
>  19     global count, container_id
>  20     # count add 1
>  21     count += 1
>  22     time.sleep(1)   # 增加
>  23
>  24     return (container_id, count)
> ```


```
➜  code_start git:(master) ✗ python invoke.py -mw 10 -it 10 -l workshop-fei-CodeStart-BNP2UW5HUAXB | awk '{print $1}' | sort | uniq -c
  10
   1 6YHLKQ:
   1 7Z3BLS:
   1 9WSNU1:
   1 E9YQX9:
   1 MZJAW1:
   1 OPQQH7:
   1 RVXGOU:
   1 UH5YL7:
   1 UL920R:
   1 WE4UXH:
➜  code_start git:(master) ✗ python invoke.py -mw 10 -it 100 -l workshop-fei-CodeStart-BNP2UW5HUAXB | awk '{print $1}' | sort | uniq -c
 100
  10 6YHLKQ:
  10 7Z3BLS:
  10 9WSNU1:
  10 E9YQX9:
  10 MZJAW1:
  10 OPQQH7:
  10 RVXGOU:
  10 UH5YL7:
  10 UL920R:
  10 WE4UXH:
➜  code_start git:(master) ✗
➜  code_start git:(master) ✗ python invoke.py -mw 10 -it 30 -l workshop-fei-CodeStart-BNP2UW5HUAXB | awk '{print $1}' | sort | uniq -c
  30
   3 1L15GC:
   3 4C4USH:
   3 5JLHWK:
   3 90Q7GJ:
   3 K1U0HS:
   3 MZJAW1:
   3 QHZG2L:
   3 T0FP54:
   3 TCU5QA:
   3 WE4UXH:
➜  code_start git:(master) ✗
```

结论：

- 在未进行代码调整前，worker 数量和 container 的数量是不一致的，每个 container 被复用的次数也是不同的；
- 在为 lambda.py 函数增加延迟后，可以看到 worker 数量和 container 的数量是相同的，并且每个 container 被复用的次数也是相同的；

### performance 测试

> 对应 https://github.com/lazydragon/asap/tree/master/performance

略

结论：

- 资源使用主要基于 memory 进行控制（CPU 的使用和 memory 成线性关系，但不能直接设置）
- 在 memory 设置为 2048MB 时，程序执行实现最少，性能最大化；