# Adattárolás a felhőben II. #
## #1 ##
```cs
public class Item
{
    [JsonProperty(PropertyName="id")]
    public string Id { get; set; }

    [JsonProperty(PropertyName="name")]
    public string Name { get; set; }

    [JsonProperty(PropertyName = "desc")]
    public string Description { get; set; }

    [JsonProperty(PropertyName="isComplete")]
    public bool Completed { get; set; }    
}
```
------------------------------------------------------
## #2 ##
```cs
using Microsoft.Azure.Documents; 
using Microsoft.Azure.Documents.Client; 
using System.Threading.Tasks;
```
------------------------------------------------------
## #3 ##
```cs
public static class DocumentDBRepository
{
  private static string databaseId = "ToDoDatabase";
  private static string collectionId = "Items";
  private static string endpoint = "";
  private static string authKey = "";

  private static DocumentClient client = new DocumentClient( new Uri( endpoint ), authKey );

  private static Database ReadOrCreateDatabase()
  {
    var db = client.CreateDatabaseQuery().AsEnumerable().Where( d => d.Id == databaseId ).FirstOrDefault();
    if( db == null )
    {
      db = client.CreateDatabaseAsync( new Database { Id = databaseId } ).Result;
    }
    return db;
  }
  
  private static DocumentCollection ReadOrCreateCollection( string databaseLink )
  {
    var col = client.CreateDocumentCollectionQuery( databaseLink ).Where( c => c.Id == collectionId ).AsEnumerable().FirstOrDefault();
    if( col == null )
    {
       var collectionSpec = new DocumentCollection { Id = collectionId };                
       col = client.CreateDocumentCollectionAsync( databaseLink, collectionSpec ).Result;
    }
    return col;
  }

  private static Database database;
  private static Database Database
  {
      get
      {
         if( database == null ) database = ReadOrCreateDatabase();
         return database;
     }
  }
  
  private static DocumentCollection collection;
  private static DocumentCollection Collection
  {
    get
    {
       if( collection == null )  collection = ReadOrCreateCollection( Database.SelfLink );
       return collection;
    }
  }                               
}
```
------------------------------------------------------
## #4 ##
```cs
public static IEnumerable<Item> GetIncompleteItems()
{
  return client.CreateDocumentQuery<Item>( Collection.DocumentsLink ).Where( item=>!item.Completed).AsEnumerable().ToList();                
}
```
------------------------------------------------------
## #5 ##
```cs
public static async Task<Document> CreateItemAsync( Item item )
{
  return await client.CreateDocumentAsync( Collection.SelfLink, item );
}
```
------------------------------------------------------
## #6 ##
```cs
public ActionResult Index()
{
  var items = DocumentDBRepository.GetIncompleteItems();
  return View( items );
}
```
-------------------------------------------------------
## #7 ##
```cs
public ActionResult Create()
{
  return View();
}

[HttpPost]        
public async Task<ActionResult> Create(Item item )
{
  if( ModelState.IsValid )
  {
    await DocumentDBRepository.CreateItemAsync( item );
    return RedirectToAction( "Index" );
  }
  return View( item );
}
```
-------------------------------------------------------
## #8 ##
```cs
public static Item GetItem( string id )
{
  return client.CreateDocumentQuery<Item>( Collection.DocumentsLink ).Where( i => i.Id == id ).AsEnumerable().FirstOrDefault();
}

public static async Task<Document> UpdateItemAsync( string id, Item item )
{
  var doc = client.CreateDocumentQuery( Collection.DocumentsLink ).Where( i => i.Id == id ).AsEnumerable().FirstOrDefault();
  return await client.ReplaceDocumentAsync( doc.SelfLink, item );
}
```
-------------------------------------------------------
## #9 ##
```cs
public ActionResult Edit( string id )
{          
  Item item = (Item)DocumentDBRepository.GetItem( id );
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
    await DocumentDBRepository.UpdateItemAsync( item.Id, item );
   return RedirectToAction( "Index" );
  }
  return View( item );
}
```
-------------------------------------------------------
## #10 ##
```cs
public static class CacheManager
{
    private const string connectionString = "(A portálról letöltött connection string.)";
    private static readonly ConnectionMultiplexer connection = ConnectionMultiplexer.Connect(connectionString);

    public static string GetValue(int key)
    {
        IDatabase cache = connection.GetDatabase();
        var value = cache.StringGet(key.ToString());
        if (value.IsNull)
        {
            return null;
        }
        else
        {
            return value.ToString();
        }
    }

    public static void SetValue(int key, string newValue)
    {
        IDatabase cache = connection.GetDatabase();
        cache.StringSet(key.ToString(), newValue);
    }
}
```
-------------------------------------------------------
## #11 ##
```cs
static void Main(string[] args)
{
    Console.WriteLine("Add meg a kulcsot! (1-10)");
    int key = int.Parse(Console.ReadLine());
    string val = CacheManager.GetValue(key);
    if (val == null)
    {
        Console.WriteLine("Nincs tárolt elem!");
    }
    else
    {
        Console.WriteLine($"Tárolt érték: {val}");
    }

    Console.WriteLine("Add meg az új értéket!");
    string newValue = Console.ReadLine();
    CacheManager.SetValue(key, newValue);
    Console.WriteLine("Nyomj meg egy billentyűt a kilépéshez!");
    Console.ReadKey();
}
```
-------------------------------------------------------
