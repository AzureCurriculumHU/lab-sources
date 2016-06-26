# Fejlesztői szolgáltatások
## #1 ##
```cs
var tc = new TelemetryClient();
var resultText = result.Succeeded ? "Success" : "ERROR: " + result.Errors.First();
var properties = new Dictionary<string, string> { {"Email", model.Email }, {"Result", resultText} };
var metrics = new Dictionary<string, double> { {"Registration Success", result.Succeeded ? 1 : 0} };
tc.TrackEvent("Registration", properties, metrics);
```
------------------------------------------------------
## #2 ##
```cs
public ActionResult Contact()
{
    ViewBag.Message = "Your contact page.";

    using (var db = new ApplicationDbContext())
    {
        db.Users.First().Roles.Add(new IdentityUserRole() { RoleId = "valami hibás" });
        db.SaveChanges();
    }

    return View();
}
```
------------------------------------------------------
## #3 ##
```xaml
<StackPanel>
    <Button Content="Hiba!" Click="Error_OnClicked"/>
</StackPanel>
```
------------------------------------------------------
## #4 ##
```cs
private void Error_OnClicked(object sender, RoutedEventArgs e)
{
    throw new Exception("teszt hiba!");
}
```
------------------------------------------------------
## #5 ##
```xaml
<Button Content="Észrevétel küldése" Click="Feedback_OnClick"/>
```
------------------------------------------------------
## #6 ##
```cs
private void Feedback_OnClick(object sender, RoutedEventArgs e)
{
    HockeyClient.Current.ShowFeedback();
}
```
------------------------------------------------------
## #7 ##
```cs
protected override void OnActivated(IActivatedEventArgs args)
{
    HockeyClient.Current.HandleReactivationOfFeedbackFilePicker(args);
}   
```
------------------------------------------------------