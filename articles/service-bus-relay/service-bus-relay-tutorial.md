---
title: Azure Service Bus WCF Relay tutorial | Microsoft Docs
description: Build a client and service application using WCF Relay.
services: service-bus-relay
documentationcenter: na
author: spelluru
manager: timlt
editor: ''

ms.assetid: 53dfd236-97f1-4778-b376-be91aa14b842
ms.service: service-bus-relay
ms.devlang: na
ms.topic: article
ms.tgt_pltfrm: na
ms.workload: na
ms.date: 11/02/2017
ms.author: spelluru

---
# Azure WCF Relay tutorial

This tutorial describes how to build a simple WCF Relay client application and service using Azure Relay. For a similar tutorial that uses [Service Bus messaging](../service-bus-messaging/service-bus-messaging-overview.md), see [Get started with Service Bus queues](../service-bus-messaging/service-bus-dotnet-get-started-with-queues.md).

Working through this tutorial gives you an understanding of the steps that are required to create a WCF Relay client and service application. Like their original WCF counterparts, a service is a construct that exposes one or more endpoints, each of which exposes one or more service operations. The endpoint of a service specifies an address where the service can be found, a binding that contains the information that a client must communicate with the service, and a contract that defines the functionality provided by the service to its clients. The main difference between WCF and WCF Relay is that the endpoint is exposed in the cloud instead of locally on your computer.

After you work through the sequence of topics in this tutorial, you will have a running service, and a client that can invoke the operations of the service. The first topic describes how to set up an account. The next steps describe how to define a service that uses a contract, how to implement the service, and how to configure the service in code. They also describe how to host and run the service. The service that is created is self-hosted and the client and service run on the same computer. You can configure the service by using either code or a configuration file.

The final three steps describe how to create a client application, configure the client application, and create and use a client that can access the functionality of the host.

## Prerequisites

To complete this tutorial, you'll need the following:

