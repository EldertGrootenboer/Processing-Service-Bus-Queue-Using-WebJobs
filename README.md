# Processing Service Bus Queue Using WebJobs
A WebJob is a simple way to set up a background job, which can process continuously or on a schedule. WebJobs differ from a cloud service as it gives you get less fine-grained control over your processing environment, making it a more true PaaS service. This sample is part of my blogpost which you can find [here](http://blog.eldert.net/?p=1355).

## Description
We will need a Web App to host our WebJob, so lets create one in the Azure Portal. You can create a new Web App by going to App Services, and selecting New.

![](https://code.msdn.microsoft.com/site/view/file/151078/1/image_thumb5.png)

To simplify our deployment later on, we will download the publish profile for our Web App once it has been created.

![](https://code.msdn.microsoft.com/site/view/file/151079/1/image_thumb-22.png)

Next we will create a new project for our WebJob, so be sure to install the [Azure WebJob SDK](https://azure.microsoft.com/en-us/documentation/articles/websites-dotnet-webjobs-sdk/) if you don’t have it yet.

![](https://code.msdn.microsoft.com/site/view/file/151080/1/image-21.png)

Once the project has been created, start by going to the App.Config, and setting the connection strings for the dashboard and storage. This should be in the format DefaultEndpointsProtocol=https;AccountName=NAME;AccountKey=KEY. Both the name and the key can be found in the settings of your storage account.

![](https://code.msdn.microsoft.com/site/view/file/151081/1/image_thumb7_thumb.png)

We will also need to set the connection string for our Service Bus Queue, for which we will need a Shared Access Key with Manage permissions, as required by the WebJob’s job host.

![](https://code.msdn.microsoft.com/site/view/file/151082/1/image_thumb-23.png)

And finally, we will also need to add the connection string to our Azure SQL database, which we will use from our Entity Framework library to communicate with the database.  

```XML
<connectionStrings> 
    <add name="AzureWebJobsDashboard" connectionString="DefaultEndpointsProtocol=https;AccountName=eldertiot;AccountKey=xxxxxxxxxxxxxxxxxxxxxxxxxxxxx" /> 
    <add name="AzureWebJobsStorage" connectionString="DefaultEndpointsProtocol=https;AccountName=eldertiot;AccountKey=xxxxxxxxxxxxxxxxxxxxxxxxxxxxx" /> 
    <add name="AzureWebJobsServiceBus" connectionString="Endpoint=sb://eldertiot.servicebus.windows.net/;SharedAccessKeyName=administrationconsole;SharedAccessKey=xxxxxxxxxxxxxxxxxxxxxxxxxxxxx" /> 
    <add name="IoTDatabaseContext" connectionString="Server=tcp:eldertiot.database.windows.net,1433;Database=eldertiot;User ID=Eldert@eldertiot;Password=xxxxxxxxxxxxxxx;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;" providerName="System.Data.SqlClient"/> 
</connectionStrings> 
```

We also have to add the AzureWebJobsDashboard connection string to the Application settings of the Web App in the Azure portal, so the logs can be stored in your storage.

![](https://code.msdn.microsoft.com/site/view/file/151083/1/image_thumb9_thumb.png)

By default a trigger is added to the WebJob for storage queues, however as we want to work with a Service Bus Queue, we will need to add the Microsoft.Azure.WebJob.ServiceBus NuGet package to our project.

  
![](https://code.msdn.microsoft.com/site/view/file/151084/1/image-24.png)

Now that we have all configuration in place, we’ll go and implement the code in our WebJob. Open up the Functions class which was created with inside your WebJob project. We will change the trigger type to ServiceBusTrigger so we can get triggers from our Service Bus Queue. As we are using a Service Bus trigger, we will also need to change the type of the message to be a BrokeredMessage instead of a string. When we have received the message, we will save its contents to the database, using [this library](https://code.msdn.microsoft.com/Entity-Framework-Code-e9000ebc).

```C#
using System; 
using System.IO; 
  
using Eldert.IoT.Data.DataTypes; 
  
using Microsoft.Azure.WebJobs; 
using Microsoft.ServiceBus.Messaging; 
  
namespace Eldert.IoT.Azure.ServiceBusQueueProcessor 
{ 
    public class Functions 
    { 
        private static readonly IoTDatabaseContext database = new IoTDatabaseContext(); 
  
        /// <summary> 
        /// This function will get triggered/executed when a new message is written on an Azure Service Bus Queue. 
        /// </summary> 
        public static void ProcessQueueMessage([ServiceBusTrigger("queueerrorsandwarnings")] BrokeredMessage message, TextWriter log) 
        { 
            try 
            { 
                log.WriteLine($"Processing message: {message.Properties["exceptionmessage"]} Ship: {message.Properties["ship"]}"); 
  
                // Add the message we received from our queue to the database 
                database.ErrorAndWarningsEntries.Add(new ErrorAndWarning() 
                { 
                    CreatedDateTime = DateTime.Parse(message.Properties["time"].ToString()), 
                    ShipName = message.Properties["ship"].ToString(), 
                    Message = message.Properties["exceptionmessage"].ToString() 
                }); 
  
                // Save changes in the database 
                database.SaveChanges(); 
            } 
            catch (Exception exception) 
            { 
                log.WriteLine($"Exception in ProcessQueueMessage: {exception}"); 
            } 
        } 
    } 
}
```
 
Next we will update the Program class, as we will need to register our Service Bus extension in the configuration of our job host.

```C#
using Microsoft.Azure.WebJobs; 
  
namespace Eldert.IoT.Azure.ServiceBusQueueProcessor 
{ 
    // To learn more about Microsoft Azure WebJobs SDK, please see http://go.microsoft.com/fwlink/?LinkID=320976 
    class Program 
    { 
        // Please set the following connection strings in app.config for this WebJob to run: 
        // AzureWebJobsDashboard and AzureWebJobsStorage 
        static void Main() 
        { 
            // Create job host configuration 
            var config = new JobHostConfiguration(); 
  
            // Tell configuration we want to use Azure Service Bus 
            config.UseServiceBus(); 
  
            // Add the configuration to the job host 
            var host = new JobHost(config); 
  
            // The following code ensures that the WebJob will be running continuously 
            host.RunAndBlock(); 
        } 
    } 
} 
```

## More Information
If you would like more information on how to publish and debug the application, you can find this in [my blogpost](http://blog.eldert.net/?p=1355). 
