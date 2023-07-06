
---
layout: post
title:  "Bastion hosts in Private VPC over SSM no SSH key"
date:   2023-07-03 22:06:00 +0100
categories: jekyll update
---

<!--more-->

# Bastion hosts in Private VPC over SSM no SSH key

## Objective 
- Improve security of AWS bastion hosts by removingpublic access
- Log all connections to the bastion host
- Access control enforced with MFA andAWS Single Sign OnProposal 
- Use SSH tunnelsthrough AWS Systems Manager :  
https://aws.amazon.com/premiumsupport/knowledge-center/systems-manager-ssh-vpc-resources/
- Use EC2 Connect to push temporary SSH Keys

## Benefits:
- No open inbound ports  
- Bastion hostsreside in the private network  
- CloudTrail logging will log all connections to Bastion host  
- Access controlled with IAM permissions (SSO accounts/roles). Protected by MFA as default  
- No need to share SSH keys

## Configure your Bastion host  
- For simplicity use the amazon Linux image (this has the SSM agent installed by default)
- To install on non-Amazon Linux images:  
https://aws.amazon.com/premiumsupport/knowledge-center/install-ssm-agent-ec2-linux
- EC2 instance IAM instance profileneedsSSM access.Use the default EC2SSMprofile. Reference: https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started-instance-profile.html
- The EC2 Connect Agent installed and running on the instance (Installed by default on Amazon Linux instances).https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-connect-set-up.html#ec2-instance-connect-install•Place in the private subnet (private IP only)
- For access to SSM configure Security group to allow outbound http/https.
- Configure your bastion security group to allow access to required resources. Your bastion host should only have access to the resources it needs (e.g., if you need to access to an RDS instance your bastion security group rules should be restricted to only allow traffic to this instance).  


## Configure your local Machine 
- Session Manager Plugin(installed on your machine)  
https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html
- Install ec2 connect Plugin(installed on your machine)  
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-connect-set-up.html#ec2-instance-connect-install-eic-CLI  

    ```
    command: pip install ec2instanceconnectcl
    ```

### Configure your ssh_config file Windows:

- Create the configfile in the following location  
C:\Users\<username>\.ssh
- Add the following text:  

    ```
    # SSH over Session Manager
    host i-* mi-*
    ProxyCommand C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters portNumber=%p"
    ```

### Linux:  
- Create the configfile in the following location:  
  ~/.ssh▪Add the following text
  
  ```
  # SSH over SessionManager
  host i-* mi-*
  ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession 
  ```

## SSH tunnel Examples  

Reference :  

https://aws.amazon.com/premiumsupport/knowledge-center/systems-manager-ssh-vpc-resources/  

### Linux  

- Configure your AWS cli:  

    ```
    aws configure sso --profile 204336659665
    ```
- Create a temporary SSH key pair  

    ```
    ssh-keygen -t rsa -N ''-f ./key-name-pem <<< y>/dev/null 2>&1ssh-keygen -m PEM -t rsa -C "key-name-pem" -f key-name-pem  
    ```
- EC2 Connect to push your temp key Pair to the bastion host

    ```
    awsec2-instance-connect send-ssh-public-key \  
    --instance-id i-0a515674b13235bc4\  
    --availability-zone eu-west-1b\  
    --instance-os-user ec2-user \  
    --ssh-public-key file://key-name-pem.pub \  
    --region eu-west-1\  
    --profile 204336659665  
    ```
- Create SSH tunnel to RDS instance

    ```
    AWS_PROFILE='204336659665' ssh -v -i key-name-pem ec2-user@i-0a515674b13235bc4 -N -L 9090:kt-test-p-appdb.cwjbxgi8senp.eu-west-1.rds.amazonaws.com:3306
    ```
- When the tunnel is established you will be able to access the RDS instance using the MySQL tools on your local machine 

    ```
    mysql -u test -h 127.0.0.1 -P 9090 -p
    ```

- Create a tunnel to SSH to a private EC2 instance A
    ```
    AWS_PROFILE='204336659665' ssh -v -i key-name-pem ec2-user@i-0a515674b13235bc4 -N -L 9091:10.119.1.164:22
    ```
- When the tunnel is established you will be able to access the SSH to your private EC2 instance.(Use your instance SSH key when connecting)  

    ```
    ssh -i <key-name-pem>ec2-user@127.0.0.1 -p 9091
    ```
- Create a tunnel to RDPto a private EC2 instance 

    ```
    AWS_PROFILE='204336659665' ssh -v -i key-name-pemec2-user@i-0a515674b13235bc4-N -L 9080:10.119.1.177:3389
    ```
- When the tunnel is established tunnel, you will be able to RDP using : localhost:9080

