# Többplatformos mobilalkalmazások
## #1 ##
```cs
private MessageDialog dialog = new MessageDialog("");
private bool isOpen = false;
```

## #2 ##
```cs
private async Task ShowDialogAsync(string content, string title = "Hiba")
{
    if (isOpen)
    {
        return;
    }
    dialog.Title = title;
    dialog.Content = content;
    isOpen = true;
    await dialog.ShowAsync();
    isOpen = false;
}
```

## #3 ##
```cs
private async Task SyncAsync()
{
    string content;
    try
    {
        await App.MobileService.SyncContext.PushAsync();
        await todoTable.PullAsync("todoItems", todoTable.CreateQuery());
    }
    catch (MobileServicePushFailedException ex)
    {
        content = "Szinkronizáció nem sikerült, lehet, hogy kapcsolat nélküli állapotban vagy.\n" + 
           "Üzenet: " + ex.Message + "\nÁllapot: " + ex.PushResult.Status.ToString();
        await ShowDialogAsync(content);
    } 
    catch (Exception ex)
    {
        content = "Szinkronizáció sikertelen " + ex.Message +
                  "\n\nHa még mindig kapcsolat nélküli állapotban vagy, " +
                  "próbálj frissíteni a kapcsolat felépítését követően.";

        await ShowDialogAsync(content);
    }
}
```
------------------------------------------------------
## #4 ##
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
## #5 ##
```cs
private async Task InitNotificationsAsync()
{
    var channel = await PushNotificationChannelManager.CreatePushNotificationChannelForApplicationAsync();
    await MobileService.GetPush().RegisterAsync(channel.Uri, null);
}
```
------------------------------------------------------
