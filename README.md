# Ciberseguridad en redes informáticas en AWS

![Diagrama de la Arquitectura](./img/DMZ-IMG-2.png)


## 1. Cimentación de la red (VPC y subredes)

El primer paso fue definir el espacio de red virtual aislado donde residirán todos los recursos.

*   **Creación de la VPC:** Se creó la VPC `vpc-herzfelder` con el bloque CIDR `10.0.0.0/16`, proporcionando un amplio rango de direcciones IP internas.
*   **Segmentación de la red (subredes):** La VPC se dividió en tres segmentos con responsabilidades de seguridad diferenciadas:
    *   **Subred pública:** `public-subnet` (`10.0.1.0/24`), en la zona de disponibilidad `eu-south-2a`. Se habilitó la asignación automática de IPv4 pública para que los recursos desplegados aquí (como el WebServer) sean accesibles desde Internet.
    *   **Subredes privadas para datos:** Se crearon dos subredes para la base de datos, lo que permite replicación y alta disponibilidad:
        *   `private-subnet-BBDD-1` (`10.0.2.0/24`) en la zona `eu-south-2a`.
        *   `private-subnet-BBDD-2` (`10.0.3.0/24`) en la zona `eu-south-2b`.

## 2. Conectividad exterior (Internet Gateway y enrutamiento)

Para habilitar la comunicación de la subred pública con Internet, se realizaron las siguientes configuraciones:

*   **Internet Gateway (IGW):** Se creó un Internet Gateway y se adjuntó a la VPC `vpc-herzfelder`. Este componente actúa como puente entre la red de AWS e Internet.
*   **Tablas de enrutamiento:**
    *   Se creó una tabla de enrutamiento dedicada al tráfico público.
    *   Se añadió una ruta hacia `0.0.0.0/0` apuntando al Internet Gateway.
    *   **Asociación:** La tabla se asoció explícitamente a la `public-subnet`.

## 3. Despliegue de la capa de computación (EC2)

Se aprovisionó el servidor encargado de atender las peticiones de los usuarios.

*   **Instancia EC2 (WebServer):**
    *   **AMI:** Amazon Linux 2023 (Kernel 6.1).
    *   **Tipo de instancia:** `t3.micro` — adecuado para entornos de prueba y cargas ligeras.
    *   **Acceso seguro:** Se generó el par de claves SSH `webserver-keypair-herzfelder` en formato `.pem` para la administración remota.
    *   **Red:** Vinculada a `vpc-herzfelder` y ubicada dentro de la `public-subnet`.

## 4. Capa de datos de alta disponibilidad (RDS)

Para la persistencia de datos, se preparó el entorno para una base de datos gestionada con réplica.

*   **Grupo de subredes de DB:** Se creó un *DB Subnet Group* que agrupa las dos subredes privadas (`private-subnet-BBDD-1` y `private-subnet-BBDD-2`). Este requisito de RDS permite que la instancia principal y su réplica residan en zonas de disponibilidad distintas, garantizando continuidad del servicio ante el fallo de una zona.
*   **Instancias RDS:** Se desplegó una base de datos principal junto con una réplica de lectura, sincronizadas entre sí para garantizar la integridad de los datos.

## 5. Configuración de seguridad (firewalls)

Se implementó una estrategia de *defensa en profundidad* mediante dos capas de control de acceso:

*   **Network ACL (NACL):** Lista de control de acceso a nivel de subred para filtrar el tráfico entrante y saliente.
*   **Security Groups (firewalls de instancia):**
    *   **WebServer-SecurityGroup:** Reglas de entrada configuradas para permitir tráfico HTTP (80), HTTPS (443) y SSH (22).
    *   **DB Security Group:** Siguiendo el principio de Zona Desmilitarizada (DMZ), el firewall de la base de datos solo acepta tráfico originado desde el Security Group del WebServer, bloqueando cualquier acceso directo externo.

## 6. Acceso remoto y gestión del tráfico

Para la administración de la infraestructura se establecieron los siguientes mecanismos:

*   **Port Forwarding:** Se configuró una estrategia de reenvío de puertos para atravesar los firewalls de forma segura, permitiendo acceder a servicios web internos o gestionar la base de datos sin exponerla directamente a Internet.
*   **Validación:** Se verificó la conectividad mediante las claves SSH generadas, confirmando que el flujo de tráfico entrante y saliente está correctamente orquestado por el Internet Gateway.


---

# Creación de AMI y balanceador de carga (ALB)

![Diagrama de la Arquitectura](./img/DMZ-IMG-3.png)

## 7. Configuración de la infraestructura de red

Antes de duplicar los servidores, se amplió el entorno de red en la VPC:

*   **Nueva subred:** Se creó `DMZ-subnet2` dentro de `vpc-herzfelder`.
*   **Zona y CIDR:** Configurada en la zona de disponibilidad `eu-south-2b` con el bloque CIDR `10.0.5.0/24`.
*   **Enrutamiento:** Se verificó que la tabla de rutas de `DMZ-subnet2` contara con una ruta activa hacia `0.0.0.0/0` a través de la interfaz de red (ENI) correspondiente, habilitando el tráfico externo.

## 8. Creación de la imagen personalizada (AMI)

Para garantizar que los nuevos servidores sean idénticos al original, se generó una Amazon Machine Image (AMI):

*   **Origen:** Instancia `WebServer`.
*   **Generación:** Desde **Acciones > Imagen y plantillas**, se seleccionó **Crear imagen**.
*   **Detalles:** Nombre `Herzfelder-WebServer-AMI`; se habilitó la opción **Reiniciar instancia** para asegurar la coherencia de los datos en el momento de la captura.
*   **Verificación:** Se monitoreó el estado de la AMI en la sección de Imágenes hasta que cambió de *Pendiente* a *Disponible*.

## 9. Despliegue de la segunda instancia

A partir de la imagen creada, se lanzó un segundo servidor:

*   **Lanzamiento desde AMI:** Se seleccionó `Herzfelder-WebServer-AMI` y se ejecutó la opción **Lanzar instancia a partir de una AMI**.
*   **Identificación:** La nueva instancia se denominó `WebServer2`.
*   **Configuración de red y seguridad:**
    *   VPC `vpc-herzfelder` y subred `DMZ-subnet2`.
    *   **Asignación automática de IP pública** habilitada.
    *   Security Group existente `WebServer-SecurityGroup` asociado.
    *   Par de claves `webserver-keypair-herzfelder` para el acceso SSH.

## 10. Configuración del balanceador de carga (ALB)

Por último, se configuró el servicio encargado de distribuir el tráfico entre ambas instancias:

*   **Navegación:** En la consola de EC2, se accedió a **Equilibrio de carga > Balanceadores de carga**.
*   **Tipo seleccionado:** **Application Load Balancer (ALB)** — diseñado para gestionar tráfico HTTP/HTTPS a nivel de aplicación (Capa 7).
*   **Creación:** Se inició el asistente de configuración del balanceador para conectar las instancias `WebServer` y `WebServer2` bajo un único punto de entrada.
