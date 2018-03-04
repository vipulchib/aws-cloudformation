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
I have broken down every section of the template and provided my thought process with regards to what I am doing and how.  We will deploy everything in the 'ca-central-1' region of AWS for this exercise.  We will need to know AMI ID's for Arista vEOS Router instance and the Amaazon Linux instance we will instantiate.  

1.  Following AWS CLI command outputs the Arista vEOS Router AMI ID of **ami-42922926**:
     ```
     aws ec2 describe-images --region ca-central-1 --filters Name=name,Values="EOS-4.20.1FX*" Name=is-public,Values=true --
     query 'Images[*].{ID:ImageId}' --output text
     ```
    
2.  Following AWS CLI command outputs the Amazon Linux AMI ID of **ami-a954d1cd**:
     ```
     aws ec2 describe-images --owners amazon --filters --region ca-central-1 'Name=name,Values=amzn-ami-hvm-*-x86_64-gp2' --
     query 'sort_by(Images, &CreationDate) | [-1].ImageId'
     ```

## Building the Cloudformation YAML template.

1. **AWSTemplateFormatVersion: "version date"** - The AWS CloudFormation template version that the template conforms to.
     ```
     AWSTemplateFormatVersion: '2010-09-09'
     ```

2. **Description** - A text string that describes the template.
     ```
     Description: VPC with Arista vEOS Router, subnets, route tables, igw, sg, Linux VMs
     ```

