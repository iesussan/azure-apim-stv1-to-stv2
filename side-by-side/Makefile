#VARIABLES
#GENERAL VARIABLES
TENANT_ID ?= 72f988bf-86f1-41af-91ab-2d7cd011db47
SUBSCRIPTION_ID ?= 98fc3001-5ba8-4215-acde-94ca7d64d2fa
RESOURCE_GROUP ?= APIM-MIGRATION-RG
LOCATION ?= eastus

#AZURE API MANAGEMENT VARIABLES - ACTUAL
APIM_ACTUAL_NAME ?= actual-apimtomigrate
APIM_ACTUAL_SKU ?= Developer
APIM_ACTUAL_PUBLISHER_NAME ?= Actual Banking APIM
APIM_ACTUAL_PUBLISHER_EMAIL ?= jesussan@microsoft.com
APIM_ACTUAL_VNET_NAME ?= actual_apim_vnet
APIM_ACTUAL_VNET_ADDRESS_SPACE ?= 192.168.2.0/24
APIM_ACTUAL_VNET_SUBNET_NAME ?= actual_apim_subnet
APIM_ACTUAL_VNET_SUBNET_ADDRESS_SPACE ?= 192.168.2.0/24
APIM_ACTUAL_PUBLIC_IP_NAME ?= actual-apim-ip
APIM_ACTUAL_PUBLIC_IP_DNS_NAME ?= actual-apim
APIM_ACTUAL_NSG_NAME ?= actual-apim-nsg
APIM_ACTUAL_GATEWAY_URL ?= https://actual-apimtomigrate.azure-api.net

#AZURE API MANAGEMENT APIS VARIABLES
API_ACTUAL_SERVICE_NAME ?= actual-apim-api
API_ACTUAL_SERVICE_DISPLAY_NAME ?= actual-apim-api
API_ACTUAL_API_TYPE ?= http
API_ACTUAL_API_PROTOCOL ?= https


#AZURE API MANAGEMENT VARIABLES - NEW
APIM_NEW_NAME ?= new-apimtomigrate
APIM_NEW_SKU ?= Developer
APIM_NEW_PUBLISHER_NAME ?= New Banking APIM
APIM_NEW_PUBLISHER_EMAIL ?= jesussan@microsoft.com
APIM_NEW_VNET_NAME ?= new_apim_vnet
APIM_NEW_VNET_ADDRESS_SPACE ?= 192.168.3.0/24
APIM_NEW_VNET_SUBNET_NAME ?= new_apim_subnet
APIM_NEW_VNET_SUBNET_ADDRESS_SPACE ?= 192.168.3.0/24
APIM_NEW_PUBLIC_IP_NAME ?= new-apim-ip
APIM_NEW_PUBLIC_IP_DNS_NAME ?= new-apim
APIM_NEW_NSG_NAME ?= new-apim-nsg
APIM_NEW_GATEWAY_URL ?= https://new-apimtomigrate.azure-api.net

check_azure_cli:
ifeq (, $(shell which az))
	@echo "Error: Azure CLI no se encuentra en el PATH del sistema. Por favor, instálelo y vuelva a intentarlo."
	exit 1
else
	@echo "Azure CLI encontrado"
endif

validate_resource_group:
ifndef RESOURCE_GROUP
	@echo "Error: La variable RESOURCE_GROUP no está definida. Por favor, defínala y vuelva a intentarlo."
	exit 1
else
	@echo "La variable RESOURCE_GROUP está definida como: $(RESOURCE_GROUP)"
endif

create_resource_group: check_azure_cli validate_resource_group
	@echo "Creando grupo de recursos $(RESOURCE_GROUP) en la ubicación $(LOCATION)..."
	az group create --name $(RESOURCE_GROUP) --location $(LOCATION)
	@echo "Grupo de recursos creado con éxito."

