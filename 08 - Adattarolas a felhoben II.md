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
using Microsoft.Azure.Documents.Linq; 
using System.Configuration;
using System.Linq.Expressions;
using System.Threading.Tasks;
```
------------------------------------------------------
## #3 ##
```cs
public static class DocumentDBRepository<T>
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
public class CacheEnabledRepository
{
  private DbFakeSource dataSource;
  private ConnectionMultiplexer connection;		
}
```
-------------------------------------------------------
## #11 ##
```cs
public CacheEnabledRepository()
{
  dataSource = new DbFakeSource();
  var connectionString = "azlabcache.redis.cache.windows.net,ssl=true,password=[password]";
  connection = ConnectionMultiplexer.Connect( connectionString );
}
```
-------------------------------------------------------
## #12 ##
```cs
public Product GetProduct( int productId )
{
  
  IDatabase cache = connection.GetDatabase();			  
  var serializedProduct = cache.StringGet( productId.ToString() );
			  
  if( serializedProduct.IsNull )
  {    
    cache.StringSet( productId.ToString(), RedisHelper.GetStringFromProduct( product ) );
    return product;
  }      
  return RedisHelper.GetProductFromString( serializedProduct );  
}
```
-------------------------------------------------------
## #13 ##
```cs
public void InsertProduct( Product p )
{  
  p.ProductId = dataSource.Products.Select( prod => prod.ProductId ).Max() + 1;
			  
  dataSource.Products.Add( p );
			   
  IDatabase cache = connection.GetDatabase();
	cache.StringSet( p.ProductId.ToString(), RedisHelper.GetStringFromProduct( p ) );
}
 
public void UpdateProduc( Product p )
{  
  var productToRemove = dataSource.Products.Single( prod => prod.ProductId == p.ProductId );
  dataSource.Products.Remove( productToRemove );
  dataSource.Products.Add( p );

  IDatabase cache = connection.GetDatabase();
  cache.StringSet( p.ProductId.ToString(), RedisHelper.GetStringFromProduct( p ) );
}
 
public void DeleteProduct( int productId )
{  
  var productToRemove = dataSource.Products.Single( prod => prod.ProductId == productId );
  dataSource.Products.Remove( productToRemove );			
   
  IDatabase cache = connection.GetDatabase();
  cache.StringSet( productId.ToString(), (string)null );
}
```
-------------------------------------------------------