# A dolgok internete (IoT) #
## #1 ##
```cs
static string connectionString = "{connection string}";
static RegistryManager registryManager;

```
------------------------------------------------------
## #2 ##
```cs
static async Task AddDeviceAsync()
{
    Console.WriteLine("Kérem adja meg az eszköz azonosítóját:");
    string deviceId = Console.ReadLine();
    Device device;
    try
    {
        device = 
            await registryManager.AddDeviceAsync(new Device(deviceId) {Status = DeviceStatus.Enabled});
    }
    catch (DeviceAlreadyExistsException)
    {
        device = await registryManager.GetDeviceAsync(deviceId);
    }
    Console.WriteLine("Generált kulcs: {0}", device.Authentication.SymmetricKey.PrimaryKey);
}

```
------------------------------------------------------
## #3 ##
```cs
static void Main(string[] args)
{
    registryManager = RegistryManager.CreateFromConnectionString(connectionString);
    AddDeviceAsync().Wait();
    Console.ReadKey();
}

```
------------------------------------------------------
## #4 ##
```cs
using Microsoft.ServiceBus.Messaging;
using System.Threading;
```
------------------------------------------------------
## #5 ##
```cs
static string connectionString = "{iothub connection string}";
static string iotHubD2cEndpoint = "messages/events";
static EventHubClient eventHubClient;
```
------------------------------------------------------
## #6 ##
```cs
private static async Task ReceiveMessagesFromDeviceAsync(string partition, CancellationToken ct) 
{
   var eventHubReceiver = 
   eventHubClient.GetDefaultConsumerGroup().CreateReceiver(partition,DateTime.UtcNow);
   while (true)
   {
     if (ct.IsCancellationRequested) break;
     EventData eventData = await eventHubReceiver.ReceiveAsync();
     if (eventData == null) continue;

     string data = Encoding.UTF8.GetString(eventData.GetBytes());
     Console.WriteLine("Üzenet érkezett. Partíció: {0} Adat: '{1}'", partition, data);
}}

```
------------------------------------------------------
## #7 ##
```cs
Console.WriteLine("Üzenetek fogadása. Kilépés: Ctrl + C");
eventHubClient = EventHubClient.CreateFromConnectionString(connectionString, iotHubD2cEndpoint);
```
------------------------------------------------------
## #8 ##
```cs
CancellationTokenSource cts = new CancellationTokenSource();

System.Console.CancelKeyPress += (s, e) =>
{
    e.Cancel = true;
    cts.Cancel();
    Console.WriteLine("Kilépés...");
};
```
------------------------------------------------------
## #9 ##
```cs
var d2cPartitions = eventHubClient.GetRuntimeInformation().PartitionIds;

var tasks = new List<Task>();
foreach (string partition in d2cPartitions)
{
    tasks.Add(ReceiveMessagesFromDeviceAsync(partition, cts.Token));
}
Task.WaitAll(tasks.ToArray());
```
------------------------------------------------------
## #10 ##
```cs
using Microsoft.Azure.Devices.Client;
using Newtonsoft.Json;
```
------------------------------------------------------
## #11 ##
```cs
static DeviceClient deviceClient;
static string iotHubUri = "{iot hub hostname}";
static string deviceKey = "{device key}";
static string deviceId = "myDevice";
```
------------------------------------------------------
## #12 ##
```cs
public class MessageContent
{
    public string deviceId;
    public double windSpeed;
}
```
------------------------------------------------------
## #13 ##
```cs
private static async void SendDeviceToCloudMessagesAsync()
{
    Random rand = new Random();

    while (true)
    {
        double currentWindSpeed = 10 + rand.NextDouble() * 4 - 2;

        var messageContent = new MessageContent();
        messageContent.deviceId = deviceId;
        messageContent.windSpeed = currentWindSpeed;
        
        var messageString = JsonConvert.SerializeObject(messageContent);
        var message = new Message(Encoding.ASCII.GetBytes(messageString));

        await deviceClient.SendEventAsync(message);
        Console.WriteLine("{0} > Üzenet küldése: {1}", DateTime.Now, messageString);

        Task.Delay(1000).Wait();
    }
}
```
------------------------------------------------------
## #14 ##
```cs
static void Main(string[] args)
{
    Console.WriteLine("Szimulált eszköz\n");
    deviceClient = 
        DeviceClient.Create(iotHubUri, new DeviceAuthenticationWithRegistrySymmetricKey(deviceId, deviceKey));

    SendDeviceToCloudMessagesAsync();
    Console.ReadLine();
}
```
------------------------------------------------------
## #15 ##
```cs
private static async void ReceiveC2dAsync()
{
    Console.WriteLine("\nÜzenetek fogadása az IoT Hub-tól");
    while (true)
    {
        Message receivedMessage = await deviceClient.ReceiveAsync();
        if (receivedMessage == null) continue;

        Console.ForegroundColor = ConsoleColor.Yellow;
        Console.WriteLine("Üzenet érkezett: {0}", Encoding.ASCII.GetString(receivedMessage.GetBytes()));
        Console.ResetColor();
        
        await deviceClient.CompleteAsync(receivedMessage);
    }
}
```
------------------------------------------------------
## #16 ##
```cs
static void Main(string[] args)
{
    Console.WriteLine("Szimulált eszköz\n");
    deviceClient = 
        DeviceClient.Create(iotHubUri, new DeviceAuthenticationWithRegistrySymmetricKey(deviceId, deviceKey));

    SendDeviceToCloudMessagesAsync();
    ReceiveC2dAsync();
    Console.ReadLine();
}
```
------------------------------------------------------
## #17 ##
```cs
using Microsoft.Azure.Devices;
```
------------------------------------------------------
## #18 ##
```cs
static ServiceClient serviceClient;
```
------------------------------------------------------
## #19 ##
```cs
private async static Task SendCloudToDeviceMessageAsync()
{
    for (int i = 0; i < 3; i++)
    {
        await Task.Delay(1000);
        Console.WriteLine("Üzenet küldése az eszköznek\n");
        var commandMessage = new Message(Encoding.ASCII.GetBytes($"{i}. üzenet az eszköznek!"));
        commandMessage.Ack = DeliveryAcknowledgement.Full;
        await serviceClient.SendAsync("myDevice", commandMessage);
    }
}
```
------------------------------------------------------
## #20 ##
```cs
private async static Task ReceiveFeedbackAsync(CancellationToken ct)
{
    var feedbackReceiver = serviceClient.GetFeedbackReceiver();

    Console.WriteLine("\nVisszajelzés fogadásának indítása.");
    while (true)
    {
        if (ct.IsCancellationRequested) break;
        var feedbackBatch = await feedbackReceiver.ReceiveAsync();
        if (feedbackBatch == null) continue;

        Console.ForegroundColor = ConsoleColor.Blue;
        Console.WriteLine("Visszajelzések: {0}", 
            string.Join(", ", feedbackBatch.Records.Select(f => f.StatusCode)));
        Console.ResetColor();

        await feedbackReceiver.CompleteAsync(feedbackBatch);
    }
}
```
------------------------------------------------------
## #21 ##
```cs
foreach (string partition in d2cPartitions)
{
    tasks.Add(ReceiveMessagesFromDeviceAsync(partition, cts.Token));
}

serviceClient = ServiceClient.CreateFromConnectionString(connectionString);
tasks.Add(SendCloudToDeviceMessageAsync());
tasks.Add(ReceiveFeedbackAsync(cts.Token));

Task.WaitAll(tasks.ToArray());
```
