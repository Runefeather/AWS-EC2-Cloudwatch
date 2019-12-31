# AWS-EC2-Cloudwatch
Tutorial for setting up an EC2 instance configured to send logs to Cloudwatch. 

## CloudWatch roles
The below sections describe how to create CloudWatch rules for two scenarios:
1. To detect changes in the state of an EC2 instance
2. To detect the execution of privileged commands by a user in an EC2 instance.

### Creating CloudWatch rules to detect a change in the state of an EC2 instance
1. Access the CloudWatch portal at https://console.aws.amazon.com/cloudwatch/home
2. Create a new rule by selecting **Rules** from the sidebar, and clicking _Create New Rule_
3. Choose the below-listed options under each of the selectable categories: 
	-	__Event Source__: Event Pattern
	-	__Service Name__: EC2
	-	__Event Type__: EC2 Instance State-change Notification
	-	__Specific state(s):__ Choose from _shutting-down_, _running_, _terminated_ 
4. Now, the event pattern preview will look like this:
~~~~
{
  "source": [
    "aws.ec2"
  ],
  "detail-type": [
    "EC2 Instance State-change Notification"
  ],
  "detail": {
    "state": [
      "shutting-down",
      "running",
      "terminated"
    ]
  }
}
~~~~
5. Select a **Target** (A Target is what gets invoked when the above-defined pattern is found):
	- Choose CloudWatch log group 
	- Create a new log group or choose an existing one.
6. Click __Configure__
7. Optionally, metrics can be created so as to trigger an alarm when the state of the machine is changed

### Creating CloudWatch filters to detect the execution of privileged commands
_The following steps outline how to create filters/alarms to detect sudo commands on an Ubuntu 16.04 flavored server running on an EC2 instance. However, they can also be expanded to work for other kinds of machines._

**First, we set up the IAM role and attach it to the instance.  Navigate to the IAM settings at https://console.aws.amazon.com/iam/**
1. From the Navigation Pane, select **Roles** and click on Create Role
2. Choose the **Trusted Entity** as AWS, and the **Service** as EC2.
3. Next, select, Create Policy. This opens a new tab. Select the JSON option and paste in the following code block:
~~~~
			{
			  "Version": "2012-10-17",
			  "Statement": [
			    {
			      "Effect": "Allow",
			      "Action": [
			        "logs:CreateLogGroup",
			        "logs:CreateLogStream",
			        "logs:PutLogEvents",
			        "logs:DescribeLogStreams"
			    ],
			      "Resource": [
			        "arn:aws:logs:*:*:*"
			    ]
			  }
			 ]
			}
~~~~
4. Click Review Policy. This validates the syntax of the JSON to ensure everything is ok.
5. On the next page, enter a Name and Description, and Create Policy. Then, going back to the Roles page, select the policy just created by searching for the name of the policy.
6. Review all the information in the Role (Add Role name, Role description and tags, optionally) and click Create Role.
7. Finally, attach the IAM role to an already running EC2 instance. (Assuming one is already created, if not, you can begin with a t2.micro instance) To do so, 
	- Go to the EC2 console at https://console.aws.amazon.com/ec2/v2/home, and right-click the instance that is created, and select **Attach/Replace IAM Role** from the drop-down menu.
	- Select the newly created Cloudwatch role and click **Apply**.

**Next, we log all commands executed locally on the EC2 instance. Log into the EC2 instance and execute the following commands.**

1. Modify the system-wide BASH runtime config to log commands by appending the below line in `bash.bashrc`:
```
    sudo nano /etc/bash.bashrc
    export PROMPT_COMMAND='RETRN_VAL=$?;logger -p local6.debug "$(whoami) [$$]: $(history 1 | sed "s/^[ ]*[0-9]\+[ ]*//" ) [$RETRN_VAL]"'
```
2. Set up Logging for local6 to a new file named commands.log in `bash.conf`:
```
		sudo nano /etc/rsyslog.d/bash.conf
		local6.*    /var/log/commands.log
```
3. Set up Log rotation for commands.log in the file `/etc/logrotate.d/rsyslog`:
```		
    sudo nano /etc/logrotate.d/rsyslog
```    
There will be a list of log files to rotate, that look like this:
```
    /var/log/mail.warn
		/var/log/mail.err
		[...]
		/var/log/message
```
Add the new bash-commands log file to that list:
```    
		/var/log/commands.log
```    
4. Restart `rsyslog`
```
sudo service rsyslog restart
```
5. Finally, logout from SSH and login again. The logs of all the commands executed should be present in /var/log/commands.log

**Next, we set up the AWS Logs Agent on the instance. Login to the EC2 instance and execute the below commands**

(_Note: The commands described below are for an EC2 inscance running Ubuntu. For other setups, please refer to the official documentation for setting up the Logs Agent - https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/QuickStartEC2Instance.html_)

1. First, update the packages by running `apt-get update` on the machine. 
2. Then, run the following command to download the Logs Agent package:
```
curl https://s3.amazonaws.com/aws-cloudwatch/downloads/latest/awslogs-agent-setup.py -O
```
3. Use python to run the install for the package. If `python` doesn't work in the below command, try `python3`
```
sudo python ./awslogs-agent-setup.py --region us-east-1
```
(_Note: You can install the CloudWatch Logs agent by specifying the us-east-1, us-west-1, us-west-2, ap-south-1, ap-northeast-2, ap-southeast-1, ap-southeast-2, ap-northeast-1, eu-central-1, eu-west-1, or sa-east-1 regions._)

4.The setup will prompt you for certain information such as AWS key ID or secret. Ensure this information is accurate. Additionally, make sure that the **location you enter is the same as the location the your EC2 instance is running in**. 

5. Add commands.log to for AWS Logs Agent to sync to CloudWatch:
```
sudo nano /var/awslogs/etc/config/command_logs.conf
```
Add the following lines to the file
```
[ec2-commands-log]
datetime_format = %Y-%m-%d %H:%M:%S
file = /var/log/commands.log
log_stream_name = {instance_id}-commands-log
log_group_name = ec2-commands-log
```
6. Save and then restart the AWS Logs Agent:
```
sudo service awslogs restart
```

That's it!
Now you should be able to see logs of all the commands executed on the instance inside CloudWatch Logs.

Now, you can either filter the events with the keyword `sudo` or `su` to find privileged commands, or you can create a metric filter on the log group to alert you when a privileged command is executed by email or SMS. 
Additionally, the logged commands can be used to detect other actions, such as the querying of the Metadata URL.  
