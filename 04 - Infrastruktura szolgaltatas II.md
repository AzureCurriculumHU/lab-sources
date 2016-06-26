# Infrastruktúra szolgáltatás II. - Üzemeltetés #
## #1 ##
```cs
dism /online /enable-feature /all /featurename:IIS-ASPNET45
del C:\inetpub\wwwroot\iisstart.htm
echo "<%@ Page Language=""C#"" %><%=System.Net.Dns.GetHostName() %>">C:\inetpub\wwwroot\default.aspx
```
------------------------------------------------------
## #2 ##
```json
,
"scriptFile": {
      "type": "string",
      "metadata": {
        "description": "The script file location."
      }
    },
    "scriptName": {
      "type": "string",
      "metadata": {
        "description": "Name of the script file"
      }
    }

```
------------------------------------------------------
## #3 ##
```json
,
{
  "type": "Microsoft.Compute/virtualMachines/extensions",
  "name": "MyCustomScriptExtension",
  "location": "[resourceGroup().location]",
  "properties": {
    "publisher": "Microsoft.Compute",
    "type": "CustomScriptExtension",
    "typeHandlerVersion": "1.2",
    "settings": {
      "fileUris": [ "[parameters('scriptFile')]" ],
      "commandToExecute": "[concat('powershell -ExecutionPolicy Unrestricted -file ',parameters('scriptName'))]"
    }
  }
}
```
------------------------------------------------------
## #4 ##
```json
,
        "loadBalancingRules": [
          {
            "properties": {
              "frontendIPConfiguration": { "id": "[variables('frontEndIPConfigID')]"},
              "backendAddressPool": { 
               "id":"[concat(variables('lbID'),'/backendAddressPools/', variables('bePoolName'))]"},
              "probe": { "id": "[concat(variables('lbID'), '/probes/lbprobe')]"},
              "protocol": "Tcp",
              "frontendPort": 80,
              "backendPort": 80,
              "idleTimeoutInMinutes": 15
            },
            "name": "lbrule"
          }
        ],
        "probes": [
          {
            "properties": {
              "protocol": "Tcp",
              "port": 80,
              "intervalInSeconds": 15,
              "numberOfProbes": 2
            },
            "name": "lbprobe"
          }
        ]
```
------------------------------------------------------
## #5 ##
```cs
Login-AzureRmAccount
Get-AzureRMSubscription
Get-AzureRmSubscription -SubscriptionName "{előfizetés neve}" | Select-AzureRmSubscription
Remove-AzureRmResource -ResourceId /subscriptions/{előfizetés id}/resourceGroups/{erőforrás csoport} -ApiVersion 2014-04-01 -Force
```
