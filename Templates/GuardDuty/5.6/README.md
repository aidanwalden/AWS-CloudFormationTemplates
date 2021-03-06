# FortiGate GuardDuty Integration

The purpose of the solution set available in this folder is to deploy serverless applications in one or many regions that parse GuardDuty finding events and creates dynamic address and address group objects on FortiGates for use in blocking bad actor IP or DNS addresses seen in the findings event.

## This solution set contains 3 templates:
```
1. Main event parsing templates (ie 'aws-gd-processor...') that:

  - Creates a CloudWatch event rule that triggers when GuardDuty finding events are seen in the local region
    which triggers a local lambda function to parse the actual event and push dynamic address objects
    and address group objects to one or many FortiGates via the FortiOS REST API (ie HTTPS). 
	
  - The dynamic address objects are created as either IPv4 or DNS objects based on the GuardDuty event type seen.  

  - These address objects are then appended to dynamic address groups that map to each known GuardDuty finding
    type to provide selective use within your firewall policy.

  - There is also a nested aggregate address group created which contains all the dynamic address group objects
    created for each finding type.
	
  - Creates KMS keys for encrypting Lambda environment variables that contain sensitive information such as
    FortiGate IPs and credentials.  Encryption of the variables is optional and can be disabled at any time.
    Decryption of cipher text can be achieved quickly with the use of AWS CLI and the KMS service.

2. Secondary event forwarding template (ie 'aws-gd-forwarder...') that:

  - Creates a CloudWatch event rule that triggers when GuardDuty finding events are seen in the local region
    which triggers a local lambda function to forward the event data from the local region and invokes the remote
    Lambda function (deployed by the previous templates above) to parse the actual event and create dynamic
    address objects and address group objects to one or many FortiGates via the FortiOS REST API (HTTPS). 
```

There are two main event parsing templates which deploy the same solution set described above with a key difference in where the Lambda function runs.  Lambda functions by default run within an AWS owned VPC and will connect to AWS and other services it interacts with (including FortiGates) using any public AWS public IP within that local region.  These functions can be configured to run within your own VPC which would allow you to use private IP addressing to reach your FortiGates as well as connect to other FortiGates using known public IPs as well.

## Lambda running within AWS VPCs
If you use the main template named **aws-gd-processor_defaultVpcSettings.sam.template.yaml**, this uses the default VPC settings (ie use AWS owned VPC) so keep in mind you will need to allow HTTPS (TCP 443) traffic from any AWS public IP to your FortiGates that you want the event parsing Lambda function to communicate with.

Below is a reference diagram showing this use case.

### Reference Diagram:
---

