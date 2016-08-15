# Platformszolgáltatások - Service Bus #
## #1 ##
```cs
[Serializable]
public class CustomMessage
{
    public string Body { get; set; }
    public DateTime Date { get; set; }
}
```
------------------------------------------------------
## #2 ##
```cs
public ActionResult Index()
{
    var queues = namespaceManager.GetQueues();
    var result = new List<string>();
    foreach (var item in queues)
    {
        result.Add(item.Path);
    }
    return View(result);
}
```
------------------------------------------------------
## #3 ##
```cs
public long MessageCount(string queueName)
{
    var queueDescription = this.namespaceManager.GetQueue(queueName);
    if (queueDescription == null)
    {
        return 0;
    }
    return queueDescription.MessageCount;
}
```
------------------------------------------------------
## #4 ##
```cs
[HttpPost]
public void SendMessage( string queueName, string messageBody )
  {
  QueueClient queueClient = QueueClient.CreateFromConnectionString( connectionString, queueName);
  var customMessage = new CustomMessage() { Date = DateTime.Now, Body = messageBody };
  using( var bm = new BrokeredMessage( customMessage ) )
  {
    bm.Properties["Urgent"] = "1";
    bm.Properties["Priority"] = "High";
    queueClient.Send( bm );
  }
}
```
------------------------------------------------------
## #5 ##
```cs
[HttpGet]
public JsonResult RetrieveMessage( string queueName )
{
  QueueClient queueClient = QueueClient.CreateFromConnectionString( connectionString, queueName, ReceiveMode.PeekLock );
  BrokeredMessage receivedMessage = queueClient.Receive( new TimeSpan( 0, 0, 30 ) );

  if( receivedMessage == null )
  {
    return this.Json( null, JsonRequestBehavior.AllowGet );
  }

  var receivedCustomMessage = receivedMessage.GetBody<CustomMessage>();

  var brokeredMsgProperties = new Dictionary<string, object>();
  brokeredMsgProperties.Add( "Size", receivedMessage.Size );
  brokeredMsgProperties.Add( "MessageId", receivedMessage.MessageId.Substring( 0, 15 ) + "..." );
  brokeredMsgProperties.Add( "TimeToLive", receivedMessage.TimeToLive.TotalSeconds );
  brokeredMsgProperties.Add( "EnqueuedTimeUtc", receivedMessage.EnqueuedTimeUtc.ToString( "yyyy-MM-dd HH:mm:ss" ) );
  brokeredMsgProperties.Add( "ExpiresAtUtc", receivedMessage.ExpiresAtUtc.ToString( "yyyy-MM-dd HH:mm:ss" ) );

  var messageInfo = new
  {
    Label = receivedMessage.Label,
    Date = receivedCustomMessage.Date.ToString("yyyy-MM-dd HH:mm:ss"),
    Message = receivedCustomMessage.Body,
    Properties = receivedMessage.Properties.ToArray(),
    BrokeredMsgProperties = brokeredMsgProperties.ToArray()
  };

  receivedMessage.Complete();
  return this.Json(messageInfo, JsonRequestBehavior.AllowGet );
}
```
-------------------------------------------------------
