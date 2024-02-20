* Create a new APIM instance in stv2 with VNet integration. Additionally review internal, application gateway integration and force tunneling.
* Backup the old APIM instance.
* Restore the backup to the new APIM instance.
* Migrate developer portal contents from the old instance to the new instance.
* Setup service configurations that are not included in the backup.
* Validate the configuration.
* Update the DNS records to point to the new instance.

# Guía de Migración de Azure API Management (stv1 a stv2)


Esta guía proporciona un conjunto de instrucciones detalladas para realizar la migración de una instancia de Azure API Management (APIM) de la versión stv1 a stv2, utilizando un enfoque "side by side". Abarca desde la creación de una nueva instancia de APIM hasta la restauración de los datos y la configuración de la instancia antigua en la nueva.

## Pre-requisitos

- Azure CLI instalado y configurado en tu sistema.
- Acceso a una suscripción de Azure con permisos de Contributor.
- Una instancia existente de Azure API Management en versión stv1 que se desea migrar.

## Pasos para la Migración

### 1. Preparación del Ambiente

En el proceso de migración "side by side", configuraremos una instancia de Azure API Management. Esta incluirá la configuración de una API que actúa como interfaz para un Azure Function en Python, el cual dispone de dos endpoints. Este escenario nos permitirá demostrar cómo se realiza el respaldo de los servicios publicados en el Azure API Management existente hacia el nuevo servicio en la versión stv2.

La estructura del directorio para este proceso incluye los siguientes archivos:

```plaintext
├── Makefile
├── README.md
├── apim-customproperties.json
├── backend.md
└── swagger.yaml
```

- **Makefile**: Contiene los comandos necesarios para ejecutar el proceso de migración paso a paso.
- **apim-customproperties.json**: Incluye la configuración para la integración de la red virtual (VNet) y la desactivación de cifrados de baja seguridad que Azure API Management configura por defecto.
- **backend.md**: Proporciona el código necesario para generar y desplegar un Azure Function con soporte para Python. Este componente, junto con el archivo **swagger.yaml**, será fundamental para probar la funcionalidad de la API tras completar el proceso de migración.

Antes de comenzar con la migración, es esencial asegurarse de que Azure CLI esté instalado y correctamente configurado en su sistema. Además, es importante verificar que el grupo de recursos donde se alojarán los recursos de Azure haya sido definido. Este paso inicial es crucial para garantizar que todos los componentes necesarios estén en su lugar y listos para la migración.

```shell
make check_azure_cli
make validate_resource_group
```

### 2. Creación del Grupo de Recursos

Permite crear un grupo de recursos en Azure para contener todos los recursos relacionados con la migración.

```shell
make create_resource_group
```

### 3. Configuración de la Red para la Instancia Actual de APIM

Para la instancia actual de APIM, se crea una red virtual (VNet) y un grupo de seguridad de red (NSG) con reglas para permitir el tráfico necesario.

```shell
make create_actual_azure_apim_vnet
```

### 4. Despliegue de la Instancia Actual de Azure API Management

Permite desplegar la instancia actual de APIM con la configuración especificada.

```shell
make deploy_actual_azure_api_management
```

### 5. Actualización de Parámetros Personalizados de la Instancia Actual de APIM

Permite actualizar los parámetros personalizados de la instancia actual de APIM, incluyendo la configuración de la red virtual, la IP pública y los ciphers

```shell
make update_actual_azure_apim_customparameters
```

### 6. Creación y Actualización de APIs en la Instancia Actual

Permite publicar y actualizar APIs en la instancia actual de APIM utilizando las especificaciones proporcionadas.

```shell
make create_acutal_publish_api_into_azure_apim
make update_actual_api_with_swagger
```

### 7. Creación de un Respaldo de la Instancia Actual de APIM

Permite crear un respaldo de la instancia actual de APIM para su posterior restauración en la nueva instancia.

```shell
make create_actual_azure_apim_backup
```

### 8. Configuración de la Red para la Nueva Instancia de APIM

Repetir el proceso de creación de la red virtual y el grupo de seguridad de red para la nueva instancia de APIM.

```shell
make create_new_azure_apim_vnet
```

### 9. Despliegue de la Nueva Instancia de Azure API Management

Desplegar la nueva instancia de APIM con la configuración deseada y prepararla para la restauración del respaldo.

```shell
make deploy_new_azure_api_management
```

### 10. Restauración del Respaldo en la Nueva Instancia de APIM

Restaurar el respaldo creado anteriormente en la nueva instancia de APIM para migrar todos los datos y configuraciones relevantes.

```shell
make restore_new_azure_apim_backup
```

## Consideraciones Post-Migración

- **Verificación de la Configuración:** Es importante validar que toda la configuración y los datos se hayan migrado correctamente a la nueva instancia de APIM. Esto incluye revisar las APIs publicadas, las políticas, las suscripciones, entre otros.
- **Monitoreo y Pruebas:** Monitorear el rendimiento y la funcionalidad de la nueva instancia de APIM para asegurar que todo funcione según lo esperado. Realizar pruebas exhaustivas para identificar y corregir cualquier problema que pueda surgir.
- **Actualización de Registros DNS:** Posterior de todas las validaciones se debe proceder actualizar los registros DNS para apuntar al nuevo servicio de APIM, asegurando así que el tráfico se redirija correctamente a la nueva instancia.


## Elementos que No se Incluyen en el Respaldo

Al realizar el respaldo y la restauración de una instancia de APIM, es importante tener en cuenta que ciertos elementos no se incluyen y deben ser configurados manualmente en la nueva instancia. Estos incluyen:

- Datos de uso utilizados para crear informes de análisis.
- Certificados TLS/SSL de dominios personalizados.
- Certificados CA personalizados.
- Configuraciones de integración de red virtual.
- Configuración de identidad administrada.
- Configuración de diagnóstico de Azure Monitor.
- Configuraciones de protocolos y cifrados.
- Contenido del portal de desarrolladores.

## Enlaces de Referencia

- Documentación de Azure API Management: [Uso de APIM con VNet (stv2)](https://learn.microsoft.com/en-us/azure/api-management/api-management-using-with-vnet?tabs=stv2)

