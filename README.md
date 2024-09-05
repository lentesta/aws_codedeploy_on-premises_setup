# aws_codedeploy_on-premises_setup
Instructions on how to set up AWS CodeDeploy for on-premises Ubuntu 22 instances. 

I'm writing this in the hopes it helps others who need to run AWS CodeDeploy on local machines. The AWS instructions for this try to be comprehensive. In trying to be comprehensive, they're long, confusing (to me, anyway), and perhaps incomplete.

Note that this setup is only needed if you're trying to deploy code using CodeDeploy to an on-premises instance. You don't need to do this if you're deploying to an EC2 instance, or if you're using ECS or AWS Lambda.

This should take you 45-60 minutes to complete if everything goes well.

### Assumptions
You already know how to use CodeDeploy for EC2 instances.

AWS has [two ways of authenticating CodeDeploy requests](https://docs.aws.amazon.com/codedeploy/latest/userguide/on-premises-instances-register.html):
  - Using an IAM role to authenticate requests. This uses temporary credentials that have to be refreshed periodically with the AWS Security Token Service. It's more work, with more things to go wrong, and is the recommended usecase by AWS, probably for security reasons.
  - Using a IAM User ARN. This is the easier, faster method. If you can easily control physical and network access to your on-premises instance, this might be a valid trade-off.


This guide uses IAM User ARNs.
This guide also uses Command-line Access Keys.
We're using the AWS region us-east-1.

### Prerequites
  - A local instance running Ubuntu 22.04 LTS
  - Superuser or sudo access to that machine
  - An AWS account with permissions to do things like create IAM users, CodeDeploy applications and deployment configurations, etc.

## Step 1 - Create an AWS IAM user with CodeDeploy permissions
From the AWS Console:
Click Services > Security, Identity, & Compliance > IAM

Click Users > Create User and use "on-premises-ubuntu" as your username. If you see an empty checkbox labeled "Provide user access to the AWS Management Console - optional", you can leave that unchecked. Once you've entered the username, click "Next".

The next IAM screen should be titled "Set Permissions." This is your chance to add the on-premises-ubuntu user to an existing group, or create a new group.  Your choice here doesn't matter for the purposes of this tutorial. The simplest thing to do here might be to create a new group and give it no permissions. The thing you want to do is be able to edit the user's Permissions Policies.

Once you've created the on-premises-ubuntu user, it's time to add the CodeDeploy permissions.  In the IAM User console you should see a tabbed section with tab titles like "Permissions | Groups | Tags | Security credentials | Access Advisor". Click Permissions, then click the "Add permissions" button on the right side of that section and choose the "Create inline policy" option.

You should end up on a screen titled "Create policy", with a section named "Specify Permissions". Here you'll create what AWS calls a "Customer inline" policy using JSON.

To do this, on the right side of that page, click the button labeled "JSON". You should see an editor titled "Specify Permissions", with some default JSON that looks like this:

<img width="1227" alt="image" src="https://github.com/user-attachments/assets/35b41e60-e395-42e9-b4c4-0b573ae1686d">

Replace all of that with the following text. 
```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Action": [
				"iam:CreateAccessKey",
				"iam:CreateUser",
				"iam:DeleteAccessKey",
				"iam:DeleteUser",
				"iam:DeleteUserPolicy",
				"iam:ListAccessKeys",
				"iam:ListUserPolicies",
				"iam:PutUserPolicy",
				"iam:GetUser",
				"codedeploy:CreateDeployment",
				"codedeploy:RegisterOnPremisesInstance",
				"codedeploy:GetOnPremisesInstance",
				"codedeploy:TagResource",
				"codedeploy:AddTagsToOnPremisesInstances",
				"codedeploy:DeregisterOnPremisesInstance"
			],
			"Resource": "*"
		}
	]
}
```

Click 'next' to save this. If you can name this, call it something understandable like 'Our_CodeDeploy_Policy_For_On-Premises_Instances'.

This policy gives the on-premises-ubuntu user the ability to do CodeDeploy actions, and also create access keys. Those access keys will come in handy shortly.

Once you've completed this step you should be back at the IAM > User console window.

Before going on to the next step, take a look at the Summary section at the top of this page. It'll contain basic information about the on-premises-ubuntu user. 

**Make note of the ARN value because you'll need it later** The ARN should look like this: arn:aws:iam::999999999999:user/on-premises-ubuntu, where 999999999999 will be a number specific to your AWS account.
                                    

## Step 2 - Create Access Keys for your AWS IAM user

You should be at the IAM > User console window.

In the IAM User console you should see a tabbed section with tab titles like "Permissions | Groups | Tags | Security credentials | Access Advisor". Click "Security credentials."

In that window should be a section titled "Access Keys". Click "Create Access Key", and select the option for Command Line Interface, and click the "I understand" checkbox that appears. Then click Next.

You'll be prompted to enter a Description tag value for these keys. Make that value something that the next person who reads it will easily understand what it's for, then click the button named "Create access key."

**When your access keys are created** make note of these name/value pairs, since you'll need them later:
- Access Key ID
- Secret Access Key

At this point, you should know the following for your IAM user:
- User ARN (from Step 1)
- Access Key ID (from this step)
- Secret Access Key (from this step)

## Step 3 - Install AWS tools on your local machine

In this step you're going to install:
- the AWS CodeDeploy agent to your local machine
- the AWS Command Line Interface (CLI)

You should be signed in to your local Ubuntu 22.04 LTS instance as a user with sudo permissions.

First, we're going to install the CodeDeploy agent. [Here's the AWS page for these steps](https://docs.aws.amazon.com/codedeploy/latest/userguide/codedeploy-agent-operations-install-ubuntu.html)

In short, here are those steps, using the command line:

```
sudo apt update
```
```
sudo apt install ruby-full
```
```
sudo apt install wget
```

Change us-east-1 in the line below if you want to use another AWS region
```
wget https://aws-codedeploy-us-east-1.s3.us-east-1.amazonaws.com/latest/install
```
```
chmod +x ./install
```
```
sudo ./install auto
```
To check that codedeploy-agent is running:
```
systemctl status codedeploy-agent
```
If it's running, then in that output you should see a line that looks like this:

> Active: active (running) since Thu 2024-12-31 23:59:59 EDT; 1h 45min ago

If you don't see that message, refer to the AWS document above for troubleshooting tips.

Next, we're going to install the AWS Command Line tools. These are useful tools for a variety of reasons. One of them is that when your local machine supports both staging and production environments, your local bash scripts can ask CodeDeploy whether it's deploying to staging or production, and make decisions based on that. (For example, you might use the CodeDeploy afterInstall.bash script to customize a crontab entry one way for staging and another for production.)

Here's how to install the AWS Command Line tools:
```
sudo snap install aws-cli --classic
```

Next, you'll need to create a **crdentials** file so that when you run these AWS command line tools, the tools know which AWS IAM user to run them as.

Refer back to Step 1 where you saved the IAM User information for your user. You'll need that information here.
```
cd $HOME
```
```
mkdir .aws
```
```
cd .aws
```
```
vi credentials
```
Then enter the following:
> [default]
> 
> aws_access_key_id = [put here the access key id you created in Step 1 and don't type the brackets here]
> 
> aws_secret_access_key = [put here the secret access key value you created in Step 1 and don't type the brackets here]
> 
> region = us-east-1
> 

Save the file then enter this command to make the file unreadable to anyone but the owner:
```
chmod 600 credentials
```

## Step 4 - Register your on-premises instance with AWS CodeDeploy

Now you need to tell AWS CodeDeploy that your on-premises instance exists. [Here's the AWS page with more detail.](https://docs.aws.amazon.com/codedeploy/latest/userguide/instances-on-premises-register-instance.html) This section is a summary of that page.

You'll need to come up with a name for your on-premises instance, if you don't already have one. We're going to use 'staging-myapp-machine409" for this example, since it specifies the environment ("staging"), the app it's running ("myapp"), and the machine name ("machine409"). 

You'll also need the IAM ARN for the IAM CodeDeploy user you created in Step 1. Remember that it looks like this: "arn:aws:iam::999999999999:user/on-premises-ubuntu", where "999999999999" will be a different number specific to your AWS account. Replace the string "arn:aws:iam::999999999999:user/on-premises-ubuntu" in the command below with your specifc values.

Here's the AWS CLI command to run to register your on-premises instance named staging-myapp-machine409:
```
aws deploy register-on-premises-instance --instance-name staging-myapp-machine409 --iam-user-arn arn:aws:iam::999999999999:user/on-premises-ubuntu --region=us-east-1 --profile=default
```
If this command works, I think there's no output. (If you get an error, refer to the AWS page above.) 

If the command worked, you'll see the instance registered in the AWS Console's [CodeDeploy On-premises instances page](https://us-east-1.console.aws.amazon.com/codesuite/codedeploy/on-premises-instances?region=us-east-1). It should look something like this:

![image](https://github.com/user-attachments/assets/5a7c6554-c3e2-4a76-9e84-3cfe20d3f667)

If you want to test CodeDeploy to just this one instance, it's helpful to add a tag here. 

Click the instance name, which is a link, and you'll be shown a new screen. On that screen is a section named "On-premises instances tags." Add a tag with these values:

> Key: instanceName
> Value: staging-myapp-machine409

The reason for adding this tag is that you can create a CodeDeploy Deployment Group that includes these on-premises instances, by telling the Deployment Group to look for on-premises instances with instanceName(s) that match whatever you want. We'll do that below.

## Step 5 - Create the codedeploy.onpremises.yml file on your on-premises instance

This step is not documented well in AWS' CodeDeploy documentation. You'll need to add another config file to your local machine, which tells the CodeDeploy service which IAM ID it should use when running. 

You'll need the AWS access keys you created in Step 1, and the IAM ARN you created in Step 2.

Create the file named /etc/codedeploy-agent/conf/codedeploy.onpremises.yml on your local machine:

```
vi /etc/codedeploy-agent/conf/codedeploy.onpremises.yml
```
And add the following lines:
> ---
> 
> :region: us-east-1
> 
> :aws_access_key_id: [put here the access key id you created in Step 1 and don't type the brackets here]
> 
> :aws_secret_access_key: [put here the secret access key value you created in Step 1 and don't type the brackets here]
> 
> :iam_user_arn: arn:aws:iam::999999999999:user/on-premises-ubuntu
> 

Save the file. 

I know that the format of this file doesn't look anything like other AWS credentials file. Also note that some AWS documentation does not show the leading ":" symbol at the beginning of the tokens. I think that symbol is required.

## Next Steps and General Advice

Those are the basic setup steps. 

From here you can create a CodeDeploy Deployment Group for your on-premises instance. When you do that, you can identify those on-premises instances using the instanceName tag you created in Step 4.

## Troubleshooting
On Ubuntu 22.04, the AWS CodeDeploy agent stores its log file in /var/log/aws/codedeploy-agent/codedeploy-agent.log.

Common On-Premises CodeDeploy errors you might see in codedeploy-agent.log, and how to solve them:

> 2024-12-31T12:59:59 INFO  [codedeploy-agent(2353390)]: On Premises config file does not exist or not readable

Make sure you've created the /etc/codedeploy-agent/conf/codedeploy.onpremises.yml file and it looks like this:

```
---
:region: us-east-1
:aws_access_key_id: SECRET_KEY_ID_HERE
:aws_secret_access_key: SECRET_ACCESS_KEY_HERE
:iam_user_arn: arn:aws:iam::999999999999:user/on-premises-ubuntu
```
I'm pretty sure the first line needs to be those three dash symbols.

If you see a bunch of ruby errors in the log file that include this line, it's probably a missing codedeploy.onpremises.yml file:

> 2024-12-31T11:03:01 ERROR [codedeploy-agent(2353390)]: booting child: error during start or run: Errno::EHOSTUNREACH - Failed to open TCP connection to 169.254.169.254:80 (No route to host - connect(2) for "169.254.169.254" port 80) - /usr/lib/ruby/3.0.0/net/http.rb:987:in `initialize'、



















  




    

    

    
    

    
    
    

  
