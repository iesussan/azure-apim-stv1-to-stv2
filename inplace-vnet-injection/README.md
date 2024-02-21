La documentación de Microsoft para la migración "in-place" de una instancia de Azure API Management (APIM) inyectada en una red a la plataforma stv2 implica actualizar la configuración de red existente para utilizar nuevas configuraciones de red. Este proceso de actualización activa efectivamente la migración, lo cual es confirmado por la documentación. A continuación, se presenta una documentación estructurada basada en la información proporcionada, enfocada en el proceso de actualización y migración.

# Documentación para la Migración "In-Place" a Azure API Management stv2

Este documento detalla el proceso para migrar una instancia de Azure API Management inyectada en una red hacia la plataforma stv2 forzando la actualización de la configuración de la red virtual (VNet). La migración se activa con la asignación de una nueva subnet.


Para migrar una instancia de APIM a la plataforma stv2, actualizaremos la configuración de red existente a una nueva. Este proceso puede incluir, opcionalmente, la migración de regreso a la VNet y subnet original utilizada previo a la migración.

### Consideraciones Importantes

- La dirección publica de su instancia de APIM cambiará.
- Las solicitudes de API permanecerán activas durante la migración.
- La configuración como dominios personalizados, zonas y certificados CA estaran bloqueadas durante 30 minutos.
- Tras la migración, será necesario actualizar todas las dependencias de red, incluyendo DNS, reglas de firewall y VNets, para usar la nueva dirección VIP.

## Actualización de la Configuración de VNet

Actualice la configuración de la VNet en cada ubicación donde esté desplegada la instancia de APIM.

### Prerrequisitos

- Una nueva subnet en la red virtual actual, o una subnet en una VNet diferente en la misma región y suscripción que su instancia de APIM. Se debe adjuntar un grupo de seguridad de red (NSG) a la subnet, y configurar las reglas NSG para APIM.
- Un recurso de dirección IPv4 pública SKU Estándar en la misma región y suscripción que su instancia de APIM.

### Pasos para la Actualización de la Configuración de VNet

