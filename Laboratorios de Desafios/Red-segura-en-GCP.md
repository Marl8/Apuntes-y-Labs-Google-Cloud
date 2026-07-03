## Crea una Red Segura en Google Cloud - Laboratorio de Dasafío - GSP342

### Paso 1: Quitar las reglas demasiado permisivas

El laboratorio te pide revisar y eliminar reglas laxas. Por lo general, estos laboratorios crean una regla llamada open-everything o similar que permite todo el tráfico.

- Ve a Red de VPC > Firewall en la consola.

- Busca reglas que tengan rangos ``0.0.0.0/0`` con todos los puertos abiertos.

- Selecciónala y haz clic en Borrar.

### Paso 2: Iniciar la instancia del host de bastión

- Ve a Compute Engine > Instancias de VM.

- Selecciona la instancia llamada bastion.

- Haz clic en Iniciar / Reanudar en la barra superior.

### Paso 3: Permitir SSH al Bastión a través de IAP

El rango de IPs oficial y exclusivo que utiliza el servicio de Identity-Aware Proxy (IAP) de Google para el reenvío de TCP es ``35.235.240.0/20``. Debes aplicar la etiqueta exacta provista por tu instrucción: ``permit-ssh-iap-ingress-ql-808``.

#### Opción A: Por comando en Cloud Shell (Recomendado)

Ejecuta el siguiente comando para crear la regla de firewall:

````Bash
gcloud compute firewall-rules create allow-ssh-from-iap-to-bastion \
    --direction=INGRESS \
    --network=lab-net \
    --action=ALLOW \
    --rules=tcp:22 \
    --source-ranges=35.235.240.0/20 \
    --target-tags=permit-ssh-iap-ingress-ql-808
````

#### Opción B: Por Consola Web

- Ve a Firewall > Crear regla de firewall.

- Nombre: ``allow-ssh-from-iap-to-bastion``

- Etiquetas de destino: ``permit-ssh-iap-ingress-ql-808``

- Rangos de IP de origen: ``35.235.240.0/20``

- Protocolos y puertos: Selecciona TCP y escribe ``22``.

⚠️ ***¡IMPORTANTE! Ahora debes asignarle la etiqueta a la VM. Ve a Compute Engine > Instancias de VM > Edita la instancia bastion > En el campo Etiquetas de red, escribe permit-``ssh-iap-ingress-ql-808`` y guarda los cambios.***

### Paso 4: Permitir tráfico HTTP público a juice-shop

Debes permitir que cualquiera acceda al puerto 80 de la aplicación usando la etiqueta ``permit-http-ingress-ql-808``.

#### Opción A: Por comando en Cloud Shell (Recomendado)

````Bash
gcloud compute firewall-rules create allow-http-to-juice-shop \
    --direction=INGRESS \
    --action=ALLOW \
    --rules=tcp:80 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=permit-http-ingress-ql-808
````

#### Opción B: Por Consola Web

- Nombre: ``allow-http-to-juice-shop``

- Etiquetas de destino: ``permit-http-ingress-ql-808``

- Rangos de IP de origen: ``0.0.0.0/0``

- Protocolos y puertos: Selecciona TCP y escribe ``80``.

- ⚠️ Asigna la etiqueta: Edita la instancia juice-shop, agrega la etiqueta ``permit-http-ingress-ql-808`` en sus etiquetas de red y guarda.

### Paso 5: Permitir SSH a juice-shop SOLO desde el Bastión

El laboratorio especifica que el origen debe ser la subred del bastión: ``acme-mgmt-subnet``. Primero necesitamos saber cuál es el rango de IPs (CIDR) de esa subred.

#### 1. Averigua el rango de la subred

Ejecuta esto en Cloud Shell para ver los datos de la subred:

````Bash
gcloud compute networks subnets describe acme-mgmt-subnet --region=us-central1
````

(Si el laboratorio usa otra región, cambia us-central1 por la tuya. Busca la línea que dice ipCidrRange, por ejemplo: ``192.168.10.0/24`` o ``10.x.x.x/x``).

#### 2. Crea la regla de firewall

Reemplaza [RANGO_DE_ACME_MGMT_SUBNET] por el CIDR numérico que obtuviste en el paso anterior y usa la etiqueta permit-ssh-internal-ingress-ql-808:

````Bash
gcloud compute firewall-rules create allow-ssh-internal-to-juice-shop \
    --direction=INGRESS \
    --action=ALLOW \
    --rules=tcp:22 \
    --source-ranges=[RANGO_DE_ACME_MGMT_SUBNET] \
    --target-tags=permit-ssh-internal-ingress-ql-808
````

⚠️ ***Asigna la etiqueta: Edita la instancia juice-shop, añade la etiqueta permit-ssh-internal-ingress-ql-808 en sus etiquetas de red y guarda.***

### Paso 6: Conectarse por SSH a juice-shop mediante el Bastión

- Ve a Compute Engine > Instancias de VM.

- Busca la instancia bastion y haz clic en el botón SSH que aparece a su derecha. Esto abrirá una ventana de terminal externa conectándose de manera segura a través de IAP.

- Una vez dentro del terminal del bastión, conéctate de forma interna a juice-shop usando su nombre o su IP privada:

```Bash
- gcloud compute ssh juice-shop --internal-ip
```
