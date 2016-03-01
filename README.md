#AWS CloudFormation

Collection of templates and docs related to Cloud Formation during my studies

## Notes 

## To-Do

* Review the latest release notes

### Goals

* Understand how the AWS::CloudFormation::Authentication works
* Be comfortable reading JSON templates
* Be able to easily search for AWS resources of any type
* Understand how a custom resouce works if you don't use lambda
* Understand WaitConditions
* Understand the various CFN-related tools (cloud-init vs. cfn-init, cfn-hup and cfn-signal)
* Hold a wide breadth of understanding of various AWS Services, and how they would be built on CFN
* Be able to explain the following: WaitConditions, Ref calls, Parameters, Mappings 
* Understand the relationship between CFN calls and the HTTPS APIs for the various services. 
* Understand how to use custom resources (Lambda functions, SNS topics, etc)
* Learn how cloudformation sets up AutoScaling/LaunchConfiguration to perform migrations with zero downtime
* Learn possible ways to customize the cfn-init in a way that to optimize timeout periods

### Features

* Template-based
* Only "Resources" is mandatory in template
* A few resourcers can't be created through CFN, such as KeyPairs, R53 HostedZones, etc
* Every resource can contain a Metadata, which basically appends a JSON-structured value to such resource. You can retrieve resource metadata from the CFN stacks in 2 ways:

1. AWS CLI. Example: 

```bash
$ aws cloudformation describe-stack-resource --stack-name ec2-bootstrapped-webserver --logical-resource-id WebServerSecurityGroup

STACKRESOURCEDETAIL	2015-11-05T18:02:20.381Z	WebServerSecurityGroup	{"Object1":"whatever1","Object2":"whatever2"}	ec2-bootstrapped-webserver-WebServerSecurityGroup-1L8GBCZDGRR7G	UPDATE_COMPLETE	AWS::EC2::SecurityGroup	arn:aws:cloudformation:eu-west-1:429230952994:stack/ec2-bootstrapped-webserver/0a3ff450-83de-11e5-8605-50a68645b2d2	ec2-bootstrapped-webserver
```

