

# Demo Steps

### Get Ready with Azure CLI
Download and install Azure CLI
> https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest

## Demo 1
### Create a new container group with Azure CLI
- az group create
- az container create
- az container show
- az container logs
- az group delete


```ps
# declare variables
$resourceGroup = "acidemoRG"
$location = "westeurope"

# create resource group
az group create -n $resourceGroup -l $location

$containerGroupName = "ghost-blog1"

# create container with ghost image
az container create -g $resourceGroup `
        -n $containerGroupName --image ghost `
        -ports 2368 –ip-address public --dns-name-label ghostaci

# get container details
az container show -g $resourceGroup –n $containerGroupName

# get conteiner logs
az container logs –g $resourceGroup -n $containerGroupName

# delete resource group
az group delete -n $resourceGroup -y

```


## Demo 2
### Create a Windows container and host a dotnet app with additional params:
- --os-type
- --memory
- --cpu
- --restart

```ps
# declare variables
$resourceGroup = "acidemoRG"
$location = "westeurope"

# create resource group
az group create -n $resourceGroup -l $location

# create container with miniblog image
$containerGroupName = "minblog-win" 

az container create -g $resourceGroup `
    --image markheath/miniblogcore:v1 `
    --ip-address public `
    --dns-name-label miniblog-win `
    --os-type windows `
    --memory 2 --cpu 2 `
    --restart-policy OnFailure



# get container details
az container show -g $resourceGroup –n $containerGroupName

# get conteiner logs
az container logs –g $resourceGroup -n $containerGroupName

# delete resource group
az group delete -n $resourceGroup -y

```



## Demo 3
### Run an image from a private container repository
- create a private repository
- push custom image into the repository
- run image from private repository in ACI



```ps
# declare variables
$resourceGroup = "acidemoRG"
$location = "westeurope"

# create resource group
az group create -n $resourceGroup -l $location

# create Azure Container Repository
$acrName = "demoacr"
az acr create -g $resourceGroup -n $acrName `
    --sku Basic --admin-enabled true

# get Azure Container Repository password
$acrPassword = az acr credential show -n $acrName `
    --query "passwords[0].value" -o tsv

# get login url for Azure Container Repository
$loginServer = az acr show -n $acrName `
    --query loginServer -o tsv


# login into private container repository using the credentials
docker login -u $acrName -p $acrPassword $loginServer

# declare custom image tag
$image = "mystaticsite:v1"
$imageTag = "$loginServer/$image"

# push the image to our registry
docker push $imageTag


# see what images are in our registry
az acr repository list -n $acrName --output table

# create a new container group using the image from the private registry

$containerGroupName = "aci-from-private-repository"

az container create -g $resourceGroup `
    -n $containerGroupName `
    --image $imageTag --cpu 1 --memory 1 `
    --registry-username $loginServer `
    --registry-password $acrPassword `
    --dns-name-label "aciacr" --ports 80


# get the site address and launch in a browser
$fqdn = az container show -g $resourceGroup -n $containerGroupName `
    --query ipAddress.fqdn -o tsv

Start-Process "http://$($fqdn)"


# view the logs for our container
az container logs -n $containerGroupName -g $resourceGroup

# delete the resource group (ACR and container group)
az group delete -n $resourceGroup -y

```


## Demo 4
### Mounting Volumes


```ps
$resourceGroup = "AciVolumeDemo"
$location = "westeurope"
az group create -n $resourceGroup -l $location

$storageAccountName = "acishare$(Get-Random `
    -Minimum 1000 -Maximum 10000)"

# create a storage account
az storage account create -g $resourceGroup `
    -n $storageAccountName `
    --sku Standard_LRS

# get the connection string for our storage account
$storageConnectionString = `
    az storage account show-connection-string `
    -n $storageAccountName -g $resourceGroup `
    --query connectionString -o tsv
# export it as an environment variable
$env:AZURE_STORAGE_CONNECTION_STRING = $storageConnectionString

# Create the file share
$shareName="acishare"
az storage share create -n $shareName

# upload 
$filename = "sampleVideo.mp4"
$localFile = "D:\$filename"
az storage file upload -s $shareName --source "$localFile"

# get the key for this storage account
$storageKey=$(az storage account keys list `
    -g $resourceGroup --account-name $storageAccountName `
    --query "[0].value" --output tsv)

$containerGroupName = "transcode"

$commandLine = "ffmpeg -i /mnt/azfile/$filename -vf" + `
    " ""thumbnail,scale=640:360"" -frames:v 1 /mnt/azfile/thumb.png"

az container create `
    -g $resourceGroup `
    -n $containerGroupName `
    --image jrottenberg/ffmpeg `
    --restart-policy never `
    --azure-file-volume-account-name $storageAccountName `
    --azure-file-volume-account-key $storageKey `
    --azure-file-volume-share-name $shareName `
    --azure-file-volume-mount-path "/mnt/azfile" `
    --command-line $commandLine

az container logs -g $resourceGroup -n $containerGroupName 

az container show -g $resourceGroup -n $containerGroupName --query provisioningState

az storage file list -s $shareName -o table

$downloadThumbnailPath = "Z:\thumb.png"
az storage file download -s $shareName -p "thumb.png" `
    --dest $downloadThumbnailPath
Start-Process $downloadThumbnailPath

#az container delete -g $resourceGroup -n $containerGroupName

# delete the resource group (file share and container group)
az group delete -n $resourceGroup -y

```
