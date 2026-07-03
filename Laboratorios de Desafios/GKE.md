## Google Kubernetes Engine - GKE - GSP510

https://partner.skills.google/paths/77/course_templates/783/labs/612117

### Variables de entorno iniciales

````bash
export PROJECT_ID=qwiklabs-gcp-03-4fef0ff5075f
export REGION=europe-west4
export ZONE=europe-west4-c
export CLUSTER_NAME=hello-world-yx1m
export NAMESPACE=gmp-xqso
````

### Tarea 1: Crea un clúster de GKE


````bash
gcloud container clusters create $CLUSTER_NAME \
    --zone $ZONE \
    --release-channel regular \
    --enable-autoscaling \
    --num-nodes 3 \
    --min-nodes 2 \
    --max-nodes 6
````
Una vez creado, obtén las credenciales para autenticarte con él:

````bash
gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE
````

### Tarea 2: Habilita Prometheus administrado en el clúster de GKE

1. Habilitar Managed Prometheus:


````bash
gcloud container clusters update $CLUSTER_NAME --zone $ZONE --enable-managed-prometheus
````
2. Crear el espacio de nombres:

````bash
kubectl create namespace $NAMESPACE
````
3. Descargar y modificar la aplicación de ejemplo:

````bash
gcloud storage cp gs://spls/gsp510/prometheus-app.yaml .
````

4. Modifica los campos <todo> utilizando los valores solicitados

5. Descargar y modificar Pod Monitoring:

````bash
gcloud storage cp gs://spls/gsp510/pod-monitoring.yaml .
````
6. Modifica los campos <todo> utilizando los valores solicitados

````bash
kubectl apply -f pod-monitoring.yaml -n $NAMESPACE
````

### Tarea 3: Implementa una aplicación en el clúster de GKE


Descargar los archivos:

````bash
gcloud storage cp -r gs://spls/gsp510/hello-app/ .
````
Desplegar el manifiesto erróneo:

````bash
kubectl apply -f hello-app/manifests/helloweb-deployment.yaml -n $NAMESPACE
````
Si verificas el estado, verás el error esperado (InvalidImageName / ErrImagePull):

````bash
kubectl get pods -n $NAMESPACE
````

### Tarea 4: Crea una métrica basada en registros y una política de alertas

#### Paso 1: Crear la métrica en Cloud Logging

Dirígete a la consola web de GCP -> Logging (Explorador de registros).

````bash
resource.type="k8s_container"
severity=WARNING
"invalid reference format"
````
- Haz clic en Run query (Ejecutar consulta). Deberías ver el error que indica Failed to apply default image tag "<todo>".

- Haz clic en el botón Create metric (Crear métrica).

- Configura los siguientes campos:

    - Metric type: Counter

    - Log metric name: pod-image-errors

- Deja lo demás por defecto y haz clic en Create metric.

#### Paso 2: Crear la política de alertas

Puedes ir directamente a Monitoring -> Alerting o darle clic en los tres puntos de tu métrica recién creada y seleccionar "Create alert from metric".
Configura la condición con los siguientes valores exactos:

- Rolling window: ``10 min``

- Rolling window function: ``count``

- Time series aggregation: ``sum``

- Condition type: ``Threshold``

- Alert trigger: ``Any time series violates``

- Threshold position: ``Above threshold``

- Threshold value: ``0``

- Notification channels: Desactívalo o desmarca "Use notification channel".

- Alert policy name: ``Pod Error Alert``

- Guarda la política de alertas (Save).

#### Tarea 5: Actualiza y vuelve a implementar tu app

1. Modificar el archivo manifiesto:

Abre el archivo ``hello-app/manifests/helloweb-deployment.yaml``. Puedes usar sed para reemplazar el marcador:

````bash
sed -i 's|image: <todo>|image: us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.
````
2. Borrar la implementación rota:

````bash
kubectl delete deployment helloweb -n $NAMESPACE
````
3. Volver a implementar el manifiesto corregido:

````bash
kubectl apply -f hello-app/manifests/helloweb-deployment.yaml -n $NAMESPACE
````
4. Verifica que el pod ahora esté en estado Running:

````bash
kubectl get pods -n $NAMESPACE
````

### Tarea 6: Aloja tu código en contenedores e impleméntalo en el clúster


1. Autenticar Docker con Artifact Registry:

````bash
gcloud auth configure-docker europe-west4-docker.pkg.dev
````
2. Actualizar el código de la aplicación (v2.0.0):

Abre ``hello-app/main.go`` y edita la línea 49 para que diga exactamente Version: ``2.0.0.`` O hazlo con este comando:

````bash
sed -i 's|Version: 1.0.0|Version: 2.0.0|g' hello-app/main.go
````
3. Construir la imagen de Docker:

Navega a la carpeta y compila la imagen usando la ruta de tu repositorio ``hello-repo``:

````bash
cd hello-app
docker build -t europe-west4-docker.pkg.dev/$PROJECT_ID/hello-repo/hello-app:v2 .
````
4. Enviar la imagen a Artifact Registry:

````bash
docker push europe-west4-docker.pkg.dev/$PROJECT_ID/hello-repo/hello-app:v2
````
5. Actualizar la implementación en GKE con la imagen v2:

````bash
kubectl set image deployment/helloweb helloweb=europe-west4-docker.pkg.dev/$PROJECT_ID/hello-repo/hello-app:v2 -n $NAMESPACE
````
6. Exponer la aplicación mediante un LoadBalancer:

````bash
kubectl expose deployment helloweb --name=helloweb-service-pm5y --type=LoadBalancer --port=8080 --target-port=8080 -n $NAMESPACE
````
7. Verificar el servicio:

Revisa el progreso hasta que se asigne una IP externa al servicio:

````bash
kubectl get service helloweb-service-pm5y -n $NAMESPACE
````
8. Una vez que veas la External-IP, copia y pega en tu navegador web: ``http://<EXTERNAL_IP>:8080``.

Deberías ver el mensaje con la actualización:

````bash
Hello, world!
Version: 2.0.0
...
````
