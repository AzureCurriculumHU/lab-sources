# Infrastruktúra szolgáltatás I. - Gépek a felhőben #
## #1 ##
```cs
# Login your account
Login-AzureRmAccount

# List your suubscriptions
Get-AzureRMSubscription | Sort SubscriptionName | Select SubscriptionName

#Select Azure-Pass subscription
$subscr="Azure Pass"
Get-AzureRmSubscription –SubscriptionName $subscr | Select-AzureRmSubscription 

# Create custom postfix for resource's names
$date = Get-Date
$postifx = $date.ToString("yymmddMMHH") 

# Set resourcegroup, location and storge account name
$rgName="LinuxRG"+$postifx
$locName="North Europe"
$saName="linuxsa"+$postifx 

# Create Resource group
New-AzureRmResourceGroup -Name $rgName -Location $locName 

# Create StorageAccunt
$saType="Standard_LRS"
New-AzureRmStorageAccount -Name $saName -ResourceGroupName $rgName –Type $saType -Location $locName

# Set virtual network informations
$vnetName="LinuxVNet"+$postifx
$subNetName = "FrontEnd"
$addressPrefix = "192.168.0.0/16"
$subnetIndex=0

# Create virtual network
New-AzureRmVirtualNetwork -ResourceGroupName $rgName -Name $vnetName -AddressPrefix $addressPrefix -Location $locName   
$vnet=Get-AzureRMVirtualNetwork -Name $vnetName -ResourceGroupName $rgName

# Create subnet
$subnetAddressPrefix = "192.168.1.0/24"
Add-AzureRmVirtualNetworkSubnetConfig -Name $subNetName -VirtualNetwork $vnet -AddressPrefix $subnetAddressPrefix

# Upload VNet's changes
Set-AzureRmVirtualNetwork -VirtualNetwork $vnet 

# Get virtual network
$vnet=Get-AzureRMVirtualNetwork -Name $vnetName -ResourceGroupName $rgName

# Create the NIC
$nicName="NIC1"
$domName="ubuntu-vm-nic1-"+$postifx
$pip=New-AzureRmPublicIpAddress -Name $nicName -ResourceGroupName $rgName -DomainNameLabel $domName -Location $locName -AllocationMethod Dynamic
$nic=New-AzureRmNetworkInterface -Name $nicName -ResourceGroupName $rgName -Location $locName -SubnetId $vnet.Subnets[$subnetIndex].Id -PublicIpAddressId $pip.Id

# Specify the name, size
$vmName="UbuntuServer-"+$postifx
$vmSize="Standard_A3"
$vm=New-AzureRmVMConfig -VMName $vmName -VMSize $vmSize

# Add a 100 GB additional data disk
$diskSize=200
$diskLabel="DataStorage"
$diskName="DISK02"
$storageAcc=Get-AzureRmStorageAccount -ResourceGroupName $rgName -Name $saName
$vhdURI=$storageAcc.PrimaryEndpoints.Blob.ToString() + "vhds/" + $vmName + $diskName  + ".vhd"
Add-AzureRmVMDataDisk -VM $vm -Name $diskLabel -DiskSizeInGB $diskSize -VhdUri $vhdURI -CreateOption empty

#Find all the available publishers
Get-AzureRmVMImagePublisher -Location $locName 

# Specify the image and local administrator account, and then add the NIC
$pubName="Canonical"
$offerName="UbuntuServer"
$skuName="15.10"
$cred=Get-Credential -Message "Type the name and password of the local administrator account. (Don't use admin for username and simple password! (Example: ubuntuuser, Passowrd1_)"
$vm=Set-AzureRmVMOperatingSystem -VM $vm -Linux -Credential $cred -ComputerName $vmName
$vm=Set-AzureRmVMSourceImage -VM $vm -PublisherName $pubName -Offer $offerName -Skus $skuName -Version "latest"
$vm=Add-AzureRmVMNetworkInterface -VM $vm -Id $nic.Id

# Specify the OS disk name and create the VM
$diskName="OSDisk"
$storageAcc=Get-AzureRmStorageAccount -ResourceGroupName $rgName -Name $saName
$osDiskUri=$storageAcc.PrimaryEndpoints.Blob.ToString() + "vhds/" + $vmName + $diskName  + ".vhd"
$vm=Set-AzureRmVMOSDisk -VM $vm -Name $diskName -VhdUri $osDiskUri -CreateOption fromImage
New-AzureRmVM -ResourceGroupName $rgName -Location $locName -VM $vm 

```
------------------------------------------------------
## #2 ##
```cs
# Login your account
Login-AzureRmAccount

# Create custom postfix for resource's names
$date = Get-Date
$postifx = $date.ToString("yymmddMMHH")

#Select Azure-Pass subscription
$subscr="Azure Pass"
Get-AzureRmSubscription –SubscriptionName $subscr | Select-AzureRmSubscription

# Set resourcegroup, location and storge account name
$rgName="LinuxRG"+$postifx
$locName="North Europe"
$saName="linuxsa"+$postifx

# Create Resource group
New-AzureRmResourceGroup -Name $rgName -Location $locName

# Create StorageAccunt
$saType="Standard_LRS"
New-AzureRmStorageAccount -Name $saName -ResourceGroupName $rgName –Type $saType -Location $locName

# Change your VHD'path 
$sourceVHD = "D:\Downloads\dsl-4.4.6-x86\VDI\dsl-4_vhd.vhd"
$imageName = "dawn-small-linux" 

$storageAcc = Get-AzureRmStorageAccount -ResourceGroupName $rgName -Name $saName
$destionationVHD = $storageAcc.PrimaryEndpoints.Blob.ToString() + "vhds/" + $imageName + ".vhd" 

Add-AzureRmVhd -ResourceGroupName $rgName -Destination $destionationVHD -LocalFilePath $sourceVHD 

# Specify the image and local administrator account, and then add the NIC
#$pubName="Canonical"
#$offerName="UbuntuServer"
#$skuName="15.10"
#$cred=Get-Credential -Message "Type the name and password of the local administrator account. (Don't use admin for username and simple password! (Example: ubuntuuser, Passowrd1_)"
#$vm=Set-AzureRmVMOperatingSystem -VM $vm -Linux -Credential $cred -ComputerName $vmName
#$vm=Set-AzureRmVMSourceImage -VM $vm -PublisherName $pubName -Offer $offerName -Skus $skuName -Version "latest"
$vm=Add-AzureRmVMNetworkInterface -VM $vm -Id $nic.Id

# Specify the OS disk name and create the VM
#$diskName="OSDisk"
#$storageAcc=Get-AzureRmStorageAccount -ResourceGroupName $rgName -Name $saName
#$osDiskUri=$storageAcc.PrimaryEndpoints.Blob.ToString() + "vhds/" + $vmName + $diskName  + ".vhd"
$osDiskUri = "https://linuxsa1617150412.blob.core.windows.net/vhds/dawn-small-linux.vhd" 

#$vm=Set-AzureRmVMOSDisk -VM $vm -Name $diskName -VhdUri $osDiskUri -CreateOption fromImage
$vm = Set-AzureRmVMOSDisk -VM $vm -VhdUri $osDiskUri -name $diskName -CreateOption attach -Linux -Caching ReadWrite 

```
