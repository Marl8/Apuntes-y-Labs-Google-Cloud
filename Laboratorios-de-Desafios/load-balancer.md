## Laboratorio de Desafío Load Balancer - GSP313

### Tarea 1: Crea varias instancias de servidor web

En esta tarea crearemos las 3 VMs en la red default, instalaremos Apache usando un script de inicio (startup-script) y abriremos el firewall.

#### 1.1. Crear las 3 instancias de VM

````bash
# Definir variables comunes
REGION=europe-west4
ZONE=europe-west4-b

# Crear VM 1 (web1)
gcloud compute instances create web1 \
    --zone=$ZONE \
    --machine-type=e2-small \
    --tags=network-lb-tag \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install apache2 -y
    service apache2 restart
    echo "<h3>Web Server: web1</h3>" | tee /var/www/html/index.html'

# Crear VM 2 (web2)
gcloud compute instances create web2 \
    --zone=$ZONE \
    --machine-type=e2-small \
    --tags=network-lb-tag \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install apache2 -y
    service apache2 restart
    echo "<h3>Web Server: web2</h3>" | tee /var/www/html/index.html'

# Crear VM 3 (web3)
gcloud compute instances create web3 \
    --zone=$ZONE \
    --machine-type=e2-small \
    --tags=network-lb-tag \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install apache2 -y
    service apache2 restart
    echo "<h3>Web Server: web3</h3>" | tee /var/www/html/index.html'
````

#### 1.2. Crear la regla de firewall

Para que el tráfico HTTP pueda llegar a las VMs, necesitamos abrir el puerto 80 para las máquinas que tengan la etiqueta ``network-lb-tag``.

````bash
gcloud compute firewall-rules create www-firewall-network-lb \
    --target-tags=network-lb-tag \
    --allow=tcp:80
````

### Tarea 2: Configura el servicio de balanceo de cargas (Network Load Balancer)

Esta tarea requiere un balanceador de cargas de red (Capa 4). Vamos a crear la IP estática, el grupo de destino (``target pool``) y la regla de reenvío (``forwarding rule``).

#### 2.1. Crear la IP externa estática

````bash
gcloud compute addresses create network-lb-ip-1 \
    --region=$REGION
````

#### 2.2. Crear el http-health-check

````bash
gcloud compute http-health-checks create basic-check
````

#### 2.2. Crear el grupo de destino y añadir las VMs

````bash
# Crear el grupo de destino www-pool
gcloud compute target-pools create www-pool \
    --region=$REGION \
    --http-health-check=basic-check

# Añadir las 3 VMs creadas al grupo
gcloud compute target-pools add-instances www-pool \
    --instances=web1,web2,web3 \
    --instances-zone=$ZONE
````

#### 2.3. Crear la regla de reenvío

````bash
gcloud compute forwarding-rules create www-rule \
    --region=$REGION \
    --ports=80 \
    --address=network-lb-ip-1 \
    --target-pool=www-pool
````

### Tarea 3: Crea un balanceador de cargas HTTP (Application Load Balancer)

Para el balanceador HTTP (Capa 7), el laboratorio te pide usar un Grupo de Instancias Administrado (MIG). Primero crearemos la plantilla, luego el grupo, y finalmente el balanceador con sus componentes.

#### 2.1. Crear la regla de firewall para los Health Checks de Google Cloud

Los balanceadores HTTP de Google Cloud necesitan comunicarse con las VMs desde rangos de IP específicos de Google para verificar si están activas (130.211.0.0/22 y 35.191.0.0/16).

````bash
gcloud compute firewall-rules create fw-allow-health-check \
    --network=default \
    --action=allow \
    --direction=ingress \
    --source-ranges=130.211.0.0/22,35.191.0.0/16 \
    --target-tags=allow-health-check \
    --rules=tcp:80
````
#### 3.2. Crear la plantilla de instancias (Instance Template)


````bash
gcloud compute instance-templates create lb-backend-template \
    --region=$REGION \
    --network=default \
    --subnet=default \
    --tags=allow-health-check \
    --machine-type=e2-medium \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install apache2 -y
    service apache2 restart
    echo "<h3>Web Server: lb-backend-group</h3>" | tee /var/www/html/index.html'
````

#### 3.3. Crear el Grupo de Instancias Administrado (MIG)

````bash
gcloud compute instance-groups managed create lb-backend-group \
    --template=lb-backend-template \
    --size=2 \
    --zone=$ZONE
````

#### 3.4. Configurar los componentes del Balanceador de Cargas HTTP

Aquí creamos el Health Check, el servicio de backend, el mapa de URLs, el Proxy HTTP y la IP global.


````bash
# 1. Crear la verificación de estado (Health Check)
gcloud compute health-checks create http http-basic-check \
    --port=80

# 2. Crear el servicio de backend global
gcloud compute backend-services create web-backend-service \
    --protocol=HTTP \
    --port-name=http \
    --health-checks=http-basic-check \
    --global

# 3. Añadir el grupo de instancias (MIG) al servicio de backend
gcloud compute instance-groups managed set-named-ports lb-backend-group \
    --named-ports http:80 \
    --zone=$ZONE

gcloud compute backend-services add-backend web-backend-service \
    --instance-group=lb-backend-group \
    --instance-group-zone=$ZONE \
    --global

# 4. Crear el mapa de URLs (URL Map)
gcloud compute url-maps create web-map-http \
    --default-service=web-backend-service

# 5. Crear el Proxy HTTP de destino
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map=web-map-http

# 6. Reservar una IP externa global para el balanceador HTTP
gcloud compute addresses create lb-ipv4-1 \
    --ip-version=IPV4 \
    --global

