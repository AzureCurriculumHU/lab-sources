# Többplatformos mobilalkalmazások
## #1 ##
```cs
private async Task SyncAsync()
{
    try
    {
        await App.MobileService.SyncContext.PushAsync();
        await todoTable.PullAsync("todoItems", todoTable.CreateQuery());
    }
    catch (MobileServicePushFailedException ex)
    {
        await new MessageDialog(
            "Szinkronizáció nem sikerült, lehet kapcsolat nélküli állapotban vagy.\nÜzenet: " +
            ex.Message + "\nÁllapot: " + ex.PushResult.Status.ToString())
            .ShowAsync();
    }
    catch (Exception ex)
    {
        await new MessageDialog(
            "Szinkronizáció sikertelen " + ex.Message +
            "\n\nHa még mindig kapcsolat nélküli állapotban vagy, " +
            "próbálj frissíteni, akkor ha él a kapcsolat.")
            .ShowAsync();
    }
}
```
------------------------------------------------------
## #2 ##
```cs
private async Task SendNotification(TodoItem item)
{
    MobileAppSettingsDictionary settings =
        Configuration.GetMobileAppSettingsProvider().GetMobileAppSettings();

    string notificationHubName = settings.NotificationHubName;
    string notificationHubConnection = 
        settings.Connections[MobileAppSettingsKeys.NotificationHubConnectionString].ConnectionString;

    NotificationHubClient hub = NotificationHubClient.CreateClientFromConnectionString(
        notificationHubConnection, notificationHubName);

    var windowsToastPayload = 
        @"<toast><visual><binding template=""ToastText01""><text id=""1"">" +
        item.Text + @" Completed :)</text></binding></visual></toast>";
    try
    {
        var result = await hub.SendWindowsNativeNotificationAsync(windowsToastPayload);
        Configuration.Services.GetTraceWriter().Info(result.State.ToString());
    }
    catch (System.Exception ex)
    {
        Configuration.Services.GetTraceWriter().Error(ex.Message, null, "Push.SendAsync Error");
    }
}
```
------------------------------------------------------
## #3 ##
```cs
private async Task InitNotificationsAsync()
{
    var channel = await PushNotificationChannelManager.CreatePushNotificationChannelForApplicationAsync();
    await MobileService.GetPush().RegisterAsync(channel.Uri, null);
}
```
------------------------------------------------------