create_actual_azure_apim_vnet: 
	@echo "Creando Network Security Group (NSG)..."
	az network nsg create \
		--name $(APIM_ACTUAL_NSG_NAME) \
		--resource-group $(RESOURCE_GROUP) \
		--location $(LOCATION)
	
	# Agregar regla para permitir tráfico entrante en el puerto 3443
	@echo "Agregando regla Allow3443Inbound al NSG..."
	az network nsg rule create \
		--nsg-name $(APIM_ACTUAL_NSG_NAME) \
		--resource-group $(RESOURCE_GROUP) \
		--name Allow3443Inbound \
		--priority 2711 \
		--direction Inbound \
		--access Allow \
		--protocol Tcp \
		--source-address-prefixes ApiManagement \
		--source-port-ranges "*" \
		--destination-address-prefixes VirtualNetwork \
		--destination-port-ranges 3443
	
	# Agregar regla para permitir tráfico entrante en el puerto 443
	@echo "Agregando regla AllowHTTPSInbound al NSG..."
	az network nsg rule create \
		--nsg-name $(APIM_ACTUAL_NSG_NAME) \
		--resource-group $(RESOURCE_GROUP) \
		--name AllowHTTPSInbound \
		--priority 2721 \
		--direction Inbound \
		--access Allow \
		--protocol Tcp \
		--source-address-prefixes Internet \
		--source-port-ranges "*" \
		--destination-address-prefixes VirtualNetwork \
		--destination-port-ranges 443

	@echo "Creando la VNET $(APIM_ACTUAL_VNET_NAME) para el Api Management que se va a migrar..."
	az network vnet create \
	 	--name $(APIM_ACTUAL_VNET_NAME) \
	 	--resource-group $(RESOURCE_GROUP) \
	 	--location $(LOCATION) \
 	 	--address-prefixes $(APIM_ACTUAL_VNET_ADDRESS_SPACE) \
		--network-security-group $(APIM_ACTUAL_NSG_NAME) -o none 

	az network vnet subnet create \
	 	--name $(APIM_ACTUAL_VNET_SUBNET_NAME) \
	 	--resource-group $(RESOURCE_GROUP) \
	 	--vnet-name $(APIM_ACTUAL_VNET_NAME) \
	 	--address-prefixes $(APIM_ACTUAL_VNET_SUBNET_ADDRESS_SPACE)

	@echo "Agregando service endpoints a la subred $(APIM_ACTUAL_VNET_SUBNET_NAME)..."
	az network vnet subnet update \
		--name $(APIM_ACTUAL_VNET_SUBNET_NAME) \
		--vnet-name $(APIM_ACTUAL_VNET_NAME) \
		--resource-group $(RESOURCE_GROUP) \
		--service-endpoints Microsoft.EventHub Microsoft.KeyVault Microsoft.ServiceBus Microsoft.Sql Microsoft.Storage

deploy_actual_azure_api_management:
	@echo "Desplegando Azure API Management..."
	az apim create --name $(APIM_ACTUAL_NAME) \
		--publisher-email $(APIM_ACTUAL_PUBLISHER_EMAIL) \
		--publisher-name "$(APIM_ACTUAL_PUBLISHER_NAME)" \
		--enable-managed-identity true \
		--virtual-network External \
		--resource-group $(RESOURCE_GROUP) \
		--sku-name $(APIM_SKU) \
		--location $(LOCATION)

