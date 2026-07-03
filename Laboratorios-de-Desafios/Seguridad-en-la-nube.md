## Lad de Desafío - Implementa los aspectos básicos de seguridad en la nube en Google Cloud - GSP342

### Sugerencias y trucos

- **Sugerencia 1**. Asegúrate de usar gke-gcloud-auth-plugin, que se necesita para el uso continuo de kubectl. Puedes ejecutar los siguientes comandos para instalarlo. Asegúrate de reemplazar el nombre y la zona de clúster de GKE, al igual que el ID del proyecto.

````bash
sudo apt-get install google-cloud-sdk-gke-gcloud-auth-plugin
echo "export USE_GKE_GCLOUD_AUTH_PLUGIN=True" >> ~/.bashrc
source ~/.bashrc
gcloud container clusters get-credentials <your cluster name> --internal-ip --project=<project ID> --zone <cluster zone>
````

- **Sugerencia 2**. Cuando agregues la dirección IP interna de la máquina orca-jumphost a la lista de direcciones autorizadas del clúster privado de Kubernetes Engine, debes usar una máscara de red /32 para garantizar que esté autorizada solamente la instancia de procesamiento específica.

- **Sugerencia 3**. No puedes conectarte directamente a un clúster privado de Kubernetes Engine desde una VPC o desde otra red fuera de la VPC donde se implementó el clúster privado si se especificó la opción enable-private-endpoint. Esta es la opción de mayor seguridad de un clúster privado. Debes usar un jumphost, o un proxy que se encuentre en la misma VPC que el clúster, para conectarte a la dirección IP de administración interna del clúster.


Antes de comenzar crea las varibles de entorno

````bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION="europe-west4"
export ZONE="europe-west4-a"
````

### Tarea 1. Crea un rol de seguridad personalizado

Debes crear un rol personalizado de IAM llamado ``orca_storage_editor_323`` con los permisos específicos para Cloud Storage.

Ejecuta el siguiente comando en tu Cloud Shell:

````bash
gcloud iam roles create orca_storage_editor_323 \
    --project=gcp-04-8bbca537b1fd \
    --title="Orca Storage Editor" \
    --description="Permisos para agregar y actualizar objetos en Cloud Storage" \
    --permissions="storage.buckets.get,storage.objects.get,storage.objects.list,storage.objects.update,storage.objects.create" \
    --stage="GA"
````

### Tarea 2. Crea una cuenta de servicio

Crea la cuenta de servicio dedicada llamada orca-private-cluster-691-sa.


````bash
gcloud iam service-accounts create orca-private-cluster-691-sa \
    --display-name="Orca Private Cluster Service Account"
````

### Tarea 3: Vincula un rol de seguridad personalizado a una cuenta de servicio

Asigna los 3 roles integrados requeridos por GKE más el rol personalizado que creaste en la Tarea 1 a la cuenta de servicio recién creada.

````bash
# 1. Vincular Monitoring Viewer
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:orca-private-cluster-691-sa@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/monitoring.viewer"

# 2. Vincular Monitoring Metric Writer
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:orca-private-cluster-691-sa@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/monitoring.metricWriter"

# 3. Vincular Logging Log Writer
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:orca-private-cluster-691-sa@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/logging.logWriter"

# 4. Vincular el Rol Personalizado de la Tarea 1
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:orca-private-cluster-691-sa@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="projects/$PROJECT_ID/roles/orca_storage_editor_323"
````

### Tarea 4: Crea y configura un nuevo clúster privado de Kubernetes Engine

Para configurar la red autorizada, primero necesitamos averiguar la IP interna de la máquina virtual orca-jumphost. Consíguela ejecutando:


````bash
export JUMPHOST_IP=$(gcloud compute instances describe orca-jumphost \
    --zone=$ZONE \
    --format='get(networkInterfaces[0].networkIP)')

echo "La IP interna de orca-jumphost es: $JUMPHOST_IP"
````

Ahora, crea el clúster privado ``orca-cluster-825`` cumpliendo con todas las condiciones de seguridad solicitadas (el comando usa la máscara ``/32`` sugerida para la IP del jumphost):


````bash
gcloud container clusters create orca-cluster-825 \
    --zone=$ZONE \
    --num-nodes=1 \
    --network=orca-build-vpc \
    --subnetwork=orca-build-subnet \
    --service-account="orca-private-cluster-691-sa@$PROJECT_ID.iam.gserviceaccount.com" \
    --enable-private-nodes \
    --enable-private-endpoint \
    --enable-ip-alias \
    --enable-master-authorized-networks \
    --master-authorized-networks="$JUMPHOST_IP/32"
````

***Nota: La creación del clúster privado puede tardar entre 5 y 10 minutos.***

### Tarea 5. Implementa una aplicación en un clúster privado de Kubernetes Engine

Debido a que el extremo público está inhabilitado por completo (``enable-private-endpoint``), no puedes administrar el clúster desde Cloud Shell. Debes conectarte por S``SSH`` al jumphost.

#### Paso 1: Conéctate por SSH a orca-jumphost

En la consola de Google Cloud, ve a Compute Engine > VM Instances, busca orca-jumphost y haz clic en el botón ``SSH``.

Alternativamente, si usas la terminal con gcloud, puedes acceder mediante:

````bash
gcloud compute ssh orca-jumphost --zone=europe-west4-a
````

#### Paso 2: Configura el entorno dentro del Jumphost

Una vez dentro de la terminal de la máquina orca-jumphost, define el ID de tu proyecto (reemplaza <TU_PROJECT_ID> por el ID real del laboratorio):


````bash
export PROJECT_ID=gcp-04-8bbca537b1fd
export ZONE="europe-west4-a"
````

#### Paso 3: Instala el plugin de autenticación y obtén las credenciales

Ejecuta los comandos de la Sugerencia 1 para preparar kubectl:

````bash
sudo apt-get update && sudo apt-get install google-cloud-sdk-gke-gcloud-auth-plugin -y
echo "export USE_GKE_GCLOUD_AUTH_PLUGIN=True" >> ~/.bashrc
source ~/.bashrc

# Obtén las credenciales usando obligatoriamente la IP interna
gcloud container clusters get-credentials orca-cluster-825 --internal-ip --project=$PROJECT_ID --zone=$ZONE
````

#### Paso 4: Implementa la aplicación de prueba

Crea el despliegue de la aplicación de muestra:

````bash
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
````

Para verificar que todo se haya creado correctamente y asegurar la puntuación de la tarea, puedes ejecutar:

````bash
kubectl get deployments
````
