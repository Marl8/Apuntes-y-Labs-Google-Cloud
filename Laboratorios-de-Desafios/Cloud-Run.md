## Lab de Desafío Cloud Run - GSP328

### Aprovisiona el entorno para el lab

Durante el lab, asegúrate de que el entorno esté aprovisionado para admitir la implementación de los recursos.


- Abre Cloud Shell en una ventana de Incógnito del navegador.

- Configura el proyecto predeterminado para el entorno:

````bash
gcloud config set project \
$(gcloud projects list --format='value(PROJECT_ID)' \
--filter='qwiklabs-gcp')
````

- Establece la región para las implementaciones de ejecución:

````bash
gcloud config set run/region REGION
````

- Configura el tipo de plataforma de Cloud Run:

````bash
gcloud config set run/platform managed
````

- Clona el repositorio con el código de Pet Theory:

````bash
git clone https://github.com/rosera/pet-theory.git && cd pet-theory/lab07
````

***Nota: Se te otorgó acceso al repositorio de desarrollo. Ten en cuenta la ubicación y usa los recursos para crear la solución según los requisitos establecidos.***

### Tarea 1. Habilita un servicio público

En esta tarea vas a compilar el código de la API de facturación de pruebas (staging) e implementarlo de forma pública.

- Navega al directorio del código:

````bash
cd ~/pet-theory/lab07/unit-api-billing
  ````  

- Compila la imagen usando Cloud Build:

````bash
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/billing-staging-api:0.1
````

- Implementa en Cloud Run de forma sin autenticar (pública):
    
````bash
gcloud run deploy public-billing-service-664 \
    --image gcr.io/$GOOGLE_CLOUD_PROJECT/billing-staging-api:0.1 \
    --allow-unauthenticated
````

### Tarea 2. Implementa un servicio de frontend

Ahora compilarás y desplegarás la interfaz gráfica de pruebas.

- Navega al directorio del frontend:
 
````bash
cd ~/pet-theory/lab07/staging-frontend-billing
````
- Compila la imagen:

````bash
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-staging:0.1
````

- Implementa en Cloud Run de forma pública:

````bash
gcloud run deploy frontend-staging-service-942 \
    --image gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-staging:0.1 \
    --allow-unauthenticated
````

### Tarea 3. Implementa un servicio privado

El equipo quiere proteger la API. Primero borraremos el servicio público anterior, compilaremos la versión 0.2 y la haremos privada.

- Elimina el servicio público anterior (es un requisito de la evaluación):

````bash
gcloud run services delete public-billing-service-664 --quiet
````

- Navega al directorio correspondiente:

````bash
cd ~/pet-theory/lab07/staging-api-billing
````

- Compila la nueva versión (0.2) de la imagen:

````bash
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/billing-staging-api:0.2
````

- Implementa en Cloud Run requiriendo autenticación (privado):

````bash
gcloud run deploy private-billing-service-838 \
    --image gcr.io/$GOOGLE_CLOUD_PROJECT/billing-staging-api:0.2 \
    --no-allow-unauthenticated
````

- Guarda la URL del servicio y haz una prueba de conexión (los comandos que dio el lab):

````bash
BILLING_URL=$(gcloud run services describe private-billing-service-838 --platform managed --region europe-west4 --format "value(status.url)")

curl -X get -H "Authorization: Bearer $(gcloud auth print-identity-token)" $BILLING_URL
````

### Tarea 4: Crea una cuenta de servicio de billing

Vamos a preparar el entorno de producción creando una identidad propia para el servicio de facturación.

  - Crea la cuenta de servicio:

````bash
gcloud iam service-accounts create billing-service-sa-930 \
    --display-name "Servicio de facturación de Cloud Run"
````

### Tarea 5. Implementa el servicio de facturación de producción

Compilaremos la API de producción y le asignaremos la cuenta de servicio que acabamos de crear.

- Navega al directorio de producción de la API:

````bash
cd ~/pet-theory/lab07/prod-api-billing
````

- Compila la imagen de producción:

````bash
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/billing-prod-api:0.1
````

- Implementa el servicio privado asociando la cuenta de servicio:

````bash
gcloud run deploy billing-prod-service-152 \
    --image gcr.io/$GOOGLE_CLOUD_PROJECT/billing-prod-api:0.1 \
    --no-allow-unauthenticated \
    --service-account billing-service-sa-930@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
````

***Ojo: La instrucción de tu lab tiene una pequeña errata en el comando PROD_BILLING_URL provisto (reutiliza el nombre del servicio de staging). Usa este comando corregido para testearlo si lo deseas:***
    
````bash
PROD_BILLING_URL=$(gcloud run services describe billing-prod-service-152 --platform managed --region europe-west4 --format "value(status.url)")
curl -X get -H "Authorization: Bearer $(gcloud auth print-identity-token)" $PROD_BILLING_URL
````

### Tarea 6. Cuenta de servicio de frontend

Para que el frontend de producción pueda "hablarle" a la API privada de facturación, necesita el rol de Invocador (``run.invoker``).

- Crea la cuenta de servicio para el frontend:

````bash
gcloud iam service-accounts create frontend-service-sa-544 \
    --display-name "Invocador del servicio de facturación de Cloud Run"
````

- Dar permiso a esa cuenta de servicio para invocar el servicio de facturación de producción (``billing-prod-service-152``):
    
````bash
gcloud run services add-iam-policy-binding billing-prod-service-152 \
    --member="serviceAccount:frontend-service-sa-544@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com" \
    --role="roles/run.invoker"
````

### Tarea 7. Vuelve a implementar el servicio de frontend

Por último, compilamos el frontend de producción y lo desplegamos públicamente para los usuarios, pero corriendo internamente bajo la identidad de la cuenta de servicio autorizada.

- Navega al directorio del frontend de producción:
    
````bash
cd ~/pet-theory/lab07/prod-frontend-billing
````
- Compila la imagen:
    
````bash
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-prod:0.1
````

- Implementa en Cloud Run de forma pública (``--allow-unauthenticated``) pero asignándole su cuenta de servicio dedicada:
    
````bash
gcloud run deploy frontend-prod-service-801 \
    --image gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-prod:0.1 \
    --allow-unauthenticated \
    --service-account frontend-service-sa-544@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount
````      
