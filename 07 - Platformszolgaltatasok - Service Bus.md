# Platformszolgáltatások - Service Bus #
## #1 ##
```cs
[Serializable]
public class CustomMessage
{
  private DateTime date;
  private string body;

  public DateTime Date
  {
    get { return this.date; }
    set { this.date = value; }
  }

  public string Body
  {
    get { return this.body; }
    set { this.body = value; }
  }
}
```
------------------------------------------------------
## #2 ##
```cs
private string connectionString= "";
private NamespaceManager namespaceManager;
public HomeController()
{        
  namespaceManager = NamespaceManager.CreateFromConnectionString( connectionString );
}
```
------------------------------------------------------
## #3 ##
```cs
public JsonResult Queues()
{
   var queues = this.namespaceManager.GetQueues().Select( c => new { Name = c.Path, Messages = c.MessageCount } ).ToArray();
   return this.Json( queues, JsonRequestBehavior.AllowGet );
}
```
------------------------------------------------------
## #4 ##
```cs
public long GetMessageCount( string queueName )
{
   var queueDescription = this.namespaceManager.GetQueue( queueName );
   return queueDescription.MessageCount;
}
```
------------------------------------------------------
## #5 ##
```cs
[HttpPost]
public void SendMessage( string queueName, string messageBody )
{
  QueueClient queueClient = QueueClient.CreateFromConnectionString( connectionString, "azuretananyag" );
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
## #6 ##
```cs
[HttpGet]
public JsonResult RetrieveMessage( string queueName )
{
  QueueClient queueClient = QueueClient.CreateFromConnectionString( connectionString, "azuretananyag", ReceiveMode.PeekLock );
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
    Date = receivedCustomMessage.Date,
    Message = receivedCustomMessage.Body,
    Properties = receivedMessage.Properties.ToArray(),
    BrokeredMsgProperties = brokeredMsgProperties.ToArray()
  };

  receivedMessage.Complete();
  return this.Json( new { MessageInfo = messageInfo }, JsonRequestBehavior.AllowGet );
}
```
-------------------------------------------------------