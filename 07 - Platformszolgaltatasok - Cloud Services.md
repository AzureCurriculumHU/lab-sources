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
public class GuestBookDataSource
{
    private static CloudStorageAccount account;

    static GuestBookDataSource()
    {
        account = CloudStorageAccount.Parse(CloudConfigurationManager.GetSetting("DataConnectionString"));
        account.CreateCloudTableClient().GetTableReference("GuestBookEntry").CreateIfNotExists();
    }

    private CloudTable guestBookTable;

    private CloudTable GuestBookTable
    {
        get
        {
            if (guestBookTable == null)
            {
                guestBookTable = account.CreateCloudTableClient().GetTableReference("GuestBookEntry");
            }
            return guestBookTable;
        }
    }

    public IEnumerable<GuestBookEntry> GuestBookEntries()
    {
        TableQuery<GuestBookEntry> query = new TableQuery<GuestBookEntry>()
            .Where(TableQuery.GenerateFilterCondition("PartitionKey", 
                QueryComparisons.Equal, DateTime.UtcNow.ToString("MMddyyyy")));
        return GuestBookTable.ExecuteQuery(query);
    }

    public void AddGuestBookEntry(GuestBookEntry newItem)
    {
        TableOperation operation = TableOperation.Insert(newItem);
        GuestBookTable.Execute(operation);
    }

    public void UpdateImageThumbnail(string partitionKey, string rowKey, string thumbUrl)
    {
        TableOperation retrieveOperation = TableOperation.Retrieve<GuestBookEntry>(partitionKey, rowKey);

        TableResult retrievedResult = GuestBookTable.Execute(retrieveOperation);
        GuestBookEntry updateEntity = (GuestBookEntry)retrievedResult.Result;

        if (updateEntity != null)
        {
            updateEntity.ThumbnailUrl = thumbUrl;
            TableOperation replaceOperation = TableOperation.Replace(updateEntity);
            GuestBookTable.Execute(replaceOperation);
        }
    }
}
```
------------------------------------------------------
## #3 ##
```cs
public static class CloudManager
{
    private static bool storageInitialized = false;
    private static object gate = new object();
    private static CloudBlobClient blobStorage;
    private static CloudQueueClient queueStorage;

    private static void InitializeStorage()
    {
        if (!storageInitialized)
        {
            lock (gate)
            {
                if (!storageInitialized)
                {
                    var storageAccount = CloudStorageAccount.Parse(CloudConfigurationManager
                        .GetSetting("DataConnectionString"));
                    blobStorage = storageAccount.CreateCloudBlobClient();
                    CloudBlobContainer container = blobStorage.GetContainerReference("guestbookpics");
                    container.CreateIfNotExists();

                    var permissions = container.GetPermissions();
                    permissions.PublicAccess = BlobContainerPublicAccessType.Container;
                    container.SetPermissions(permissions);
                    storageInitialized = true;
                }
            }
        }
    }

    public static UploadBlobResult UploadBlob(HttpPostedFileBase file)
    {
        InitializeStorage();
        string uniqueBlobName = string.Format("guestbookpics/image_{0}{1}",
            Guid.NewGuid().ToString(), Path.GetExtension(file.FileName));
        CloudBlockBlob blob = blobStorage.GetContainerReference("guestbookpics")
            .GetBlockBlobReference(uniqueBlobName);
        blob.Properties.ContentType = file.ContentType;
        blob.UploadFromStream(file.InputStream);
        return new UploadBlobResult
        {
            UniqueBlobName = uniqueBlobName,
            Uri = blob.Uri.ToString()
        };
    }
}
```
------------------------------------------------------
## #4 ##
```cs
[HttpPost]
[ValidateAntiForgeryToken]
public ActionResult Index([Bind(Include = "File, Name, Message")]NewGuestBookEntryViewModel vm)
{
    if (vm.File != null && vm.File.ContentLength != 0 && ModelState.IsValid)
    {
        var result = CloudManager.UploadBlob(vm.File);
        GuestBookEntry entry = new GuestBookEntry()
        {
            GuestName = vm.Name,
            Message = vm.Message,
            PhotoUrl = result.Uri,
            ThumbnailUrl = result.Uri
        };
        DataSource.AddGuestBookEntry(entry);
        return RedirectToAction("Index");
    }
    else
    {
        ModelState.AddModelError("", "Fájl feltöltése kötelező!");
        return View(CreateViewModel(vm));
    }
}
```
------------------------------------------------------
## #5 ##
```cs
queueStorage = storageAccount.CreateCloudQueueClient();
CloudQueue queue = queueStorage.GetQueueReference("guestthumbs");
queue.CreateIfNotExists();
```
------------------------------------------------------
## #6 ##
```cs
public static void AddImageToQueue(string uniqueBlobName, string partitionKey, string rowKey)
{
    InitializeStorage();
    var queue = queueStorage.GetQueueReference("guestthumbs");
    var message = new CloudQueueMessage(string.Format("{0},{1},{2}", 
        uniqueBlobName, partitionKey, rowKey));
    queue.AddMessage(message);
}
```
-------------------------------------------------------
## #7 ##
```cs
private CloudQueue queue;
private CloudBlobContainer container;
```
-------------------------------------------------------
## #8 ##
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
## #9 ##
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
## #10 ##
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