1. Identificamos el Azure Api Management actualizar.
```shell
export SUBSCRIPTION_ID="98fc3001-5ba8-4215-acde-94ca7d64d2fa"
export RESOURCE_GROUP="APIM-MIGRATION-RG"
export LOCATION="eastus"
export APIM_NAME="actual-apimtomigrate"

APIMID=$(az apim show -n $APIM_NAME --resource-group  $RESOURCE_GROUP --query id -o tsv)
```
2. Crearemos el NSG y las reglas que son parte de los pre - requisitos:
```shell
export APIM_ACTUAL_NSG_NAME="new-nsg-apim-migration"
az network nsg create --name $APIM_ACTUAL_NSG_NAME --resource-group $RESOURCE_GROUP --location $LOCATION

# Internet request
az network nsg rule create \
  --name Allow80and443InboundFromInternet \
  --nsg-name $APIM_ACTUAL_NSG_NAME \
  --resource-group $RESOURCE_GROUP \
  --priority 1000 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes Internet \
  --source-port-ranges "*" \
  --destination-address-prefixes VirtualNetwork \
  --destination-port-ranges "80-443"

#Management Port
az network nsg rule create \
  --nsg-name $APIM_ACTUAL_NSG_NAME \
  --resource-group $RESOURCE_GROUP \
  --name Allow3443InboundManagement \
  --priority 1020 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes ApiManagement \
  --source-port-ranges "*" \
  --destination-address-prefixes VirtualNetwork \
  --destination-port-ranges 3443

# Azure Infrastructure load balancer port
az network nsg rule create \
  --nsg-name $APIM_ACTUAL_NSG_NAME \
  --resource-group $RESOURCE_GROUP \
  --name Allow6390InboundFromAzureLB \
  --priority 1030 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes AzureLoadBalancer \
  --source-port-ranges "*" \
  --destination-address-prefixes VirtualNetwork \
  --destination-port-ranges 6390

# Azure Traffic Manager (Puerto 443)
az network nsg rule create \
  --nsg-name $APIM_ACTUAL_NSG_NAME \
  --resource-group $RESOURCE_GROUP \
  --name Allow443InboundFromATM \
  --priority 1040 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes AzureTrafficManager \
  --source-port-ranges "*" \
  --destination-address-prefixes VirtualNetwork \
  --destination-port-ranges 443

#Reglas de Salida (Outbound)
#Dependencia de Azure Storage (Puerto 443)

az network nsg rule create \
  --nsg-name $APIM_ACTUAL_NSG_NAME \
  --resource-group $RESOURCE_GROUP \
  --name Allow443OutboundToStorage \
  --priority 1050 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes VirtualNetwork \
  --source-port-ranges "*" \
  --destination-address-prefixes Storage \
  --destination-port-ranges 443

#Acceso a Endpoints de Azure SQL (Puerto 1433)
az network nsg rule create \
  --nsg-name $APIM_ACTUAL_NSG_NAME \
  --resource-group $RESOURCE_GROUP \
  --name Allow1433OutboundToSQL \
  --priority 1060 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes VirtualNetwork \
  --source-port-ranges "*" \
  --destination-address-prefixes Sql \
  --destination-port-ranges 1433

#Acceso a Azure Key Vault (Puerto 443)
az network nsg rule create \
  --nsg-name $APIM_ACTUAL_NSG_NAME \
  --resource-group $RESOURCE_GROUP \
  --name Allow443OutboundToKeyVault \
  --priority 1070 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes VirtualNetwork \
  --source-port-ranges "*" \
  --destination-address-prefixes AzureKeyVault \
  --destination-port-ranges 443

#Publicación de Diagnósticos y Métricas a Azure Monitor (Puertos 1886, 443)
az network nsg rule create \
  --nsg-name $APIM_ACTUAL_NSG_NAME \
  --resource-group $RESOURCE_GROUP \
  --name Allow1886OutboundToAzureMonitor \
  --priority 1080 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes VirtualNetwork \
  --source-port-ranges "*" \
  --destination-address-prefixes AzureMonitor \
  --destination-port-ranges 1886

az network nsg rule create \
  --nsg-name $APIM_ACTUAL_NSG_NAME \
  --resource-group $RESOURCE_GROUP \
  --name Allow443OutboundToAzureMonitor \
  --priority 1090 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes VirtualNetwork \
  --source-port-ranges "*" \
  --destination-address-prefixes AzureMonitor \
  --destination-port-ranges 443
```
3.- Creación de la nueva Vnet/Subnet con la asociación del NSG previo creado. 
```shell
export APIM_VNET_NAME="vnet-apim-tomigrating"
export APIM_VNET_ADDRESS_SPACE="192.168.4.0/24"
export APIM_VNET_SUBNET_NAME="vnet-subnet-apim-tomigrating"
az network vnet create \
    --name $APIM_VNET_NAME \
	--resource-group $RESOURCE_GROUP \
	--location $LOCATION \
 	--address-prefixes $APIM_VNET_ADDRESS_SPACE \
	--network-security-group $APIM_ACTUAL_NSG_NAME

az network vnet subnet create \
    --name $APIM_VNET_SUBNET_NAME \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $APIM_VNET_NAME \
    --address-prefixes $APIM_VNET_ADDRESS_SPACE

az network vnet subnet update \
    --name $APIM_VNET_SUBNET_NAME \
	--vnet-name $APIM_VNET_NAME \
	--resource-group $RESOURCE_GROUP \
	--service-endpoints Microsoft.EventHub Microsoft.KeyVault Microsoft.ServiceBus Microsoft.Sql Microsoft.Storage \
	--network-security-group $APIM_ACTUAL_NSG_NAME
```
4.- Preparar y ejecutar la migración usando la actualización de vnet/subnet
```shell
    export APIM_PUBLIC_IP_NAME="actual-apim-pip"
    export APIM_PUBLIC_IP_DNS_NAME="actual-apim-tag"
	az network public-ip create \
	    --name $APIM_PUBLIC_IP_NAME \
	    --resource-group $RESOURCE_GROUP \
	    --location $LOCATION \
	    --allocation-method Static \
	    --dns-name $APIM_PUBLIC_IP_DNS_NAME

	export APIM_PUBLIC_IP_ID=$(az network public-ip show --name $APIM_PUBLIC_IP_NAME --resource-group $RESOURCE_GROUP --query id --out tsv)

	TMPFILE=$(mktemp) 
	cp ./apim-customproperties.json $TMPFILE 
	sed -i.bak "s|__SUBSCRIPTION__|$SUBSCRIPTION_ID|g" $TMPFILE
	sed -i.bak "s|__RESOURCE_GROUP__|$RESOURCE_GROUP|g" $TMPFILE
	sed -i.bak "s|__VNET_NAME__|$APIM_VNET_NAME|g" $TMPFILE
	sed -i.bak "s|__SUBNET_NAME__|$APIM_VNET_SUBNET_NAME)|g" $TMPFILE
	sed -i.bak "s|__PUBLIC_IP_ID__|$APIM_PUBLIC_IP_ID|g" $TMPFILE
	cat $TMPFILE

	TOKEN=$(az account get-access-token --resource https://management.azure.com/ --query "accessToken" -o tsv)

	curl -X PUT \
	-H "Authorization: Bearer $TOKEN" \
	-H "Content-Type: application/json" \
	-d @$TMPFILE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ApiManagement/service/$APIM_ACTUAL_NAME?api-version=2022-08-01"

	rm $TMPFILE
    
    echo "Parámetros personalizados de Azure API Management actualizados con éxito."
```

#### Verificación de la Migración

Para verificar el éxito de la migración, compruebe la versión de la plataforma de su instancia de APIM; después de una migración exitosa, el valor debe set stv2. Además, verifique el estado de la red para asegurar la conectividad de la instancia con sus dependencias.

#### Actualización de Dependencias de Red

Tras una migración exitosa, actualice todas las dependencias de red para utilizar la nueva dirección VIP/espacio de direcciones de subnet.


### (Opcional) Migrar de Regreso a la VNet y Subnet Original

Se puede optar por migrar de regreso a la VNet y subnet original en cada región después de la migración a la plataforma stv2, actualizando nuevamente la configuración de VNet.

#### Prerrequisitos

- La subnet y VNet original con un NSG adjunto y reglas NSG configuradas para APIM.
- Uns dirección IPv4 pública SKU Estándar en la misma región y suscripción que su instancia de APIM.
