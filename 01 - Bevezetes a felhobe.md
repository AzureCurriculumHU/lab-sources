# Bevezetés a felőbe. #
## #1 ##
```cs
Add-AzureAccount
Get-AzureSubscription
Select-AzureSubscription "Azure Pass"
Get-AzureService
Get-AzureDeployment -Slot Staging -ServiceName lab04cloudservice
Move-AzureDeployment -ServiceName lab04cloudservice
Get-AzureDeployment -ServiceName lab04cloudservice
```
------------------------------------------------------
## #2 ##
```cs
azure
azure login
azure login -u microsoft@account.com
azure account set "Azure Pass"
azure storage account list
azure storage account delete lab04cloudservice
azure storage account delete mysttorageaccountkb
```