# 7. Crear la regla de reenvío global (Forwarding Rule)
gcloud compute forwarding-rules create http-content-rule \
    --address=lb-ipv4-1 \
    --global \
    --target-http-proxy=http-lb-proxy \
    --ports=80
````

### Explicación

### Paso 1: Crear la infraestructura base y la red

Antes de balancear tráfico, necesitamos servidores reales a los cuales enviar ese tráfico.

#### 1.1. Crear las tres Máquinas Virtuales (VMs)

Ejecutamos tres comandos gcloud compute instances create para web1, web2 y web3.

- ``--machine-type=e2-small``: Define el tamaño de la máquina (CPU y RAM).

- ``--tags=network-lb-tag``: Esto es crucial. Le ponemos una "etiqueta" a las máquinas para luego poder aplicarles reglas de firewall o balanceo masivamente.

- ``--metadata=startup-script="..."``: Es un script que la VM ejecuta automáticamente apenas se enciende. Actualiza el sistema, instala el servidor web Apache, lo reinicia y escribe un archivo index.html con el nombre de la máquina para que sepamos cuál nos está respondiendo.

#### 1.2. Crear la regla de firewall (www-firewall-network-lb)

Por defecto, Google Cloud bloquea todo el tráfico entrante del exterior por seguridad.

- Con gcloud compute firewall-rules create, creamos una regla que permite el protocolo TCP en el puerto ``80 (HTTP)``.

- Usamos ``--target-tags=network-lb-tag`` para decirle a Google Cloud: "Aplica esta regla únicamente a las máquinas que tengan esta etiqueta".

### Paso 2: Configurar el Balanceador de Cargas de Red (Network Load Balancer)

Este tipo de balanceador trabaja en la Capa 4 (Transporte). Solo mira direcciones IP y puertos, distribuyendo el tráfico de forma muy rápida y eficiente.

#### 2.1 Reservar una IP estática (network-lb-ip-1)

Con gcloud compute addresses create, le pedimos a Google Cloud que nos reserve una dirección IP pública fija en la región de Europa. Esta será la IP pública a la que los usuarios accederán.

#### 2.2 Crear el Health Check (basic-check)

Con gcloud compute http-health-checks create, configuramos un sistema de "monitoreo de salud". El balanceador enviará pings constantemente a las VMs para asegurarse de que Apache sigue vivo. Si una VM se cae, deja de enviarle tráfico.

#### 2.3 Crear el Grupo de Destino (www-pool) y añadir las VMs

Un balanceador de red necesita un Target Pool (grupo de destino) para saber a dónde redirigir a los usuarios.

- Lo creamos asociándole el basic-check que hicimos en el paso anterior.

- Luego, con add-instances, metemos a web1, web2 y web3 dentro de esa bolsa (pool).

#### 2.4 Crear la Regla de Reenvío (www-rule)

Este es el "frente" del balanceador. Conecta la IP estática externa con nuestro grupo de VMs internas en el puerto ``80``. Cuando el tráfico llega a la IP estática, esta regla lo empuja hacia el ``www-pool``.

### Paso 3: Configurar el Balanceador de Cargas HTTP (Application Load Balancer)

Este balanceador trabaja en la Capa 7 (Aplicación). Puede tomar decisiones inteligentes basadas en la URL (por ejemplo, enviar los videos a un servidor y las imágenes a otro). Requiere un Grupo de Instancias Administrado (MIG), lo que significa que Google Cloud puede crear o destruir VMs automáticamente según la demanda.

#### 3.1 Firewall para los Health Checks de Google

A diferencia del balanceador de red, el balanceador HTTP de Google utiliza unos rangos de IP específicos de Google (``130.211.0.0/22`` y ``35.191.0.0/16``) para revisar si las máquinas están vivas. Creamos la regla fw-allow-health-check para permitir que esas IPs de Google hablen con nuestras VMs en el puerto ``80``.

#### 3.2 Crear la Plantilla de Instancia (lb-backend-template)

Para que Google Cloud pueda crear máquinas automáticamente si el tráfico sube, necesita una plantilla: se define que cada máquina nueva que se cree debe ser ``e2-medium``, tener ``Debian 12`` y llevar el script que instala Apache.

#### 3.3 Crear el Grupo de Instancias Administrado (lb-backend-group)

Usando la plantilla del paso anterior, le decimos a Google: "Crea un grupo automatizado y asegúrate de que siempre haya 2 máquinas activas (``--size=2``)". Google Cloud creará dos VMs nuevas de forma automática con nombres aleatorios basados en la plantilla.

#### 3.4 Unir las piezas del Balanceador HTTP

Aquí es donde se conecta todo mediante la arquitectura de Capa 7:

- Health Check (``http-basic-check``): Otra verificación de estado, pero esta vez optimizada para HTTP global.

- Backend Service (``web-backend-service``): Es el componente del balanceador que gestiona los grupos de instancias. Le asignamos nuestro grupo administrado (lb-backend-group) y el Health Check.

- URL Map (``web-map-http``): El enrutador. Define qué rutas van a qué backends. Como aquí es un laboratorio simple, le decimos que todo el tráfico vaya por defecto a nuestro servicio de backend único.

- Target HTTP Proxy (``http-lb-proxy``): Recibe las peticiones del exterior, analiza el mapa de URLs y las pasa al backend.

- IP Global (``lb-ipv4-1``) y Regla de Reenvío (``http-content-rule``): Creamos una IP pública global (válida en todo el mundo) y configuramos la regla final para que todo el tráfico que llegue a esa IP en el puerto 80 sea entregado al Proxy HTTP.
