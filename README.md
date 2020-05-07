# lambda-deploy

This is basic demonstration of creating, building, and deploying a .NET Core AWS Lambda function with Github Actions.

# What is Lambda?

Lambda is a [Function as a Service](https://en.wikipedia.org/wiki/Function_as_a_service) offering from AWS
allowing you to create individual functions that can be triggered from a variety of events, such as web
requests via AWS API Gateway or an AWS SQS message being published.

# Walkthrough

## System Configuration

Amazon provides a number of .NET Core project templates to simplify creating new Lambda projects.  These can be
installed by running the following at a console:

```
dotnet new -i Amazon.Lambda.Templates
```

Once these are installed, you will see a number of new templates available:

```
$ dotnet new lambda --list
Usage: new [options]

Options:
  -h, --help          Displays help for this command.
  -l, --list          Lists templates containing the specified name. If no name is specified, lists all templates.
  -n, --name          The name for the output being created. If no name is specified, the name of the current directory is used.
  -o, --output        Location to place the generated output.
  -i, --install       Installs a source or a template pack.
  -u, --uninstall     Uninstalls a source or a template pack.
  --nuget-source      Specifies a NuGet source to use during install.
  --type              Filters templates based on available types. Predefined values are "project", "item" or "other".
  --dry-run           Displays a summary of what would happen if the given command line were run if it would result in a template creation.
  --force             Forces content to be generated even if it would change existing files.
  -lang, --language   Filters templates based on language and specifies the language of the template to create.
  --update-check      Check the currently installed template packs for updates.
  --update-apply      Check the currently installed template packs for update, and install the updates.


Unable to determine the desired template from the input template name: lambda.
The following templates partially match the input. Be more specific with the template name and/or language.

Templates                                                 Short Name                                        Language      Tags
-----------------------------------------------------------------------------------------------------------------------------------------------
Order Flowers Chatbot Tutorial                            lambda.OrderFlowersChatbot                        [C#]          AWS/Lambda/Function
Lambda Custom Runtime Function (.NET Core 3.0)            lambda.CustomRuntimeFunction                      [C#], F#      AWS/Lambda/Function
Lambda Detect Image Labels                                lambda.DetectImageLabels                          [C#], F#      AWS/Lambda/Function
Lambda Empty Function                                     lambda.EmptyFunction                              [C#], F#      AWS/Lambda/Function
Lex Book Trip Sample                                      lambda.LexBookTripSample                          [C#]          AWS/Lambda/Function
Lambda Simple Application Load Balancer Function          lambda.SimpleApplicationLoadBalancerFunction      [C#]          AWS/Lambda/Function
Lambda Simple DynamoDB Function                           lambda.DynamoDB                                   [C#], F#      AWS/Lambda/Function
Lambda Simple Kinesis Firehose Function                   lambda.KinesisFirehose                            [C#]          AWS/Lambda/Function
Lambda Simple Kinesis Function                            lambda.Kinesis                                    [C#], F#      AWS/Lambda/Function
Lambda Simple S3 Function                                 lambda.S3                                         [C#], F#      AWS/Lambda/Function
Lambda Simple SNS Function                                lambda.SNS                                        [C#]          AWS/Lambda/Function
Lambda Simple SQS Function                                lambda.SQS                                        [C#]          AWS/Lambda/Function
```

## Creating the Project

With the templates installed, creating a Lambda function is simply:

```
dotnet new lambda.EmptyFunction --name MyFunction --output .
```

The options specified here are:
 * `lambda.EmptyFunction` - The template to use; in this case **Lambda Empty Function**
 * `--name MyFunction` - The name of the .NET project to create
 * `--output .` - Where to create the project; this is optional, but by default it will put everything
   into a folder with the same name as the `name` parameter, which I don't like

To make it easier to work with the `MyFunction` library and `MyFunction.Tests` test project,
we can create a Visual Studio Solution file and add references to the projects:

```
dotnet new sln --name LambdaDeploy

dotnet sln add src/MyFunction/
dotnet sln add test/MyFunction.Tests/
```

Now we can verify that everything is working as expected:
```
$ dotnet build
Microsoft (R) Build Engine version 16.4.0+e901037fe for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Restore completed in 685.81 ms for C:\src\MyFunction\test\MyFunction.Tests\MyFunction.Tests.csproj.
  Restore completed in 685.79 ms for C:\src\MyFunction\src\MyFunction\MyFunction.csproj.
  MyFunction -> C:\src\MyFunction\src\MyFunction\bin\Debug\netcoreapp2.1\MyFunction.dll
  MyFunction.Tests -> C:\src\MyFunction\test\MyFunction.Tests\bin\Debug\netcoreapp2.1\MyFunction.Tests.dll

Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:08.30

$ dotnet test --no-build
Test run for C:\src\MyFunction\test\MyFunction.Tests\bin\Debug\netcoreapp2.1\MyFunction.Tests.dll(.NETCoreApp,Version=v2.1)
Microsoft (R) Test Execution Command Line Tool Version 16.3.0
Copyright (c) Microsoft Corporation.  All rights reserved.

Starting test execution, please wait...

A total of 1 test files matched the specified pattern.

Test Run Successful.
Total tests: 1
     Passed: 1
 Total time: 4.0221 Seconds
 ```

# Deploying to AWS

> :warning: The following requires that you have configured your AWS credentials already. :warning:

There are different ways of deploying Lambda functions, including Visual Studio plugins or
manually creating a ZIP file of the build and uploading via the AWS Console, but we'll be
using a custom dotnet tool that Amazon has created that can be run via a console.  To install
this, run the following:

```
dotnet tool install -g Amazon.Lambda.Tools
```

This adds a number of commands to the `dotnet` command, but we'll just be working with two
for now:

 * `deploy-function`
 * `invoke-function`

To deploy the function to AWS, we just need to run the following:

```
dotnet lambda deploy-function MyFunction
```

# Invoking on AWS

Using the installed Amazon Lambda dotnet tools, we can test our function by running:

```
$ dotnet lambda invoke-function MyFunction --payload "this should come back uppercase"
Amazon Lambda Tools for .NET Core applications (3.3.1)
Project Home: https://github.com/aws/aws-extensions-for-dotnet-cli, https://github.com/aws/aws-lambda-dotnet

Payload:
"THIS SHOULD COME BACK UPPERCASE"

Log Tail:
START RequestId: 9a0ad78b-e320-4bbd-818a-d616370bc5c5 Version: $LATEST
END RequestId: 9a0ad78b-e320-4bbd-818a-d616370bc5c5
REPORT RequestId: 9a0ad78b-e320-4bbd-818a-d616370bc5c5
       Duration: 0.26 ms
       Billed Duration: 100 ms
       Memory Size: 256 MB
       Max Memory Used: 62 MB
```

Here we can see the response from our function, the **Payload** contains our input string
converted to uppercase, along with other diagnostic and billing information.

# Deploying from Github Actions

For an efficient CI/CD pipeline, we want to do the following:

 * Build and run unit tests on every pull request
 * Deploy the Lambda function to AWS on a merged pull request

With [Github Actions](https://github.com/features/actions), we can do all of this without
requiring an external system.  Adding actions to our repository is as simple as creating
new YAML files under the `.github/workflows` folder.  We will be making two, one for each
of the above goals.

## Build and Run Unit Tests

The following file is created in `.github/workflows/build-and-test.yml`:

```yaml
name: Build and Run Unit Tests on PR

on:
  pull_request:
    branches:
      - master

jobs:
  build:
    runs-on: ubuntu-latest
    name: Build and run unit tests on PR
    steps:
    - uses: actions/checkout@722adc63f1aa60a57ec37892e133b1d319cae598 # 2.0.0
    - name: Setup .NET Core
      uses: actions/setup-dotnet@b7821147f564527086410e8edd122d89b4b9602f # 1.4.0
      with:
        dotnet-version: 2.1.510
    - name: Setup Nuget CLI
      uses: NuGet/setup-nuget@255f46e14d51fbc603743e2aa2907954463fbeb9 # 1.0.2
    - name: Restore with nuget
      run: nuget restore
    - name: Build with dotnet
      run: dotnet build --configuration Release
    - name: Test with dotnet
      run: dotnet test --configuration Release --no-build --no-restore
```

The above is self-documenting, but for clarity the steps it performs are:

1. Checks out the code from Github for processing
1. Installs .NET Core
1. Installs NuGet
1. Runs `nuget restore` to retrieve referenced packaged
1. Runs `dotnet build` to verify that the solution builds
1. Runs `dotnet test` to verify that all unit tests pass

## Deploy to AWS Lambda on Merge

The following file is created in `.github/workflows/deploy-to-lambda-on-merge.yml`:

```yaml
name: Deploy to AWS Lambda on Merge

on:
  push:
    branches:
      - master

jobs:
  build:
    runs-on: ubuntu-latest
    name: Deploy to AWS Lambda on merge
    steps:
    - uses: actions/checkout@722adc63f1aa60a57ec37892e133b1d319cae598 # 2.0.0
    - name: Setup .NET Core
      uses: actions/setup-dotnet@b7821147f564527086410e8edd122d89b4b9602f # 1.4.0
      with:
        dotnet-version: 2.1.510
    - name: Setup Nuget CLI
      uses: NuGet/setup-nuget@255f46e14d51fbc603743e2aa2907954463fbeb9 # 1.0.2
    - name: Install dotnet lambda tool
      run: dotnet tool install --global Amazon.Lambda.Tools
    - name: Publish with dotnet lambda
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_KEY: ${{ secrets.AWS_SECRET_KEY }}
        AWS_REGION: ${{ secrets.AWS_REGION }}
      run: >-
        dotnet lambda
        deploy-function MyFunction
        --project-location src/MyFunction
        --function-role TestRole
        --aws-access-key-id $AWS_ACCESS_KEY_ID
        --aws-secret-key $AWS_SECRET_KEY
        --region $AWS_REGION
        --function-publish true
```

This is very similar to the build and test action, with a few additional steps:

 * **Install dotnet lambda tool** - installs the same `dotnet lambda` tool we used in our
   testing earlier
 * **Publish with dotnet lambda** - publishs the Lambda to AWS, using the same tooling we
   used earlier

The main difference between our local deployment and this one is the use of Github Secrets
for specifying our AWS credentials.  Since we never want to commit passwords or other secrets
to source control, we can use the Github Secrets feature to store these values and inject
them at runtime.

### Github Secrets

Under the **Settings** tab, click on the **Secrets** link.  You will need to add the
following secrets with the appropriate values for your account:

 * `AWS_ACCESS_KEY_ID` - akin to a user name or public key, one of the two pieces needed
   to authenticate with AWS
 * `AWS_SECRET_KEY` - akin to a password or private key, a secret used for authentication
 * `AWS_REGION` - while not technically a secret, using Github Secrets for this lets us
   keep this configuration out of source control

### Lambda Versioning

Even with unit tests verifying that our code is correct, we want to create a new version
of our Lambda on each deploy.  This way, if we change the signature or functionality of
it, existing callers will not be impacted until they upgrade to the latest version.  This
setting is configured by the `--function-publish true` flag in the `deploy-lambda`
command.

# TODO

 * General instructions for setting up AWS Credential
 * Description of IAM roles for running Labmda, `--function-role`
 * Integrating with AWS API Gateway
 * Calling from an application

# References

 * [What Is AWS Lambda?](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
 * [Lambda in .NET Core Walkthrough](https://docs.aws.amazon.com/lambda/latest/dg/csharp-package-cli.html)
 * [AWS Lambda for .NET Core @ Github](https://github.com/aws/aws-lambda-dotnet)
 * [AWS Extensions for .NET CLI @ Github](https://github.com/aws/aws-extensions-for-dotnet-cli)
