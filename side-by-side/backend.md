
## Creación del Azure Function basada en python 
```bash
export LOCATION='eastus' \
export RESOURCE_GROUP='APIM-MIGRATION-AZFUNCTIONPY' \
export STORAGE_ACCOUNT_SKU='Standard_LRS' \
export STORAGE_ACCOUNT_NAME='mrstorageaccountazfuncpy'

#### Creamos el Azure Resource Group
az group create --name $RESOURCE_GROUP --location $LOCATION
#### Creamos el Azure Storage Account
az storage account create --name $STORAGE_ACCOUNT_NAME \
--location "$LOCATION" \
--resource-group $RESOURCE_GROUP \
--sku $STORAGE_ACCOUNT_SKU
```

#### Creamos el Azure function container app.
##### Nota: El sku del app service es consumption, por eso la existencia del parametro --consumption-plan-location $LOCATION
```bash
export AZURE_FUNCTION_NAME='containerapp-python-functions' \
export AZURE_FUNCTION_OS_TYPE='Linux' \
export AZURE_FUNCTION_RUNTIME='Python' \
export AZURE_FUNCTION_RUNTIME_VERSION='3.11' \
export AZURE_FUNCTIONS_VERSION='4'

#### Creare Azure Function
az functionapp create --name $AZURE_FUNCTION_NAME \
--storage-account $STORAGE_ACCOUNT_NAME \
--consumption-plan-location $LOCATION \
--resource-group $RESOURCE_GROUP \
--os-type $AZURE_FUNCTION_OS_TYPE \
--runtime $AZURE_FUNCTION_RUNTIME \
--runtime-version $AZURE_FUNCTION_RUNTIME_VERSION \
--functions-version $AZURE_FUNCTIONS_VERSION --assign-identity 
```

#### Creamos directorios base para el codigo del function
```bash
mkdir -p functions
```

```bash
cd functions
```
#### Generamos nuestra Nueva function
```bash
func new --authlevel anonymous \
--language python \
--name python-functions \
--template "HTTP trigger" \
--worker-runtime python
```
#### Desplegamos la function
```bash
zip -r functionpythonapp.zip .
az functionapp deployment source config-zip \
-g $RESOURCE_GROUP \
-n $AZURE_FUNCTION_NAME \
--src functionpythonapp.zip
```
#### Obtenemos el master key del function el cual sera usado en el request:
```bash
export azure_function_key=$(az functionapp keys list -g $RESOURCE_GROUP -n $AZURE_FUNCTION_NAME --query functionKeys.default -o tsv)
```


#### Usamos  curl para realizar  un request al function en cuestion:
```bash
curl https://$AZURE_FUNCTION_NAME.azurewebsites.net/api/python-functions?code=$azure_function_key
```