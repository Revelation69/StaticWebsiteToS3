

# CodeBuild YAML file

## Version

```yaml
version: 0.2
env:
  variables:
    APP_ENV: 'prod'
```

The file begins with the version declaration `version: 0.2`, which specifies the version of the build specification syntax. This tells AWS CodeBuild to interpret the file according to the rules of version `0.2`.

Next, we have the `env` section where environment variables are defined. In this case, there is a variable `APP_ENV` that is set to `prod`. This variable is used to signify that the build is for the **production** environment, and you can use it in your application to alter configurations based on the environment.

## Phases

```yaml
phases:
  # use the install phase only for installing packages in the build environment
  install:
    runtime-versions:
       nodejs: 18
    commands:
      - echo Install Angular CLI and all node_modules 
      - cd $CODEBUILD_SRC_DIR/frontend/app-for-aws
      - npm install && npm install -g @angular/cli
  build:
    commands:
      - echo Build process started now
      - cd $CODEBUILD_SRC_DIR/frontend/app-for-aws
      - ng build --configuration=production
  post_build:
    commands:
      - echo Build process finished, upload artifacts to S3 bucket
      - cd dist/app-for-aws
      - ls -la
```

Following that, the `phases` section describes the different stages of the build process: **install**, **build**, and **post_build**.

In the `install` phase, we specify the runtime version by using `runtime-versions`. Here, the Node.js runtime is set to version 18, meaning that CodeBuild will use this version of Node.js when running the build process. Then, we have a series of commands that execute during the install phase. The first command is an `echo` statement, which just prints a message indicating that the Angular CLI and dependencies are being installed. After this, the command `cd $CODEBUILD_SRC_DIR/frontend/app-for-aws` is used to navigate to the Angular application directory within the source code. The `$CODEBUILD_SRC_DIR` is an environment variable that points to the root directory of the checked-out source code. The next command installs all necessary dependencies by running `npm install` and globally installs the Angular CLI with `npm install -g @angular/cli`.

Moving on to the `build` phase, the first command again echoes a message, letting us know that the build process has started. Then, it changes the directory to the same path as before with `cd $CODEBUILD_SRC_DIR/frontend/app-for-aws`. The key command in this phase is `ng build --configuration=production`, which uses Angular’s CLI to build the application with the `production` configuration. This configuration typically ensures that optimizations such as Ahead-of-Time (AOT) compilation, minification, and other performance improvements are applied to the final build.

After the build process is complete, the `post_build` phase runs. This phase typically handles post-processing tasks such as preparing the built files for deployment. The first command here simply prints a message indicating that the build process has finished. Next, the command `cd dist/app-for-aws` navigates into the `dist/app-for-aws` directory, where the built production files are located. Finally, the command `ls -la` lists all files in this directory, which helps to verify that the build was successful and to ensure that the necessary files are present.

## Artifacts

```yaml
artifacts:
  #  get files within the dist folder only
  base-directory: 'frontend/app-for-aws/dist*'
  discard-paths: yes
  files:
    - '**/*'  
```

The `artifacts` section defines what files will be included in the output of the build process. The `base-directory` is set to `frontend/app-for-aws/dist*`, which means that CodeBuild will look in the `dist` folder within the `frontend/app-for-aws` directory for the build artifacts. The wildcard `*` ensures that any folder under `dist`, such as `dist/app-for-aws`, is included. The `discard-paths: yes` setting ensures that the folder structure is not preserved when uploading to the destination, meaning all the files will be placed directly in the root of the S3 bucket. Finally, the `files` section specifies that all files within the `dist` directory and its subdirectories (denoted by `**/*`) should be included in the artifact upload.

# CloudFormation

This CloudFormation template essentially automates the process of setting up AWS resources to deploy our static website, using GitHub as the source for the website's code, AWS CodeBuild to build the website, and an S3 bucket to store the built website. 

