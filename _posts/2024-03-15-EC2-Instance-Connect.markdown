---
layout: post
title:  "EC2-Instance-Connect"
date:   2024-03-26 12:13:26 +0000
tags: [AWS,ECS]
---



## EC2 Instance Connect Endpoint

Price :

Free - check this 
https://aws.amazon.com/about-aws/whats-new/2023/06/amazon-ec2-instance-connect-ssh-rdp-public-ip-address/

### Create the EC2 Instance Connect Endpoint

Name : EC2 Instance Connect Endpoint
Ref: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-ec2-instance-connect-endpoints.html#create-eice



### Security Group

Name : EIC Endpoint Security Group
outbould : all

You can limit this to a partuclar security group if needed.
Ref: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/eice-security-groups.html


### Iam Role

If you create an EC2 Instance Connect Endpoint, the EC2InstanceConnectEndpoint managed policy is automatically created in your AWS account and attached to the AWSServiceRoleForEC2InstanceConnect service-linked role.

AWSServiceRoleForEC2InstanceConnect 

Your EC2 instance but have the follwoing role attached ( Or the attached role must have the follwoing permisions)

Trust policy
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2-instance-connect.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}

Permissions policies

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeNetworkInterfaces",
                "ec2:DescribeAvailabilityZones"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateNetworkInterface"
            ],
            "Resource": "arn:aws:ec2:*:*:subnet/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateNetworkInterface"
            ],
            "Resource": "arn:aws:ec2:*:*:network-interface/*",
            "Condition": {
                "ForAllValues:StringEquals": {
                    "aws:TagKeys": [
                        "InstanceConnectEndpointId"
                    ]
                },
                "Null": {
                    "aws:RequestTag/InstanceConnectEndpointId": "false"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:ModifyNetworkInterfaceAttribute"
            ],
            "Resource": "arn:aws:ec2:*:*:network-interface/*",
            "Condition": {
                "Null": {
                    "aws:ResourceTag/InstanceConnectEndpointId": "false"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateTags"
            ],
            "Resource": "arn:aws:ec2:*:*:network-interface/*",
            "Condition": {
                "StringEquals": {
                    "ec2:CreateAction": "CreateNetworkInterface"
                },
                "ForAllValues:StringEquals": {
                    "aws:TagKeys": [
                        "InstanceConnectEndpointId"
                    ]
                },
                "Null": {
                    "aws:RequestTag/InstanceConnectEndpointId": "false"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DeleteNetworkInterface"
            ],
            "Resource": "*",
            "Condition": {
                "StringLike": {
                    "aws:ResourceTag/InstanceConnectEndpointId": [
                        "eice-*"
                    ]
                }
            }
        }
    ]
} 


### Client IP preservation 

Review

The instance needs at least one inbound rule to allow traffic from the EC2 Instance Connect Endpoint.

Allow inbound traffic from the EC2 Instance Connect Endpoint security group.

Allow inbound traffic from the client IP address.

Allow inbound traffic from the VPC CIDR so that any instances in the VPC can send traffic to the destination instance.



EIC Endpoint Security Group