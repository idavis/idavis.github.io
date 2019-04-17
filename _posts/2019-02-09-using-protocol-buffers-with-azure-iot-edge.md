---
layout: post
title: "Using Protocol Buffers with Azure IoT Edge"
date: 2019-02-02 6:00
published: false
categories: azure iot iotedge protobuf
---
## Introduction

Azure IoT Edge's Edge Hub communication is a bit low level and put all serialization on you the developer. Rather than moving json and pulling out untyped properties, you can use Protocol Buffers to generate contracts and serialization code. This guide assumes you have gone through the [Linux](https://docs.microsoft.com/en-us/azure/iot-edge/quickstart-linux) or [Windows](https://docs.microsoft.com/en-us/azure/iot-edge/quickstart) IoT Edge Quickstart.

1. [Creating Modules](#creating-modules)


## Creating Modules

Create a new Azure IoT Edge Solution with a Python module called `pingclient` and a C# module called `pingserver` and update the routes:
TODO FIX ROUTES
```json
"$edgeHub": {
  "properties.desired": {
    "schemaVersion": "1.0",
    "routes": {
      "pingclientToPingServer": "FROM /messages/modules/pingclient/outputs/* INTO $upstream",
      "pingserverToIoTHub": "FROM /messages/modules/pingserver/outputs/* INTO $upstream"
    }
  }
}
```

## Creating Contracts

In the root of your solution, create a `contracts.proto` file and add this content:

```
syntax = "proto3";
package protosample;

message PingMetadata {
  string requestId = 1;
  string message = 2;
  string timestamp = 3;
}
```

This says that we are going to create a contract using v3 rather than v2 of the syntax with a message called PingMetadata with some useful properties.

Install the Protocol Buffers compiler for your platform. For Ubuntu 18.04, it is found in the `protobuf-compiler` package in `apt`.

## Generating Python Contracts

Open the `requirements.txt` in the Python project add add a line for `protobuf`. Your requirements file should look like this:

```txt
azure-iothub-device-client~=1.4.3
protobuf==3.6.1
```

In the terminal run  `protoc --python_out=./modules/pingclient/ -I. contracts.proto` assuming your current directory is the edge solution root. You should now see a `contracts_pb2.py` file in your `pingclient` module.

In your `main.py` you can now add `import contracts_pb2` to access the generated templating.


## The Python Application

 For this example, we're going to create a Python application that loops sending messages and awaiting responses serializing at both ends.

```python
```

## Generating C# Contracts

Open `pingserver.csproj` and add the following lines:

```csproj
  <ItemGroup>
    <PackageReference Include="Google.Protobuf" Version="3.6.1" />
    <PackageReference Include="Grpc" Version="1.17.1" />
    <PackageReference Include="Grpc.Tools" Version="1.17.1" PrivateAssets="All" />
    <!-- https://github.com/grpc/grpc/blob/master/src/csharp/BUILD-INTEGRATION.md -->
    <Protobuf Include="../../**/*.proto" OutputDir="%(RelativePath)" CompileOutputs="false" GrpcServices="None" />
  </ItemGroup>
```

If you now run `dotnet restore` and `dotnet build` you will see a `Contracts.cs` file appear in your project.

## Making Syntax Cleaner

We can add a couple of extension methods to make our module code look nice. Create an `Extensions.cs` file with the contents:

```cs
using protosample;
using Google.Protobuf;
using Microsoft.Azure.Devices.Client;

namespace pingserver
{
    internal static class Extensions
    {
        public static PingMetadata ToPingMetadata(this Message message)
        {
            using (var bodyStream = message.GetBodyStream())
            {
                return PingMetadata.Parser.ParseFrom(bodyStream);
            }
        }

        public static Message ToMessage(this PingMetadata message)
        {
            return new Message(message.ToByteArray());
        }
    }
}
```

## Summary

