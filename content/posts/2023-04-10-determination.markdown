+++
title = 'GCP -> AWS Migration: Determination'
date = 2023-04-16T16:05:46+01:00
draft = false
tags = ['AWS copilot','Cloudformation', 'RDS']
[cover]
    image = 'img/determination.jpg'
+++


> You know that feeling ?, when you're escaping a bad documentation, instead crawling around searching for a solution, and you find a snippet of code on Stack Overflow or Reddit, after you copy and paste it, it doesn't work then your mind tells you "you need to change it a little bit".  
> So you start changing the code to solve your problem, and guess what? A hell of a lot of new terminology and ideas enter your mind, and you start to get confused. Well, I Hamza Bou Issa am in that state of mind.

The last time I was left in "Deploying RDS with copilot using CloudFormation", Yeah my approach to solving the problem is the same as before, typing a bunch of keywords and questions into Google, clicking on the first few links, if Stack Overflow then I detect responses with green mark and copy code, if it's an article, I find snippets and copy the ones that have RDS or DB on them

I took the time to understand CloudFormation file structure and a few resource types

At first, I found this snippet

    # Set AWS template version
    AWSTemplateFormatVersion: "2010-09-09"
    # Set Parameters
    Parameters:
      EngineVersion:
        Description: PostgreSQL version.
        Type: String
        Default: "14.1"
      SubnetIds:
        Description: Subnets
        Type: "List<AWS::EC2::Subnet::Id>"
      VpcId:
        Description: Insert your existing VPC id here
        Type: String
    
    Resources:
      DBSubnetGroup:
        Type: "AWS::RDS::DBSubnetGroup"
        Properties:
          DBSubnetGroupDescription: !Ref "AWS::StackName"
          SubnetIds: !Ref SubnetIds
    
      DatabaseSecurityGroup:
        Type: "AWS::EC2::SecurityGroup"
        Properties:
          GroupDescription: The Security Group for the database instance.
          VpcId: !Ref VpcId
          SecurityGroupIngress:
            - IpProtocol: tcp
              FromPort: 5432
              ToPort: 5432
    
      DBInstance:
        Type: "AWS::RDS::DBInstance"
        Properties:
          AllocatedStorage: "30"
          DBInstanceClass: db.t4g.medium
          DBName: "postgres"
          DBSubnetGroupName: !Ref DBSubnetGroup
          Engine: postgres
          EngineVersion: !Ref EngineVersion
          MasterUsername: username
          MasterUserPassword: password
          StorageType: gp2
          MonitoringInterval: 0
          VPCSecurityGroups:
          - !Ref DatabaseSecurityGroup

The following code will create 3 resources: **DbInstance** , **DatabaseSecurityGroup** , **DBSubnetGroup** , From my understanding the connection between those resources is a Database need to be created on private **Subnets(DBSubnetGroup)** on the other side for the database to accept connection it needs a security group( **DatabaseSecurityGroup** ) which should be on the same **VPC** as the **Subnets**

Now before I paste this code into <u>environment/addons/rds.yml</u>, I'm going to remove the parameters as we have an alternate method of passing the **SubnetIds** and **VpcId**.

CloudFormation gives the ability to import resources from previously created stacks with `Fn::ImportValue` function. In this case, after I run `copilot env deploy --name test` . Copilot create 2 CloudFormation stacks

<figure class="kg-card kg-image-card"><img src="/assets/img/image_o-7.png" class="kg-image" alt loading="lazy" ></figure>

The first stack is the interesting one, after we open on **mycompany-app-test** stack, we click on the Outputs panel, and it should show us the created resources with export names that can be imported on our RDS stack.

<figure class="kg-card kg-image-card"><img src=" /assets/img/stacks_o.png" class="kg-image" alt loading="lazy" ></figure>

The two interesting export names are **mycompany-app-test-PrivateSubnets** , **mycompany-app-test-VpcId** , let's refactor our rds.yml file and add them

    # Set AWS template version
    AWSTemplateFormatVersion: "2010-09-09"
    # Set Parameters
    Parameters:
      App:
        Type: String
        Description: Your application's name.
      Env:
        Type: String
        Description: The environment name your service, job, or workflow is being deployed
      Name:
        Type: String
        Description: The name of the service, job, or workflow being deployed.
    
    Resources:
      DBSubnetGroup:
        Type: "AWS::RDS::DBSubnetGroup"
        Properties:
          DBSubnetGroupDescription: !Ref "AWS::StackName"
          SubnetIds:
            !Split [',', { 'Fn::ImportValue': !Sub '${App}-${Env}-PrivateSubnets'}]
      DatabaseSecurityGroup:
        Type: "AWS::EC2::SecurityGroup"
        Properties:
          GroupDescription: The Security Group for the database instance.
          VpcId:
            Fn::ImportValue: !Sub '${App}-${Env}-VpcId'
          SecurityGroupIngress:
            - IpProtocol: tcp
              FromPort: 5432
              ToPort: 5432
              CidrIp: 0.0.0.0/0
    
      DBInstance:
        Type: "AWS::RDS::DBInstance"
        Properties:
          AllocatedStorage: "30"
          DBInstanceClass: db.t4g.medium
          DBName: "postgres"
          DBSubnetGroupName: !Ref DBSubnetGroup
          Engine: postgres
          EngineVersion: "14.1"
          MasterUsername: username
          MasterUserPassword: password
          StorageType: gp2
          MonitoringInterval: 0
          VPCSecurityGroups:
          - !Ref DatabaseSecurityGroup