### 1. `AWSTemplateFormatVersion`
At the very beginning, you have:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
```

This just specifies the version of the CloudFormation template format you're using. It's telling AWS to use the format introduced on September 9th, 2010. This ensures compatibility with all the CloudFormation features we're going to be using.

---

### 2. `Description`
Next, there's a `Description` section:

```yaml
Description: >-
  AWS CloudFormation sample template
  - Create a new CodeBuild project that gets the source code from GitHub and drops the build artifacts into S3 bucket that hosts the static website
  - Create a new role for CodeBuild
```

This part is there to explain **what this template is doing**. It's like a brief summary of what will happen when this template is applied. In this case, it's telling us that:

- **A new CodeBuild project** will be created to pull the website source code from GitHub and push the build artifacts (the compiled website) into an S3 bucket.
- **A new role** will be created for AWS CodeBuild to allow it to access the necessary AWS resources (like S3 and CloudWatch).

---

### 3. `Parameters`
The `Parameters` section is where you define the **inputs** that you (or anyone else using this template) need to provide. They are variables that you can customize when creating the stack.

```yaml
Parameters:
  paramPersonalGitHubAccessToken:
    Type: String
    MinLength: 10
    ConstraintDescription: Personal GitHub access token is missing
    Description: Provide your personal GitHub access token for CodeBuild to access your GitHub repo
```

The first parameter is `paramPersonalGitHubAccessToken`, which is a string where you’ll provide your **GitHub access token**. This token is required for AWS CodeBuild to access your GitHub repository and get the source code.

```yaml
  paramCodeBuildProjectName:
    Description: Specify CodeBuild project name
    Type: String
    MinLength: 10
    MaxLength: 255
    AllowedPattern: ^[a-zA-Z][-a-zA-Z0-9]*$
    Default: michael-codebuild-for-website-hosting
```

Next, you have `paramCodeBuildProjectName`. Here, you're specifying the name of the CodeBuild project. This can be anything you like, but it must follow certain rules, like starting with a letter and only containing letters, numbers, and hyphens. If you don’t provide a name, it defaults to `michael-codebuild-for-website-hosting`.

```yaml
  paramGitHubLink:
    Type: String
    Description: Provide your GitHub link with source code for static website
    Default: https://github.com/Revelation69/StaticWebsiteToS3
```

`paramGitHubLink` is the URL to your GitHub repository where the source code for the static website is stored. By default, it points to a sample GitHub repository.

```yaml
  paramBuildSpecRelativePathInGitHub:
    Description: Provide a relative path for BuildSpec yaml file in source code for static website
    Type: String
    Default: frontend/app-for-aws/buildspec.yml
```

The `paramBuildSpecRelativePathInGitHub` specifies the relative path to the **buildspec.yml** file in your GitHub repository. This file contains instructions for CodeBuild on how to build your project (like installing dependencies, running tests, etc.). The default is `frontend/app-for-aws/buildspec.yml`.

```yaml
  paramStaticWebsiteHostingBucketName:
    Description: Specify an existing S3 bucket that hosts static website
    Type: String
    Default: michael-website-hosting-bucket
```

`paramStaticWebsiteHostingBucketName` lets you specify the **S3 bucket** where the static website will be hosted. This is the bucket where CodeBuild will upload the built website files. The default is `michael-website-hosting-bucket`.

```yaml
  paramUniqueTagName:
    Description: Specify a unique name for tag
    Type: String
    Default: static-website-hosting-to-s3
    AllowedPattern: "[\\x20-\\x7E]*"
    ConstraintDescription: Must contain only ASCII characters
