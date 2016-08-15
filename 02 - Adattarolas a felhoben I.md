# Adattárolás a felhőben I. #
## #1 ##
```cs
public static class CloudManager
{
   private static readonly string connectionString = ""
   private static CloudStorageAccount account = CloudStorageAccount.Parse( connectionString );
}
```
------------------------------------------------------
## #2 ##
```cs
public static List<CloudBlobContainer> GetContainers()
{
  var client = account.CreateCloudBlobClient();
  return client.ListContainers().ToList();
}
```
------------------------------------------------------
## #3 ##
```cs
private void PopulateContainers()
{
  lvContainers.Items.Clear();
  foreach (var container in CloudManager.GetContrainers())
  {
     lvContainers.Items.Add(new ListViewItem {Text=container.Name, Tag=container});
  }
}
```
------------------------------------------------------
## #4 ##
```cs
private void Form1_Load( object sender, EventArgs e )
{
  PopulateContainers();
}
```
------------------------------------------------------
## #5 ##
```cs
public static void CreateContainer( string containerName )
{
  var client = account.CreateCloudBlobClient();
  var container = client.GetContainerReference( containerName );
  container.CreateIfNotExists();
  container.SetPermissions( new BlobContainerPermissions { PublicAccess = BlobContainerPublicAccessType.Off } );
}
```
------------------------------------------------------
## #6 ##
```cs
private void createContainerToolStripMenuItem_Click( object sender, EventArgs e )
{
  string containerName = Prompt.ShowDialog( "Container name", "Create container" );
  CloudManager.CreateContainer( containerName );            
  PopulateContainers();
}
```
-------------------------------------------------------
## #7 ##
```cs
public static void UploadBlob( string file, CloudBlobContainer container )
{
  var blob = container.GetBlockBlobReference( Path.GetFileName( file ) );
  blob.UploadFromFile( file,FileMode.Open );
}
```
-------------------------------------------------------
## #8 ##
```cs
private void uploadBlobToolStripMenuItem_Click( object sender, EventArgs e )
{
  if( lvContainers.SelectedItems.Count != 1 )
     return;

  OpenFileDialog getFileDialog = new OpenFileDialog();
  if( getFileDialog.ShowDialog() == DialogResult.OK )
  {
         CloudManager.UploadBlob( getFileDialog.FileName, (CloudBlobContainer)lvContainers.SelectedItems[0].Tag );
  }
}
```
-------------------------------------------------------
## #9 ##
```cs
public static List<IListBlobItem> GetBlobs( CloudBlobContainer container )
{
  return container.ListBlobs().ToList();
}
```
-------------------------------------------------------
## #10 ##
```cs
private void PopulateBlobs()
{
    lvBlobs.Items.Clear();
    if (lvContainers.SelectedItems.Count != 0)
    {
        var blobs = CloudManager.GetBlobs((CloudBlobContainer) lvContainers.SelectedItems[0].Tag);
        foreach (var blob in blobs)
        {
            lvBlobs.Items.Add(new ListViewItem {Text = blob.Uri.ToString(), Tag = blob.Uri});
        }
    }
}
```
-------------------------------------------------------
## #11 ##
```cs
private void lvContainers_SelectedIndexChanged( object sender, EventArgs e )
{
  PopulateBlobs();
}
```
-------------------------------------------------------
## #12 ##
```cs
private void uploadBlobToolStripMenuItem_Click( object sender, EventArgs e )
{
  if( lvContainers.SelectedItems.Count != 1 )
  {
     return;
  }
  OpenFileDialog getFileDialog = new OpenFileDialog();
  if( getFileDialog.ShowDialog() == DialogResult.OK )
  {
         CloudManager.UploadBlob( getFileDialog.FileName, 
                                  (CloudBlobContainer)lvContainers.SelectedItems[0].Tag );
  }
  PopulateBlobs();
}
```
-------------------------------------------------------
## #13 ##
```cs
public static byte[] DownloadBlob(Uri uri)
{
  var client = account.CreateCloudBlobClient();
  var blob=client.GetBlobReferenceFromServer( uri );
  blob.FetchAttributes();
  var result = new byte[blob.Properties.Length];
  blob.DownloadToByteArray( result, 0 );
  return result;
}
```
-------------------------------------------------------
## #14 ##
```cs
private void lvBlobs_DoubleClick(object sender, EventArgs e)
{
    if (lvBlobs.SelectedItems.Count != 1) return;

    var selectedItem = lvBlobs.SelectedItems[0];
    var uri = (Uri)selectedItem.Tag;
    SaveFileDialog saveFile = new SaveFileDialog()
    {
        FileName = Path.GetFileName(uri.ToString())
    };
    if (saveFile.ShowDialog() == DialogResult.OK)
    {
        var contents = CloudManager.DownloadBlob(uri);
        File.WriteAllBytes(saveFile.FileName, contents);
    }
}
```
-------------------------------------------------------
## #15 ##
```cs
public class Item : TableEntity
{    
    [JsonProperty(PropertyName="name")]
    public string Name { get; set; }

    [JsonProperty(PropertyName = "desc")]
    public string Description { get; set; }

    [JsonProperty(PropertyName="isComplete")]
    public bool Completed { get; set; }    
}
```
-------------------------------------------------------
## #16 ##
```cs
@foreach( var item in Model )
{
  <tr>
    <td>
       @Html.DisplayFor( modelItem => item.Name )
    </td>
    <td>
       @Html.DisplayFor( modelItem => item.Description )
    </td>
    <td>
      @Html.DisplayFor( modelItem => item.Completed )
    </td>
     <td>
       @Html.ActionLink( "Edit", "Edit", new { id = item.RowKey } )             
     </td>
  </tr>
}
```
-------------------------------------------------------
## #17 ##
```cs
<h4>Item</h4>
<hr />
@Html.ValidationSummary( true, "", new { @class = "text-danger" } )        
@Html.HiddenFor( model => model.PartitionKey );
@Html.HiddenFor( model => model.RowKey );
```
-------------------------------------------------------
## #18 ##
```cs
public static class TableRepository
{
  private static readonly string connectionString = "";
  private static CloudStorageAccount account = CloudStorageAccount.Parse( connectionString );

  private static CloudTable todoTable;
  private static CloudTable ToDoTable
  {
      get
      {
         if( todoTable == null )
         {
            var client = account.CreateCloudTableClient();
            todoTable = client.GetTableReference( "TodoItems" );
            todoTable.CreateIfNotExists();
        }
      return todoTable;
   }
}
```
-------------------------------------------------------
## #19 ##
```cs
public static IEnumerable<Item> GetIncompleteItems()
{
  TableQuery<Item> query = new TableQuery<Item>().Where
  (
    TableQuery.GenerateFilterConditionForBool( "Completed", QueryComparisons.Equal, false )
   );
  return ToDoTable.ExecuteQuery( query ).ToList();            
}
```
-------------------------------------------------------
## #20 ##
```cs
public static Item CreateItem( Item item )
{
  item.PartitionKey = DateTime.Today.ToString();
  item.RowKey = Guid.NewGuid().ToString();
  TableOperation insertOperation = TableOperation.Insert( item );
  ToDoTable.Execute( insertOperation );
  return item;       
}
```
-------------------------------------------------------
## #21 ##
```cs
public ActionResult Index()
{
  var items = TableRepository.GetIncompleteItems();
  return View( items );
}
```
-------------------------------------------------------
## #22 ##
```cs
public ActionResult Create()
{
  return View();
}

[HttpPost]        
public ActionResult Create(Item item )
{
  if( ModelState.IsValid )
  {
     await TableRepository.CreateItem( item );
     return RedirectToAction( "Index" );
  }
  return View( item );
}
```
-------------------------------------------------------
## #23 ##
```cs
public static Item GetItem( string rowKey )
{
   TableOperation retrieveOperation = TableOperation.Retrieve<Item>( DateTime.Today.ToString(), rowKey );
   var result = ToDoTable.Execute( retrieveOperation ).Result;
  return (Item)result;
}

public static Item UpdateItem( Item item )
{
  item.ETag = "*";
  TableOperation updateOperation = TableOperation.Replace( item );
  ToDoTable.Execute( updateOperation );
  return item;
} 
```
-------------------------------------------------------
## #24 ##
```cs
public ActionResult Edit( string id )
{          
  Item item = TableRepository.GetItem( id );
  if( item == null )
  {
    return HttpNotFound();
  }
  return View( item );
}

[HttpPost]        
public async Task<ActionResult> Edit(  Item item )
{
  if( ModelState.IsValid )
  {
    TableRepository.UpdateItem( item );
    return RedirectToAction( "Index" );
  }
  return View( item );
}
```
-------------------------------------------------------
## #25 ##
```cs
public static class Prompt
{
  public static string ShowDialog( string text, string caption )
  {
    Form prompt = new Form()
    {
      Width = 500,
      Height = 150,
      FormBorderStyle = FormBorderStyle.FixedDialog,
      Text = caption,
      StartPosition = FormStartPosition.CenterScreen
    };
    Label textLabel = new Label() { Left = 50, Top = 20, Text = text };
    TextBox textBox = new TextBox() { Left = 50, Top = 50, Width = 400 };
    Button confirmation = new Button() { Text = "Ok", 
                                         Left = 350, Width = 100, Top = 70, 
                                         DialogResult = DialogResult.OK 
                                       };
    confirmation.Click += ( sender, e ) => { prompt.Close(); };
    prompt.Controls.Add( textBox );
    prompt.Controls.Add( confirmation );
    prompt.Controls.Add( textLabel );
    prompt.AcceptButton = confirmation;
    return prompt.ShowDialog() == DialogResult.OK ? textBox.Text : "";
  }
}
```
-------------------------------------------------------
