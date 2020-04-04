# aws-lambda-swift-sprinter

[![Swift 5](https://img.shields.io/badge/Swift-5.0-blue.svg)](https://swift.org/download/) [![Swift 5.1.5](https://img.shields.io/badge/Swift-5.1.5-blue.svg)](https://swift.org/download/) [![Swift 5.2.1](https://img.shields.io/badge/Swift-5.2.1-blue.svg)](https://swift.org/download/)  ![](https://img.shields.io/badge/version-1.0.0-green) ![](https://travis-ci.com/swift-sprinter/aws-lambda-swift-sprinter.svg?branch=master) ![](https://img.shields.io/badge/@AWS-Open-orange.svg)

![](./images/aws-lambda-swift-sprinter.png)

The goal of this project is to provide an environment to build an [AWS Lambda Custom Runtime](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html) for the **Swift** programming language and provide the required support to run [swift-nio 2.0](https://github.com/apple/swift-nio).

The support of **swift-nio 2.0** is crucial to allow HTTPS requests inside the Swift Lambda.

The project helps building a Swift Lambda based on the framework [LambdaSwiftSprinter](https://github.com/swift-sprinter/aws-lambda-swift-sprinter-core).

The project contains also some Examples:

- [HelloWorld](https://github.com/swift-sprinter/aws-lambda-swift-sprinter/blob/master/Examples/HelloWorld): A basic Lambda Swift example
- [HTTPSRequest](https://github.com/swift-sprinter/aws-lambda-swift-sprinter/blob/master/Examples/HTTPSRequest): A basic example showing how to perform an HTTPS request from the Swift Lambda using the [LambdaSwiftSprinterNioPlugin](https://github.com/swift-sprinter/aws-lambda-swift-sprinter-nio-plugin)
- [S3Test](https://github.com/swift-sprinter/aws-lambda-swift-sprinter/blob/master/Examples/S3Test): A basic example showing how to access an S3 bucket from the Swift Lambda using [https://github.com/swift-aws/aws-sdk-swift](https://github.com/swift-aws/aws-sdk-swift).
- [RedisDemo](https://github.com/swift-sprinter/aws-lambda-swift-sprinter/blob/master/Examples/RedisDemo): A basic example showing how to connect to Redis from the Swift Lambda using the [RediStack](https://gitlab.com/mordil/swift-redi-stack.git) client.
- [PostgreSQLDemo](https://github.com/swift-sprinter/aws-lambda-swift-sprinter/tree/master/Examples/PostgreSQLDemo): A basic example showing how to connect to Redis from the Swift Lambda using the [PostgresNIO](https://github.com/vapor/postgres-nio.git) client.

# Introduction

The AWS Lambdas run on [Amazon Linux](https://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html). Unfortunately, there is no support at the moment for Swift on Amazon Linux. This means that the Lambda cannot be built using `swift build` inside the Amazon Linux image.

The work-around to build swift on Amazon Linux is achieved by:
 - building the code on the [official Docker Swift](https://hub.docker.com/_/swift/)
 - extracting the build and all the runtime's shared libraries
 - packaging the artefacts and use them as [AWS Lambda Custom Runtime](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html)

The artefacts required to run the Swift Lambda are split in two parts:
- [Lambda Layer](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html): A layer is a ZIP archive that contains libraries, a custom runtime, all the shared libraries required to run the Swift lambda.
- Lambda code depends on the [LambdaSwiftSprinter](https://github.com/swift-sprinter/aws-lambda-swift-sprinter-core) framework: A zip package containing the Swift Lambda main executable and third-party libraries.

# Lambda Development Workflow

#### 1) Requirements
- Install Docker from [here](https://docs.docker.com/install/)
- Clone this repository. From the command line type:

```console
git clone https://github.com/swift-sprinter/aws-lambda-swift-sprinter
cd aws-lambda-swift-sprinter
```
- Ensure you can run `make`:

```console
make --version
```

the `Makefile` was developed with this version:
```
GNU Make 3.81
Copyright (C) 2006  Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE.

This program built for i386-apple-darwin11.3.0
```

#### 2) Prepare a custom docker image

The docker image [`swift:latest`](https://hub.docker.com/_/swift) contains the latest release of Swift. This image does not contain the libraries `libssl-dev` and `libicu-dev` required to build `swift-nio`.

The `Dockerfile` contains the recipe to build a custom docker image based on the `swift:latest` with the addition of the `swift-nio` requirements.

To build the image use the command:
```console
make docker_build
```
This will build and tag a local docker image called `nio-swift:latest`.

It's possible to check it by using the following command:

```console
docker image ls
```

```
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nio-swift           latest              b2b9d0b2a68d        8 days ago          1.45GB
swift               latest              1e9a2a744b48        13 days ago         1.35GB
```

#### 3) Build the lambda layer

To prepare the AWS Lambda layer containing all the **swift runtime libraries** and the **bootstrap** file:

```console
make package_layer
```

The output of this command is a ZIP file called `swift-lambda-runtime.zip` under the folder `.build`

#### 4) Write the lambda code

Here a basic example of the Lambda code:
```swift
import Foundation
import LambdaSwiftSprinter

struct Event: Codable {
    let name: String
}

struct Response: Codable {
    let message: String
}

let syncLambda: SyncCodableLambda<Event, Response> = { (event, context) throws -> Response in
    let message = "Hello World! Hello \(event.name)!"
    return Response(message: message)
}

//...

do {
    let sprinter = try SprinterCURL()
    sprinter.register(handler: "helloWorld", lambda: syncLambda)
    try sprinter.run()
} catch {
    log(String(describing: error))
}
```

More details on how to code a Swift Lambda are documented under the Examples folder:

- [HelloWorld](https://github.com/swift-sprinter/aws-lambda-swift-sprinter/blob/master/Examples/HelloWorld)
- [HTTPSRequest](https://github.com/swift-sprinter/aws-lambda-swift-sprinter/blob/master/Examples/HTTPSRequest)
- [S3Test](https://github.com/swift-sprinter/aws-lambda-swift-sprinter/blob/master/Examples/S3Test)

Refer to the [LambdaSwiftSprinter framework documentation](https://github.com/swift-sprinter/aws-lambda-swift-sprinter-core) to know more.

#### 5) Build the lambda

By default, the `Makefile` will build the `HelloWorld` example contained in this repository.

The following command will build and zip the Lambda:

```
make package_lambda
```
The output of this command is a ZIP file called `lambda.zip` under the folder `.build`.

# Lambda Configuration

## Parametrized calls / Automation

Can pass values to your `make` command either before or after the call like below. This will fit better in a script per example to set up continuous integration in your project.

### Before `make`

```
SWIFT_EXECUTABLE=HTTPSRequest \
SWIFT_PROJECT_PATH=Examples/HTTPSRequest \
LAMBDA_FUNCTION_NAME=HTTPSRequest \
LAMBDA_HANDLER=${SWIFT_EXECUTABLE}.getHttps \
    make invoke_lambda
```

### After `make`

```
make invoke_lambda \
    SWIFT_EXECUTABLE=HTTPSRequest \
    SWIFT_PROJECT_PATH=Examples/HTTPSRequest \
    LAMBDA_FUNCTION_NAME=HTTPSRequest \
    LAMBDA_HANDLER=HTTPSRequest.getHttps
```

### Parameters you can pass

| Key | Usage | Default |
| --- | --- | --- |
| AWS_PROFILE | An AWS AIM profile you create to authenticate to your account. | default |
| IAM_ROLE_NAME | The execution role created that will be assumed by the Lambda. | lambda_sprinter_basic_execution |
| AWS_BUCKET | The AWS S3 bucket where the layer and lambdas zip files get uploaded. | aws-lambda-swift-sprinter |
| SWIFT_VERSION | Version of Swift used / Matches Dockerfile location too from `docker/` folder. | 5.2.1 |
| LAYER_VERSION | Version of the Swift layer that will be created and uploaded for the Lambda to run on. | 5-2-1 |
| SWIFT_EXECUTABLE | Name of the binary file. | HelloWorld |
| SWIFT_PROJECT_PATH | Path to your Swift project. | Examples/HelloWorld |
| LAMBDA_FUNCTION_NAME | Display name of your Lambda in AWS. | HelloWorld |
| LAMBDA_HANDLER | Name of your lambda handler function. If you declare it using `sprinter.register(handler: "FUNCTION_NAME", lambda: syncLambda)` you should declare it as `<SWIFT_EXECUTABLE>.<FUNCTION_NAME>`. | $(SWIFT_EXECUTABLE).helloWorld |

## Manual change

You can also edit the `Makefile` to build a different Example by commenting the following lines and uncommenting the line related to the example you want to build.
```
...

SWIFT_EXECUTABLE?=HelloWorld
SWIFT_PROJECT_PATH?=Examples/HelloWorld
LAMBDA_FUNCTION_NAME?=HelloWorld
LAMBDA_HANDLER?=$(SWIFT_EXECUTABLE).helloWorld

...
```

# Lambda Deployment Workflow

The following tutorial describes how to deploy the lambda in your AWS account from the command line using [AWS Command Line Interface](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html) through the `Makefile`.

The goal of the deployment is to create and configure a lambda with `lambda.zip` and the layer with `swift-lambda-runtime.zip`.

Steps:
- Create a Lambda layer with the **swift-lambda-runtime.zip** 
- Create the lambda with the **lambda.zip** code and the correct configuration
    - Name
    - Runtime
    - Handler
    - Execution role
    - Function code
    - Test event

There are many ways to achieve a lambda deployment (AWS Console, SAM, CloudFormation ...), please refer to the latest [AWS Lambda documentation](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) to know more.

## Requirements

- an AWS account for test purpose.
- aws cli: Install the aws cli. Here the [instructions](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html).

- If you want to deploy your lambdas and layers using S3 you need to make sure the bucket in the Makefile already exists. If it doesn't you can create it using the command `make create_s3_bucket` which will use the value of the variable `AWS_BUCKET` as a name.

- if your AWS account it doesn't have admin priviledges:
    - Review(*) the policy contained in the file **LambdaDeployerPolicyExample.json**
    - Attach the policy to your user. [Here](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_manage-attach-detach.html) is the official documentation.

(*) Note:
It would be better to restrict the policy to the function and layer you want to create.
It's suggested to test the following scripts before using in production.

#### 1) Upload the Lambda layer
Create a new lambda layer using the `swift-lambda-runtime.zip` file. This step is required once, however you are free to run it if you need to update your layer.

##### Upload the Lambda using S3

```console
make upload_lambda_layer_with_s3
```

Datetime based versions are created and uploaded to S3 every time your version is created.

##### Upload the Lambda directly

```console
make upload_lambda_layer
```

### 2) Create the Lambda

You can create a new lambda which might take a few minutes using one of the options below:

##### Create the Lambda using S3

```console
make create_lambda_with_s3
```

Datetime based versions are created and uploaded to S3 every time your version is created.

##### Create the Lambda directly

```console
make create_lambda
```

The lambda is created with the following parameters:

- function-name: `$(LAMBDA_FUNCTION_NAME)`
    - HelloWorld
- runtime: provided
- handler: `$(LAMBDA_HANDLER)`
    - HelloWorld.helloWorld
- role: `"$(IAM_ROLE_ARN)"`
    - a new role will be created with the name: `lambda_sprinter_basic_execution`
- zip-file: $(LAMBDA_BUILD_PATH)/$(LAMBDA_ZIP) --> `./build/lambda/lambda.zip`
- layers: `$(LAMBDA_LAYER_ARN)`
    - it will use the arn contained in the file generated by the `make upload_lambda_layer`

This step is required once, if you need to update the lambda use the step 5.

### 4) Invoke the Lambda
Now the lambda function is ready for testing. The following command invokes the lambda with using the file **event.json** contained in the project folder.

```console
make invoke_lambda
```

The output:
```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```
```
Result:
```
```json
{"message":"Hello World! Hello Swift-Sprinter!"}
```
Note:

The lambda invocation may require some policy to access other AWS Resources. Check the `S3Test` example to know more.

### 5) Update the Lambda (optional)

If needed, you will also be able to update your Lambda using one of the commands below:

##### Update the Lambda using S3

```console
make update_lambda_with_s3
```

##### Update the Lambda directly

```console
make update_lambda
```

### 6) Nuclear option

Sometimes you want to go back to a clean slate and we have a command for that which will rely on the parameters you use.

```console
make nuke
```

That command will clean your local build folders then delete lambdas and layers created based off the configuration you use.

### 7) Delete S3 bucket

You can delete the S3 bucket where you hold your lambdas and layers with the command below:

```console
make delete_s3_bucket
```

# Update the lambda layer

The docker image could require security updates or could require updates.

To patch the lambda with a new layer:

1. Rebuild the docker image
```
make docker_build
```
2. package the layer
```
make package_layer
```
3. upload the layer to AWS
```
make upload_lambda_layer
```
4. update the lambda
```
make update_lambda
```

# Local development

Development and test can be simplified by using a sandboxed local environment replicating of the AWS Lambda by using the [lambci/lambda:provided](https://github.com/lambci/docker-lambda) Docker image.

The local sandboxed Lambda can be launched just to test the Lambda once or more than once.

To run the sandboxed Lambda environment it's required to share the files contained in the layer, the boostrap and the files required by the build with the ```lambci/lambda:provided``` Docker image.

```
docker run --rm \
-v "$(PWD)/$(LOCAL_LAMBDA_PATH)":/var/task:ro,delegated \
-v "$(PWD)/bootstrap":/opt/bootstrap:ro,delegated \
-v "$(PWD)/$(SHARED_LIBS_FOLDER)":/opt/swift-shared-libs:ro,delegated \
lambci/lambda:provided $(LAMBDA_HANDLER) $(LOCAL_LAMBDA_EVENT)
```

- ```$LOCAL_LAMBDA_PATH```: the path containing the build
- ```$SHARED_LIBS_FOLDER```: the path containing the extracted swift runtime libraries
- ```$LAMBDA_HANDLER```: the lambda handler
- ```$LOCAL_LAMBDA_EVENT```: the event JSON string


To run the Lambda locally follow the [Lambda development workflow](https://github.com/swift-sprinter/aws-lambda-swift-sprinter#lambda-development-workflow) to the step 4:

1) Requirements: Clone the repository and install Docker
2) Prepare a custom docker image: ```make docker_build```
3) Build the lambda layer: ```make package_layer```
4) Write the lambda code

To simplify the local execution swift-sprinter added some useful commands:

## Lambda Local

### Build lambda locally

Build the lambda and copy the artefacts under ```$LOCAL_LAMBDA_PATH```

```make build_lambda_local```

### Invoke lambda local once

Invoke the Lambda locally once with Docker and LambCI on port ```9001```

```make invoke_lambda_local_once```

### Start lambda local environment

Start the local environment with Docker and LambCI on port ```9001```

```make start_lambda_local_env```

Keep it running and open a new terminal window to invoke the lambda.

Use ```ctrl^C``` to stop it.

### Invoke the lambda locally

Invoke the lambda locally using the endpoint ```http://localhost:9001``` with the aws cli.

```make invoke_lambda_local```

## Use of Docker Compose

To test lambda locally with a more complex environment it's possible to use Docker Compose.

All the examples in the repository have  ```docker-compose.yml``` file to run the example locally listening on the port ```9001```.

### Start the Docker Compose 

Start the docker-compose test environment

```make start_docker_compose_env```

### Invoke the lambda locally

Invoke the lambda locally using the endpoint ```http://localhost:9001``` with the aws cli.

```make invoke_lambda_local```

### Stop the Docker Compose

Stop the docker-compose test environment

```make stop_docker_compose_env```

# Manual deployment from AWS Console

### Requirements:
- Use MacOS or Linux
- Install [Docker](https://docs.docker.com/docker-for-mac/install/)
- Clone the repo `git clone https://github.com/swift-sprinter/aws-lambda-swift-sprinter.git`

### Build:
- From the command line run `make docker_build`
- From the command line run `make package_layer`
- From the command line run `make package_lambda`

### Configure:
- Go to `AWS Lambda -> Layers` in AWS Console and create a new layer from scratch
- Enter layer name "swift-lambda-runtime-5-1-5"
- Upload the zip file `build/swift-lambda-runtime-5-1-5.zip`
- Leave "Compatible runtimes" empty.
- Click "Create"
- Copy the `arn` from the created layer, it's required to set up the lambda.
- Go to `AWS Lambda` in AWS Console and create a new function from scratch
- Enter function name "benchmark-swift-hello" and select "Provide your own bootstrap" as runtime
- Choose the execution role created above
- Upload the zip file created with function code, it can be found under `build/lambda.zip`
- Input "HelloWorld.helloWorld" in "Handler" (`Executable.Handler`)
- Click "Layers"
- Click "Add Layer"
- Click "Provide a layer version"
- Add the `arn` of the layer you have uploaded previously.
- Click "Add"
- Click "Save"

### Test:
- Test the function by clicking `Test` in the top right corner and add the following event:
```
{
    "name": "Swift-Sprinter"
}
```
- Give a name to the event, save and then test it.

# Contributions

Contributions are more than welcome! Follow [this guide](https://github.com/swift-sprinter/aws-lambda-swift-sprinter/blob/master/CONTRIBUTING.md) to contribute.

# Acknowledgements

This project has been inspired by the amazing work of the following people:

- Matthew Burke, Capital One : https://medium.com/capital-one-tech/serverless-computing-with-swift-f515ff052919

- Justin Sanders : https://medium.com/@gigq/using-swift-in-aws-lambda-6e2a67a27e03

- Claus Höfele : https://medium.com/@claushoefele/serverless-swift-2e8dce589b68

- Kohki Miki, Cookpad : https://github.com/giginet/aws-lambda-swift-runtime

- Toni Sutter : https://github.com/tonisuter/aws-lambda-swift

- Sébastien Stormacq :  https://github.com/sebsto/swift-custom-runtime-lambda

A special thanks to [BJSS](https://www.bjss.com) to sustain me in delivering this project.

This project has been awarded AWS Open Source Promotional Credits for 2020. [@AWSOpen](https://twitter.com/awsopen)