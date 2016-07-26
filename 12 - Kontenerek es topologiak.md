# Konténerek és topológiák #
## #1 ##
```cs
docker info
docker search microsoft
docker pull microsoft/aspnet

powershell

wget https://raw.githubusercontent.com/Microsoft/Virtualization-Documentation/live/windows-server-container-tools/Update-ContainerHost/Update-ContainerHost.ps1 -OutFile Update-ContainerHost.ps1
./Update-ContainerHost.ps1

exit

docker images

docker run microsoft/aspnet cmd

docker ps

docker ps -a

docker run --name myaspnet -i -t microsoft/aspnet cmd

mkdir \dockerrox

docker ps -a
```
------------------------------------------------------

## #2 ##
```cs

docker rm konténer_neve

docker ps -a

docker start -i myaspnet

docker ps

docker attach myaspnet

```
------------------------------------------------------

## #3 ##
```cs
dnvm
dnu


dir \
powershell
wget https://github.com/aspnet/Home/archive/dev.zip -OutFile dev.zip

Expand-Archive dev.zip -DestinationPath \dockerrox
cd C:\dockerrox\Home-dev\samples\1.0.0-rc1-final\
Copy-Item .\HelloMvc -Destination c:\HelloMvc -Recurse

cd c:\HelloMvc
dnu restore

cd \
dnx -p c:\HelloMVC web

docker commit myaspnet local/hellomvc

docker run --name hellomvc -p 80:5004 -it local/hellomvc dnx -p c:\HelloMVC web

```
------------------------------------------------------
