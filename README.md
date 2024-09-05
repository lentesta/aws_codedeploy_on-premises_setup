# aws_codedeploy_on-premises_setup
Instructions on how to set up AWS CodeDeploy for on-premises Ubuntu 22 instances. 

I'm writing this in the hopes it helps others who need to run AWS CodeDeploy on local machines. The AWS instructions for this try to be comprehensive. In trying to be comprehensive, they're long, confusing (to me, anyway), and perhaps incomplete.

Note that this setup is only needed if you're trying to deploy code using CodeDeploy to an on-premises instance. You don't need to do this if you're deploying to an EC2 instance, or if you're using ECS or AWS Lambda.

Assuming everything goes well, this should take you 45-60 minutes to complete.

AWS has two ways of authenticating CodeDeploy requests [Details here: https://docs.aws.amazon.com/codedeploy/latest/userguide/on-premises-instances-register.html]
  - Using an IAM role to authenticate requests. This uses temporary credentials that have to be refreshed periodically with the AWS Security Token Service. It's more work, with more things to go wrong, and is the recommended usecase by AWS, probably for security reasons.
  - Using a IAM User ARN. This is the easier, faster method. If you can easily control physical and network access to your on-premises instance, this might be a valid trade-off.
  - This guide uses IAM User ARNs.

Prerequites:
  - A local instance running Ubuntu 22.04 LTS
  - Superuser or sudo access to that machine
  - An AWS account with permissions to do things like create IAM users, CodeDeploy applications and deployment configurations, etc.

##Step 1 - Create an AWS IAM user with CodeDeploy permissions

  From the AWS Console:
    Click Services > Security, Identity, & Compliance > IAM

    Click Users > Create User and use "on-premises-ubuntu" as your username. If you see an empty checkbox labeled "Provide user access to the AWS Management Console - optional", you can leave that unchecked. Once you've entered the username, click "Next".

    The next IAM screen should be titled "Set Permissions." This is your chance to add the on-premises-ubuntu user to an existing group, or create a new group.  Your choice here doesn't matter for the purposes of this tutorial. The simplest thing to do here might be to create a new group and give it no permissions. The thing you want to do is be able to edit the user's Permissions Policies.

    Once you've created the on-premises-ubuntu user, it's time to add the CodeDeploy permissions.  In the IAM User console you should see a tabbed section with tab titles like "Permissions | Groups | Tags | Security credentials | Access Advisor". Click Permissions, then click the "Add permissions" button on the right side of that section and choose the "Create inline policy" option.

    You should end up on a screen titled "Create policy", with a section named "Specify Permissions". 
    
    On the right side of that page, click the button labeled "JSON". You should see an editor titled "Sepcify Permissions", with some default JSON that looks like this:

    <img width="1227" alt="image" src="https://github.com/user-attachments/assets/35b41e60-e395-42e9-b4c4-0b573ae1686d">


    

    

    
    

    
    
    

  
