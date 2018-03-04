# Overview
Purpose of this post is to outline the basics of creating AWS resources (infrastructure) using Cloud Formation as a service.

# What is CloudFormation
CloudFormation can be interpreted as the ability to make an underlying API call to AWS to provision and configure AWS resources.  Its an automated way to create, modify and delete AWS resources.  CloudFormation templates can either be created using the AWS CloudFormation Designer (https://console.aws.amazon.com/cloudformation/designer) or a a JSON or YAML-formatted document.  In this document we will go through CloudFormation examples using the YAML format.

We will create a 'stack' (a collection of all the AWS resrouces we plan to create) using a CloudFormation YAML-formatted script.  This can be done from the CloudFormation AWS Console reachable via this URL - https://console.aws.amazon.com/cloudformation/ or using the AWS CLI.  More information on AWS CLI installation is available here - http://docs.aws.amazon.com/cli/latest/userguide/installing.html

For more information on AWS CloudFormation please reference this URL - http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html

# Need for Cloudformation Templates
I often have the need to spin up Arista vEOS Router instances in AWS to demonstrate Arista's Any Cloud capabilities.  Doing so from AWS consoles - VPC & EC2 is rather painful with numerous clicks and lots of back and forth.  I've published a document on Arista EOS Central which can be leveraged to build whatever I am demonstrating here using the AWS VPC & EC2 consoles - https://eos.arista.com/arista-any-cloud-platform-hybrid-cloud-veos-router-in-aws-deployment-guide/

With help from a colleague I put together a CloudFormation template that helps me automating all the AWS underlay components - VPC, Subnets, Route Tables, Internet Gateways, EC2 instances, VPC peerings and eventually update all the neccessary route tables for demonstration of an overlay built with Arista vEOS Routers.

# Cloudformation Template creation in YAML
AWS has done a tremendous job in listing out all the fine details and the documentation is pretty thorough. I will reference a lot of points from this URL - https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-guide.html. Templates include several major sections. The Resources section is the only required section. Some sections in a template can be in any order. 

## Template Sections
I have broken down every section of the template and provided my thought process with regards to what I am doing and how.

1. **AWSTemplateFormatVersion: "version date"** - The AWS CloudFormation template version that the template conforms to.
     ```
     AWSTemplateFormatVersion: '2010-09-09'
     ```

2. **Description** - A text string that describes the template.
     ```
     Description: VPC with Arista vEOS Router, subnets, route tables, igw, sg, Linux VMs
     ```

3. **Parameters** - Parameters enable you to input custom values to your template each time you create or update a stack.  

   A. We will create a Parameters for the VPC ID, provide a Description (VPC ID) and then specify the Type of Parameter 
   with a 'Value (default)' as **Arista**
     
   B. We will create a Parameters for the VPC CIDR, provide a Description (VPC Supernet) and then specify the Type of 
   Parameter with a 'Value (default)' as **10.1.0.0/16**

   C. We will create a Parameters for the 1st Subnet in the VPC, provide a Description (VPC Subnet A-1) and then specify the 
   Type of Parameter with a 'Value (default)' as **10.1.1.0/24**
     
   D. We will create a Parameters for the 2nd Subnet in the VPC, provide a Description (VPC Subnet A-2) and then specify the 
   Type of Parameter with a 'Value (default)' as **10.1.11.0/24**
     
   ```
   Parameters: 
    ID:
      Description: Arista VPC ID
      Type: String
      Default: Arista
    VPCCidr: 
      Description: VPC Supernet
      Type: String
      Default: 10.1.0.0/16
    SubnetA1Cidr: 
      Description: VPC Subnet A-1
      Type: String
      Default: 10.1.1.0/24
    SubnetA2Cidr: 
      Description: VPC Subnet A-2
      Type: String
      Default: 10.1.11.0/24
   ```

4. **Resources** -  The required Resources section declares the AWS resources that you want to include in the stack, such as 
     an Amazon EC2 instance.
     
     A. We will create a Resource for VPC creation and we will name it **AristaVPC**.  For *'CidrBlock'* section of the 
     Properties we will reference the *'VPCCidr'* Parameter we previously defined.  VPC will be named as *'VPC-Arista'*  but 
     *'Arista'* portion of VPC's name will be derived from the VPC ID we defined in the Parameters section.
      ```
      AristaVPC:
       Type: AWS::EC2::VPC
       Properties:
        CidrBlock: !Ref VPCCidr
        Tags:
          - Key: Name
            Value: !Sub
            - VPC-${ID}
            - {ID: !Ref ID}
      ```
     B. We will create a Resource for Security Group and we will name it **AristaSecurityGroup**.  For *'VpcId'* section 
     of the Properties we will reference the *'AristaVPC'* Parameter we previously defined.  In the following example we are 
     allowing SSH (TCP port 22) and ICMP from 'everywhere' (0.0.0.0/0).  The Security Group will be named as *'VPC-Arista-
     SecurityGroup'* but *'Arista'* portion of VPC's name will be derived from the VPC ID we defined in the Parameter 
     section.
      ```
      AristaSecurityGroup:
       Type: AWS::EC2::SecurityGroup
       Properties:
        GroupName: Arista VPC Security Group
        GroupDescription: Allow SSH and ICMP from Everywhere
        VpcId: !Ref 'AristaVPC'
        SecurityGroupIngress:
         - IpProtocol: tcp
           FromPort: '22'
           ToPort: '22'
           CidrIp: 0.0.0.0/0
         - IpProtocol: icmp
           FromPort: -1
           ToPort: -1
           CidrIp: 0.0.0.0/0
        Tags:
        - Key: Name
          Value: !Sub
          - VPC-${ID}-SecurityGroup
          - {ID: !Ref ID}
      ```
     C. We will create a Resource for the Internet Gateway and we will name is **AristaVPCIGW**. The Internet Gateway will be 
     named as *'VPC-Arista-IGW'* but *'Arista'* portion of VPC's name will be derived from the VPC ID we defined in 
     the Parameter section.
      ```
      AristaVPCIGW:
       Type: AWS::EC2::InternetGateway
       Properties:
        Tags:
         - Key: Name
           Value: !Sub
           - VPC-${ID}-IGW
           - {ID: !Ref ID}
      ```
      A. We will create a Resource for VPC creation and we will name is **AristaVPC**.  For *'CidrBlock'* section of the 
     Properties we will reference the *'VPCCidr'* Parameter we previously defined.
      ```
      AristaVPC:
       Type: AWS::EC2::VPC
       Properties:
        CidrBlock: !Ref VPCCidr
        Tags:
          - Key: Name
            Value: !Sub
            - VPC-${ID}
            - {ID: !Ref ID}
      ```
     A. We will create a Resource for VPC creation and we will name is **AristaVPC**.  For *'CidrBlock'* section of the 
     Properties we will reference the *'VPCCidr'* Parameter we previously defined.
      ```
      AristaVPC:
       Type: AWS::EC2::VPC
       Properties:
        CidrBlock: !Ref VPCCidr
        Tags:
          - Key: Name
            Value: !Sub
            - VPC-${ID}
            - {ID: !Ref ID}
      ```
     A. We will create a Resource for VPC creation and we will name is **AristaVPC**.  For *'CidrBlock'* section of the 
     Properties we will reference the *'VPCCidr'* Parameter we previously defined.
      ```
      AristaVPC:
       Type: AWS::EC2::VPC
       Properties:
        CidrBlock: !Ref VPCCidr
        Tags:
          - Key: Name
            Value: !Sub
            - VPC-${ID}
            - {ID: !Ref ID}
      ```
     A. We will create a Resource for VPC creation and we will name is **AristaVPC**.  For *'CidrBlock'* section of the 
     Properties we will reference the *'VPCCidr'* Parameter we previously defined.
      ```
      AristaVPC:
       Type: AWS::EC2::VPC
       Properties:
        CidrBlock: !Ref VPCCidr
        Tags:
          - Key: Name
            Value: !Sub
            - VPC-${ID}
            - {ID: !Ref ID}
      ```
     A. We will create a Resource for VPC creation and we will name is **AristaVPC**.  For *'CidrBlock'* section of the 
     Properties we will reference the *'VPCCidr'* Parameter we previously defined.
      ```
      AristaVPC:
       Type: AWS::EC2::VPC
       Properties:
        CidrBlock: !Ref VPCCidr
        Tags:
          - Key: Name
            Value: !Sub
            - VPC-${ID}
            - {ID: !Ref ID}
      ```
     A. We will create a Resource for VPC creation and we will name is **AristaVPC**.  For *'CidrBlock'* section of the 
     Properties we will reference the *'VPCCidr'* Parameter we previously defined.
      ```
      AristaVPC:
       Type: AWS::EC2::VPC
       Properties:
        CidrBlock: !Ref VPCCidr
        Tags:
          - Key: Name
            Value: !Sub
            - VPC-${ID}
            - {ID: !Ref ID}
      ```
     A. We will create a Resource for VPC creation and we will name is **AristaVPC**.  For *'CidrBlock'* section of the 
     Properties we will reference the *'VPCCidr'* Parameter we previously defined.
      ```
      AristaVPC:
       Type: AWS::EC2::VPC
       Properties:
        CidrBlock: !Ref VPCCidr
        Tags:
          - Key: Name
            Value: !Sub
            - VPC-${ID}
            - {ID: !Ref ID}
      ```
     A. We will create a Resource for VPC creation and we will name is **AristaVPC**.  For *'CidrBlock'* section of the 
     Properties we will reference the *'VPCCidr'* Parameter we previously defined.
      ```
      AristaVPC:
       Type: AWS::EC2::VPC
       Properties:
        CidrBlock: !Ref VPCCidr
        Tags:
          - Key: Name
            Value: !Sub
            - VPC-${ID}
            - {ID: !Ref ID}
      ```
     A. We will create a Resource for VPC creation and we will name is **AristaVPC**.  For *'CidrBlock'* section of the 
     Properties we will reference the *'VPCCidr'* Parameter we previously defined.
      ```
      AristaVPC:
       Type: AWS::EC2::VPC
       Properties:
        CidrBlock: !Ref VPCCidr
        Tags:
          - Key: Name
            Value: !Sub
            - VPC-${ID}
            - {ID: !Ref ID}
      ```
# Building a Stack
We will build the Stack and use AWS CLI to create, monitor, update and delete stacks.
```
aws cloudformation create-stack --stack-name AristaVPCStack --template-body file://Arista-VPC-Cloudformation.yaml
```





