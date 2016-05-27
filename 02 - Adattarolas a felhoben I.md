# Adattárolás a felhőben I. #
## Snipet #1 ##
`public static class CloudManager`  
`{`  
`   private static readonly string connectionString = ""`  
`   private static CloudStorageAccount account = CloudStorageAccount.Parse( connectionString );`  
`}`  
------------------------------------------------------
## Snipet #2 ##
`public static List<CloudBlobContainer> GetContainers()`  
`{`  
`  var client = account.CreateCloudBlobClient();`  
`  return client.ListContainers().ToList();`  
`}`  
------------------------------------------------------
