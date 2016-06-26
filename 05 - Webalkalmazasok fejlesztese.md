# Webalkalmazások fejlesztése #
## #1 ##
```cs
public static void Resize([BlobTrigger("images/{name}")] WebImage input, [Blob("images-resized/{name}")] out WebImage output)
{
  var width = 180;
  var height = Convert.ToInt32(input.Height * 180 / input.Width);
  output = input.Resize(width, height);
}
```
------------------------------------------------------
## #2 ##
```cs
public class WebImageBinder : ICloudBlobStreamBinder<WebImage>
{
  public Task<WebImage> ReadFromStreamAsync(Stream input, CancellationToken cancellationToken)
  {
     return Task.FromResult<WebImage>(new WebImage(input));
  }
  
  public Task WriteToStreamAsync(WebImage value, Stream output, CancellationToken cancellationToken)
  {
     var bytes = value.GetBytes();
     return output.WriteAsync(bytes, 0, bytes.Length, cancellationToken);
  }
}
```
------------------------------------------------------
## #3 ##
```cs
<connectionStrings>
    <add name="AzureWebJobsDashboard" connectionString="DefaultEndpointsProtocol=https;AccountName=[accountname];AccountKey=[accesskey]"/>
    <add name="AzureWebJobsStorage" connectionString="DefaultEndpointsProtocol=https;AccountName=[accountname];AccountKey=[accesskey]"/>
</connectionStrings>
```
------------------------------------------------------
## #4 ##
```cs
<div class="row">
  <div class="col-md-4">
  @using( Html.BeginForm( "Upload", "Home", FormMethod.Post, new { enctype = "multipart/form-data" } ) )
  {
     <input type="file" name="file" />
     <input type="submit" name="Submit" id="Submit" value="Upload" />
  }
  </div>
</div>
```
------------------------------------------------------
## #5 ##
```cs
private static readonly string connectionString = ""
private static CloudStorageAccount account = CloudStorageAccount.Parse( connectionString );       

[HttpPost]
public ActionResult Upload( HttpPostedFileBase file )
{
  var client = account.CreateCloudBlobClient();
  var container = client.GetContainerReference( "images" );
  container.CreateIfNotExists();
  container.SetPermissions( new BlobContainerPermissions { 
                                         PublicAccess = BlobContainerPublicAccessType.Off } );
  var blob = container.GetBlockBlobReference( file.FileName );
  var bytes = new byte[file.InputStream.Length];
  file.InputStream.Read( bytes, 0, (int)file.InputStream.Length );
  blob.UploadFromByteArray( bytes , 0, bytes.Length );
  return RedirectToAction( "Index" );
}
```
------------------------------------------------------
## #6 ##
```javascript
var http = require('http')
var port = process.env.PORT || 1337;
http.createServer(function(req, res) {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello Node\n');
}).listen(port);

```
-------------------------------------------------------