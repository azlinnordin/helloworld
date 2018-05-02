# node-helloworld
Sample app for demos

## Setup a default registry

```sh
export ACR_NAME=jengademos
az configure --defaults acr=$ACR_NAME
```
## Clone the repo
```sh
git clone github.com/
```

## Local Build
```sh
# Build
az acr build -t helloworld:{{.Build.ID}} . 
#List Images
az acr repository show-tags --repository helloworld
```
 Common Environment Variables
```sh
# Replace these values for your configuration
# I've left our values in, as we use this for our demos, providing some examples
export ACR_NAME=jengademos
export RESOURCE_GROUP=$ACR_NAME
# fully qualified url of the registry. 
# This is where your registry would be
# Accounts for registries in dogfood or other clouds like .gov, Germany and China
export REGISTRY_NAME=${ACR_NAME}.azurecr.io/ 
export AKV_NAME=$ACR_NAME-vault # name of the keyvault
export GIT_TOKEN_NAME=stevelasker-git-access-token # keyvault secret name
```

## Create a build task
- Populate your GIT Personal Access Token
  ```sh
  export PAT=$(az keyvault secret show \
                --vault-name $AKV_NAME \
                --name $GIT_TOKEN_NAME \
                --query value -o tsv)
  ```
- Create the build task
  ```sh
  az acr build-task create \
    -n helloworld \
    -c https://github.com/demo42/helloworld \
    -t demo42/helloworld:{{.Build.ID}} \
    --cpu 2 \
    --git-access-token=$PAT
  ```

- Commit a code change
  
  Monitor the current builds
  ```sh
  watch -n1 az acr build-task list-builds 
  ```

- View the current executing builds

  ```sh
  az acr build-task logs
  ```

## Base Image Updates

- Update the base image

  ```sh
  az acr build -t baseimages/node:9 \
    -f node-jessie.Dockerfile \
    .
  ```
## Deploy to AKS

- Get the cluster you're working with
  ```sh
  az aks list
  ```

- get credentials for the cluster

  ```sh
  az aks get-credentials -n [name] -g [group]
  ```
- Set vaiables

  ```sh
  export HOST=http://demo42-helloworld.eastus.cloudapp.azure.com/
  export ACR_NAME=jengademos
  export TAG=aa42
  export AKV_NAME=jengademoskv
  ```

- Deploy with Helm

  Set the 
  ```sh
  helm install ./release/helm/ -n helloworld \
  --set helloworld.host=$HOST \
  --set helloworld.image=jengademos.azurecr.io/demo42/helloworld:$TAG \
  --set imageCredentials.registry=$ACR_NAME.azurecr.io \
  --set imageCredentials.username=$(az keyvault secret show \
                                         --vault-name $AKV_NAME \
                                         --name $ACR_NAME-pull-usr \
                                         --query value -o tsv) \
  --set imageCredentials.password=$(az keyvault secret show \
                                         --vault-name $AKV_NAME \
                                         --name $ACR_NAME-pull-pwd \
                                         --query value -o tsv)
```
## Upgrade
```sh
helm upgrade helloworld ./release/helm/ \
--reuse-values \
  --set helloworld.image=jengademos.azurecr.io/demo42/helloworld:$TAG \
  --set imageCredentials.registry=$ACR_NAME.azurecr.io \
  --set imageCredentials.username=$(az keyvault secret show \
                                         --vault-name $AKV_NAME \
                                         --name $ACR_NAME-pull-usr \
                                         --query value -o tsv) \
  --set imageCredentials.password=$(az keyvault secret show \
                                         --vault-name $AKV_NAME \
                                         --name $ACR_NAME-pull-pwd \
                                         --query value -o tsv)

  ```