+++
title = 'Deployment Preview with AWS CloudFront'
date = 2023-01-10T11:39:04+01:00
tags = ['AWS','CloudFront','CI/CD pipeline','FrontEnd']

+++

### **Introduction**

Deploy Previews allow you and your team to experience changes to any part of your site without having to publish them to production.

With a deploy previews feature you and your teammates can see the changes of every pull request you make without merging it, this will reduce the burden of rolling back the environment when bugs happen as you can review the changes before.

In this tutorial, you’ll learn about creating a CI pipeline with CodeBuild that gets triggered on every pull request creation or update, for every build we host react build folder on an S3 bucket and serve it with Cloudfront, finally after merging the pull request, we delete the build folder from S3.

In the end, this tutorial will expand your knowledge of AWS Services and help you speed up your development and deployment process.

Before starting, here is a playlist to enjoy reading with

<figure class="kg-card kg-embed-card"><iframe style="border-radius: 12px" width="100%" height="352" title="Spotify Embed: Sythnwave n Chill" frameborder="0" allowfullscreen allow="autoplay; clipboard-write; encrypted-media; fullscreen; picture-in-picture" loading="lazy" src="https://open.spotify.com/embed/playlist/7tspLoZXjrHIPAbHnSmlwj?si=32ab2c15569e4b2c&amp;utm_source=oembed"></iframe></figure>
## **Prerequisites**

To accomplish this tutorial, you’ll need:

- an AWS account with administrative permissions
- an AWS CLI with a minimum version of 2.4.6
- a runnable React app hosted on GitHub
- basic knowledge of YAML file structure
- clone this [simple react app](https://github.com/hamzabouissi/simple-react-app)

## **Creating CI Pipeline**

To build a resilient system, you need to add tests on your code then run them locally and on every push to version control. In this step, you will build a CI pipeline that triggers every pull request creation or update on your GitHub repository, the pipeline will run the desired tests and return a badge containing the status of the tests to your GitHub.

Create the file &nbsp;pull-request.yml &nbsp;in your text editor:

    nano pull-request.yml

Add the following CloudFormation code to the file, which builds a CodeBuildProject that defines a CI pipeline with GitHub integration:

    Resources:
      CodeBuildProject:
        Type: AWS::CodeBuild::Project
        Properties:
          Name: FrontBuildOnPull
          ServiceRole: !GetAtt BuildProjectRole.Arn
          LogsConfig:
            CloudWatchLogs:
              GroupName: !Ref BuildLogGroup
              Status: ENABLED
              StreamName: front_pull_request
          EncryptionKey: "alias/aws/s3"
          Artifacts:
            Type: S3
            Location: !Ref ArtifactBucket
            Name: FrontPullRequest
            OverrideArtifactName: true
            EncryptionDisabled: true
          Environment:
            Type: LINUX_CONTAINER
            ComputeType: BUILD_GENERAL1_MEDIUM
            Image: aws/codebuild/amazonlinux2-x86_64-standard:4.0
           
          Source:
            Type: GITHUB
            Location: "https://github.com/projectX/repoY"
            ReportBuildStatus: true
          Triggers:
            BuildType: BUILD
            Webhook: true
            FilterGroups:
              - - Type: EVENT
                  Pattern: PULL_REQUEST_CREATED,PULL_REQUEST_UPDATED
          Visibility: PUBLIC_READ
          ResourceAccessRole: !Ref PublicReadRole

I will briefly explain the CloudFormation template structure, so you can get a general understanding of CloudFormation files that guides you through the next steps.

The majority of CloudFormation template files will contain Parameters(Optional) and Resources(Required), Outputs(Optional), each block on the Resources is a Service, and every Service contains Type and Properties. The type is referred to as AWS Service or a Custom Function and each Type has a different set of properties.

Let’s analyze the above Code to understand better. Here we have a Resources block that contains three Services. The CodeBuildProject refers to AWS::CodeBuild::Project which is an AWS Service for [CodeBuild](https://aws.amazon.com/codebuild/). The CodeBuild has a set of properties, you can find them here [properties](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-codebuild-project.html), I will explain each property functionality and why we’re adding it.

### **Give CodeBuild the required permissions**

For CodeBuild to work we need to assign the ServiceRole an IAM Role to give CodeBuild the right to access other AWS services like secrets, S3, and logs.

      ...
      ServiceRole: !GetAtt BuildProjectRole.Arn
      ...
      ...

<u>BuildProjectRole</u> is a role that assumes the principal “codebuild.amazonaws.com” the permission to use a set of services on our behalf of us, more reading about [AWS::IAM::ROLE](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-role.html)

After, we need to ask ourselves what services CodeBuild must have access to and what we want to access on each of the services:

- Codebuild needs to create and update the test reports.
- CodeBuild needs to store the CI process logs inside a logs group.
- CodeBuild needs to access the secrets manager to get some secret credentials ex: CYPRESS\_KEY. - CodeBuild needs to get and store React build folder on S3.

Now after we defined our requirements, we create BuildProjectPolicy which is an [AWS::IAM::Policy](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-policy.html) for BuildProjectRole that contains the list of statements, and each statement is composed of a set of actions.

    BuildProjectRole:
        Type: AWS::IAM::Role
        Properties:
          AssumeRolePolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Principal:
                  Service:
                    - codebuild.amazonaws.com
                Action:
                  - sts:AssumeRole
          
      BuildProjectPolicy:
        Type: AWS::IAM::Policy
        DependsOn: BuildProjectRole
        Properties:
          PolicyName: !Sub ${AWS::StackName}-CodeBuildPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - codebuild:CreateReportGroup
                  - codebuild:CreateReport
                  - codebuild:UpdateReport
                  - codebuild:BatchPutTestCases
                  - codebuild:BatchPutCodeCoverages
                # Create and update test report
    
                Resource: "*"
              - Effect: Allow
                Action:
                  - s3:PutObject
                  - s3:GetObject
                  - s3:GetObjectVersion
                # Get and store React build folder on S3
    
                Resource: "*"
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                # Store the CI process logs inside a logs group
                Resource: arn:aws:logs:*:*:*
              - Effect: Allow
                Action:
                  - secretsmanager:*
                # Get secret creedentials ex: CYPRESS_KEY
                
                Resource: "*"
          Roles:
            - !Ref BuildProjectRole

### **Access and Store CodeBuild Logs**

Logs are an important key of DevOps because it adds the visibility aspect to the infrastructure, so gather logs as much as you can. In our case, to enable logs for <u>CodeBuild</u> project we create <u>LogsConfig</u> property.

      ...
      ...
      LogsConfig:
        CloudWatchLogs:
          GroupName: !Ref BuildLogGroup
          Status: ENABLED
          StreamName: front_pull_request
      ...
      ...

Now, we need to create a <u>LogGroup</u> to which CodeBuild pushes logs to

    BuildLogGroup:
        Type: AWS::Logs::LogGroup
        Properties:
          LogGroupName: frontend_build_on_pull
          RetentionInDays: 7

<u>RetentionInDays</u> is the period after which the logs expire.

### **Access and Store React Builds**

After successfully building and testing the project, we should store the build folder on S3 so we can access it again when the developer wants to see his changes. So we create an Artifacts property that stores build on an S3 bucket called “ArtifactBucket”

      ...
      ...
      Artifacts:
        Type: S3
        Location: !Ref ArtifactBucket
        Name: FrontPullRequest
        OverrideArtifactName: true
        EncryptionDisabled: true
      ...
      ...

<u>OverrideArtifactName</u> we use to customize the path where we store the builds.

<u>EncryptionDisabled</u> we disabled it because we don’t need custom encryption in our case, we will be using the default one instead.

    ArtifactBucket:
      Type: 'AWS::S3::Bucket'
      DeletionPolicy: Delete
      Properties:
        BucketName: "some-random-unique-bucket-name"
        BucketEncryption:
          ServerSideEncryptionConfiguration:
            - ServerSideEncryptionByDefault:
                SSEAlgorithm: AES256
        PublicAccessBlockConfiguration:
          BlockPublicAcls: true
          BlockPublicPolicy: true
          IgnorePublicAcls: true
          RestrictPublicBuckets: true

<u>ArtifactBucket</u> is a resource block that refers to AWS::S3::Bucket, also we should add some properties to restrict public access to build folders and define how the data must be encrypted when it gets stored.

<u>Name</u>: you need to change to your unique bucket name, the name must be unique across all AWS buckets.

<u>BucketEncryption</u> we use Amazon S3 server-side encryption which uses 256-bit Advanced Encryption Standard(AES256).

<u>PublicAccessBlockConfiguration</u> this property blocks all public access to the public and ignores any policy that allows public visibility of the bucket.

### **CodeBuild CI Environment**

For the CI to work, you must provide information about the CI environment. A build environment represents a combination of the operating system, programming language runtime, and tools that CodeBuild uses to run a build.

      ...
      ...
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_MEDIUM
        Image: aws/codebuild/amazonlinux2-x86_64-standard:4.0
      ...

<u>Type</u> this is the OS type, which is Linux for our case.

<u>ComputeType</u>: this provides the CodeBuild with information about the computing resources it’s allowed to use (7 GB Memory,4vCPU,128GB Disk ).

<u>Image</u>: A docker image to use during the build which provides the environment with a set of tools and runtimes, here is the [documentation](https://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-available.html) for better details

### **CodeBuild Trigger and Repo Linking**

To fire up the build process whenever a pull request is created or updated, you need to add two properties: Source, Triggers

      ...
      ...
      Source:
        Type: GITHUB
        Location: "https://github.com/projectX/repoY"
        ReportBuildStatus: true
      Triggers:
        BuildType: BUILD
        Webhook: true
        FilterGroups:
          - - Type: EVENT
              Pattern: PULL_REQUEST_CREATED,PULL_REQUEST_UPDATED
      ...
      ...

<u>Source</u> defines the type of version control and the location, you need to change the <u>Location</u> property to your desired repository, on the other side to report the build status back to version control(Github) you turn <u>ReportBuildStatus</u> to true. <u>Triggers</u> specify webhooks that trigger The AWS CodeBuild build.

To allow Codebuild access to your repository, you need to create a personal access token and pass it to CodeBuild so it can access your repo on your behalf, This can be done in two steps:

1- Generate a personal token from your GitHub account:

Visit the [Personal access tokens Github page](https://github.com/settings/tokens/new), then select “Full control of private repositories” and “Full control of repository hooks” and click Generate token.

2- Pass down the generated token to AWS Codebuild credentials: Run the following command to generate an import-source-credentials.json &nbsp;file on your local :

    aws codebuild import-source-credentials --generate-cli-skeleton

After, we need to modify &nbsp;import-source-credentials.json &nbsp;and fill it with your credentials:

- auth\_type: PERSONAL\_ACCESS\_TOKEN
- serverType: GITHUB
- username: your GitHub username
- token: the generated token from the previous step.

    aws codebuild import-source-credentials --cli-input-json file://import-source-credentials.json

To test that your command runs successfully, run the following command to list your CodeBuild credentials:

    aws codebuild list-source-credentials

Now when CodeBuild starts running it will search in your source code for a file named buildspec.yml, by definition :

A _<u>buildspec</u>_ is a collection of build commands and related settings, in YAML format, that CodeBuild uses to run a build.

Here is an example of buildspec.yml from our demo app :

    version: 0.2
    
    phases:
      install:
        commands:
          - npm install 
      pre_build:
        on-failure: ABORT
        commands:
          - echo "run some pre-check"
          #- npm run format:check
          #- npm run lint:check
      build:
        on-failure: ABORT
        commands:
          - echo "run tests"
          # - npm run-script start & npx wait-on http://localhost:3000
          # - npx cypress run --record --key ${CYPRESS_KEY}
    
      post_build:
        commands:
          - |
              npm run build
    
    artifacts:
      # include all files required to run application
      # we include only the static build files
      files:
        - '**/*'
      name: $CODEBUILD_WEBHOOK_TRIGGER
      base-directory: 'dist'

Our CI contains different phases (install, pre\_build …etc), finally, post\_build will run when all the previous steps exited successfully.

<u>post_build</u> will create a &nbsp;dist folder and then CodeBuild will upload the dist folder to S3 under CODEBUILD\_WEBHOOK\_TRIGGER path that will be &nbsp;/pr/{pr\_number}.

For example, if a pull request gets created with the number 1, then after CI builds successfully the build folder will get stored on ArtifactBucket under path **/pr/1/**.

### **CI logs Public Visibility To Users**

The following property gives developers access to see build logs without having an AWS account :

    ...
    ...
    Visibility: PUBLIC_READ
    ResourceAccessRole: !Ref PublicReadRole
    ...

then the PublicReadRole must allow access to logs and S3

    
      PublicReadRole:
        Type: AWS::IAM::Role
        Properties:
          AssumeRolePolicyDocument:
            Statement:
              - Action: ['sts:AssumeRole']
                Effect: Allow
                Principal:
                  Service: [codebuild.amazonaws.com]
            Version: '2012-10-17'
          Path: /
    
      PublicReadPolicy:
        Type: 'AWS::IAM::Policy'
        Properties:
          PolicyName: PublicBuildPolicy
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - "logs:GetLogEvents"
                Resource:
                  - !Sub "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:${BuildLogGroup}:*"
              - Effect: Allow
                Action:
                  - "s3:GetObject"
                  - "s3:GetObjectVersion"
                Resource:
                  - Fn::Sub:
                    - "${ArtifcatArn}/*"
                    - {ArtifcatArn: !GetAtt ArtifactBucket.Arn}
          Roles:
            - !Ref PublicReadRole

### **Sum It Up**

After you understand the full picture of CodeBuild Service and its dependency, you can edit pull-request.yml:

    nano pull-request.yml

replace its content with the following code

    Resources:
      CodeBuildProject:
        Type: AWS::CodeBuild::Project
        Properties:
          Name: FrontBuildOnPull
          ServiceRole: !GetAtt BuildProjectRole.Arn
          LogsConfig:
            CloudWatchLogs:
              GroupName: !Ref BuildLogGroup
              Status: ENABLED
              StreamName: front_pull_request
          EncryptionKey: "alias/aws/s3"
          Artifacts:
            Type: S3
            Location: !Ref ArtifactBucket
            Name: FrontPullRequest
            OverrideArtifactName: true
            EncryptionDisabled: true
          Environment:
            Type: LINUX_CONTAINER
            ComputeType: BUILD_GENERAL1_MEDIUM
            Image: aws/codebuild/amazonlinux2-x86_64-standard:4.0
           
          Source:
            Type: GITHUB
            Location: "https://github.com/projectX/repoY"
            ReportBuildStatus: true
          Triggers:
            BuildType: BUILD
            Webhook: true
            FilterGroups:
              - - Type: EVENT
                  Pattern: PULL_REQUEST_CREATED,PULL_REQUEST_UPDATED
          Visibility: PUBLIC_READ
          ResourceAccessRole: !Ref PublicReadRole
    
      BuildProjectRole:
        Type: AWS::IAM::Role
        Properties:
          AssumeRolePolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Principal:
                  Service:
                    - codebuild.amazonaws.com
                Action:
                  - sts:AssumeRole
          
      BuildProjectPolicy:
        Type: AWS::IAM::Policy
        DependsOn: BuildProjectRole
        Properties:
          PolicyName: !Sub ${AWS::StackName}-CodeBuildPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - codebuild:CreateReportGroup
                  - codebuild:CreateReport
                  - codebuild:UpdateReport
                  - codebuild:BatchPutTestCases
                  - codebuild:BatchPutCodeCoverages
                # Create and update test report
    
                Resource: "*"
              - Effect: Allow
                Action:
                  - s3:PutObject
                  - s3:GetObject
                  - s3:GetObjectVersion
                # Get and store React build folder on S3
    
                Resource: "*"
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                # Store the CI process logs inside a logs group
                Resource: arn:aws:logs:*:*:*
              - Effect: Allow
                Action:
                  - secretsmanager:*
                # Get secret creedentials ex: CYPRESS_KEY
                
                Resource: "*"
          Roles:
            - !Ref BuildProjectRole
    
      BuildLogGroup:
        Type: AWS::Logs::LogGroup
        Properties:
          LogGroupName: frontend_build_on_pull
          RetentionInDays: 7
      
      ArtifactBucket:
        Type: 'AWS::S3::Bucket'
        DeletionPolicy: Delete
        Properties:
          BucketName: "some-random-unique-bucket-name"
          BucketEncryption:
            ServerSideEncryptionConfiguration:
              - ServerSideEncryptionByDefault:
                  SSEAlgorithm: AES256
          PublicAccessBlockConfiguration:
            BlockPublicAcls: true
            BlockPublicPolicy: true
            IgnorePublicAcls: true
            RestrictPublicBuckets: true
      
      PublicReadRole:
        Type: AWS::IAM::Role
        Properties:
          AssumeRolePolicyDocument:
            Statement:
              - Action: ['sts:AssumeRole']
                Effect: Allow
                Principal:
                  Service: [codebuild.amazonaws.com]
            Version: '2012-10-17'
          Path: /
    
      PublicReadPolicy:
        Type: 'AWS::IAM::Policy'
        Properties:
          PolicyName: PublicBuildPolicy
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - "logs:GetLogEvents"
                Resource:
                  - !Sub "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:${BuildLogGroup}:*"
              - Effect: Allow
                Action:
                  - "s3:GetObject"
                  - "s3:GetObjectVersion"
                Resource:
                  - Fn::Sub:
                    - "${ArtifcatArn}/*"
                    - {ArtifcatArn: !GetAtt ArtifactBucket.Arn}
          Roles:
            - !Ref PublicReadRole

To test our code, move into the pull-request.yml file path and run

    aws cloudformation create-stack --stack-name pull-request-preview-stack --template-body file://pull-request.yml --capabilities CAPABILITY_NAMED_IAM

Now visit [Cloudformation Console](https://console.aws.amazon.com/cloudformation/home) and you should see a stack with the name pull-request-preview-stack and its status CREATE\_IN\_PROGRESS or CREATE\_COMPLETE

## **Serving Content With CloudFront**

CloudFront is a content delivery network(CDN), CDN can speed up the serving of your static content by caching in the edges that are near you, it is less expensive and it allows you to add a TLS certificate(HTTP) and domain to your static content.

On the other side, we don’t need all of these things but what we want from CloudFront is Lambda@Edge which will allow us to redirect requests by the header to right S3 bucket folder. To explain more, let’s suppose we made 2 pull-request by numbers 121,122, after 2 of them are tested and built successfully, they will get stored in a folder named pr on the Artifact Bucket with the following structure:

pr/

121/

- assets
- index.html
- …
- …

122/

- assets
- index.html
- …
- …

Now when Lambda@Edge comes to play, every request that comes to the CloudFront host(which is a random subdomain under CloudFront xxxxx.cloudfront.net) with a pull-request header and value 122 will be redirected to the folder “122” to serve “122” pull request contents.

In this section, you will learn how to serve S3 content from a CDN and customize users requests by different criteria like (header,query-params…etc) and also build a trigger whenever a build runs successfully, it clears the cache of a specific pull request on the CDN for the new contents.

### **Creating CloudFront**

Let’s edit pull-request.yml

    nano pull-request.yml

and add a CloudFront Distribution Service that will primarily serve index.html from ArtifactBucket

    Distribution:
      Type: AWS::CloudFront::Distribution
      Properties:
        DistributionConfig:
          Origins:
            - DomainName: !Sub ${ArtifactBucket}.s3.${AWS::Region}.amazonaws.com
              Id: S3Origin
              S3OriginConfig:
                OriginAccessIdentity: !Sub origin-access-identity/cloudfront/${OriginAccessIdentity}
          Enabled: true
          DefaultRootObject: index.html
          Logging:
            Bucket: !Sub ${DistributionBucket}.s3.${AWS::Region}.amazonaws.com
          DefaultCacheBehavior:
            TargetOriginId: S3Origin
            CachePolicyId: !Ref DistributionCachingPolicy
            ViewerProtocolPolicy: https-only
          PriceClass: PriceClass_100

Cloudfront has 4 major properties that we will explain briefly

<u>Origins</u>: This property refers to where CloudFront retrieves the content, we will be adding only Artifact Bucket, also there is a sub-property <u>OriginAccessIdentity</u> that restricts access to ArtifactBucket bucket only from CloudFront.

<u>DefaultRootObject</u>: The default file that CloudFront will render from the bucket

<u>PriceClass</u>: To how many regions should our content get cached? we choose PriceClass\_100 that’s the minimum value because we don’t care about that.

<u>Logging</u>: We should collect access logs, So we store them in a bucket “DistributionBucket”

<u>DefaultCacheBehavior</u> is a required property so we will add it, but we don’t need any caching behavior from CloudFront to test the pull requests. DefaultCacheBehavior contains 3 required attributes:

- **TargetOriginId** points to one of our Origins which is S3Origin the only origin we have.
- **ViewerProtocolPolicy** controls whether the user is redirected to HTTPS or uses HTTP or both, we chose https-only.
- **CachePolicyId** , is an important property because we need to allow CloudFront to pass certain headers to Lambda@Edge and the available [managed cache policies](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-managed-cache-policies.html#managed-cache-policies-list) don’t allow that, so creating a custom cache policy will help. here is the code for the custom cache policy that allows passing the pull-request header to Lambda@Edge.

    DistributionCachingPolicy:
      Type: AWS::CloudFront::CachePolicy
      Properties:
        CachePolicyConfig: 
          Comment: "Allow pass of header and query params to cloudfront"
          DefaultTTL: 86400
          MaxTTL: 31536000
          MinTTL: 80400
          Name: CacheForHeaderAndQuery
          ParametersInCacheKeyAndForwardedToOrigin:
            CookiesConfig:
              CookieBehavior: none
            EnableAcceptEncodingGzip: false
            HeadersConfig: 
              HeaderBehavior: whitelist
              Headers: 
                - pull-request
            QueryStringsConfig: 
              QueryStringBehavior: all

Now the problem we’re facing is that our pull requests content is stored under the PR folder so we cannot serve them as Cloudfront doesn’t allow dynamic selection of contents. So the solution is rerouting the user request before it reaches the S3 bucket and choosing the right path depending on the custom header(pull-request) he passes on the request.

### **Choose The Right Path with Lambda@Edge**

To add Lambda@Edge to CloudFront we will modify DefaultCacheBehavior to look like this:

    DefaultCacheBehavior:
      TargetOriginId: S3Origin
      ViewerProtocolPolicy: redirect-to-https
      CachePolicyId: !Ref DistributionCachingPolicy
      LambdaFunctionAssociations:
        - EventType: 'origin-request'
          LambdaFunctionARN: !Ref VersionedLambdaFunction

Let’s create the lambda function, the programming language will be **python** :

    LambdaFunction:
      Type: 'AWS::Lambda::Function'
      Properties:
        FunctionName: "cloudfront_lambda"
        Code:
          ZipFile: !Sub |
            import re
            import boto3 
    
            def handler(event, context):
                client = boto3.client("s3")
                bucket = "front-end-preview-bucket"
                response = client.list_objects_v2(Bucket=bucket,Prefix="pr/",Delimiter="/")
                prs = [p['Prefix'].split('/')[1] for p in response['CommonPrefixes']]
    
                request = event['Records'][0]['cf']['request']
                print(request)
    
                headers = request['headers']
                pr = headers.get('pull-request',None)
                
                
                if pr is None:
                  print(f"{pr} not found")
                  response = {
                    'status': '404',
                    'statusDescription': 'No Found',
                  }
                  return response
                
                pr = pr[0]['value']
    
                if not pr.isdigit() or not (pr in prs):
                  print(f"{pr} not good")
                  response = {
                    'status': '404',
                    'statusDescription': 'No Found',
                  }
                  return response
    
                pr = int(pr)
                request['uri'] = f"/pr/{pr}{request['uri']}"
    
                print(request)
                return request
    
        Handler: 'index.handler'
        MemorySize: 128
        Role: !GetAtt 'LambdaRole.Arn'
        Runtime: 'python3.9'
        Timeout: 5
    
      LambdaRole:
        Type: 'AWS::IAM::Role'
        Properties:
          AssumeRolePolicyDocument:
            Version: '2012-10-17'
            Statement:
            - Effect: Allow
              Principal:
                Service:
                - 'lambda.amazonaws.com'
                - 'edgelambda.amazonaws.com'
              Action: 'sts:AssumeRole'
          Policies:
            - PolicyName: FetchContentFromBucket
              PolicyDocument:
                Version: "2012-10-17"
                Statement:
                  - Effect: Allow
                    Action:
                      - "s3:GetObject"
                      - "s3:ListBucket"
                      - "s3:GetObjectVersion"
                    Resource:
                      - Fn::Sub:
                        - "${ArtifcatArn}/*"
                        - {ArtifcatArn: !GetAtt ArtifactBucket.Arn}
                     
          ManagedPolicyArns:
          - 'arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole'
          - "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"

To summarize what function do, we fetch all current pull requests that are stored on the bucket and whenever a user makes a request, we check the header(pull-request) value and see if it’s available, if yes we rewrite the request URI to **“/pr/{pr}{request[‘uri’]}”**, if no we return 404 response.

Also, we assign Lambda different policies, some are managed policies like :

- AWSLambdaBasicExecutionRole
- AWSLambdaVPCAccessExecutionRole

and others are custom like FetchContentFromBucket to query React build folder from ArtifactBucket.

### **A Little Demo**

After understanding what each service does, we update our pull-request.yml file with the above code. then we update cloud formation with the following command:

    aws cloudformation update-stack --stack-name pull-request-preview-stack --template-body file://pull-request.yml --capabilities CAPABILITY_NAMED_IAM

Now visit [Cloudformation Console](https://console.aws.amazon.com/cloudformation/home) and you should see a stack with the name pull-request-preview-stack and its status **UPDATE\_IN\_PROGRESS** or **UPDATE\_COMPLETE**.

To get the CloudFront endpoint Click on pull-request-preview-stack on Cloudformation Console and Click Outputs

<figure class="kg-card kg-image-card"><img src="/assets/img/cloudformation-preview.png" class="kg-image" alt loading="lazy" width="1264" height="388" sizes="(min-width: 720px) 720px"></figure>

What’s missing now, is creating a pull request in our repository. first, we need a new branch

    git switch -c feature/test_preview

then let’s make some changes and finally add, commit, push

    git add .
    git commit -m "it's getting darker :black:"
    git push --set-upstream origin feature/test_preview

Now let’s wait until the tests pass successfully 👀

<figure class="kg-card kg-image-card"><img src="/assets/img/Untitled.png" class="kg-image" alt loading="lazy" width="940" height="327" sizes="(min-width: 720px) 720px"></figure>

If you visit the CloudFront Endpoint you will get a 404 page, this happens because you didn’t pass a pull-request number as a header. So to achieve this we need to install a chrome extension “[mobheader](https://modheader.com/)”

<figure class="kg-card kg-image-card"><img src="/assets/img/Untitled-1.png" class="kg-image" alt loading="lazy" width="1085" height="151" sizes="(min-width: 720px) 720px"></figure>

you replace 271 with your pull request number that you find on the Github pull request page

Refresh now, Taddaaaa 🎊

<figure class="kg-card kg-image-card"><img src="/assets/img/Untitled-2.png" class="kg-image" alt loading="lazy" width="1025" height="208" sizes="(min-width: 720px) 720px"></figure>
### **Invalidate CloudFront Cache**

Now, Let's suppose that you made a pull request and requested one of your teammates to make a review for you and he reclaimed something buggy is happening, so you went to investigate the issue and solved it and now you’re pushing the updates. After your CI builds and tests successfully, you visit the CloudFront URL and you find the bug still exists, why ?!.

Well it’s because CloudFront caches the build folder on its servers and you need to invalidate the cache from the servers then CloudFront will request the files again from S3

So the approach will be creating a Lambda Function that gets triggered whenever the builds run successfully, The function takes CloudFront DistributionId as an environment variable and make an invalidation request to /pr/{pr\_number} subfolder

    EventCloudFrontLambda:
      Type: 'AWS::Events::Rule'
      Properties:
        Description: Invalidate Cloudfront after a successful build
        State: ENABLED
        EventPattern:
          source:
            - aws.codebuild
          detail-type:
            - CodeBuild Build State Change
          detail:
            build-status:
              - SUCCEEDED
            project-name:
              - !Ref CodeBuildProject
        Targets:
          -
            Arn: !GetAtt InvalidateCloudFront.Arn
            Id: "TargetFunctionV1"
    
      InvalidateCloudFront:
        Type: 'AWS::Lambda::Function'
        Properties:
          FunctionName: "invalidate_cloudfront_from_codebuild_lambda"
          Environment:
            Variables:
              DistributionId: !GetAtt Distribution.Id
          Code:
            ZipFile: !Sub |
              import boto3
              import uuid
              import os
    
              def handler(event, context):
                  print(event)
                  artifict = event['detail']['additional-information']['artifact']['location']
                  pr = artifict.split("/")[2]
    
                  distribution_id = os.getenv("DistributionId")
    
                  print(distribution_id)
    
                  client = boto3.client("cloudfront")
                  response = client.create_invalidation(
                      DistributionId=distribution_id,
                      InvalidationBatch={
                          'Paths': {
                              'Quantity': 1,
                              'Items': [
                                  f'/pr/{pr}/*',
                              ]
                          },
                          'CallerReference': str(uuid.uuid4())
                      }
                  )
    
                  return {"status":200}
          Handler: 'index.handler'
          MemorySize: 128
          Role: !GetAtt 'LambdaRole.Arn'
          Runtime: 'python3.9'
          Timeout: 15
    
      InvalidateCloudFrontLogs:
        Type: AWS::Logs::LogGroup
        DependsOn: InvalidateCloudFront
        Properties:
          LogGroupName: !Sub "/aws/lambda/${InvalidateCloudFront}"
          RetentionInDays: 7
    
      PermissionForEventsToInvokeLambda:
        Type: AWS::Lambda::Permission
        Properties:
          FunctionName: !Ref InvalidateCloudFront
          Action: "lambda:InvokeFunction"
          Principal: "events.amazonaws.com"
          SourceArn: !GetAtt EventCloudFrontLambda.Arn

### **Test Updating The Pull-request &nbsp;Code**

Let’s make some changes, if you cloned my repository you can change the index.json file

and replace &nbsp;“Hello World War 3! &nbsp;with “Hello World Peace”

and let’s wait for the builds to run successfully and recheck again our preview

## **Challenge For You**

As our pull request creates a directory on S3, it is a waste of storage and money if we leave the directory on S3 after merging the pull request.

So the challenge will be creating a GitHub action workflow that will delete the folder from S3 after merging the pull request. one of the requirements is using AWS OpenID Connect.

Don’t hesitate to email me, It will be a pleasure for me to review your work 😊

## **Summary**

In this article, we walked into different AWS Services(CodeBuild, Cloudfront,S3), we understand the mechanism of AWS IAM finally we learned how to create and deploy our services with Cloudformation

Thanks for your time, stay tuned for new articles.