2. cfn-get-metadata helper script (it must be installed via aws-cfn-bootstrap package for linux/windows. Although it's installed by defaul in Amazon Linux). Example:

```bash
$ cfn-get-metadata --access-key XXXX --secret-key XXXX --stack ec2-bootstrapped-webserver --resource WebServerSecurityGroup --region eu-west-1
{
    "Object1": "whatever1",
    "Object2": "whatever2"
}
```

### Limitations

* It's not possible to migrate resources into CFN that were already manually created outside of CFN without tearing it all down and rebuilding it from CFN. So if a customer wants to migrate his current infrastructure into CFN, one option would be to use CloudFormer to create a template, and then edit it and schedulle a maintenance window to migrate them little by little, preferably by truncating the template into several nested ones as well.

* [FIXED ALREADY] There is currently no official way in CFN to specify the permissions needed for an SNS topic to trigger the Lambda function. It has to be done through a Custom Resource (Example can be found on StudyTask 08). Note: A new resource type AWS::Lambda::Permission was announced in October/2015 to fix this. I tested and added an example of its use in Study task 08.

### Tips and A-HA moments

* Always use tags where possible, so you won't have issues with resources changing ids due updates

* Create security groups with nothing, and then add the rules after (to avoid circular dependency issues)

* Whenever you create a stack through the AWS CLI, it uses Python 2.7 running Boto libs to connect to AWS via HTTP, through a regional endpoint of the service (i.e: https://cloudformation.eu-west-1.amazonaws.com/), via HTTPS, to make an HTTP POST to a rest API service, and then such api call that creates the stack. After that, it can return the following response codes:

-=- create-stack or update-stack -=-

Returns a 200 response code (HTTP OK), which will also contain some data, such as the RequestID of the stack creation, as well as the ARN of the new stack that's being created. Example captured with --debug argument of the aws CLI command:

```bash
$ aws cloudformation create-stack --stack-name ec2-bootstrapped-webserver --template-body file://~/Git/CloudFormation/StudyTasks/cfn-init/ec2-amazon-linux-apache-php.cform --parameters ParameterKey=KeyName,ParameterValue=AmazonLinuxIreland ParameterKey=InstanceType,ParameterValue=t1.micro --debug

...

2015-11-05 16:55:38,724 - MainThread - awscli.formatter - DEBUG - RequestId: 0a31c388-83de-11e5-b082-33dd0ab48d35
arn:aws:cloudformation:eu-west-1:429230952994:stack/ec2-bootstrapped-webserver/0a3ff450-83de-11e5-8605-50a68645b2d2
$
```

## Tools

### JSON Tools 

#### Validators
* JSONLint - The JSON Validator: http://jsonlint.com/
* JSON Validator: http://www.freeformatter.com/json-validator.html
* JSON formater and validator: http://jsonformatter.curiousconcept.com/

#### Viewers
* JQ - Lightweight and flexible command-line JSON processor: http://stedolan.github.io/jq/

## Documentation
* CloudFormation User-guide: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html
* Developer Tools: http://aws.amazon.com/developertools/AWS-CloudFormation?browse=1
* Official public articles and tutorials: http://aws.amazon.com/cloudformation/aws-cloudformation-articles-and-tutorials/

## Helper Scripts

AWS CloudFormation includes a set of helper scripts (cfn-init, cfn-signal, cfn-get-metadata, and cfn-hup) that are based on cloud-init. You call these helper scripts from your AWS CloudFormation templates to install, configure, and update applications on Amazon EC2 instances that are in the same template.

### CFN-Init

* It runs as root and it uses sh shell
* The cfn-init helper script reads template metadata from the AWS::CloudFormation::Init key and acts accordingly to:

- Fetch and parse metadata from CloudFormation
- Install packages
- Write files to disk
- Enable/disable and start/stop services

The required parameters are "Region", "Stack" and "Resource". Ex:

```bash
/usr/local/bin/cfn-init --stack cfn-bridge 
                        --resource InstanceCRProcessor 
                        --region eu-west-1
```

Command workflow:

1. It connects to the CloudFormation regional endpoint through https and reads the template from that stack, going through the metadata (AWS::CloudFormation::Init key) of the specified Resource.
2. Runs the ConfigSets from the C (checks for "config" key if not specified)

### CFN-Hup

The cfn-hup helper is a daemon that detects changes in resource metadata and runs user-specified actions when a change is detected. This allows you to make configuration updates on your running Amazon EC2 instances through the UpdateStack API action.

It has 2 configuration files (stored in /etc/cfn/ when installed through aws-cfn-bootstrap package for linux. Although it's installed by default in Amazon Linux):

* cfn-hup.conf: Contains Stack name and AWS credentials that the cfn-hup daemon targets. Example of /etc/cfn/cfn-hup.conf:

```Ini
[main]
stack=arn:aws:cloudformation:eu-west-1:123123123123:stack/openvpn/bdffdf60-df18-11e5-ae2e-50faeb5524d2
region=eu-west-1
```

* hooks.conf: Contains the user actions that the cfn-hup daemon calls periodically. The hooks configuration file is loaded at cfn-hup daemon startup only, so new hooks will require the daemon to be restarted. A cache of previous metadata values is stored at /var/lib/cfn-hup/data/metadata_db (not human readable)—you can delete this cache to force cfn-hup to run all post.add actions again. Example of /etc/cfn/hooks.d/cfn-auto-reloader.conf: 

```Ini
[cfn-auto-reloader-hook]
triggers=post.update
path=Resources.VPNInstanceVPC1.Metadata.AWS::CloudFormation::Init
action=/opt/aws/bin/cfn-init -v          --stack openvpn         --resource VPNInstanceVPC1          --configsets InstallAndRun          --region eu-west-1
runas=root
```

Daemon workflow:

1. Once started, the daemon checks /etc/cfn/cfn-hup.conf for stacks to check.
2. Daemon connects to the regional CloudFormation endpoint via HTTPS to retrieve the meta-data from those stacks.
3. If the HTTP header from response has a different 'x-amzn-requestid', daemon checks hooks in /etc/cfn/hooks.d/ for triggers related to the new stack state. If yes, the "action" is executed. Normally it's good practice to add the cfn-init to run here, just like it's set-up in user-data.
4. cfn-hup does NOT send a response to CFN whatsoever, so if something fails in the action, CloudFormation is not notified.

### CFN-Get-Metadata

A way to retrieve the metadata from a resource inside a stack, to be used by the cfn-get-metadata helper script on the instances. It can be useful to set-up the userdata of an instance.

* It retrieves the values in a Json (unlike aws cloudformation cli commands). Requires:

    - Credentials (Access Key and Secret Key)
    - Region
    - Stack
    - Resource

Example:

```bash
$ cfn-get-metadata --access-key XXXX --secret-key XXXX --stack ec2-bootstrapped-webserver --resource WebServerSecurityGroup --region eu-west-1
{
    "Object1": "whatever1",
    "Object2": "whatever2"
}
```

If using the -f to pass the credential-file parameter, it must have the following format (that's different than the typical cli credential file in ~/.aws/credentials):

```Ini
AWSAccessKeyId=XXXXXXXX
AWSSecretKey=XXXXXXXX
```

Command workflow:

1. It connects to the CloudFormation regional endpoint through https and reads the template from that stack, retrieving the metadata of the specified resource.
2. It disconnects from the CloudFormation regional endpoint

### CFN-Signal

A way to send signals to a resource in the stack (like a WaitConditionHandler, as part of a WaitCondition resource). It's used with the command cfn-signal and it can send customized error messages to stack too (-r or --reason attributes).

Requires:

* Region
* Stack
* Resource

Example: 

```bash
$ sudo /opt/aws/bin/cfn-signal --stack openvpn   
                               --resource VPNInstanceVPC1          
                               --region eu-west-1
```

Command workflow:

1. If the unique id (cfn-signal -i) is not passed via the parameters, the command checks the current instance id in 169.254.169.254/latest/meta-data/instance-id
2. It connects to the wait-condition handle address, which is actually a CNAME to a regional s3 bucket.
3. It posts a json-formatted response through a HTTP Request (PUT) to that pre-signed s3 URL. The Json must contain "Status", "UniqueId", "Data" and "Reason". Ex:

```json
{
  "Status" : "StatusValue",
  "UniqueId" : "instance-id",
  "Data" : "Some Data",
  "Reason" : "Some Reason"
}
```

## Custom Resources

* It's done with the AWS::CloudFormation::CustomResource or Custom::String resource type
* It requires a ServiceToken property inside the CFN resource in template, it can define either a SNS topic or a Lambda function.
* CloudFormation sends requests to the Custom Resources in a JSON format that requires many fields, like RequestType (create/update/delete), ResponseURL(s3 bucket pre-signed url), StackID, RequestID, resourcetype, ResourceProperties, LogicalResourceId, etc. Here's an example of a Custom Resource Request Object:

```json
{
   "RequestType" : "Create",    <=== automatically filled
   "ResponseURL" : "http://pre-signed-S3-url-for-response",   <=== automatically filled
   "StackId" : "arn:aws:cloudformation:us-west-2:EXAMPLE/stack-name/guid",   <=== automatically filled
   "RequestId" : "unique id for this create request",
   "ResourceType" : "Custom::TestResource",
   "LogicalResourceId" : "MyTestResource",
   "ResourceProperties" : {
      "Name" : "Value",
      "List" : [ "1", "2", "3" ]
   }
}
```

The custom resource provider processes the AWS CloudFormation request and returns a response of SUCCESS or FAILED to the pre-signed URL.

* The Custom Resource Response Object CAN INCLUDE the output data provided by the resource. Example:

```json
{
   "Status" : "SUCCESS",
   "PhysicalResourceId" : "TestResource1",
   "StackId" : "arn:aws:cloudformation:us-west-2:EXAMPLE:stack/stack-name/guid",
   "RequestId" : "unique id for this create request",
   "LogicalResourceId" : "MyTestResource",
   "Data" : {
      "OutputName1" : "Value1",
      "OutputName2" : "Value2",
   }
}
```

* After getting a response, AWS CloudFormation proceeds with the stack operation according to the response. Any output data from the custom resource is stored in the pre-signed URL location. The template developer can retrieve that data by using the Fn::GetAtt function.

### Types of Custom Resource

#### SNS + EC2 instance (aws-cfn-resource-bridge)

* It's sent to an SNS that must be subscribed to an SQS queue. This SQS queue must contain a role that contains a policy allowing the action "SQS:SendMessage" to the ARN of the queue from the "aws:SourceArn" of the SNS topic.
* That being done, the custom resource must run in an EC2 instance. There's a framework called aws-cfn-resource-bridge (https://github.com/aws/aws-cfn-resource-bridge) that makes things easier.
* Must make sure the instance that runs aws-cfn-resources-bridge framework has internet access in order to be able to access the CloudFormation endpoints via HTTPS and to send the signals (cfn-signal) to the WaitConditionHandler, as well as executing the cfn-init to run the config inside the Metadata of the template (AWS::CloudFormation::Init).
* The instance must contain a IamInstanceProfile property with a "AWS::IAM::InstanceProfile" that contains a Role which is going to be a "sts:AssumeRole" that contains policies allowing sqs:* (or at least to "sqs:ChangeMessageVisibility", "sqs:DeleteMessage" and "sqs:ReceiveMessage"). It should also contain the actions allowing other possible API calls that are going to be used by the Custom Resource.
* If the script in the sections of cfn-resource-bridge.conf has any sort of errors, it won't work and won't post the message to the SQS queue, the error is going to be "Failed to create resource. Unknown Failure" in the Custom Resource error in the CFN events.

Here is the full work flow:

1. SNS ==> Subscribes to an SQS queue.
2. EC2 Instance checks the SQS queue with the aws-cfn-resource-bridge framework and act accordingly to the JSON definition there. 
3. Response is sent through a PUT via HTTPS on the ResponseURL (pre-signed S3 bucket) 

#### Lambda

* AWS CloudFormation calls a Lambda API to invoke the function ("Action": ["lambda:InvokeFunction"]) and passes all the request data to the function, such as the request type and resource properties. 
* Lambda function must contain an execution role ("AWS::IAM::Role") that contains a ["sts:AssumeRole"] as an action to the "lambda.amazonaws.com" service. Aditionally: 
    - It must also contains the policies allowing the actions "logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents" to "arn:aws:logs:*:*:*"
    - Must contain allowed action "cloudformation:DescribeStacks" to "*"
* The response from Lambda to CloudFormation must be sent to the "event.ResponseURL", which is sent within the Custom Resource sent from CloudFormation to Lambda. It is a pre-signed S3 URL.
* Ideally, it must also implement a response for a stack deletion as well (event.RequestType == "Delete"), sending a response to the responseURL with a STATUS of "SUCCESS" too, otherwise the custom resource will timeout and fail during stack deletion.

Here is the full work flow:

CFN Stack ==> (API call to lambda::InvokeFunction) ==> Lambda function with an execution role with the right permissions (logs:* and cloudformation:DescribeStacks).

Lambda function ==> API call (PUT via HTTPS) to ResponseURL (pre-signed S3 bucket) (can contain a "Data")
