# How-to upload C++ on an AWS Lambda


This git is meant to explain how to upload a c++ app on a lambda.

You have several ways to upload a c++ app.
1. create a zip file containing the runtime
1. create a docker container containing the program

Here we will see the first case.

To build the c++ app with the runtime, we need to build the program on a x86 machine (64bits), and it's better (even no mandatory) to build it on a Amazon Linux 2 (like a AWS EC2).


# Step 1: Create a EC2 with a Amazon Linux 2 distro, and connect to it

Warning: the SDK needs at least 4Gb of RAM to build. You therefore must run at least a t2.medium instance of EC2. 

Once connected through SSH, go sudo for the rest of the session by typing:
```
sudo -i
```


# Step 2: Setup the environnment by typing:

```
yum -y update
yum -y install make
yum -y install git
yum -y install gcc-c++
yum -y install nano
yum -y install zip
yum -y install clang
yum -y install libcurl-devel
yum -y install cmake3
yum -y install zlib-devel
yum -y install openssl-devel
export CC=gcc
export CXX=g++
```


# Step 3: Install and build the C++ runtime provided by AWS:
(this is mandatory for any C++ app uploaded on a Lambda)

```
cd ~ 
git clone https://github.com/awslabs/aws-lambda-cpp.git

cd aws-lambda-cpp
mkdir build
cd build

cmake3 .. -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF -DCMAKE_INSTALL_PREFIX=~/out

make
make install
```


# Step 4: Install and build the C++ SDK provided by AWS:
(this is mandatory for any C++ app that will have to use one or more aws service, like s3)
(in the example below, only the s3 package will be installed, if you want a full build, just remove the -DBUILD_ONLY=s3 parameter below, but the compilation will take much more time and take much more memory)

```
cd ~
git clone https://github.com/aws/aws-sdk-cpp.git
cd aws-sdk-cpp
./prefetch_crt_dependency.sh

mkdir build
cd build

cmake3 .. -DCMAKE_C_COMPILER=gcc \
 -DBUILD_ONLY=s3 \
 -DBUILD_SHARED_LIBS=OFF \
 -DENABLE_UNITY_BUILD=ON \
 -DCMAKE_BUILD_TYPE=Release \
 -DCMAKE_INSTALL_PREFIX=~/out

make
make install
```




# Step 5: Install and build your program

