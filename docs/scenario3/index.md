# Scenario 3 - IAM Role credential exfiltration

After manually remediating the previous GuardDuty finding, you have finally finished your first cup of coffee when an email notification comes in alerting you to yet another finding.  You finish reading the first email and then a minute or so later you see the relevant remediation email, meaning Alice has already put in place a remediation for this finding.  The other findings you looked at dealt with EC2 instances and AWS IAM credentials separately, but this finding appears to be related to an AWS IAM Role associated with an EC2 instance.  You decide to take a closer look at the finding and remediation.

**Scenario Note**
	None of your IAM credentials have actually been compromised or exposed in any way. The finding was triggered when you used AWS Systems Manager for accessing the temporary security credentials of the EC2 instance and then ran a number of CLI commands locally from your computer.

## Investigation

### Browse to the GuardDuty console to investigate

To view the findings:

1.  Navigate to the [GuardDuty console](https://ap-southeast-1.console.aws.amazon.com/guardduty/home?) (ap-southeast-1) and then, in the navigation pane on the left, choose **Current**. 
2.  Click the  ![Refresh](images/refreshicon.png "Refresh") icon to refresh the GuardDuty console. You should see a finding with the type **UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration**. 

3.  Click on the **UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration** finding to view the full details. 

Looking at the finding details you can see that this is actually a **High Severity** finding.  This finding informs you of attempts to run AWS API operations from a host outside of EC2, using temporary AWS credentials that were created on an EC2 instance in your AWS account.  This means your EC2 instance has been compromised, and the temporary credentials from the instance have been exfiltrated to a remote host outside of AWS.

> You will notice that each GuardDuty finding has an assigned severity level and value (low, medium, or high) that can help you determine your response to a potential security issue that is highlighted by the finding.  These severity levels are preset by AWS but we have seen customers modify these values in their automation workflows to better align the risk of that finding in the context of their environment and requirements.

### View the CloudWatch Event rule

1.	Navigate to the [CloudWatch console](https://ap-southeast-1.console.aws.amazon.com/cloudwatch/home?) (ap-southeast-1) and on the left navigation, under the **Events** section, click **Rules**.
2.	Click on the rule that Alice configured for this particular finding (**GuardDuty-Event-IAMUser-InstanceCredentialExfiltration**). 

Take a closer look at the **Event Pattern**.  The pattern Alice setup for all the rules specifies particular findings.  

> Like Alice, you can create CloudWatch Event Rules that are triggered for particular findings but you can also create a rule that is triggered based on any GuardDuty finding in order to have a centralized workflow.  Below is an example of an Event Pattern that would trigger for any GuardDuty finding:

```
{
  "detail-type": [
    "GuardDuty Finding"
  ],
  "source": [
    "aws.guardduty"
  ]
}
```

### View the remediation Lambda function

Alice also set up a remediation for this threat. Look through the Lambda Function code to better understand the remediation.

Go to the [Lambda console](https://ap-southeast-1.console.aws.amazon.com/lambda/home?) (ap-southeast-1) and review the function named **GuardDuty-Example-Remediation-InstanceCredentialExfiltration**.

The Lambda Function retrieves the Role name from the finding details and then attaches an IAM policy that revokes all active sessions for the role.

> What permissions does the Lambda Function need to perform the remediation?  Is there a risk associated with this level of permissions?

### Verify that the remediation was successful

To verify that the **InstanceCredentialExfiltration** finding was remediated, you can run one of the CLI commands you ran earlier.

```
aws dynamodb list-tables --profile attacker
```

You should see a response that states that there is an explicit deny for that action. Go view the Role to evaluate the policy that was attached.

1.  Browse to the [AWS IAM](https://console.aws.amazon.com/iam/home?region=ap-southeast-1) console.
2.  Click **Roles** in the left navigation.
3.  Click on the Role you identified in the GuardDuty finding and email notifications (**GuardDuty-Example-EC2-Compromised**).
4.  Click the **Permissions** tab.
5.  Click on the **RevokeOldSessions** Policy.

## Questions

**What are the risks involved with this remediation?**

**What other EC2 instances are using this Role?**

## Summary:

[Summary](https://github.com/securityroadshow/amazon-guardduty-hands-on/blob/master/docs/summary.md)
