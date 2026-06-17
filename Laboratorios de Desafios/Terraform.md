## Laboratorio de Dasafío - Crea una infraestructura con Terraform 

### Tarea 1. Crea los archivos de configuración

#### Paso 1.1: Crear la estructura de directorios y archivos

Ejecuta los siguientes comandos en Cloud Shell para crear todas las carpetas y archivos vacíos necesarios:

````Bash
mkdir -p modules/instances modules/storage
touch main.tf variables.tf
touch modules/instances/instances.tf modules/instances/outputs.tf modules/instances/variables.tf
touch modules/storage/storage.tf modules/storage/outputs.tf modules/storage/variables.tf
````

#### Paso 1.2: Completar las variables globales y de los módulos

Debes agregar las mismas tres variables en tres archivos diferentes: ``variables.tf``, ``modules/instances/variables.tf`` y ``modules/storage/variables.tf``.

Abre los archivos (puedes usar el editor de Cloud Shell o nano) y pega lo siguiente en cada uno de ellos. Recuerda reemplazar <TU_PROJECT_ID_AQUÍ> por tu ID de proyecto real de GCP:

````python
Terraform

variable "region" {
  type    = string
  default = "us-east4"
}

variable "zone" {
  type    = string
  default = "us-east4-c"
}

variable "project_id" {
  type    = string
  default = "qwiklabs-gcp-04-51af90b50e44"
}
````

#### Paso 1.3: Configurar el archivo raíz main.tf e inicializar

Abre el archivo ``main.tf`` en la raíz de tu proyecto y agrega el bloque de configuración de Terraform y el proveedor de Google:

````python
Terraform

terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 4.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}
````

Ahora, ejecuta el comando para inicializar el directorio:

````Bash
terraform init
````

### Tarea 2. Importa infraestructura

#### Paso 2.1: Agregar la referencia del módulo en main.tf

Modifica el archivo ``main.tf`` de la raíz agregando la llamada al módulo de instancias al final:

````python
Terraform

module "instances" {
  source     = "./modules/instances"
  project_id = var.project_id
  region     = var.region
  zone       = var.zone
}
````

Vuelve a inicializar para que Terraform reconozca el nuevo módulo:

````Bash
terraform init
````

#### Paso 2.2: Escribir la configuración mínima en modules/instances/instances.tf

Ve a la consola de GCP -> Compute Engine -> Instancias de VM para verificar el tipo de máquina e imagen de ``tf-instance-1 ``y ``tf-instance-2``.

Escribe lo siguiente en ``modules/instances/instances.tf``:

````python
# Terraform

resource "google_compute_instance" "tf_instance_1" {
  name         = "tf-instance-1"
  machine_type = "e2-micro" 
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11" 
    }
  }

  network_interface {
    network = "default"
  }

  metadata_startup_script   = "#!/bin/bash"
  allow_stopping_for_update = true
}

resource "google_compute_instance" "tf_instance_2" {
  name         = "tf-instance-2"
  machine_type = "e2-micro"
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    network = "default"
  }

  metadata_startup_script   = "#!/bin/bash"
  allow_stopping_for_update = true
}
````

#### Paso 2.3: Importar las instancias reales al estado de Terraform

Para realizar la importación hacia el módulo, ejecuta los siguientes comandos desde la raíz de tu proyecto en Cloud Shell. (Nota: Necesitaremos el ID de cada instancia, pero si usas el nombre de la instancia funciona igual en entornos estándar de GCP).

````Bash
terraform import module.instances.google_compute_instance.tf_instance_1 tf-instance-1
terraform import module.instances.google_compute_instance.tf_instance_2 tf-instance-2
````

Una vez importadas con éxito, ejecuta:

````Bash
terraform plan
terraform apply -auto-approve
````

### Tarea 3. Configura un backend remoto

#### Paso 3.1: Configurar el módulo de almacenamiento

Abre ``modules/storage/storage.tf`` y añade el recurso del bucket:

````python
# Terraform

resource "google_storage_bucket" "remote_backend" {
  name                        = "tf-bucket-084158"
  location                    = "US"
  force_destroy               = true
  uniform_bucket_level_access = true
}
````

#### Paso 3.2: Llamar al módulo en main.tf y crearlo

Abre el archivo ``main.tf`` raíz y añade la referencia al módulo al final del archivo:

````python
Terraform

module "storage" {
  source     = "./modules/storage"
  project_id = var.project_id
  region     = var.region
  zone       = var.zone
}
````

Ejecuta los comandos para inicializar y aplicar (crear el bucket):