* [Microsoft Visual Studio 2015 or higher](https://visualstudio.com). This tutorial uses Visual Studio 2017.
* An active Azure account. If you don't have one, you can create a free account in just a couple of minutes. For details, see [Azure Free Trial](https://azure.microsoft.com/free/).

## Create a service namespace

The first step is to create a namespace, and to obtain a [Shared Access Signature (SAS)](../service-bus-messaging/service-bus-sas.md) key. A namespace provides an application boundary for each application exposed through the relay service. A SAS key is automatically generated by the system when a service namespace is created. The combination of service namespace and SAS key provides the credentials for Azure to authenticate access to an application. Follow the [instructions here](relay-create-namespace-portal.md) to create a Relay namespace.

## Define a WCF service contract

The service contract specifies what operations (the web service terminology for methods or functions) the service supports. Contracts are created by defining a C++, C#, or Visual Basic interface. Each method in the interface corresponds to a specific service operation. Each interface must have the [ServiceContractAttribute](https://msdn.microsoft.com/library/system.servicemodel.servicecontractattribute.aspx) attribute applied to it, and each operation must have the [OperationContractAttribute](https://msdn.microsoft.com/library/system.servicemodel.operationcontractattribute.aspx) attribute applied to it. If a method in an interface that has the [ServiceContractAttribute](https://msdn.microsoft.com/library/system.servicemodel.servicecontractattribute.aspx) attribute does not have the [OperationContractAttribute](https://msdn.microsoft.com/library/system.servicemodel.operationcontractattribute.aspx) attribute, that method is not exposed. The code for these tasks is provided in the example following the procedure. For a larger discussion of contracts and services, see [Designing and Implementing Services](https://msdn.microsoft.com/library/ms729746.aspx) in the WCF documentation.

### Create a relay contract with an interface

1. Open Visual Studio as an administrator by right-clicking the program in the **Start** menu and selecting **Run as administrator**.
2. Create a new console application project. Click the **File** menu and select **New**, then click **Project**. In the **New Project** dialog, click **Visual C#** (if **Visual C#** does not appear, look under **Other Languages**). Click the **Console App (.NET Framework)** template, and name it **EchoService**. Click **OK** to create the project.

    ![][2]

3. Install the Service Bus NuGet package. This package automatically adds references to the Service Bus libraries, as well as the WCF **System.ServiceModel**. [System.ServiceModel](https://msdn.microsoft.com/library/system.servicemodel.aspx) is the namespace that enables you to programmatically access the basic features of WCF. Service Bus uses many of the objects and attributes of WCF to define service contracts.

    In Solution Explorer, right-click the project, and then click **Manage NuGet Packages...**. Click the **Browse** tab, then search for **WindowsAzure.ServiceBus**. Ensure that the project name is selected in the **Version(s)** box. Click **Install**, and accept the terms of use.

    ![][3]
4. In Solution Explorer, double-click the Program.cs file to open it in the editor, if it is not already open.
5. Add the following using statements at the top of the file:

    ```csharp
    using System.ServiceModel;
    using Microsoft.ServiceBus;
    ```
6. Change the namespace name from its default name of **EchoService** to **Microsoft.ServiceBus.Samples**.

   > [!IMPORTANT]
   > This tutorial uses the C# namespace **Microsoft.ServiceBus.Samples**, which is the namespace of the contract-based managed type that is used in the configuration file in the [Configure the WCF client](#configure-the-wcf-client) step. You can specify any namespace you want when you build this sample; however, the tutorial will not work unless you then modify the namespaces of the contract and service accordingly, in the application configuration file. The namespace specified in the App.config file must be the same as the namespace specified in your C# files.
   >
   >
7. Directly after the `Microsoft.ServiceBus.Samples` namespace declaration, but within the namespace, define a new interface named `IEchoContract` and apply the `ServiceContractAttribute` attribute to the interface with a namespace value of `http://samples.microsoft.com/ServiceModel/Relay/`. The namespace value differs from the namespace that you use throughout the scope of your code. Instead, the namespace value is used as a unique identifier for this contract. Specifying the namespace explicitly prevents the default namespace value from being added to the contract name. Paste the following code after the namespace declaration:

    ```csharp
    [ServiceContract(Name = "IEchoContract", Namespace = "http://samples.microsoft.com/ServiceModel/Relay/")]
    public interface IEchoContract
    {
    }
    ```

   > [!NOTE]
   > Typically, the service contract namespace contains a naming scheme that includes version information. Including version information in the service contract namespace enables services to isolate major changes by defining a new service contract with a new namespace and exposing it on a new endpoint. In this manner, clients can continue to use the old service contract without having to be updated. Version information can consist of a date or a build number. For more information, see [Service Versioning](https://go.microsoft.com/fwlink/?LinkID=180498). For the purposes of this tutorial, the naming scheme of the service contract namespace does not contain version information.
   >
   >
8. Within the `IEchoContract` interface, declare a method for the single operation the `IEchoContract` contract exposes in the interface and apply the `OperationContractAttribute` attribute to the method that you want to expose as part of the public WCF Relay contract, as follows:

    ```csharp
    [OperationContract]
    string Echo(string text);
    ```
9. Directly after the `IEchoContract` interface definition, declare a channel that inherits from both `IEchoContract` and also to the `IClientChannel` interface, as shown here:

    ```csharp
    public interface IEchoChannel : IEchoContract, IClientChannel { }
    ```

    A channel is the WCF object through which the host and client pass information to each other. Later, you will write code against the channel to echo information between the two applications.
10. From the **Build** menu, click **Build Solution** or press **Ctrl+Shift+B** to confirm the accuracy of your work so far.

### Example

The following code shows a basic interface that defines a WCF Relay contract.

```csharp
using System;
using System.ServiceModel;

namespace Microsoft.ServiceBus.Samples
{
    [ServiceContract(Name = "IEchoContract", Namespace = "http://samples.microsoft.com/ServiceModel/Relay/")]
    public interface IEchoContract
    {
        [OperationContract]
        String Echo(string text);
    }

    public interface IEchoChannel : IEchoContract, IClientChannel { }

    class Program
    {
        static void Main(string[] args)
        {
        }
    }
}
```

Now that the interface is created, you can implement the interface.

## Implement the WCF contract

Creating an Azure relay requires that you first create the contract, which is defined by using an interface. For more information about creating the interface, see the previous step. The next step is to implement the interface. This involves creating a class named `EchoService` that implements the user-defined `IEchoContract` interface. After you implement the interface, you then configure the interface using an App.config configuration file. The configuration file contains necessary information for the application, such as the name of the service, the name of the contract, and the type of protocol that is used to communicate with the relay service. The code used for these tasks is provided in the example following the procedure. For a more general discussion about how to implement a service contract, see [Implementing Service Contracts](https://msdn.microsoft.com/library/ms733764.aspx) in the WCF documentation.

1. Create a new class named `EchoService` directly after the definition of the `IEchoContract` interface. The `EchoService` class implements the `IEchoContract` interface.

    ```csharp
    class EchoService : IEchoContract
    {
    }
    ```

    Similar to other interface implementations, you can implement the definition in a different file. However, for this tutorial, the implementation is located in the same file as the interface definition and the `Main` method.
2. Apply the [ServiceBehaviorAttribute](https://msdn.microsoft.com/library/system.servicemodel.servicebehaviorattribute.aspx) attribute to the `IEchoContract` interface. The attribute specifies the service name and namespace. After doing so, the `EchoService` class appears as follows:

    ```csharp
    [ServiceBehavior(Name = "EchoService", Namespace = "http://samples.microsoft.com/ServiceModel/Relay/")]
    class EchoService : IEchoContract
    {
    }
    ```
3. Implement the `Echo` method defined in the `IEchoContract` interface in the `EchoService` class.

    ```csharp
    public string Echo(string text)
    {
        Console.WriteLine("Echoing: {0}", text);
        return text;
    }
    ```
4. Click **Build**, then click **Build Solution** to confirm the accuracy of your work.

### Define the configuration for the service host

1. The configuration file is very similar to a WCF configuration file. It includes the service name, endpoint (that is, the location that Azure Relay exposes for clients and hosts to communicate with each other), and the binding (the type of protocol that is used to communicate). The main difference is that this configured service endpoint refers to a [NetTcpRelayBinding](/dotnet/api/microsoft.servicebus.nettcprelaybinding) binding, which is not part of the .NET Framework. [NetTcpRelayBinding](/dotnet/api/microsoft.servicebus.nettcprelaybinding) is one of the bindings defined by the service.
2. In **Solution Explorer**, double-click the App.config file to open it in the Visual Studio editor.
3. In the `<appSettings>` element, replace the placeholders with the name of your service namespace, and the SAS key that you copied in an earlier step.
4. Within the `<system.serviceModel>` tags, add a `<services>` element. You can define multiple relay applications in a single configuration file. However, this tutorial defines only one.

    ```xml
    <?xmlversion="1.0"encoding="utf-8"?>
    <configuration>
      <system.serviceModel>
        <services>

        </services>
      </system.serviceModel>
    </configuration>
    ```
5. Within the `<services>` element, add a `<service>` element to define the name of the service.

    ```xml
    <service name="Microsoft.ServiceBus.Samples.EchoService">
    </service>
    ```
6. Within the `<service>` element, define the location of the endpoint contract, and also the type of binding for the endpoint.

    ```xml
    <endpoint contract="Microsoft.ServiceBus.Samples.IEchoContract" binding="netTcpRelayBinding"/>
    ```

    The endpoint defines where the client will look for the host application. Later, the tutorial uses this step to create a URI that fully exposes the host through Azure Relay. The binding declares that we are using TCP as the protocol to communicate with the relay service.
7. From the **Build** menu, click **Build Solution** to confirm the accuracy of your work.

### Example

The following code shows the implementation of the service contract.

```csharp
[ServiceBehavior(Name = "EchoService", Namespace = "http://samples.microsoft.com/ServiceModel/Relay/")]

    class EchoService : IEchoContract
    {
        public string Echo(string text)
        {
            Console.WriteLine("Echoing: {0}", text);
            return text;
        }
    }
```

The following code shows the basic format of the App.config file associated with the service host.

```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <system.serviceModel>
    <services>
      <service name="Microsoft.ServiceBus.Samples.EchoService">
        <endpoint contract="Microsoft.ServiceBus.Samples.IEchoContract" binding="netTcpRelayBinding" />
      </service>
    </services>
    <extensions>
      <bindingExtensions>
        <add name="netTcpRelayBinding"
                    type="Microsoft.ServiceBus.Configuration.NetTcpRelayBindingCollectionElement, Microsoft.ServiceBus, Culture=neutral, PublicKeyToken=31bf3856ad364e35"/>
      </bindingExtensions>
    </extensions>
  </system.serviceModel>
</configuration>
```

## Host and run a basic web service to register with the relay service

This step describes how to run an Azure Relay service.

### Create the relay credentials

1. In `Main()`, create two variables in which to store the namespace and the SAS key that are read from the console window.

    ```csharp
    Console.Write("Your Service Namespace: ");
    string serviceNamespace = Console.ReadLine();
    Console.Write("Your SAS key: ");
    string sasKey = Console.ReadLine();
    ```

    The SAS key will be used later to access your project. The namespace is passed as a parameter to `CreateServiceUri` to create a service URI.
2. Using a [TransportClientEndpointBehavior](/dotnet/api/microsoft.servicebus.transportclientendpointbehavior) object, declare that you will be using a SAS key as the credential type. Add the following code directly after the code added in the last step.

    ```csharp
    TransportClientEndpointBehavior sasCredential = new TransportClientEndpointBehavior();
    sasCredential.TokenProvider = TokenProvider.CreateSharedAccessSignatureTokenProvider("RootManageSharedAccessKey", sasKey);
    ```

### Create a base address for the service

After the code you added in the last step, create a `Uri` instance for the base address of the service. This URI specifies the Service Bus scheme, the namespace, and the path of the service interface.

```csharp
Uri address = ServiceBusEnvironment.CreateServiceUri("sb", serviceNamespace, "EchoService");
```

"sb" is an abbreviation for the Service Bus scheme, and indicates that we are using TCP as the protocol. This was also previously indicated in the configuration file, when [NetTcpRelayBinding](https://msdn.microsoft.com/library/microsoft.servicebus.nettcprelaybinding.aspx) was specified as the binding.

For this tutorial, the URI is `sb://putServiceNamespaceHere.windows.net/EchoService`.

### Create and configure the service host

1. Set the connectivity mode to `AutoDetect`.

    ```csharp
    ServiceBusEnvironment.SystemConnectivity.Mode = ConnectivityMode.AutoDetect;
    ```

    The connectivity mode describes the protocol the service uses to communicate with the relay service; either HTTP or TCP. Using the default setting `AutoDetect`, the service attempts to connect to Azure Relay over TCP if it is available, and HTTP if TCP is not available. Note that this differs from the protocol the service specifies for client communication. That protocol is determined by the binding used. For example, a service can use the [BasicHttpRelayBinding](https://msdn.microsoft.com/library/microsoft.servicebus.basichttprelaybinding.aspx) binding, which specifies that its endpoint communicates with clients over HTTP. That same service could specify **ConnectivityMode.AutoDetect** so that the service communicates with Azure Relay over TCP.
2. Create the service host, using the URI created earlier in this section.

    ```csharp
    ServiceHost host = new ServiceHost(typeof(EchoService), address);
    ```

    The service host is the WCF object that instantiates the service. Here, you pass it the type of service you want to create (an `EchoService` type), and also to the address at which you want to expose the service.
3. At the top of the Program.cs file, add references to [System.ServiceModel.Description](https://msdn.microsoft.com/library/system.servicemodel.description.aspx) and [Microsoft.ServiceBus.Description](/dotnet/api/microsoft.servicebus.description).

    ```csharp
    using System.ServiceModel.Description;
    using Microsoft.ServiceBus.Description;
    ```
4. Back in `Main()`, configure the endpoint to enable public access.

    ```csharp
    IEndpointBehavior serviceRegistrySettings = new ServiceRegistrySettings(DiscoveryType.Public);
    ```

    This step informs the relay service that your application can be found publicly by examining the ATOM feed for your project. If you set **DiscoveryType** to **private**, a client would still be able to access the service. However, the service would not appear when it searches the Relay namespace. Instead, the client would have to know the endpoint path beforehand.
5. Apply the service credentials to the service endpoints defined in the App.config file:

    ```csharp
    foreach (ServiceEndpoint endpoint in host.Description.Endpoints)
    {
        endpoint.Behaviors.Add(serviceRegistrySettings);
        endpoint.Behaviors.Add(sasCredential);
    }
    ```

    As stated in the previous step, you could have declared multiple services and endpoints in the configuration file. If you had, this code would traverse the configuration file and search for every endpoint to which it should apply your credentials. However, for this tutorial, the configuration file has only one endpoint.

### Open the service host

1. Open the service.

    ```csharp
    host.Open();
    ```
2. Inform the user that the service is running, and explain how to shut down the service.

    ```csharp
    Console.WriteLine("Service address: " + address);
    Console.WriteLine("Press [Enter] to exit");
    Console.ReadLine();
    ```
3. When finished, close the service host.

    ```csharp
    host.Close();
    ```
4. Press **Ctrl+Shift+B** to build the project.

### Example

Your completed service code should appear as follows. The code includes the service contract and implementation from previous steps in the tutorial, and hosts the service in a console application.

```csharp
using System;
using System.ServiceModel;
using System.ServiceModel.Description;
using Microsoft.ServiceBus;
using Microsoft.ServiceBus.Description;

namespace Microsoft.ServiceBus.Samples
{
    [ServiceContract(Name = "IEchoContract", Namespace = "http://samples.microsoft.com/ServiceModel/Relay/")]
    public interface IEchoContract
    {
        [OperationContract]
        String Echo(string text);
    }

    public interface IEchoChannel : IEchoContract, IClientChannel { };

    [ServiceBehavior(Name = "EchoService", Namespace = "http://samples.microsoft.com/ServiceModel/Relay/")]
    class EchoService : IEchoContract
    {
        public string Echo(string text)
        {
            Console.WriteLine("Echoing: {0}", text);
            return text;
        }
    }

    class Program
    {
        static void Main(string[] args)
        {

            ServiceBusEnvironment.SystemConnectivity.Mode = ConnectivityMode.AutoDetect;         

            Console.Write("Your Service Namespace: ");
            string serviceNamespace = Console.ReadLine();
            Console.Write("Your SAS key: ");
            string sasKey = Console.ReadLine();

           // Create the credentials object for the endpoint.
            TransportClientEndpointBehavior sasCredential = new TransportClientEndpointBehavior();
            sasCredential.TokenProvider = TokenProvider.CreateSharedAccessSignatureTokenProvider("RootManageSharedAccessKey", sasKey);

            // Create the service URI based on the service namespace.
            Uri address = ServiceBusEnvironment.CreateServiceUri("sb", serviceNamespace, "EchoService");

            // Create the service host reading the configuration.
            ServiceHost host = new ServiceHost(typeof(EchoService), address);

            // Create the ServiceRegistrySettings behavior for the endpoint.
            IEndpointBehavior serviceRegistrySettings = new ServiceRegistrySettings(DiscoveryType.Public);

            // Add the Relay credentials to all endpoints specified in configuration.
            foreach (ServiceEndpoint endpoint in host.Description.Endpoints)
            {
                endpoint.Behaviors.Add(serviceRegistrySettings);
                endpoint.Behaviors.Add(sasCredential);
            }

            // Open the service.
            host.Open();

            Console.WriteLine("Service address: " + address);
            Console.WriteLine("Press [Enter] to exit");
            Console.ReadLine();

            // Close the service.
            host.Close();
        }
    }
}
```

## Create a WCF client for the service contract

The next step is to create a client application and define the service contract you will implement in later steps. Note that many of these steps resemble the steps used to create a service: defining a contract, editing an App.config file, using credentials to connect to the relay service, and so on. The code used for these tasks is provided in the example following the procedure.

1. Create a new project in the current Visual Studio solution for the client by doing the following:

   1. In Solution Explorer, in the same solution that contains the service, right-click the current solution (not the project), and click **Add**. Then click **New Project**.
   2. In the **Add New Project** dialog box, click **Visual C#** (if **Visual C#** does not appear, look under **Other Languages**), select the **Console App (.NET Framework)** template, and name it **EchoClient**.
   3. Click **OK**.
      <br />
2. In Solution Explorer, double-click the Program.cs file in the **EchoClient** project to open it in the editor, if it is not already open.
3. Change the namespace name from its default name of `EchoClient` to `Microsoft.ServiceBus.Samples`.
4. Install the [Service Bus NuGet package](https://www.nuget.org/packages/WindowsAzure.ServiceBus): in Solution Explorer, right-click the **EchoClient** project, and then click **Manage NuGet Packages**. Click the **Browse** tab, then search for `Microsoft Azure Service Bus`. Click **Install**, and accept the terms of use.

    ![][3]
5. Add a `using` statement for the [System.ServiceModel](https://msdn.microsoft.com/library/system.servicemodel.aspx) namespace in the Program.cs file.

    ```csharp
    using System.ServiceModel;
    ```
6. Add the service contract definition to the namespace, as shown in the following example. Note that this definition is identical to the definition used in the **Service** project. You should add this code at the top of the `Microsoft.ServiceBus.Samples` namespace.

    ```csharp
    [ServiceContract(Name = "IEchoContract", Namespace = "http://samples.microsoft.com/ServiceModel/Relay/")]
    public interface IEchoContract
    {
        [OperationContract]
        string Echo(string text);
    }

    public interface IEchoChannel : IEchoContract, IClientChannel { }
    ```
7. Press **Ctrl+Shift+B** to build the client.

### Example

The following code shows the current status of the Program.cs file in the **EchoClient** project.

```csharp
using System;
using Microsoft.ServiceBus;
using System.ServiceModel;

namespace Microsoft.ServiceBus.Samples
{

    [ServiceContract(Name = "IEchoContract", Namespace = "http://samples.microsoft.com/ServiceModel/Relay/")]
    public interface IEchoContract
    {
        [OperationContract]
        string Echo(string text);
    }

    public interface IEchoChannel : IEchoContract, IClientChannel { }


    class Program
    {
        static void Main(string[] args)
        {
        }
    }
}
```

## Configure the WCF client

In this step, you create an App.config file for a basic client application that accesses the service created previously in this tutorial. This App.config file defines the contract, binding, and name of the endpoint. The code used for these tasks is provided in the example following the procedure.

1. In Solution Explorer, in the **EchoClient** project, double-click **App.config** to open the file in the Visual Studio editor.
2. In the `<appSettings>` element, replace the placeholders with the name of your service namespace, and the SAS key that you copied in an earlier step.
3. Within the system.serviceModel element, add a `<client>` element.

    ```xml
    <?xmlversion="1.0"encoding="utf-8"?>
    <configuration>
      <system.serviceModel>
        <client>
        </client>
      </system.serviceModel>
    </configuration>
    ```

    This step declares that you are defining a WCF-style client application.
4. Within the `client` element, define the name, contract, and binding type for the endpoint.

    ```xml
    <endpoint name="RelayEndpoint"
                    contract="Microsoft.ServiceBus.Samples.IEchoContract"
                    binding="netTcpRelayBinding"/>
    ```

    This step defines the name of the endpoint, the contract defined in the service, and the fact that the client application uses TCP to communicate with Azure Relay. The endpoint name is used in the next step to link this endpoint configuration with the service URI.
5. Click **File**, then click **Save All**.

## Example

The following code shows the App.config file for the Echo client.

```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <system.serviceModel>
    <client>
      <endpoint name="RelayEndpoint"
                      contract="Microsoft.ServiceBus.Samples.IEchoContract"
                      binding="netTcpRelayBinding"/>
    </client>
    <extensions>
      <bindingExtensions>
        <add name="netTcpRelayBinding"
                    type="Microsoft.ServiceBus.Configuration.NetTcpRelayBindingCollectionElement, Microsoft.ServiceBus, Culture=neutral, PublicKeyToken=31bf3856ad364e35"/>
      </bindingExtensions>
    </extensions>
  </system.serviceModel>
</configuration>
```

## Implement the WCF client
In this step, you implement a basic client application that accesses the service you created previously in this tutorial. Similar to the service, the client performs many of the same operations to access Azure Relay:

1. Sets the connectivity mode.
2. Creates the URI that locates the host service.
3. Defines the security credentials.
4. Applies the credentials to the connection.
5. Opens the connection.
6. Performs the application-specific tasks.
7. Closes the connection.

However, one of the main differences is that the client application uses a channel to connect to the relay service, whereas the service uses a call to **ServiceHost**. The code used for these tasks is provided in the example following the procedure.

### Implement a client application
1. Set the connectivity mode to **AutoDetect**. Add the following code inside the `Main()` method of the **EchoClient** application.

    ```csharp
    ServiceBusEnvironment.SystemConnectivity.Mode = ConnectivityMode.AutoDetect;
    ```
2. Define variables to hold the values for the service namespace, and SAS key that are read from the console.

    ```csharp
    Console.Write("Your Service Namespace: ");
    string serviceNamespace = Console.ReadLine();
    Console.Write("Your SAS Key: ");
    string sasKey = Console.ReadLine();
    ```
3. Create the URI that defines the location of the host in your Relay project.

    ```csharp
    Uri serviceUri = ServiceBusEnvironment.CreateServiceUri("sb", serviceNamespace, "EchoService");
    ```
4. Create the credential object for your service namespace endpoint.

    ```csharp
    TransportClientEndpointBehavior sasCredential = new TransportClientEndpointBehavior();
    sasCredential.TokenProvider = TokenProvider.CreateSharedAccessSignatureTokenProvider("RootManageSharedAccessKey", sasKey);
    ```
5. Create the channel factory that loads the configuration described in the App.config file.

    ```csharp
    ChannelFactory<IEchoChannel> channelFactory = new ChannelFactory<IEchoChannel>("RelayEndpoint", new EndpointAddress(serviceUri));
    ```

    A channel factory is a WCF object that creates a channel through which the service and client applications communicate.
6. Apply the credentials.

    ```csharp
    channelFactory.Endpoint.Behaviors.Add(sasCredential);
    ```
7. Create and open the channel to the service.

    ```csharp
    IEchoChannel channel = channelFactory.CreateChannel();
    channel.Open();
    ```
8. Write the basic user interface and functionality for the echo.

    ```csharp
    Console.WriteLine("Enter text to echo (or [Enter] to exit):");
    string input = Console.ReadLine();
    while (input != String.Empty)
    {
        try
        {
            Console.WriteLine("Server echoed: {0}", channel.Echo(input));
        }
        catch (Exception e)
        {
            Console.WriteLine("Error: " + e.Message);
        }
        input = Console.ReadLine();
    }
    ```

    Note that the code uses the instance of the channel object as a proxy for the service.
9. Close the channel, and close the factory.

    ```csharp
    channel.Close();
    channelFactory.Close();
    ```

## Example

Your completed code should appear as follows, showing how to create a client application, how to call the operations of the service, and how to close the client after the operation call is finished.

```csharp
using System;
using Microsoft.ServiceBus;
using System.ServiceModel;

namespace Microsoft.ServiceBus.Samples
{
    [ServiceContract(Name = "IEchoContract", Namespace = "http://samples.microsoft.com/ServiceModel/Relay/")]
    public interface IEchoContract
    {
        [OperationContract]
        String Echo(string text);
    }

    public interface IEchoChannel : IEchoContract, IClientChannel { }

    class Program
    {
        static void Main(string[] args)
        {
            ServiceBusEnvironment.SystemConnectivity.Mode = ConnectivityMode.AutoDetect;


            Console.Write("Your Service Namespace: ");
            string serviceNamespace = Console.ReadLine();
            Console.Write("Your SAS Key: ");
            string sasKey = Console.ReadLine();



            Uri serviceUri = ServiceBusEnvironment.CreateServiceUri("sb", serviceNamespace, "EchoService");

            TransportClientEndpointBehavior sasCredential = new TransportClientEndpointBehavior();
            sasCredential.TokenProvider = TokenProvider.CreateSharedAccessSignatureTokenProvider("RootManageSharedAccessKey", sasKey);

            ChannelFactory<IEchoChannel> channelFactory = new ChannelFactory<IEchoChannel>("RelayEndpoint", new EndpointAddress(serviceUri));

            channelFactory.Endpoint.Behaviors.Add(sasCredential);

            IEchoChannel channel = channelFactory.CreateChannel();
            channel.Open();

            Console.WriteLine("Enter text to echo (or [Enter] to exit):");
            string input = Console.ReadLine();
            while (input != String.Empty)
            {
                try
                {
                    Console.WriteLine("Server echoed: {0}", channel.Echo(input));
                }
                catch (Exception e)
                {
                    Console.WriteLine("Error: " + e.Message);
                }
                input = Console.ReadLine();
            }

            channel.Close();
            channelFactory.Close();

        }
    }
}
```

## Run the applications

1. Press **Ctrl+Shift+B** to build the solution. This builds both the client project and the service project that you created in the previous steps.
2. Before running the client application, you must make sure that the service application is running. In Solution Explorer in Visual Studio, right-click the **EchoService** solution, then click **Properties**.
3. In the solution properties dialog box, click **Startup Project**, then click the **Multiple startup projects** button. Make sure **EchoService** appears first in the list.
4. Set the **Action** box for both the **EchoService** and **EchoClient** projects to **Start**.

    ![][5]
5. Click **Project Dependencies**. In the **Projects** box, select **EchoClient**. In the **Depends on** box, make sure **EchoService** is checked.

    ![][6]
6. Click **OK** to dismiss the **Properties** dialog.
7. Press **F5** to run both projects.
8. Both console windows open and prompt you for the namespace name. The service must run first, so in the **EchoService** console window, enter the namespace and then press **Enter**.
9. Next, you are prompted for your SAS key. Enter the SAS key and press ENTER.

    Here is example output from the console window. Note that the values provided here are for example purposes only.

    `Your Service Namespace: myNamespace`
    `Your SAS Key: <SAS key value>`

    The service application prints to the console window the address on which it's listening, as seen in the following example.

    `Service address: sb://mynamespace.servicebus.windows.net/EchoService/`
    `Press [Enter] to exit`
10. In the **EchoClient** console window, enter the same information that you entered previously for the service application. Follow the previous steps to enter the same service namespace and SAS key values for the client application.
11. After entering these values, the client opens a channel to the service and prompts you to enter some text as seen in the following console output example.

    `Enter text to echo (or [Enter] to exit):`

    Enter some text to send to the service application and press Enter. This text is sent to the service through the Echo service operation and appears in the service console window as in the following example output.

    `Echoing: My sample text`

    The client application receives the return value of the `Echo` operation, which is the original text, and prints it to its console window. The following is example output from the client console window.

    `Server echoed: My sample text`
12. You can continue sending text messages from the client to the service in this manner. When you are finished, press Enter in the client and service console windows to end both applications.

## Next steps

This tutorial showed how to build an Azure Relay client application and service using the WCF Relay capabilities of Service Bus. For a similar tutorial that uses [Service Bus Messaging](../service-bus-messaging/service-bus-messaging-overview.md), see [Get started with Service Bus queues](../service-bus-messaging/service-bus-dotnet-get-started-with-queues.md).

To learn more about Azure Relay, see the following topics.

* [Azure Relay overview](relay-what-is-it.md)
* [How to use the WCF relay service with .NET](relay-wcf-dotnet-get-started.md)

[2]: ./media/service-bus-relay-tutorial/create-console-app.png
[3]: ./media/service-bus-relay-tutorial/install-nuget.png
[5]: ./media/service-bus-relay-tutorial/set-projects.png
[6]: ./media/service-bus-relay-tutorial/set-depend.png
