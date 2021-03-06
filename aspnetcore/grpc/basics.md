---
title: 使用 C# 的 gRPC 服务
author: juntaoluo
description: 了解用C#编写 gRPC services 的基本概念。
monikerRange: '>= aspnetcore-3.0'
ms.author: johluo
ms.date: 07/03/2019
uid: grpc/basics
ms.openlocfilehash: 8d99d036fd4b00fc4568e67ea5225dc006dea4b1
ms.sourcegitcommit: 73e255e846e414821b8cc20ffa3aec946735cd4e
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 10/03/2019
ms.locfileid: "71925190"
---
# <a name="grpc-services-with-c"></a>gRPC 服务与 C\#

本文档概述了在中C#编写[gRPC](https://grpc.io/docs/guides/)应用程序所需的概念。 本文介绍的主题适用于基于[C](https://grpc.io/blog/grpc-stacks)和 ASP.NET Core 的 gRPC 应用。

## <a name="proto-file"></a>proto 文件

gRPC 使用协定优先方法进行 API 开发。 默认情况下，协议缓冲区（protobuf）用作接口设计语言（IDL）。 *\*的 proto*文件包含：

* GRPC 服务的定义。
* 客户端和服务器之间发送的消息。

有关 protobuf 文件语法的详细信息，请参阅[官方文档（protobuf）](https://developers.google.com/protocol-buffers/docs/proto3)。

例如，请考虑[gRPC 服务入门](xref:tutorials/grpc/grpc-start)中使用的*fibonacci*文件：

* 定义 `Greeter` 服务。
* `Greeter` 服务定义 `SayHello` 调用。
* `SayHello` 发送 `HelloRequest` 消息并收到 `HelloReply` 消息：

[!code-protobuf[](~/tutorials/grpc/grpc-start/sample/GrpcGreeter/Protos/greet.proto)]

## <a name="add-a-proto-file-to-a-c-app"></a>将一个 proto 文件添加到 C\# 应用

在项目中，将 *\*proto*文件添加到 `<Protobuf>` 项组：

[!code-xml[](~/tutorials/grpc/grpc-start/sample/GrpcGreeter/GrpcGreeter.csproj?highlight=2&range=7-9)]

## <a name="c-tooling-support-for-proto-files"></a>.proto 文件的 C# 工具支持

需要工具包[Grpc](https://www.nuget.org/packages/Grpc.Tools/)才能从C# *\*proto*文件生成资产。 生成的资产（文件）：

* 每次生成项目时都按需生成。
* 不会添加到项目中，也不会签入源控件。
* 是包含在*obj*目录中的生成项目。

服务器项目和客户端项目都需要此包。 `Grpc.AspNetCore` 元包包括对 `Grpc.Tools`的引用。 服务器项目可以使用 Visual Studio 中的包管理器或通过将 `<PackageReference>` 添加到项目文件来添加 `Grpc.AspNetCore`：

[!code-xml[](~/tutorials/grpc/grpc-start/sample/GrpcGreeter/GrpcGreeter.csproj?highlight=1&range=12)]

客户端项目应直接引用 `Grpc.Tools` 使用 gRPC 客户端所需的其他包。 运行时不需要工具包，因此，该依赖项标记为 `PrivateAssets="All"`：

[!code-xml[](~/tutorials/grpc/grpc-start/sample/GrpcGreeterClient/GrpcGreeterClient.csproj?highlight=3&range=9-11)]

## <a name="generated-c-assets"></a>生成C#的资产

工具包将生成表示C#在包含的 *\*proto*文件中定义的消息的类型。

对于服务器端资产，会生成抽象服务基类型。 基类型包含*proto*文件中包含的所有 gRPC 调用的定义。 创建一个派生自此基类型的具体服务实现，并为 gRPC 调用实现逻辑。 对于 `greet.proto`（前面所述的示例），将生成一个包含虚拟 `SayHello` 方法的抽象 `GreeterBase` 类型。 具体的实现 `GreeterService` 重写方法，并实现处理 gRPC 调用的逻辑。

[!code-csharp[](~/tutorials/grpc/grpc-start/sample/GrpcGreeter/Services/GreeterService.cs?name=snippet)]

对于客户端资产，会生成具体的客户端类型。 *Proto*文件中的 gRPC 调用被转换为具体类型上的方法，可以调用该类型。 在 `greet.proto`中，将生成一个具体 `GreeterClient` 类型。 调用 `GreeterClient.SayHelloAsync` 以启动对服务器的 gRPC 调用。

[!code-csharp[](~/tutorials/grpc/grpc-start/sample/GrpcGreeterClient/Program.cs?name=snippet)]

默认情况下，将为 `<Protobuf>` 项组中包含的每个 *\*proto*文件生成服务器和客户端资产。 若要确保服务器项目中仅生成服务器资产，则 `GrpcServices` 属性设置为 `Server`。

[!code-xml[](~/tutorials/grpc/grpc-start/sample/GrpcGreeter/GrpcGreeter.csproj?highlight=2&range=7-9)]

同样，特性设置为在客户端项目中 `Client`。

[!INCLUDE[](~/includes/gRPCazure.md)]

## <a name="additional-resources"></a>其他资源

* <xref:grpc/index>
* <xref:tutorials/grpc/grpc-start>
* <xref:grpc/aspnetcore>
* <xref:grpc/client>