update_actual_azure_apim_customparameters:
	az network public-ip create \
	--name $(APIM_ACTUAL_PUBLIC_IP_NAME) \
	--resource-group $(RESOURCE_GROUP) \
	--location $(LOCATION) \
	--allocation-method Static \
	--dns-name $(APIM_ACTUAL_PUBLIC_IP_DNS_NAME)

	@APIM_PUBLIC_IP_ID=$$(az network public-ip show --name $(APIM_ACTUAL_PUBLIC_IP_NAME) --resource-group $(RESOURCE_GROUP) --query id --out tsv); \
	TMPFILE=$$(mktemp); \
	cp ./apim-customproperties.json $$TMPFILE; \
	sed -i.bak 's|__SUBSCRIPTION__|$(SUBSCRIPTION_ID)|g' $$TMPFILE; \
	sed -i.bak 's|__RESOURCE_GROUP__|$(RESOURCE_GROUP)|g' $$TMPFILE; \
	sed -i.bak 's|__VNET_NAME__|$(APIM_ACTUAL_VNET_NAME)|g' $$TMPFILE; \
	sed -i.bak 's|__SUBNET_NAME__|$(APIM_ACTUAL_VNET_SUBNET_NAME)|g' $$TMPFILE; \
	sed -i.bak "s|__PUBLIC_IP_ID__|$$APIM_PUBLIC_IP_ID|g" $$TMPFILE; \
	cat $${TMPFILE}; \
	echo "Actualizando parámetros personalizados de Azure API Management $(APIM_ACTUAL_NAME)..."; \
	TOKEN=$$(az account get-access-token --resource https://management.azure.com/ --query "accessToken" -o tsv); \
	curl -X PUT \
	-H "Authorization: Bearer $$TOKEN" \
	-H "Content-Type: application/json" \
	-d @$${TMPFILE} \
	"https://management.azure.com/subscriptions/$(SUBSCRIPTION_ID)/resourceGroups/$(RESOURCE_GROUP)/providers/Microsoft.ApiManagement/service/$(APIM_ACTUAL_NAME)?api-version=2022-08-01"; \
	rm $$TMPFILE && \
	echo "Parámetros personalizados de Azure API Management actualizados con éxito."

get_actual_azure_apim_subscription_key:
	$(eval APIMID=$(shell az apim show -n $(APIM_ACTUAL_NAME) --resource-group  $(RESOURCE_GROUP) --query id -o tsv))
	az rest --method post  --uri $(APIMID)/subscriptions/master/listSecrets?api-version=2021-08-01 --query primaryKey -o tsv

create_acutal_publish_api_into_azure_apim:
	az apim api create \
	--resource-group ${RESOURCE_GROUP} \
	--service-name ${APIM_ACTUAL_NAME} \
	--api-id ${API_ACTUAL_SERVICE_NAME} \
	--path / \
	--display-name "$(API_ACTUAL_SERVICE_DISPLAY_NAME)" \
	--service-url $(APIM_ACTUAL_GATEWAY_URL) \
	--protocols $(API_ACTUAL_API_PROTOCOL)

update_actual_api_with_swagger:
	az apim api import \
	--path "/" \
	--api-id ${API_ACTUAL_SERVICE_NAME} \
	--resource-group ${RESOURCE_GROUP}  \
	--service-name ${APIM_ACTUAL_NAME} \
	--specification-path ./swagger.yaml \
	--specification-format OpenApi

create_actual_azure_apim_backup:
	@STORAGE_ACCOUNT_NAME=$$(echo bck-$(APIM_ACTUAL_NAME) | sed 's/-//g') && \
	az storage account create \
	--name $$STORAGE_ACCOUNT_NAME \
	--resource-group $(RESOURCE_GROUP) \
	--location $(LOCATION) \
	--sku Standard_LRS && \
	STORAGE_ACCOUNT_KEY=$$(az storage account keys list \
	--resource-group $(RESOURCE_GROUP) \
	--account-name $$STORAGE_ACCOUNT_NAME \
	--query "[0].value" -o tsv) && \
	az apim backup \
	--name $(APIM_ACTUAL_NAME) \
	--resource-group $(RESOURCE_GROUP) \
	--backup-name $$(date +'%Y-%m-%d')-$(APIM_ACTUAL_NAME)-backup \
	--storage-account-name $$STORAGE_ACCOUNT_NAME \
	--storage-account-container backups \
	--storage-account-key $$STORAGE_ACCOUNT_KEY


################################################# NEW APIM #################################################

create_new_azure_apim_vnet: 
	@echo "Creando Network Security Group (APIM_NEW_NSG_NAME)..."
	az network nsg create \
		--name $(APIM_NEW_NSG_NAME) \
		--resource-group $(RESOURCE_GROUP) \
		--location $(LOCATION)
	
	# Agregar regla para permitir tráfico entrante en el puerto 3443
	@echo "Agregando regla Allow3443Inbound al NSG..."
	az network nsg rule create \
		--nsg-name $(APIM_NEW_NSG_NAME) \
		--resource-group $(RESOURCE_GROUP) \
		--name Allow3443Inbound \
		--priority 2711 \
		--direction Inbound \
		--access Allow \
		--protocol Tcp \
		--source-address-prefixes ApiManagement \
		--source-port-ranges "*" \
		--destination-address-prefixes VirtualNetwork \
		--destination-port-ranges 3443
	
	# Agregar regla para permitir tráfico entrante en el puerto 443
	@echo "Agregando regla AllowHTTPSInbound al NSG..."
	az network nsg rule create \
		--nsg-name $(APIM_NEW_NSG_NAME) \
		--resource-group $(RESOURCE_GROUP) \
		--name AllowHTTPSInbound \
		--priority 2721 \
		--direction Inbound \
		--access Allow \
		--protocol Tcp \
		--source-address-prefixes Internet \
		--source-port-ranges "*" \
		--destination-address-prefixes VirtualNetwork \
		--destination-port-ranges 443

	@echo "Creando la VNET $(APIM_NEW_VNET_NAME) para el Api Management que se va a migrar..."
	az network vnet create \
	 	--name $(APIM_NEW_VNET_NAME) \
	 	--resource-group $(RESOURCE_GROUP) \
	 	--location $(LOCATION) \
 	 	--address-prefixes $(APIM_NEW_VNET_ADDRESS_SPACE) \
		--network-security-group $(APIM_NEW_NSG_NAME) -o none 

	az network vnet subnet create \
	 	--name $(APIM_NEW_VNET_SUBNET_NAME) \
	 	--resource-group $(RESOURCE_GROUP) \
	 	--vnet-name $(APIM_NEW_VNET_NAME) \
	 	--address-prefixes $(APIM_NEW_VNET_SUBNET_ADDRESS_SPACE)

	@echo "Agregando service endpoints a la subred $(APIM_NEW_VNET_SUBNET_NAME)..."
	az network vnet subnet update \
		--name $(APIM_NEW_VNET_SUBNET_NAME) \
		--vnet-name $(APIM_NEW_VNET_NAME) \
		--resource-group $(RESOURCE_GROUP) \
		--service-endpoints Microsoft.EventHub Microsoft.KeyVault Microsoft.ServiceBus Microsoft.Sql Microsoft.Storage

deploy_new_azure_api_management:
	@echo "Desplegando Azure API Management..."
	az apim create --name $(APIM_NEW_NAME) \
		--publisher-email $(APIM_NEW_PUBLISHER_EMAIL) \
		--publisher-name "$(APIM_NEW_PUBLISHER_NAME)" \
		--enable-managed-identity true \
		--virtual-network External \
		--resource-group $(RESOURCE_GROUP) \
		--sku-name $(APIM_NEW_SKU) \
		--location $(LOCATION)

update_new_azure_apim_customparameters:
	# az network public-ip create \
	# --name $(APIM_NEW_PUBLIC_IP_NAME) \
	# --resource-group $(RESOURCE_GROUP) \
	# --location $(LOCATION) \
	# --allocation-method Static \
	# --dns-name $(APIM_NEW_PUBLIC_IP_DNS_NAME)

	@APIM_NEW_PUBLIC_IP_ID=$$(az network public-ip show --name $(APIM_NEW_PUBLIC_IP_NAME) --resource-group $(RESOURCE_GROUP) --query id --out tsv); \
	TMPFILE=$$(mktemp); \
	cp ./apim-customproperties.json $$TMPFILE; \
	sed -i.bak 's|__SUBSCRIPTION__|$(SUBSCRIPTION_ID)|g' $$TMPFILE; \
	sed -i.bak 's|__RESOURCE_GROUP__|$(RESOURCE_GROUP)|g' $$TMPFILE; \
	sed -i.bak 's|__VNET_NAME__|$(APIM_NEW_VNET_NAME)|g' $$TMPFILE; \
	sed -i.bak 's|__SUBNET_NAME__|$(APIM_NEW_VNET_SUBNET_NAME)|g' $$TMPFILE; \
	sed -i.bak "s|__PUBLIC_IP_ID__|$$APIM_NEW_PUBLIC_IP_ID|g" $$TMPFILE; \
	cat $${TMPFILE}; \
	echo "Actualizando parámetros personalizados de Azure API Management $(APIM_NEW_NAME)..."; \
	TOKEN=$$(az account get-access-token --resource https://management.azure.com/ --query "accessToken" -o tsv); \
	curl -X PUT \
	-H "Authorization: Bearer $$TOKEN" \
	-H "Content-Type: application/json" \
	-d @$${TMPFILE} \
	"https://management.azure.com/subscriptions/$(SUBSCRIPTION_ID)/resourceGroups/$(RESOURCE_GROUP)/providers/Microsoft.ApiManagement/service/$(APIM_NEW_NAME)?api-version=2022-08-01"; \
	rm $$TMPFILE && \
	echo "Parámetros personalizados de Azure API Management actualizados con éxito."

restore_new_azure_apim_backup:
	@STORAGE_ACCOUNT_NAME=$$(echo bck-$(APIM_ACTUAL_NAME) | sed 's/-//g') && \
	STORAGE_ACCOUNT_KEY=$$(az storage account keys list \
	--resource-group $(RESOURCE_GROUP) \
	--account-name $$STORAGE_ACCOUNT_NAME \
	--query "[0].value" -o tsv) && \
	az apim restore \
	--name $(APIM_NEW_NAME) \
	--resource-group $(RESOURCE_GROUP) \
	--backup-name $$(date +'%Y-%m-%d')-$(APIM_ACTUAL_NAME)-backup \
	--storage-account-name $$STORAGE_ACCOUNT_NAME \
	--storage-account-container backups \
	--storage-account-key $$STORAGE_ACCOUNT_KEY