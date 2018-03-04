# Overview
Purpose of this post is to outline the basics of creating AWS resources (infrastructure) using Cloud Formation as a service.

# What is CloudFormation
CloudFormation can be interpreted as the ability to make an underlying API call to AWS to provision and configure AWS resources.  Its an automated way to create, modify and delete AWS resources.  CloudFormation templates can either be created using the AWS CloudFormation Designer (https://console.aws.amazon.com/cloudformation/designer) or a a JSON or YAML-formatted document.  In this document we will go through CloudFormation examples using the YAML format.

We will create a 'stack' (a collection of all the AWS resrouces we plan to create) using a CloudFormation YAML-formatted script.  This can be done from the CloudFormation AWS Console reachable via this URL - https://console.aws.amazon.com/cloudformation/ or using the AWS CLI.  More information on AWS CLI installation is available here - http://docs.aws.amazon.com/cli/latest/userguide/installing.html

For more information on AWS CloudFormation please reference this URL - http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html

# Need for Cloudformation Templates
I often have the need to spin up Arista vEOS Router instances in AWS to demonstrate Arista's Any Cloud capabilities.  Doing so from AWS consoles - VPC & EC2 is rather painful with numerous clicks and lots of back and forth.  With help from a colleague I put together a CloudFormation template that helps me automating all the AWS underlay components - VPC, Subnets, Route Tables, Internet Gateways, EC2 instances, VPC peerings and eventually update all the neccessary route tables for demonstration of an overlay built with Arista vEOS Routers.

# Cloudformation Template creation in YAML
AWS has done a tremendous job in listing out all the fine details and the documentation is pretty thorough. I will reference a lot of points from this URL - https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-guide.html. Templates include several major sections. The Resources section is the only required section. Some sections in a template can be in any order. 

## Template Sections

1. **AWSTemplateFormatVersion: "version date"** - The AWS CloudFormation template version that the template conforms to.
     ```
     AWSTemplateFormatVersion: '2010-09-09'
     ```

2. **Description** - A text string that describes the template.
     ```
     Description: VPC with Arista vEOS Router, subnets, route tables, igw, sg, Linux VMs
     ```

3. **Parameters** - Parameters enable you to input custom values to your template each time you create or update a stack.  

..A. We will create a Parameters for the VPC ID, provide a Description (VPC ID) and then specify the Type of Parameter 
    with a 'Value (default)' as **Arista**
     
..B. We will create a Parameters for the VPC CIDR, provide a Description (VPC Supernet) and then specify the Type of 
    Parameter with a 'Value (default)' as **10.100.0.0/16**
 
 C. We will create a Parameters for the 1st Subnet in the VPC, provide a Description (VPC Subnet A-1) and then specify the 
    Type of Parameter with a 'Value (default)' as **10.100.1.0/24**
     
 D. We will create a Parameters for the 2nd Subnet in the VPC, provide a Description (VPC Subnet A-2) and then specify the     
    Type of Parameter with a 'Value (default)' as **10.100.111.0/24**
     
     ```
     Parameters: 
       ID:
         Description: VPC ID
         Type: String
         Default: Arista
       VPCCidr: 
         Description: VPC Supernet
         Type: String
         Default: 10.100.0.0/16
       SubnetA1Cidr: 
         Description: VPC Subnet A-1
         Type: String
         Default: 10.100.1.0/24
       SubnetA2Cidr: 
         Description: VPC Subnet A-2
         Type: String
         Default: 10.100.11.0/24
     ```

4. **Resources** -  The required Resources section declares the AWS resources that you want to include in the stack, such as an Amazon EC2 instance.  
     ```
     Resources: 
     ```
Here is a list of resources we will create:

   A. We will create a Resource for the VPC Creation:
     
     ``````
         TransitVPC:
         Type: AWS::EC2::VPC
         Properties:
           CidrBlock: !Ref VPCCidr
           Tags:
             - Key: Name
               Value: !Sub
               - VPC-${ID}
               - {ID: !Ref ID}
     ``````  
     
   B. We

# Building a Stack
We will build the Stack and use AWS CLI to create, monitor, update and delete stacks.
```
aws cloudformation create-stack --stack-name AristaVPCStack --template-body file://Arista-VPC-Cloudformation.yaml
```





