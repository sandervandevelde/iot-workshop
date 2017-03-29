# The Things Network & Azure IoT: a perfect combination
## Passing commands back to a device

This is an example of how downlink commands are sent back to a device. In this workshop, we will send commands back to faulty devices, using an Azure Function, to start them up again. 

![alt tag](img/arch/azure-telemetry-pipeline-commands.png)

This part of the workshop supports both to the [TTN Node](TheThingsNetwork.md) and to the [UWP app](UwpToIotHub.md). 

*Note: In this workshop, we will create uniquely named Azure resources. The suggested names could be reserved already.*

### Prerequisites

1. Azure account [create here](https://azure.microsoft.com/en-us/free/) _(Azure passes will be present for those who have no Azure account (please check your email for final confirmation))_
2. a running NodeJs app which simulates a machine running duty cycles
4. A combination of Azure IoT Hub, Stream Analytics job, Event Hub and Azure Function which are waiting for analyzed telemetry coming from the devices
5. A running Device Explorer or IoT Hub Explorer, connected to the IoT Hub, showing the telemetry coming in

### Steps to perform in this part of the workshop

At the end of this part of the workshop, the following steps are performed

1. Creating commands to send back
2. Handle commands in the running NodeJs app

## Sending back commands for devices which are in a faulty state

![alt tag](img/msft/Picture12-connect-anything-using-flow.png)

In the [previous workshop](Azure.md), we passed the telemetry from the device to a Stream Analytics job. This job collected devices which are sending error states. Every two minutes, information about devices that are in a faulty state are passed to an Azure Function.

In this workshop, we will react on these devices by sending them a command to 'repair themselves'. 

### Updating the Azure Function with sending command logic

First, we update the Azure Function. For each device which is passed on, we send a command back.

Sending commands back to devices is a specific feature of the IoT Hub. The IoT Hub registers devices and their security policies. And the IoT Hub has built-in logic to send commands back.

1. On the left, select `Resource groups`. A list of resource groups is shown

    ![alt tag](img/azure-resource-groups.png)

2. Select the ResourceGroup `IoTWorkshop-rg`. It will open a new blade with all resources in this group
3. Select the Azure Function App `IoTWorkshop-fa`
4. To the left, the current functions are shown. Select `IoTWorkshopEventHubFunction`

    ![alt tag](img/commands/azure-functions-functions.png)

5. The Code panel is shown. The code of the function is shown. *Note: this code is saved in a file named run.scx*
6. Change the current code into

    ```csharp
    using System;
    using Microsoft.Azure.Devices;
    using System.Text;
    using Newtonsoft.Json; 

    public static void Run(string myEventHubMessage, TraceWriter log)
    {
      log.Info($"Stream Analytics produced {myEventHubMessage}");  

      // Connect to IoT Hub   
      var connectionString = "[IOT HUB connection string]";
      var serviceClient = ServiceClient.CreateFromConnectionString(connectionString);
    
      // Send commands to all devices 
      var messages = JsonConvert.DeserializeObject<StreamAnalyticsCommand[]>(myEventHubMessage);

      log.Info($"{messages.Length} messages arrived.");

      foreach(var message in messages)
      {
        var bytes= new byte[1];
        bytes[0] = 42; // restart the machine!
        var commandMessage = new Message(bytes);
        serviceClient.SendAsync(message.deviceid, commandMessage);

        // Log
        log.Info($"Machine restart command processed after {message.count} errors for {message.deviceid}");
      }
    }

    public class StreamAnalyticsCommand
    {
      public string deviceid {get; set;}
      public int count {get; set;}
    }
    ```

7. Press the `Logs` button to open the pane which shows some basic logging

    ![alt tag](img/azure-function-app-eventhubtrigger-logs.png)

8. A 'Logs' panel is shown. This 'Logs' panel works like a trace log.
9. If you try to run this code, you will notice that compilation fails. This is not that surprising: we are using certain libraries that Azure Functions has no knowledge of. Yet!
10. Press the `View Files` button to open the pane which shows a directory tree of all files.

    ![alt tag](img/commands/azure-function-app-view-files.png)

11. In the pane you can see that the file currently selected is: run.csx

    ![alt tag](img/commands/azure-function-app-view-files-pane.png)

12. Add a new file by pressing `Add`

    ![alt tag](img/commands/azure-function-app-view-files-pane-add.png)

13. Name the new file `project.json`

    ![alt tag](img/commands/azure-function-app-view-files-pane-add-file.png)

14. Press `Enter` to confirm the name of the file and an empty code editor will be shown for this file.
15. The Project.json file describes which Nuget packages have to be referenced. Fill the editor with the following code 

    ```json
    {
      "frameworks": {
        "net46": {
          "dependencies": {
            "Microsoft.AspNet.WebApi.Client": "5.2.3",
            "Microsoft.AspNet.WebApi.Core": "5.2.3",
            "Microsoft.Azure.Amqp": "1.1.5",
            "Microsoft.Azure.Devices": "1.1.0",
            "Newtonsoft.Json": "9.0.1"
          }
        }
      }
    }
    ```

16. Select `Save`. The changed C# code will be recompiled immediately *Note: you can press 'save and run', this will actually run the function, but an empty test will be passed (check out the 'Test' option to the right for more info)*
17. In the 'Logs' panel, just below 'Code', `verify the outcome` of the compilation

    ```
    2017-01-08T14:49:46.794 Packages restored.
    2017-01-08T14:49:47.113 Script for function 'IoTWorkshopEventHubFunction' changed. Reloading.
    2017-01-08T14:49:47.504 Compilation succeeded.
    ```

18. There is just one thing left to do: we have to fill in the Azure IoT Hub security policy connection string. To send commands back, we have to proof we are authorized to do this
19. In the Azure Function, replace '[IOT HUB connection string]' your *remembered* IoT Hub `Connection String-primary key`
20. Recompile again succesfully

Now, the Azure Function is ready to receive data about devices which simulate 'faulty machines'. And it can send commands back to 'repair' the 'machines'.

## Handle commands in the devices

![alt tag](img/msft/Picture05-submit-data-to-ttn.png)

Let's check if your device in already in a faulty state and see how the Azure IoT Platforms sends back a command to repair it.

### Handle commands in the javascript Node.js app

The javascript client is instrumented to fall into an error state already, every fifth "completed cycle" message results in an error state. This will put the device in an error state.

Just wait for approx. two minutes and see how the error state arrives at the IoT Hub and within two minutes the command is generated to 'fix' the machine.

Again, after five successfull cycles, the app will generate another error state, which will eventually be fixed automatically.

## Conclusion


Receiving commands from Azure completes the main part of the workshop.

We hope you did enjoy working with the Azure IoT Platform, as much as we did. Thanks for getting this far!

![alt tag](img/msft/Picture13-make-the-world-a-better-place.png)

But wait, there is still more. We added two bonus chapters to the workshop

1. [Deploying the TTN C# bridge as Azure Web Job](Webjob.md)
2. [Add basic monitoring to the platform](IoTPatformMonitoring.md)

![alt tag](img/logos/innovatos-digitalshockwaves-2017.png)
