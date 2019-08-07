# Environment Setup


> All of the links assume you are using the **ap-southeast-1 (Singapore)** region.

## Getting Started – Just Two Clicks

Follow these steps to enable GuardDuty:

1. **First Click**: Navigate to the [GuardDuty console](https://us-west-2.console.aws.amazon.com/guardduty/home?) (us-west-2) and click **Get Started**.
![Get Started](images/screenshot1.png "Get Started")

2. **Second Click**: On the next screen click **Enable GuardDuty**.
![Enable GuardDuty](images/screenshot2.png "Enable GuardDuty")

### Data sources

That is all you need to do. There are no prerequisites you need to set up, no agents to install, and no hardware to configure. From the moment you enable GuardDuty it begins analyzing all of the VPC Flow Logs, CloudTrail logs, and DNS Logs in that region. DNS Logs are generated from the default AWS DNS resolvers used for your VPCs and are not an available data source to customers.  

> If you are using a 3rd party DNS resolver or if you set up your own DNS resolvers, then GuardDuty cannot access, process, and identify threats from that data source. 

GuardDuty accesses all of these [data sources](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_data-sources.html) without any of them having to be enabled; although it is a best practice to enable CloudTrail and VPC Flow Logs for your own analysis. GuardDuty is a regional service so in order for the service to monitor these data sources in other regions you will need to enable it in those regions.  You can accomplish this by following the same steps above and enabling it through the console but most customers are using the APIs to programmatically enable it in all regions and across multiple accounts.  Regardless of the number of VPCs, IAM users, or other AWS resources in your account, there is no impact to your resources because all of the processing is being done within the managed service. 

> The pricing for GuardDuty is based on the quantity of AWS CloudTrail Events analyzed and the volume of Amazon VPC Flow Log and DNS Log data analyzed (per GB).  Each region in an AWS Account has a free 30-day trial to better forecast what the cost of the service will be.

![GuardDuty Enabled](images/screenshot3.png "GuardDuty Enabled")

### Findings

Now that GuardDuty is enabled it is actively monitoring the three data sources for malicious or unauthorized behavior as it relates to your EC2 instances and AWS IAM Principals.  You should be taken directly to the **Findings** tab which will show finding details as GuardDuty detects them. After deploying the scenario, you will start to see GuardDuty findings being detected.  Each finding is broken down into the following format to allow for a concise yet readable description of potential security issues.

**ThreatPurpose : ResourceTypeAffected / ThreatFamilyName . ThreatFamilyVariant ! Artifact**

> Click [here](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_finding-format.html) for a description of each part.

The more advanced behavioral and machine learning detections require a baseline (7 - 14 days) to be established so GuardDuty is able to learn the regular behavior and identity anomalies. An example of a finding that requires a baseline would be if an EC2 instance started communicating with a remote host on an unusual port or an IAM User who has no prior history of modifying Route Tables starts making modifications.  All of the findings generated in these scenarios will be based on signatures, so the findings will be detected 10 minutes after the completion of the CloudFormation stack.  The delay is due to the amount of time it takes for the information about a threat to appear in one of the data sources and the amount of time it takes for GuardDuty to access and analyze that particular data source.  

> Click [here](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_finding-types-active.html) for a complete list of current GuardDuty finding types. 

GuardDuty sends notifications based on Amazon CloudWatch Events when any change in the findings takes place. These notifications are sent within 5 minutes of the finding. All subsequent occurrences of an existing finding will have the same ID as the original finding and notifications will be sent every 6 hours after the initial notification.  This is to eliminate alert fatigue due to the same finding.

## Deploy the environment

??? info "AWS Sponsored Event"
    1. Navigate to the <a href="https://dashboard.eventengine.run" target="_blank">Event Engine dashboard</a>
	2. Enter your **team hash** code. 
	3. Click **AWS Console**.  The CloudFormation template for this round has already been prerun.

??? info "Individual"

    Launch the CloudFormation stack below to setup the GuardDuty environment:

    Region| Deploy
    ------|-----
    US West 2 (Oregon) | <a href="https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=GuardDuty-Hands-On&templateURL=https://s3-us-west-2.amazonaws.com/sa-security-specialist-workshops-us-west-2/guardduty-hands-on/guardduty-cfn-template.yml" target="_blank">![Deploy in us-west-2](./images/deploy-to-aws.png)</a>

    1. Click the **Deploy to AWS** button above.  This will automatically take you to the console to run the template.  

    2. Click **Next** on the **Specify Template**, **Specify Details**, and **Options** sections.

    3. Finally, acknowledge that the template will create IAM roles under **Capabilities** and click **Create**.

    This will bring you back to the CloudFormation console. You can refresh the page to see the stack starting to create. Before moving on, make sure the stack is in a **CREATE_COMPLETE**.

The initial findings will begin to show up in GuardDuty 10 minutes after the CloudFormation stack creation completes. 

### What is created?

The CloudFormation template will create the following resources:

  * Three [Amazon EC2](https://aws.amazon.com/ec2/) Instances (and supporting network infrastructure)
    * Two Instances that contain the name “*Compromised Instance*”
    * One instance that contains the name “*Malicious Instance*”
  * [AWS IAM Role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) For EC2 which will have permissions to SSM Parameter Store and DynamoDB
  * One [Amazon SNS Topic](https://docs.aws.amazon.com/sns/latest/dg/GettingStarted.html) so you will be able to receive notifications
  * Three [AWS CloudWatch Event](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/WhatIsCloudWatchEvents.html) rules for triggering the appropriate notification or remediation
  * Two [AWS Lambda](https://aws.amazon.com/lambda/) functions that will be used for remediating findings and will have permissions to modify Security Groups and revoke active IAM Role sessions (on only the IAM Role associated with this scenario)
  * [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-paramstore.html) value for storing a fake database password.

> Make sure the CloudFormation stack is in a **CREATE_COMPLETE** status before moving on.

### Generate a finding manually

All of the simulated attacks are automated in the CloudFormation template except for one; which requires you to take some manual steps.  To produce the final finding, you will need to copy the IAM temporary security credentials from the EC2 instance and make API calls from your laptop (the calls need to come from outside the AWS network). 

#### Retrieve the IAM temporary security credentials using AWS Systems Manager

To simulate this last and final attack you will need to retrieve the IAM temporary security credentials generated by the IAM Role for EC2. You can either SSH directly to the instance and query the metadata or follow the steps below to use [AWS Systems Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html) (an SSM agent was automatically started on the instance at launch). 

1.  Go to [Managed Instances](https://us-west-2.console.aws.amazon.com/systems-manager/managed-instances?region=us-west-2) within the **AWS Systems Manager** console (us-west-2).
    
    > You should see an instance named **GuardDuty-Example: Compromised Instance: Scenario 3** with a ping status of **Online**.

2.  To see the keys currently active on the instance, click on **Session Manager** on the left navigation and then click **Start Session**.
3.  To see the credentials currently active on the instance, click on the radio button next to **Compromised Instance: Scenario 3** and click **Start Session**.
4.  Run the following command in the shell:

    ```
    curl http://169.254.169.254/latest/meta-data/iam/security-credentials/GuardDuty-Example-EC2-Compromised
    ```

5. Make note of the **AccessKeyID**, **SecretAccessKey**, and **Token**.

### Create a new AWS CLI profile on your laptop to use the IAM temporary credentials 

Now that you have retrieved the IAM temporary security credentials you will need to add them to an AWS CLI profile. There are a number of ways to do this, but below are some commands to help get you started:

From a command prompt, run the following commands (replace the variables with your credentials):
```
aws configure set profile.attacker.region us-west-2
aws configure set profile.attacker.aws_access_key_id <access_key>
aws configure set profile.attacker.aws_secret_access_key <secret_key>
aws configure set profile.attacker.aws_session_token <session_token>
```

If you view your local aws credentials file, you should now see an [attacker] profile with the stolen IAM temporary credentials.

### Run commands using the IAM temporary credentials

Now that you have your named profile you can use it to make API calls. Use the commands below to query different services to see what you have access to (don't be surprised if you see some access denied responses):

**Do you have any IAM permissions:**
```
aws iam get-user --profile attacker

aws iam create-user --user-name Chuck --profile attacker
```

**What about DynamoDB?**
```
aws dynamodb list-tables --profile attacker

aws dynamodb describe-table --table-name GuardDuty-Example-Customer-DB --profile attacker
```

**Can you query the data?**
```
aws dynamodb scan --table-name GuardDuty-Example-Customer-DB --profile attacker

aws dynamodb put-item --table-name GuardDuty-Example-Customer-DB --item '{"name":{"S":"Joshua Tree"},"state":{"S":"Michigan"},"website":{"S":"https://www.nps.gov/yell/index.htm"}}' --profile attacker

aws dynamodb scan --table-name GuardDuty-Example-Customer-DB --profile attacker

aws dynamodb delete-table --table-name GuardDuty-Example-Customer-DB --profile attacker

aws dynamodb list-tables --profile attacker
```

**Do you have access to Systems Manager Parameter Store?**
```
aws ssm describe-parameters --profile attacker

aws ssm get-parameters --names "gd_prod_dbpwd_sample" --profile attacker

aws ssm get-parameters --names "gd_prod_dbpwd_sample" --with-decryption --profile attacker

aws ssm delete-parameter --name "gd_prod_dbpwd_sample" --profile attacker
```