As you see, I removed the previous parameters and replace them with a few parameters **App** , **Env** , **Name** which is copilot required add-ons parameters. Also for SubnetIds I import PrivateSubnets and split it because it must be passed as separate values

After I run `copilot env deploy --name test` a nested stack will get created

<figure class="kg-card kg-image-card"><img src="/assess/img/rds_cloudformation.png" class="kg-image" alt loading="lazy" ></figure>

But how I'm going to test if the database is working or accepting connection while it's not reachable on the public internet, well it seems I can create an ec2 instance on the public subnet and allow connection with the security group.

Here is the refactored code

    # Set AWS template version
    AWSTemplateFormatVersion: "2010-09-09"
    # Set Parameters
    Parameters:
      App:
        Type: String
        Description: Your application's name.
      Env:
        Type: String
        Description: The environment name your service, job, or workflow is being deployed to.
    
     
    Resources:
      DBSubnetGroup:
        Type: "AWS::RDS::DBSubnetGroup"
        Properties:
          DBSubnetGroupDescription: !Ref "AWS::StackName"
          SubnetIds:
            !Split [',', { 'Fn::ImportValue': !Sub '${App}-${Env}-PrivateSubnets' }]
      
      DatabaseSecurityGroup:
        Type: "AWS::EC2::SecurityGroup"
        Properties:
          GroupDescription: The Security Group for the database instance.
          VpcId: 
            Fn::ImportValue:
              !Sub '${App}-${Env}-VpcId'
          SecurityGroupIngress:
            - IpProtocol: tcp
              FromPort: 5432
              ToPort: 5432
              CidrIp: 0.0.0.0/0
      
      DBInstance:
        Type: "AWS::RDS::DBInstance"
        Properties:
          AllocatedStorage: "30"
          DBInstanceClass: db.t4g.medium
          DBName: "postgres"
          DBSubnetGroupName: !Ref DBSubnetGroup
          Engine: postgres
          EngineVersion: "14.1"
          MasterUsername: username
          MasterUserPassword: password
          StorageType: gp2
          MonitoringInterval: 0
          VPCSecurityGroups:
            - !Ref DatabaseSecurityGroup
          Tags:
            - Key: Name
              Value: !Sub 'copilot-${App}-${Env}'
    
      NewKeyPair:
        Type: 'AWS::EC2::KeyPair'
        Properties:
          KeyName: !Sub ${App}-${Env}-EC2-RDS-KEYPAIR
          Tags:
            - Key: Name
              Value: !Sub 'copilot-${App}-${Env}'
      
      EC2SecuityGroup:
        Type: "AWS::EC2::SecurityGroup"
        Properties:
          GroupDescription: The Security Group for the ec2 instance.
          VpcId: 
            Fn::ImportValue:
              !Sub '${App}-${Env}-VpcId'
          SecurityGroupIngress:
            - IpProtocol: tcp
              FromPort: 22
              ToPort: 22
              CidrIp: 0.0.0.0/0
          Tags:
            - Key: Name
              Value: !Sub 'copilot-${App}-${Env}'
            
      Ec2Instance:
        Type: 'AWS::EC2::Instance'
        Properties:
          ImageId: ami-05e8e219ac7e82eba
          InstanceType: t2.micro
          KeyName: !Ref NewKeyPair
          SubnetId: !Select ["0",!Split [',', { 'Fn::ImportValue': !Sub '${App}-${Env}-PublicSubnets' }]]
          SecurityGroupIds:
            - !Ref EC2SecuityGroup
          
          UserData: |
            #!/bin/bash
            sudo apt update
            sudo apt upgrade
            sudo apt install postgresql postgresql-contrib
    
    
          Tags:
            - Key: Name
              Value: !Sub 'copilot-${App}-${Env}'
          
    
    
    Outputs:
      ServerPublicDNS:
        Description: "Public DNS of EC2 instance"
        Value: !GetAtt Ec2Instance.PublicDnsName
    
      DatabaseEndpoint:
        Description: "Connection endpoint for the database"
        Value: !GetAtt DBInstance.Endpoint.Address

A few things to notice, I added an Ec2Instance on a public subnet, a security group that allows only ssh port (22), meanwhile, the ImageId, InstanceType property are more tied to the region you're deploying to, I am using eu-west-3(I'm not a French guy -\_-). NewKeyPair is the ssh key to log into EC2, and finally the Outputs section for getting the EC2 instance and Database connection URL

I rerun `copilot env deploy --name test` , wait for the stack to update, before connecting to ec2 an ssh file must be downloaded, the ssh key pair will be saved on AWS Parameter Store, here are the following steps to download it

    aws ec2 describe-key-pairs --filters Name=key-name,Values=mycompany-app-test-E

The above command output.

    key-05abb699beEXAMPLE

and to save the ssh key value

    aws ssm get-parameter --name /ec2/keypair/key-05abb699beEXAMPLE --with-decrypt

Now, After I get the ssh file, try to connect to the server

    ssh -i new-key-pair.pem ubuntu@ec2-host

I try to connect to RDS

    psql -U username -d postgres -h database_host

<figure class="kg-card kg-image-card"><img src="/assets/img/terminal_o.png" class="kg-image" alt loading="lazy"></figure>