During this step, you will have to build your own program and to output the executable file in the ~/out directory. Below you will find the two examples provided by AWS, the hello world app and an example using S3 (based on https://aws.amazon.com/blogs/compute/introducing-the-c-lambda-runtime/)

## Step 5.1: install and build the hello world app provided by AWS:

Create a folder for the program:
```
cd ~
mkdir hello-cpp-world
cd hello-cpp-world
```

Write a main.cpp file including
```c
// main.cpp
#include <aws/lambda-runtime/runtime.h>
using namespace aws::lambda_runtime;
invocation_response my_handler(invocation_request const& request)
{
return invocation_response::success("Hello, World!", "application/json");
}

int main()
{
run_handler(my_handler);
return 0; 
}
```
Note: to create a new file, juste type
```
vi main.cpp
```
and then press I (input mode), past the code above, then press ESC, then type :wq

Write a CMakeLists.txt file including
```
cmake_minimum_required(VERSION 3.5)
set(CMAKE_CXX_STANDARD 11)
project(hello LANGUAGES CXX)
find_package(aws-lambda-runtime REQUIRED)
add_executable(hello "main.cpp")
target_link_libraries(hello PUBLIC AWS::aws-lambda-runtime)
aws_lambda_package_target(hello NO_LIBC)
```

Then make and create the endoder zip file to upload onto the lambda
```
mkdir build
cd build

cmake3 .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH=~/out

make
make aws-lambda-package-hello
```

For conveniance purpose, move the hello.zip file to the root directory from which the next AWS CLI command will be performed

```
mv hello.zip ~
cd ~
```



## Step 5.2: install and build the s3 example app provided by AWS:

Create a folder for the program:
```c
cd ~
mkdir cpp-encoder-example
cd cpp-encoder-example
```

Write a main.cpp file including
```cpp
// main.cpp
#include <aws/core/Aws.h>
#include <aws/core/utils/logging/LogLevel.h>
#include <aws/core/utils/logging/ConsoleLogSystem.h>
#include <aws/core/utils/logging/LogMacros.h>
#include <aws/core/utils/json/JsonSerializer.h>
#include <aws/core/utils/HashingUtils.h>
#include <aws/core/platform/Environment.h>
#include <aws/core/client/ClientConfiguration.h>
#include <aws/core/auth/AWSCredentialsProvider.h>
#include <aws/s3/S3Client.h>
#include <aws/s3/model/GetObjectRequest.h>
#include <aws/lambda-runtime/runtime.h>
#include <iostream>
#include <memory>

using namespace aws::lambda_runtime;

std::string download_and_encode_file(
    Aws::S3::S3Client const& client,
    Aws::String const& bucket,
    Aws::String const& key,
    Aws::String& encoded_output);

std::string encode(Aws::String const& filename, Aws::String& output);
char const TAG[] = "LAMBDA_ALLOC";

static invocation_response my_handler(invocation_request const& req, Aws::S3::S3Client const& client)
{
    using namespace Aws::Utils::Json;
    JsonValue json(req.payload);
    if (!json.WasParseSuccessful()) {
        return invocation_response::failure("Failed to parse input JSON", "InvalidJSON");
    }

    auto v = json.View();

    if (!v.ValueExists("s3bucket") || !v.ValueExists("s3key") || !v.GetObject("s3bucket").IsString() ||
        !v.GetObject("s3key").IsString()) {
        return invocation_response::failure("Missing input value s3bucket or s3key", "InvalidJSON");
    }

    auto bucket = v.GetString("s3bucket");
    auto key = v.GetString("s3key");

    AWS_LOGSTREAM_INFO(TAG, "Attempting to download file from s3://" << bucket << "/" << key);

    Aws::String base64_encoded_file;
    auto err = download_and_encode_file(client, bucket, key, base64_encoded_file);
    if (!err.empty()) {
        return invocation_response::failure(err, "DownloadFailure");
    }

    return invocation_response::success(base64_encoded_file, "application/base64");
}

std::function<std::shared_ptr<Aws::Utils::Logging::LogSystemInterface>()> GetConsoleLoggerFactory()
{
    return [] {
        return Aws::MakeShared<Aws::Utils::Logging::ConsoleLogSystem>(
            "console_logger", Aws::Utils::Logging::LogLevel::Trace);
    };
}

int main()
{
    using namespace Aws;
    SDKOptions options;
    options.loggingOptions.logLevel = Aws::Utils::Logging::LogLevel::Trace;
    options.loggingOptions.logger_create_fn = GetConsoleLoggerFactory();
    InitAPI(options);
    {
        Client::ClientConfiguration config;
        config.region = Aws::Environment::GetEnv("AWS_REGION");
        config.caFile = "/etc/pki/tls/certs/ca-bundle.crt";

        auto credentialsProvider = Aws::MakeShared<Aws::Auth::EnvironmentAWSCredentialsProvider>(TAG);
        S3::S3Client client(credentialsProvider, config);
        auto handler_fn = [&client](aws::lambda_runtime::invocation_request const& req) {
            return my_handler(req, client);
        };
        run_handler(handler_fn);
    }
    ShutdownAPI(options);
    return 0;
}

std::string encode(Aws::IOStream& stream, Aws::String& output)
{
    Aws::Vector<unsigned char> bits;
    bits.reserve(stream.tellp());
    stream.seekg(0, stream.beg);

    char streamBuffer[1024 * 4];
    while (stream.good()) {
        stream.read(streamBuffer, sizeof(streamBuffer));
        auto bytesRead = stream.gcount();

        if (bytesRead > 0) {
            bits.insert(bits.end(), (unsigned char*)streamBuffer, (unsigned char*)streamBuffer + bytesRead);
        }
    }
    Aws::Utils::ByteBuffer bb(bits.data(), bits.size());
    output = Aws::Utils::HashingUtils::Base64Encode(bb);
    return {};
}

std::string download_and_encode_file(
    Aws::S3::S3Client const& client,
    Aws::String const& bucket,
    Aws::String const& key,
    Aws::String& encoded_output)
{
    using namespace Aws;

    S3::Model::GetObjectRequest request;
    request.WithBucket(bucket).WithKey(key);

    auto outcome = client.GetObject(request);
    if (outcome.IsSuccess()) {
        AWS_LOGSTREAM_INFO(TAG, "Download completed!");
        auto& s = outcome.GetResult().GetBody();
        return encode(s, encoded_output);
    }
    else {
        AWS_LOGSTREAM_ERROR(TAG, "Failed with error: " << outcome.GetError());
        return outcome.GetError().GetMessage();
    }
}
```

Note: to create a new file, juste type
```
vi main.cpp
```
and then press I (input mode), past the code above, then press ESC, then type :wq


Write a CMakeLists.txt file including
```
cmake_minimum_required(VERSION 3.5)
set(CMAKE_CXX_STANDARD 11)
project(encoder LANGUAGES CXX)

find_package(aws-lambda-runtime REQUIRED)
find_package(AWSSDK COMPONENTS s3)

add_executable(${PROJECT_NAME} "main.cpp")
target_link_libraries(${PROJECT_NAME} PUBLIC
                      AWS::aws-lambda-runtime
                       ${AWSSDK_LINK_LIBRARIES})

aws_lambda_package_target(${PROJECT_NAME})
```

Then make and create the endoder zip file to upload onto the lambda
```
mkdir build
cd build

cmake3 .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH=~/out

make
make aws-lambda-package-encoder
```

For conveniance purpose, move the encoder.zip file to the root directory from which the next AWS CLI command will be performed

```
mv encoder.zip ~
cd ~
```


# Step 6: Deploy on lambda

## Prerequisite: Install and configure AWS CLI
```
cd ~
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Then you can test the installation by typing:
```
aws --version
```

And lastly, just have to configure the aws cli. You need to have created an IAM user.
```
aws configure
```

## Step 6.1: Create the lambda function
This step is to be done only once, if the lambda function is already created, jump into the next step directly.

### Step 6.2.1: Create a trust-policy.json 

```
{
 "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": ["lambda.amazonaws.com"]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Add the strategy "AmazonS3ReadOnlyAccess" in the created role
todo: update the trust-policy.json to add this role automatically

### Step 6.2.2: Create an iam role
```
aws iam create-role \
--role-name lambda-mycppfunction \
--assume-role-policy-document file://trust-policy.json
```

Save the arn.

In the following command, make sure:
- <specify the role arn from the previous step> is replaced with the arn provided by the "aws iam create-role" command done previously
- specify the name of the zip file by changing <NAME_OF_PROG> (can be hello.zip or encode.zip if you want to upload the examples provided by AWS)

```
aws lambda create-function \
--function-name mycppfunction \
--role <specify the role arn from the previous step> \
--runtime provided \
--timeout 15 \
--memory-size 128 \
--handler hello \
--zip-file fileb://<NAME_OF_PROG>.zip
```

## Step 6.3: Update the lambda function 
Once you have created the function, to update it, juste type :
 
```
aws lambda update-function-code \
--function-name mycppfunction \
--zip-file fileb://<NAME_OF_PROG>.zip
```


# Final step: test the function
```
aws lambda invoke --function-name mycppfunction --payload '{"s3bucket": "your_bucket_name", "s3key":"your_file_key" }' base64_image.txt
```


In AWS Console - Lambda, you can run the same lambda with the following payload :
```
{
  "s3bucket": "your_bucket_name",
  "s3key": "your_file_key"
}
```
