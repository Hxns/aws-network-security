# Guía de Configuración: Creación de AMI y Balanceador de Carga (ALB)

Este documento detalla el procedimiento seguido para preparar la infraestructura de red, crear una imagen personalizada de un servidor web y configurar el equilibrio de carga.

## 1. Configuración de la Infraestructura de Red

Antes de la duplicación de servidores, se preparó el entorno de red en la VPC:

*   **Creación de Subred:** Se creó una nueva subred denominada `DMZ-subnet2` dentro de la VPC `vpc-herzfelder`.
*   **Asignación de Zona y CIDR:** Se configuró en la zona de disponibilidad `eu-south-2b` con el bloque CIDR `10.0.5.0/24`.
*   **Configuración de Enrutamiento:** Se verificó que la tabla de rutas de la subred DMZ tuviera una ruta activa hacia el destino `0.0.0.0/0` a través de una interfaz de red (ENI) específica para permitir el tráfico externo.

## 2. Creación de la Imagen Personalizada (AMI)

Para asegurar que los nuevos servidores sean idénticos al original, se generó una Amazon Machine Image (AMI):

*   **Selección del Origen:** Se utilizó la instancia denominada `WebServer` como base.
*   **Generación de la Imagen:** Desde el menú de **Acciones > Imagen y plantillas**, se seleccionó **Crear imagen**.
*   **Detalles de la AMI:** Se le asignó el nombre `Herzfelder-WebServer-AMI` y se habilitó la opción de **Reiniciar instancia** para garantizar la coherencia de los datos durante la creación.
*   **Verificación:** Se monitoreó el estado de la AMI en la sección de Imágenes hasta que cambió de "Pendiente" a "Disponible".

## 3. Despliegue de la Segunda Instancia

Utilizando la imagen creada, se procedió a lanzar un segundo servidor:

*   **Lanzamiento desde AMI:** Se seleccionó la AMI `Herzfelder-WebServer-AMI` y se ejecutó la opción **Lanzar instancia a partir de una AMI**.
*   **Identificación:** La nueva instancia se nombró `WebServer2`.
*   **Configuración de Red y Seguridad:**
    *   Se seleccionó la VPC `vpc-herzfelder` y la subred `DMZ-subnet2`.
    *   Se habilitó la **Asignación automática de IP pública**.
    *   Se asoció el grupo de seguridad existente `WebSever-SecurityGroup`.
    *   Se utilizó el par de claves `webserver-keypair-herzfelder` para el acceso.

## 4. Configuración del Balanceador de Carga (ALB)

Finalmente, se inició la configuración del servicio para distribuir el tráfico:

*   **Navegación:** En la consola de EC2, se accedió a la sección **Equilibrio de carga > Balanceadores de carga**.
*   **Selección del Tipo:** Se eligió crear un **Application Load Balancer (ALB)**, diseñado para manejar tráfico HTTP y HTTPS a nivel de aplicación (Capa 7).
*   **Creación:** Se pulsó el botón **Crear** para iniciar el asistente de configuración del balanceador que conectará las instancias `WebServer` y `WebServer2`.