![Example Diagram](https://raw.githubusercontent.com/fortinetsolutions/AWS-CloudFormationTemplates/master/Templates/GuardDuty/5.6/net-diag-1.jpg)

---

## Lambda running within your VPCs
If you use the main template named **aws-gd-processor_localVpcSettings.sam.template.yaml**, this uses your VPC setting so keep in mind that the subnets the Lambda function is set to initiate traffic from will need to provide reachability to your FortiGate IPs provided (private or public).  AWS requires that two private subnets be provided in different availability zones for redundancy so the Lambda function should always be able to iniate traffic if one zone is down.

Additionally if the Lambda environment variables are encrypted, either a VPC endpoint for KMS needs to be deployed within the same VPC or public internet access needs to be available for Lambda to interact with the KMS API.

Below is a reference diagram showing this use case.

### Reference Diagram:
---

![Example Diagram](https://raw.githubusercontent.com/fortinetsolutions/AWS-CloudFormationTemplates/master/Templates/GuardDuty/5.6/net-diag-2.jpg)

---

## General main event parsing template instructions

With either of the main templates you will be prompted for the same parameters for Lambda environment variables and AWS user name principal for KMS key administration.  The default parameter values can be used for all parameters except for FortiGateLoginInfoList and KeyAdministrator.

The FortiGateLoginInfoList parameter is where you enter in the list of FortiGates that you want Lambda to connect to in the relevant format.  The CloudFormation template has this parameter set to hide your input so this sensitive data can't be seen by other users via the AWS console, CLI, or API.  So it is recommended to prepare your input first and then paste it into this parameter field in the AWS console.

Keep in mind, once you have deployed the template you can go to the Lambda console and scroll down to the environment variables section to validate your input is formatted correctly or correct any issues.

	If you are using only one FortiGate, then your input should follow this format:
		<fgt-ip>,<fgt-user>,<fgt-password>
		1.1.1.1,admin,i-abcdef123456

	If you are using multiple FortiGates, then your input should follow this format:
		<fgt1-ip>,<fgt1-user>,<fgt1-password>|<fgt2-ip>,<fgt2-user>,<fgt2-password>
		1.1.1.1,admin,i-abcdef123456|2.2.2.2,admin,i-abcdef123456

The KeyAdministrator parameter should be your AWS username or the username of the individual that will be in charge of encrypt\decrypting the variable containing the FortiGate login information above.  The username will be added to the KMS key policy as an administrator of the key and also be granted privileges to use the key.

Once you deploy the template, you can trigger the function by generating sample events form the GuardDuty console, use one of the sample events provided as part of this solution set to create a test event for Lambda, or generate live events by running a port scan against some instances in the same region as this Lambda function.

You can view the resulting logs from execution of the Lambda function by looking at the relevant log group in the CloudWatch console.

## (Optional) General secondary event forwarding template instructions

With the secondary event forwarding template named **aws-gd-forwarder.cf.template.yaml**, you will be prompted for parameters for the remote Lambda function so that it can be invoked from the local region when GuardDuty events are seen in the local region.

The TargetLambdaName parameter should be the name of the remote Lambda function deployed by one of the main event parsing templates.  This can be referenced by looking at the outputs from the main event parsing template.  An example value is 'stackname-LambdaFunction-ABC123XYZ'.

The TargetLambdaRegion parameter should be the name of the remote region where one of the main event parsing templates is deployed.  This can be referenced by looking at the outputs from the main event parsing template.  An example value is 'us-east-1'.

The TargetLambdaARN parameter should be the name of the remote Lambda function deployed by one of the main event parsing templates.  This can be referenced by looking at the outputs from the main event parsing template.  An example value is 'arn:aws:lambda:us-east-1:123456789012:function:stackname-LambdaFunction-ABC123XYZ'.

## (Optional) Encrypt Lambda environment variables

After confirming that the value provided for the 'fgtLOGINinfo' environment variable is correct, you can encrypt this with the KMS key created by the template.  Navigate to the Lambda console and scroll down to the environment variables section and expand 'Encryption configuration'.

- Select 'Enable helpers for encryption in transit' 
- In 'KMS key to encrypt in transit', select the key created by the template 'stackname-LambdaFunctionKey'
- Select 'Enter Value' to use that key
- Select 'Encrypt' next to the 'fgtLOGINinfo' variable
- Finally click save in the upper right hand corner of the Lambda console

If you want to decrypt the cipher text at a later time, you can do this with the AWS CLI.  This will allow you to see the original clear text information.  Make sure you are specifying the same region where the cipher text was pulled from for proper decryption.

This is an example AWS CLI command where you can simply paste in your cipher text and run this command on ubuntu:
	
	aws kms decrypt --ciphertext-blob fileb://<(echo '<paste-encrypted-string-here>' | base64 -d) --region <region-value-here> --output text --query Plaintext | base64 -d

 Here is a quick example of doing this with ciphertext from the ca-central-1 region:

	# aws kms decrypt --ciphertext-blob fileb://<(echo 'AQICAHj7dXsqRQCihL+mMyEc0NPccA5sYyPSwRwMxzpnt0BFwwGUD4Tv/Wo95fa8UoDEASt+AAAAqzCBqAYJKoZIhvcNAQcGoIGaMIGXAgEAMIGRBgkqhkiG9w0BBwEwHgYJYIZIAWUDBAEuMBEEDF67d4Q7tiTt8PnmZwIBEIBklOTKrTm0EmV75X2mh0huprQHnFVgiHYw+6aLbT/Z6zqtcIfQYt1dPz4O70wpnK1Xs7gMmAOP9O1dRXgcF4T6WYN55ImzZG2l3lUDLJDFlNWL/GyztcmxPLX+9E83as0SF/aKhw==' | base64 -d) --region ca-central-1 --output text --query Plaintext | base64 -d

	# 10.0.0.254,admin,i-0f39770c95a099070|10.0.2.254,admin,i-06fee8bd7beb35185