````Bash
terraform init
terraform apply -auto-approve
````

#### Paso 3.3: Cambiar el backend a remoto

Ahora que el bucket ya existe en Google Cloud, abre tu ``main.tf`` raíz y modifica el bloque inicial de terraform agregando el bloque backend:

````python
# Terraform

terraform {
  backend "gcs" {
    bucket = "tf-bucket-084158"
    prefix = "terraform/state"
  }
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 4.0"
    }
  }
}
````

Ejecuta el siguiente comando para migrar el estado:

````Bash
terraform init
````

### Tarea 4. Modifica y actualiza infraestructura

#### Paso 4.1: Modificar las instancias y añadir la tercera

Abre ``modules/instances/instances.tf`` y actualiza el machine_type de las dos primeras a ``e2-standard-2``. Al mismo tiempo, agrega el bloque de código para la tercera VM (``tf-instance-758949``):

````python
# Terraform

# Modifica la 1
resource "google_compute_instance" "tf_instance_1" {
  name         = "tf-instance-1"
  machine_type = "e2-standard-2"
  zone         = var.zone
  # ... (mantén el resto del bloque igual)
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  network_interface {
    network = "default"
  }
  metadata_startup_script   = "#!/bin/bash"
  allow_stopping_for_update = true
}

# Modifica la 2
resource "google_compute_instance" "tf_instance_2" {
  name         = "tf-instance-2"
  machine_type = "e2-standard-2"
  zone         = var.zone
  # ... (mantén el resto del bloque igual)
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  network_interface {
    network = "default"
  }
  metadata_startup_script   = "#!/bin/bash"
  allow_stopping_for_update = true
}

# Agrega la nueva instancia 3
resource "google_compute_instance" "tf_instance_3" {
  name         = "tf-instance-758949"
  machine_type = "e2-standard-2"
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    network = "default"
  }

  metadata_startup_script   = "#!/bin/bash"
  allow_stopping_for_update = true
}
````

Ejecuta los cambios:

````Bash
terraform apply -auto-approve
````

### Tarea 5. Destruye recursos

#### Paso 5.1: Eliminar la tercera instancia

Abre de nuevo modules/instances/instances.tf y borra por completo el bloque del recurso ``google_compute_instance "tf_instance_3"``. Guarda el archivo.

Aplica los cambios para que Terraform la destruya:

````Bash
terraform apply -auto-approve
````

### Tarea 6. Usar un módulo de Registry

#### Paso 6.1: Añadir el módulo de red de Terraform Registry

Abre tu ``main.tf`` raíz y añade la configuración para invocar la creación de la VPC y sus subredes usando el módulo oficial:

````python
# Terraform

module "vpc" {
  source  = "terraform-google-modules/network/google"
  version = "10.0.0"

  project_id   = var.project_id
  network_name = "tf-vpc-188971"
  routing_mode = "GLOBAL"

  subnets = [
    {
      subnet_name   = "subnet-01"
      subnet_ip     = "10.10.10.0/24"
      subnet_region = var.region
    },
    {
      subnet_name   = "subnet-02"
      subnet_ip     = "10.10.20.0/24"
      subnet_region = var.region
    }
  ]
}
````

Ejecuta la inicialización y aplica para construir la red:

````Bash
terraform init
terraform apply -auto-approve
````

#### Paso 6.2: Conectar las instancias a las nuevas subredes

Abre ``modules/instances/instances.tf`` y edita el bloque ``network_interface``de ambas instancias para que apunten a la nueva red y a sus respectivas subredes:

````python
Terraform

# Modifica en tf_instance_1
  network_interface {
    network    = "tf-vpc-188971"
    subnetwork = "subnet-01"
  }

# Modifica en tf_instance_2
  network_interface {
    network    = "tf-vpc-188971"
    subnetwork = "subnet-02"
  }
````

Aplica los cambios en la infraestructura:

````Bash
terraform apply -auto-approve
````
### Tarea 7. Configura un firewall

#### Paso 7.1: Crear la regla de firewall en main.tf

Abre el ``main.tf`` raíz y añade el recurso de firewall al final de todo:

````python
# Terraform

resource "google_compute_firewall" "tf_firewall" {
  name    = "tf-firewall"
  network = "projects/${var.project_id}/global/networks/tf-vpc-188971"

  allow {
    protocol = "tcp"
    ports    = ["80"]
  }

  source_ranges = ["0.0.0.0/0"]
}
````

#### Paso 7.2: Despliegue final

Ejecuta el último empujón para aplicar la regla de firewall:

````Bash
terraform apply -auto-approve
````