3. **Parameters** - Parameters enable you to input custom values to your template each time you create or update a stack.  We will create the following:

   A. A Parameters for the VPC ID, provide a Description (VPC ID) and then specify the Type of Parameter 
   with a 'Value (default)' as **Arista**
     
   B. A Parameters for the VPC CIDR, provide a Description (VPC Supernet) and then specify the Type of 
   Parameter with a 'Value (default)' as **10.1.0.0/16**

   C. AParameters for the 1st Subnet in the VPC, provide a Description (VPC Subnet A-1) and then specify the 
   Type of Parameter with a 'Value (default)' as **10.1.1.0/24**
     
   D. AParameters for the 2nd Subnet in the VPC, provide a Description (VPC Subnet A-2) and then specify the 
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
     an Amazon EC2 instance.  We will create the following:
     
     A. A Resource for VPC creation and name it **AristaVPC**.  For *'CidrBlock'* section of the Properties we 
     will reference the *'VPCCidr'* Parameter we previously defined.  VPC will be named as *'VPC-Arista'*  but *'Arista'* 
     portion of VPC's name will be derived by referencing the VPC ID we defined in the Parameters section.
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
     B. A Resource for Security Group and name it **AristaSecurityGroup**.  For *'VpcId'* section of the 
     Properties we will reference the *'AristaVPC'* Parameter we previously defined.  In the following example we are 
     allowing SSH (TCP port 22) and ICMP from 'everywhere' (0.0.0.0/0).  The Security Group will be named as *'VPC-Arista-
     SecurityGroup'* but *'Arista'* portion of VPC's name will be derived by referencing the VPC ID we defined in the 
     Parameter section.
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
     C. A Resource for the Internet Gateway and name it **AristaVPCIGW**. The Internet Gateway will be named as 
     *'VPC-Arista-IGW'* but *'Arista'* portion of VPC's name will be derived by referencing the VPC ID we defined in the 
     Parameter section.
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
     D. A Resource to Attach the Internet Gateway and name it **AttachIGW**.  For *'VpcId'* section of the 
     Properties we will reference the *'AristaVPC'* Parameter we previously defined.  Similarly for *'InternetGatewayId'* 
     section of the Properties we will reference the *'AristaVPCIGW'* Parameter we previously defined.
      ```
      AttachIGW:
       Type: AWS::EC2::VPCGatewayAttachment
       Properties:
        VpcId: !Ref 'AristaVPC'
        InternetGatewayId: !Ref 'AristaVPCIGW'
      ```
     E. A Resource for the 1st Subnet and name it **subnetA1**.
      ```
      subnetA1:
       Type: AWS::EC2::Subnet
       Properties:
        VpcId: !Ref 'AristaVPC'
        CidrBlock: !Ref SubnetA1Cidr
        AvailabilityZone: ca-central-1a
        MapPublicIpOnLaunch: false
        Tags:
         - Key: Name
           Value: !Ref SubnetA1Cidr
      ```
     F. A Resource for the 1st Subnet Routing Table and name it **subnetA1RT**
      ```
      subnetA1RT:
       Type: AWS::EC2::RouteTable
       Properties:
        VpcId: !Ref 'AristaVPC'
        Tags:
         - Key: Name
           Value: !Ref SubnetA1Cidr
      ```
     G. A Resource for Associating the 1st Subnet to Routing Table and name it **subnetA1RTAssoc**.
      ```
      subnetA1RTAssoc:
       Type: AWS::EC2::SubnetRouteTableAssociation
       Properties:
        SubnetId: !Ref subnetA1
        RouteTableId: !Ref subnetA1RT  
      ```
     H. A Resource for the 2nd Subnet and name it **subnetA2**.
      ```
      subnetA2:
       Type: AWS::EC2::Subnet
       Properties:
        VpcId: !Ref 'AristaVPC'
        CidrBlock: !Ref SubnetA2Cidr
        AvailabilityZone: ca-central-1a
        MapPublicIpOnLaunch: false
        Tags:
         - Key: Name
           Value: !Ref SubnetA2Cidr
      ```
     I. A Resource for the 2nd Subnet Routing Table and name it **subnetA2RT**
      ```
      subnetA2RT:
       Type: AWS::EC2::RouteTable
       Properties:
        VpcId: !Ref 'AristaVPC'
        Tags:
         - Key: Name
           Value: !Ref SubnetA2Cidr
      ```
     J. A Resource for Associating the 2nd Subnet to Routing Table and name it **subnetA2RTAssoc**.
      ```
      subnetA2RTAssoc:
       Type: AWS::EC2::SubnetRouteTableAssociation
       Properties:
        SubnetId: !Ref subnetA2
        RouteTableId: !Ref subnetA2RT  
      ```
     K. A Resource for Route for 10.0.0.0/8 to point to *'vEOSIntfSubnetA2'* and name it **privateRouteA2**.
      ```
      privateRouteA2:
       Type: AWS::EC2::Route
       DependsOn: vEOSIntfSubnetA2
       Properties:
        RouteTableId: !Ref 'subnetA2RT'
        DestinationCidrBlock: 10.0.0.0/8
        NetworkInterfaceId: !Ref 'vEOSIntfSubnetA2' 
      ```     
     L. A Resource for the default Route for 0.0.0.0/0 to point to *'AristaVPCIGW'* and name it **publicRouteA2**.
      ```
      publicRouteA2:
       Type: AWS::EC2::Route
       DependsOn: AttachIGW
       Properties:
        RouteTableId: !Ref 'subnetA2RT'
        DestinationCidrBlock: 0.0.0.0/0
        GatewayId: !Ref 'AristaVPCIGW'
      ``` 
     M. A Resource for Compute Instance's Interface and name it **VMIntf1a**.
      ```
      VMIntf1a:
      Type: AWS::EC2::NetworkInterface
      Properties:
       SourceDestCheck: false
       SubnetId: !Ref 'subnetA2'
       GroupSet:
       - !Ref AristaSecurityGroup
       PrivateIpAddress: 10.1.11.10
       Tags:
        - Key: Name
          Value: !Sub
          - ${ID}-Host-ETH0
          - {ID: !Ref ID}
      ```  
     N. A Resource to Create and Associate an Elastic IP to the previously created Compute Instance's Interface and name it 
     **VMIntf1aEIP**.
      ```
      VMIntf1aEIP:
       Type: AWS::EC2::EIP
       Properties:
        Domain: vpc
      AssociateVMIntf1aEIP:
       Type: AWS::EC2::EIPAssociation
       Properties:
        AllocationId: !GetAtt VMIntf1aEIP.AllocationId
        NetworkInterfaceId: !Ref VMIntf1a
      ```  
     O. A Resource for the Compute Instance and name it **Ec2Instance1a**.
      ```
      Ec2Instance1a:
       Type: AWS::EC2::Instance
       DependsOn: AristaVPCIGW
       Properties:
        AvailabilityZone: ca-central-1a
        ImageId: ami-a954d1cd
        InstanceType: t2.micro
        KeyName: Arista-vEOS-Router
        UserData: 
         Fn::Base64: !Sub | 
          #!/bin/bash -xe 
          yum update -y 
          yum --enablerepo=epel install -y iperf iperf3
         NetworkInterfaces:
          - NetworkInterfaceId: !Ref VMIntf1a
            DeviceIndex: 0
         Tags:
          - Key: Name
            Value: !Sub
            - ${ID}-Host
            - {ID: !Ref ID}
      ```  
     P. A Resource for the Arista vEOS Router's Interfaces and name them **vEOSIntfSubnetA1** and **vEOSIntfSubnetA2**.
      ```
      vEOSIntfSubnetA1:
       Type: AWS::EC2::NetworkInterface
       Properties:
        SourceDestCheck: false
        SubnetId: !Ref 'subnetA1'
        GroupSet:
        - !Ref AristaSecurityGroup
        PrivateIpAddress: 10.1.1.6
        Tags:
         - Key: Name
           Value: !Sub
           - vRouter-${ID}-ETH1
           - {ID: !Ref ID}
      vEOSIntfSubnetA2:
       Type: AWS::EC2::NetworkInterface
       Properties:
        SourceDestCheck: false
        SubnetId: !Ref 'subnetA2'
        GroupSet:
        - !Ref AristaSecurityGroup
        PrivateIpAddress: 10.1.11.6
        Tags:
         - Key: Name
           Value: !Sub
           - vRouter-${ID}-ETH2
           - {ID: !Ref ID}
      ``` 
     Q. A Resource to Create and Associate an Elastic IP to the previously created Arista vEOS Router's Ethernet2 Interface 
     and name it **vEOSIntfSubnetA2EIP**.
      ```
      vEOSIntfSubnetA2EIP:
       Type: AWS::EC2::EIP
       Properties:
        Domain: vpc
      AssociatevEOSIntfSubnetA2EIP:
       Type: AWS::EC2::EIPAssociation
       Properties:
        AllocationId: !GetAtt vEOSIntfSubnetA2EIP.AllocationId
        NetworkInterfaceId: !Ref vEOSIntfSubnetA2
      ``` 
     R. A Resource to create the Arista vEOS Router instance and name it **AristavEOSRouter**.
      ```
      AristavEOSRouter:
       Type: AWS::EC2::Instance
       DependsOn: AristaVPCIGW
       Properties:
        AvailabilityZone: ca-central-1a
        ImageId: ami-42922926
        InstanceType: t2.medium
        KeyName: Arista-vEOS-Router
        UserData: 
         Fn::Base64: "%EOS-STARTUP-CONFIG-START%\nhostname Arista-vRouter\ninterface Ethernet1\nmtu 9001\nno switchport\nip 
         address 10.1.1.6/24\ninterface Ethernet2\nmtu 9001\nno switchport\nip address 10.1.11.6/24\nip route 0.0.0.0/0 
         Ethernet2 10.1.11.1\nip routing\n%EOS-STARTUP-CONFIG-END%\n"
        NetworkInterfaces:
        - NetworkInterfaceId: !Ref vEOSIntfSubnetA1
          DeviceIndex: 0
        - NetworkInterfaceId: !Ref vEOSIntfSubnetA2
          DeviceIndex: 1
        Tags:
         - Key: Name
           Value: !Sub
           - ${ID}-vRouter
           - {ID: !Ref ID}
      ``` 
# Building a Stack
We will build the Stack using the YAML file that comprises of the Parameters and Resources we defined in the previous section.  We will use the following AWS CLI command to launch the stack:
```
aws cloudformation create-stack --stack-name AristaVPCStack --template-body file://Github-AristaVPC.yaml
```





