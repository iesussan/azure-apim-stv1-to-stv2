# Documentación de Migración y Configuración de Azure API Management

Este repositorio contiene los recursos y guías necesarios para facilitar la migración y configuración de Azure API Management (APIM) mediante varios enfoques. Actualmente solo estan disponible: migración "side by side" y "in-place VNet injection".

 A continuación, se describe el contenido y propósito de cada carpeta incluida en el repositorio.

## Estructura del Directorio

El repositorio está organizado en dos carpetas principales, cada una dedicada a un enfoque específico de migración o configuración:

```
.
├── inpace-vnet-injection
│   └── README.md
└── side-by-side
    ├── Makefile
    ├── README.md
    ├── apim-customproperties.json
    ├── backend.md
    └── swagger.yaml
```

### Carpeta `side-by-side`

La carpeta `side-by-side` está diseñada para usuarios que buscan realizar una migración "side by side" de una instancia existente de Azure API Management a una nueva instancia, generalmente para actualizarse a una versión más reciente o cambiar la configuración de red.

Contenidos:
- **Makefile**: Define los comandos necesarios para automatizar el proceso de migración, incluyendo la creación de recursos, configuración de red, y más.
- **README.md**: Ofrece una guía detallada sobre cómo realizar la migración "side by side", incluyendo requisitos previos, pasos a seguir, y consideraciones importantes.
- **apim-customproperties.json**: Contiene configuraciones para la integración de VNet y la desactivación de cifrados de baja seguridad por defecto en Azure API Management.
- **backend.md**: Proporciona el código y las instrucciones para desplegar un Azure Function en Python, el cual sirve como backend para la API que se migrará.
- **swagger.yaml**: Define la especificación OpenAPI para la API que se utilizará como prueba funcional después de la migración.

### Carpeta `inplace-vnet-injection`

La carpeta `inplace-vnet-injection` contiene información y guías para realizar la inyección de VNet "in-place" en una instancia existente de Azure API Management. Este enfoque es útil para generar la migración de una instancia de APIM con el cambio de una red virtual sin necesidad de migrar a una nueva instancia.

Contenidos:
- **README.md**: Documento que detalla los pasos y consideraciones para actualizar inplace la inyección de VNet "in-place" en una instancia de APIM, incluyendo la configuración de una nueva red virtual lo que dispara el proceso de upgrade al sku stv2.