```

Finally, `paramUniqueTagName` is used to assign a unique **tag** to all resources created by the CloudFormation stack. Tags help you organize and identify resources in your AWS account.

---

### 4. `Resources`
This section defines all the AWS resources that CloudFormation will create. Let's go through each one:

#### `myCodeBuildProjectRole`
```yaml
  myCodeBuildProjectRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub role-for-${paramCodeBuildProjectName}
      Path: /
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Action:
              - 'sts:AssumeRole'
            Effect: Allow
            Principal:
              Service:
                - codebuild.amazonaws.com
      Policies:
        - PolicyName: !Sub policy-for-${paramCodeBuildProjectName}
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Effect: Allow
                Resource:
                  - !Sub arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:${paramCodeBuildProjectName}-CloudWatchLogs
              - Action:
                  - s3:PutObject
                  - s3:GetObject
                  - s3:GetObjectVersion
                Effect: Allow
                Resource:
                  - !Sub arn:aws:s3:::${paramStaticWebsiteHostingBucketName}
                  - !Sub arn:aws:s3:::${paramStaticWebsiteHostingBucketName}/*
```

`myCodeBuildProjectRole` is an **IAM Role** for CodeBuild. This role defines permissions that CodeBuild needs to do its job. It allows CodeBuild to create logs in CloudWatch and interact with S3 (to upload and retrieve website files).

#### `myCodeBuildSourceCredential`
```yaml
  myCodeBuildSourceCredential:
    Type: AWS::CodeBuild::SourceCredential
    Properties:
      AuthType: PERSONAL_ACCESS_TOKEN
      ServerType: GITHUB
      Token: !Ref paramPersonalGitHubAccessToken
```

`myCodeBuildSourceCredential` defines the credentials that CodeBuild will use to access your GitHub repository. This is where you provide the **personal GitHub access token** you set up earlier.

#### `myCodeBuildProject`
```yaml
  myCodeBuildProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: !Ref paramCodeBuildProjectName
      Description: CodeBuild project for automatically build of static website hosted on s3
      Source:
        Type: GITHUB
        Location: !Ref paramGitHubLink
        GitCloneDepth: 1
        BuildSpec: !Ref paramBuildSpecRelativePathInGitHub
        Auth:
          Resource: !Ref myCodeBuildSourceCredential
          Type: OAUTH
      Triggers:
        Webhook: true
        FilterGroups:
          - - Type: EVENT
              Pattern: PUSH
            - Type: HEAD_REF
              Pattern: ^refs/heads/main
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/standard:7.0
      ServiceRole: !Ref myCodeBuildProjectRole
      TimeoutInMinutes: 5
      Artifacts:
        Type: S3
        Name: '/'
        Location: !Sub arn:aws:s3:::${paramStaticWebsiteHostingBucketName}
        EncryptionDisabled: True
      LogsConfig:
        CloudWatchLogs:
          Status: ENABLED
          GroupName: !Sub ${paramCodeBuildProjectName}-cloud-watch-logs
```

`myCodeBuildProject` creates the **CodeBuild project** itself. It ties everything together:
- It connects to the GitHub repository for the source code.
- It specifies the **build environment** (Ubuntu, using the `aws/codebuild/standard:7.0` Docker image).
- It defines the **triggers** to start a build whenever there’s a push to the `main` branch of the repository. Basically CodeBuild is configured to listen to any git pushes to main branch and then trigger the build immediately.
- The build artifacts (the resulting website files) are uploaded to the S3 bucket we specified earlier.

> To know which container, machine and image of the VM you need when using CodeBuild check here for Docker images and here for available runtimes based on the code source programming language and framework you can check for Docker images [here](https://docs.aws.amazon.com/codebuild/latest/userguide/ec2-compute-images.html) and Available runtimes [here](https://docs.aws.amazon.com/codebuild/latest/userguide/available-runtimes.html).

As we have specified NodeJS v18 in our buildspec file, we can use either Amazon Linux 2 AArch64 standard:3.0 or Ubuntu standard:7.0 image.

[Image]


Based on selected image I can get Image identifier from [this](https://docs.aws.amazon.com/codebuild/latest/userguide/ec2-compute-images.html) list.

### 5. `Outputs`
```yaml
Outputs:
  outputCodeBuildProject:
    Description: CodeBuild project name
    Value: !Ref myCodeBuildProject
```

Finally, the **Outputs** section will provide you with the **name of the CodeBuild project** once the stack is created. This is useful for referencing the project after the CloudFormation stack is complete.

---

This CloudFormation template essentially automates the process of setting up a continuous integration and deployment pipeline for a static website. It ties together GitHub, CodeBuild, and S3 to allow automatic building and hosting of your website.