# Platformszolgáltatások - Cloud Services #
## #1 ##
```cs
public class GuestBookEntry : TableEntity
{
  public GuestBookEntry()
  {
    this.PartitionKey = DateTime.UtcNow.ToString( "MMddyyyy" );
    this.RowKey = string.Format( "{0:10}_{1}", DateTime.MaxValue.Ticks - DateTime.Now.Ticks, Guid.NewGuid() );
  }

  public string GuestName { get; set; }
  public string PhotoUrl { get; set; }
  public string Message { get; set; }
  public string ThumbnailUrl { get; set; }
}
```
------------------------------------------------------
## #2 ##
```cs
public class GuestBookContext : TableServiceContext
{
  public GuestBookContext(CloudTableClient client) : base(client) { }
  public IQueryable<GuestBookEntry> GuestBookEntry
  {
    get
    {
       return this.CreateQuery<GuestBookEntry>( "GuestBookEntry" );
    }
  }
}
```
------------------------------------------------------
## #3 ##
```cs
public class GuestBookDataSource
{
  private static CloudStorageAccount account;
  private GuestBookContext ctx;
  
  static GuestBookDataSource()
  {
     account = CloudStorageAccount.Parse( "DataConnectionString" );
     account.CreateCloudTableClient().GetTableReference( "GuestBookEntry" ).CreateIfNotExists();
  }

  public GuestBookDataSource()
  {
    this.ctx = new GuestBookContext( account.CreateCloudTableClient() );
  }

  public IEnumerable<GuestBookEntry> GuestBookEntries()
  {
    var table = account.CreateCloudTableClient().GetTableReference( "GuestBookEntry" );
    TableQuery<GuestBookEntry> query = new TableQuery<GuestBookEntry>().Where( TableQuery.GenerateFilterCondition( "PartitionKey", QueryComparisons.Equal, DateTime.UtcNow.ToString( "MMddyyyy" ) ) );
    return table.ExecuteQuery( query );
  }

   public void AddGuestBookEntry( GuestBookEntry newItem )
   {
     TableOperation operation = TableOperation.Insert( newItem );
     CloudTable table = ctx.ServiceClient.GetTableReference( "GuestBookEntry" );
     table.Execute( operation );
   }

  public void UpdateImageThumbnail( string partitionKey, string rowKey, string thumbUrl )
  {
    CloudTable table = ctx.ServiceClient.GetTableReference( "GuestBookEntry" );
    TableOperation retrieveOperation = TableOperation.Retrieve<GuestBookEntry>( partitionKey, rowKey );
    TableResult retrievedResult = table.Execute( retrieveOperation );
    GuestBookEntry updateEntity = (GuestBookEntry)retrievedResult.Result;

    if( updateEntity != null )
    {
      updateEntity.ThumbnailUrl = thumbUrl;
      TableOperation replaceOperation = TableOperation.Replace( updateEntity );
      table.Execute( replaceOperation );
    }
  }
}
```
------------------------------------------------------
## #4 ##
```cs
private static bool storageInitialized = false;
private static object gate = new object();
private static CloudBlobClient blobStorage;
```
------------------------------------------------------
## #5 ##
```cs
private void InitializeStorage()
{
  if( !storageInitialized )
  {
    lock ( gate )
    {
      if( !storageInitialized )
      {
        var storageAccount = CloudStorageAccount.Parse( CloudConfigurationManager.GetSetting( "DataConnectionString" ) );
        blobStorage = storageAccount.CreateCloudBlobClient();
        CloudBlobContainer container = blobStorage.GetContainerReference( "guestbookpics" );
        container.CreateIfNotExists();

        var permissions = container.GetPermissions();
        permissions.PublicAccess = BlobContainerPublicAccessType.Container;
        container.SetPermissions( permissions );

        storageInitialized = true;
       }
     }
  }
}
```
------------------------------------------------------
## #6 ##
```cs
protected void SignButton_Click( object sender, EventArgs e )
{
  if( this.FileUpload1.HasFile )
  {
    this.InitializeStorage();
                
    string uniqueBlobName = string.Format( "guestbookpics/image_{0}{1}", Guid.NewGuid().ToString(), Path.GetExtension( this.FileUpload1.FileName ) );
    CloudBlockBlob blob = blobStorage.GetContainerReference( "guestbookpics" ).GetBlockBlobReference( uniqueBlobName );
    blob.Properties.ContentType = this.FileUpload1.PostedFile.ContentType;
    blob.UploadFromStream( this.FileUpload1.FileContent );                
                
    GuestBookEntry entry = new GuestBookEntry() { GuestName = this.NameTextBox.Text, Message = this.MessageTextBox.Text, PhotoUrl = blob.Uri.ToString(), ThumbnailUrl = blob.Uri.ToString() };
    GuestBookDataSource ds = new GuestBookDataSource();
    ds.AddGuestBookEntry( entry );                
  }

  this.NameTextBox.Text = string.Empty;
  this.MessageTextBox.Text = string.Empty;
  this.DataList1.DataBind();
}
```
------------------------------------------------------
## #7 ##
```cs
protected void Timer1_Tick( object sender, EventArgs e )
{
  this.DataList1.DataBind();
}
```
-------------------------------------------------------
## #8 ##
```cs
protected void Page_Load( object sender, EventArgs e )
{
  if( !Page.IsPostBack )
  {
    this.Timer1.Enabled = true;
  }
} 
```
-------------------------------------------------------
## #9 ##
```cs
queueStorage = storageAccount.CreateCloudQueueClient();
CloudQueue queue = queueStorage.GetQueueReference( "guestthumbs" );
queue.CreateIfNotExists();
```
-------------------------------------------------------
## #10 ##
```cs
var queue = queueStorage.GetQueueReference( "guestthumbs" );
var message = new CloudQueueMessage( string.Format( "{0},{1},{2}", uniqueBlobName, entry.PartitionKey, entry.RowKey ) );
queue.AddMessage( message );
```
-------------------------------------------------------
## #11 ##
```cs
private CloudQueue queue;
private CloudBlobContainer container;
```
-------------------------------------------------------
## #12 ##
```cs
var storageAccount = CloudStorageAccount.Parse( CloudConfigurationManager.GetSetting( "DataConnectionString" ) );                                    
CloudQueueClient queueStorage = storageAccount.CreateCloudQueueClient();
this.queue = queueStorage.GetQueueReference( "guestthumbs" );
this.queue.CreateIfNotExists();

CloudBlobClient blobStorage = storageAccount.CreateCloudBlobClient();
this.container = blobStorage.GetContainerReference( "guestbookpics" );
this.container.CreateIfNotExists();
var permissions = this.container.GetPermissions();
permissions.PublicAccess = BlobContainerPublicAccessType.Container;
this.container.SetPermissions( permissions );
```
-------------------------------------------------------
## #13 ##
```cs
public void ProcessImage( Stream input, Stream output )
{
  int width, height;
  var originalImage = new Bitmap( input );

  if( originalImage.Width > originalImage.Height )
  {
    width = 128;
    height = 128 * originalImage.Height / originalImage.Width;
  }
  else
  {
    height = 128;
    width = 128 * originalImage.Width / originalImage.Height;
   }
            
  using( var thumbnailImage = new Bitmap( width, height ) )
  using( Graphics graphics = Graphics.FromImage( thumbnailImage ) )
  {
    graphics.InterpolationMode = InterpolationMode.HighQualityBicubic;
    graphics.SmoothingMode = SmoothingMode.AntiAlias;
    graphics.PixelOffsetMode = PixelOffsetMode.HighQuality;
    graphics.DrawImage( originalImage, 0, 0, width, height );
    thumbnailImage.Save( output, ImageFormat.Jpeg );
  }            
}
```
-------------------------------------------------------
## #14 ##
```cs
private async Task RunAsync( CancellationToken cancellationToken )
{
  while( !cancellationToken.IsCancellationRequested )
  {
    CloudQueueMessage msg = this.queue.GetMessage();
    if( msg != null )
    {
      var messageParts = msg.AsString.Split( new char[] { ',' } );
      var imageBlobName = messageParts[0];
      var partitionKey = messageParts[1];
      var rowkey = messageParts[2];

      string thumbnailName = imageBlobName + "thumbnail";

      CloudBlockBlob inputBlob = this.container.GetBlockBlobReference( imageBlobName );
      CloudBlockBlob outputBlob = this.container.GetBlockBlobReference( thumbnailName );

      using( Stream input = inputBlob.OpenRead() )
      using( Stream output = outputBlob.OpenWrite() )
      {
        this.ProcessImage( input, output );                        
        outputBlob.Properties.ContentType = "image/jpeg";
        string thumbnailBlobUri = outputBlob.Uri.ToString();
        GuestBookDataSource ds = new GuestBookDataSource();
        ds.UpdateImageThumbnail( partitionKey, rowkey, thumbnailBlobUri );
        this.queue.DeleteMessage( msg );
      }
    }
    else
    {
      System.Threading.Thread.Sleep( 1000 );
    }
  }
}
```
-------------------------------------------------------
