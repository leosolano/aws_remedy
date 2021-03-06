# AWS and Remedy Integration for MultiCustomer Environments
In the following tutorial I am going to describe the step by step required to integrate aws cloudwatch with Remedy Helix ITSM. Event though it could be the same process to integrate  any other exposed  web services, during this tutorial we will share the use of SOAP web services, which could be the midleware between AWS and Remedy Helix. The set of aws services that will be part of this integration are: cloudwatch alarm, SNS,Lambda, IAM, DynamoDB, Cloud9 & EC2

![Architecture](https://github.com/leosolano/aws_remedy/blob/main/img/img1.png)

# Step 1. Create an EC2 instance t2.micro and install stress tool 
As you might now cloudwatch have the feature of cloudwatch alarm that we could use to triger an SNS topic, but the intention in this first step is to create a t2.micro aws linux 2 ami EC2 instance  with Public IP enabled in order to access it via SSH and install the stress tool. 

1. login to the EC2 instance: ssh -i "key-pair" ec2-user@public_ip
2. Enable EPEL repo : "sudo amazon-linux-extras install epel -y"
3. Install Stress : "sudo yum install stress -y"

# Step 2. Create a cloudwatch alarm 
In this step the idea is to create a cloudwatch alarm based on the instance ID that you could capture from the EC2 console. Once you have the ID thats looks as i-4567sbsgsjabcd, go to -->cloudwatch --> alarms -->create alarms --> select metric. At this point you need to look inside the EC2 metrics related with the interested variable (CPUUtilization in our case). In the "conditions" section choice Greater or Greater/Equal, than and put the value in % for your alarm. In our example it would be 90%, then hit "next". In the "Alarm state trigger" section choice *Create New Topic* and put the name of your new SNS Topic. If you didn't created a SNS topic preiously, an email address would be required, but you could create an SNS topic in advance to this step and you can avoid to add an email address to link  a Lambda function as part of this topic. 

# Step 3. Create a Cloud9 Environment
As part of the integration process it´ll be require to use some python libraries that are not present in Lamda by default, it is why you need to create a Cloud9 environment, where you could install the python modules that are not present in Lambda, as "zeep" library, required to make SOAP calls to the remedy web services. 
Go to Cloud9 environment, and create the smaller instance you can for your project. Once you have your environment ready, install the zeep package following the next comands: python -m pip install --target=./ zeep. If any other external library is required just go and use the following public post: https://aws.amazon.com/premiumsupport/knowledge-center/cloud9-deploy-lambda-external-libraries/

# Step 4. Create a DynamoDB Table (Optional)
For those cases where the information that comes from the cloudwatch alarms is not enough to fill the form to pass the alarm to the web services, might be a good idea to create a DynamoDB table including some fields that could be linked with the AWS AccountID,  as customer name, customer contact, etc. In this case the table we created included just the account ID and customer name fields, where the  account ID is the partition key in this DynamoDB table.
Go to DynamoDB-->Create Table : Section "Create DynamoDB table" Table Name <your table>, Partition Key <Account_ID> then  hit "create"
Go to your new table an create items as follow --> "Create Item", in Accound_ID value field write "<the list of your acount IDs>", then append a new String field with "<your customer name>". 

# Step 5. Create your Lambda code from Cloud9.
Now is the time to deploy the lamda fuction called aws_to_remedy.py in the Cloud9 Environment, but previos to do that you need to check the exact set of field that you wsdl endpoint requires. To do that execute the following commamnd from the Cloud9 console: python -mzeep <your wsdl endpoint>. See the attached example in "example" folder. After that, you would be able to capture all the required fields in the wsdl Method to pass later in the SOAP call, using request class. 
In the code folder you have one example of this lambda function using python3.7  as runtime, so now is the moment to pass the whole environment (including zeep library) to Lambda. To do that see the procedure as follows here: https://docs.aws.amazon.com/cloud9/latest/user-guide/lambda-toolkit.html.    Please use the "Uploading a Lambda function" section.
Remember to set the appropiate IAM Lambda execution role to allows the Lambda function to record events in cloudwatch logs and to query DynamoDB Tables. In my example I used the included IAM Policies in the "examples" folder.
            
# Step 6. Subscribe the Lamda function to the SNS Topic
You need to subscribe the lambda function to created in the previous step, to teh SNS topic created in the Step 2. 
To do that just go to SNS in console, click in "topics" -->choice your SNS Topic-->Create Subscription-->"Protocol" Section choice "AWS Lambda"-->paste tghe ARN of your AWS Lambda function-->hit "create subscription"
            
# Step 7. Test the integration
This is the final step, and the only required action is to access the EC2 instances under test and force the CPU to be higher than 90% with the stress tool.

1. login to the EC2 instance: ssh -i "key-pair" ec2-user@public_ip
2. run stress : "stress -c 2 -t 900"
3. Review the alarms in cloudwatch alarms
4. Validate if the Lambda function ends withou any error in the cloudwatch log group.

         

 

            
            






