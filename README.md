# What we are going to do
We will follow a simple IoT scenario in which a device sends a message to the IoT Hub that the web app will display in real-time (without having to refresh the page). In a more real world IoT scenario, that could be telemetry or sensor data such as temperature or humidity coming from the device.

To achieve this we will create and use the following services on Azure:

- IoT Hub
- Function App
- SignalR Service

**IoT Hub** is a managed service that acts as a central message hub for communication between your IoT application and the devices it manages. **Azure SignalR Service** is a fully managed real-time messaging platform that supports WebSockets. We’ll use it in combination with **Azure Functions** to broadcast ‘Device-To-Cloud’ (D2C) messages to the browser in a simulated enviroment. We will run a simple client web application which will display the realtime updated values of Temprature, Humidity, and Pressure coming from a simulated device. 

![Diagram showing the realtime data on website](/imgs/demo.png)

In short the process can be described as follows: we will simulate an IoT device with our development machine and send a ‘Device-To-Cloud’ (D2C) message to the IoT Hub. The Azure function will be triggered by this incoming message on the IoT Hub and has an output binding to the SignalR Service. The web app establishes a connection with the SignalR Service and will receive messages sent to the SignalR Service. The web app will update the UI based on the data in the SignalR message. In a diagram the entire process looks like this:

![Diagram showing the flow of the message through the system](/imgs/flow.png)

# Simulated IoT Device Messages
Run the simuluated device .sln file. This project will randomly generate values for Temprature, Humidity, and Pressure for a IoT Device and upload it to the IoT Hub.

```
 class Program
    {
        private static DeviceClient s_deviceClient;
        private readonly static string s_myDeviceId = "myDevice01";
        private readonly static string s_iotHubUri = "iothubjohnsoncontrol.azure-devices.net";
        // This is the primary key for the device. This is in the portal. 
        // Find your IoT hub in the portal > IoT devices > select your device > copy the key. 
        private readonly static string s_deviceKey = "Bb6kOyw/YaeFVubvPo/DlAoLJwF1C0cIi7TR/Q/enxg=";

        private static void Main(string[] args)
        {
            Console.WriteLine("Routing Tutorial: Simulated device\n"); 
            s_deviceClient = DeviceClient.Create(s_iotHubUri, new DeviceAuthenticationWithRegistrySymmetricKey(s_myDeviceId, s_deviceKey), TransportType.Mqtt);
            SendDeviceToCloudMessagesAsync();
            Console.WriteLine("Press the Enter key to stop.");
            Console.ReadLine();
        }
        private static async void SendDeviceToCloudMessagesAsync()
        {
            double minTemperature = 20;
            double minHumidity = 60;
            Random rand = new Random();

            while (true)
            {
                double currentTemperature = minTemperature + rand.NextDouble() * 15;
                double currentHumidity = minHumidity + rand.NextDouble() * 20;

                string infoString;
                string levelValue;

                if (rand.NextDouble() > 0.7)
                {
                    if (rand.NextDouble() > 0.5)
                    {
                        levelValue = "critical";
                        infoString = "This is a critical message.";
                    }
                    else
                    {
                        levelValue = "storage";
                        infoString = "This is a storage message.";
                    }
                }
                else
                {
                    levelValue = "normal";
                    infoString = "This is a normal message.";
                }

                var telemetryDataPoint = new
                {
                    deviceId = s_myDeviceId,
                    temperature = currentTemperature,
                    humidity = currentHumidity,
                    pointInfo = infoString
                };
                var telemetryDataString = JsonConvert.SerializeObject(telemetryDataPoint);

                //set the body of the message to the serialized value of the telemetry data
                var message = new Message(Encoding.ASCII.GetBytes(telemetryDataString));
                message.Properties.Add("level", levelValue);

                await s_deviceClient.SendEventAsync(message);
                Console.WriteLine("{0} > Sent message: {1}", DateTime.Now, telemetryDataString);

                await Task.Delay(5000);
            }
        }
    }

```

# Azure Function
Run the project Azure Function. This will execute the azure function, which will pull the data from the IoT Hub to SignalR in Realtime.

```

    public static class SignalR
    {
        [FunctionName("SignalR")]
        public static async Task Run(
            [IoTHubTrigger("messages/events", Connection = "IoTHubTriggerConnection", ConsumerGroup = "$Default")]EventData message,
            [SignalR(HubName = "broadcast")]IAsyncCollector<SignalRMessage> signalRMessages,
            ILogger log)
        {

            var deviceData = JsonConvert.DeserializeObject<DeviceData>(Encoding.UTF8.GetString(message.Body.Array));
            deviceData.DeviceId = Convert.ToString(message.SystemProperties["iothub-connection-device-id"]);


            log.LogInformation($"C# IoT Hub trigger function processed a message: {JsonConvert.SerializeObject(deviceData)}");
            await signalRMessages.AddAsync(new SignalRMessage()
            {
                Target = "notify",
                Arguments = new[] { JsonConvert.SerializeObject(deviceData) }
            });
        }
    }

```

# Run the Client App
Client app will contain 3 cards for Temprature, Humidity, and Pressure. Whose values gets updated on realtime by using SignalR


Original Credits: [Realtime Serverless IoT Data using Azure SignalR and Azure Functions in Angular](https://www.codeproject.com/Articles/1273206/Realtime-Serverless-IoT-Data-using-Azure-SignalR-a)
