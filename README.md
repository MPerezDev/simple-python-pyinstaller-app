# Entregable 3 - Terraform + SCV + Jenkins
Manuel Pérez Ruiz - José Luis Argudo Garrido

## Introducción
En este documento se encuentran los distintos pasos que se tienen que dar para llevar a cabo la realización de la práctica. Suponemos que no es necesario explicar en este documento la instalación de las aplicaciones necesarias como Docker o Terraform.

## Paso 1 - Creación de la imagen de Jenkins y subida a Docker Hub
Para la creación de la imagen de Jenkins necesitamos un archivo Dockerfile que contenga lo siguiente:

``` dockerfile
FROM jenkins/jenkins:2.426.2-jdk17
USER root
RUN apt-get update && apt-get install -y lsb-release
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
  https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean:1.27.9 docker-workflow:572.v950f58993843"
```

Una vez creado este fichero, contruimos la iamgen personalizada de Jenkins con el siguiente comando introducido por consola:

```
docker build -t myjenkins-blueocean:2.426.2-1 . 
```

Cuando lo hemos creado, usamos los dos siguientes comandos en la consola para subir la imagen a Docker Hub y poder usarla posteriormente:

```
docker tag myjenkins-blueocean:2.426.2-1 argudo/myjenkins-blueocean:2.426.2-1
docker push argudo/myjenkins-blueocean:2.426.2-1
```

Con todas estas instrucciones ya tenemos la imagen personalizada de Jenkins creada y disponible para usar desde Docker Hub de cara a los siguientes pasos.

## Paso 2 - Configuración y uso de Terraform

En esta fase vamos a usar el fichero main.tf, que presenta la siguiente estructura:

``` tf
terraform {
  required_providers {
    docker = {
      source = "kreuzwerker/docker"
      version = "~> 3.0.1"
    }
  }
}

provider "docker" {
  host    = "npipe:////.//pipe//docker_engine"
}

resource "docker_network" "my_network" {
  name   = "my_network"
}

resource "docker_volume" "certs" {
  name = "certs"
}

resource "docker_volume" "data" {
  name = "data"
}

resource "docker_container" "jenkins" {
  name  = "jenkins"
  image = "argudo/myjenkins-blueocean:2.426.2-1"
  privileged = true

  env = [     
    "DOCKER_HOST=tcp://docker:2376",     
    "DOCKER_CERT_PATH=/certs/client",     
    "DOCKER_TLS_VERIFY=1",     
    "JAVA_OPTS=-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true",   
    ]

  ports {
    internal = 8080
    external = 8080
  }
  networks_advanced {
    name = docker_network.my_network.name
  }
  volumes{
    volume_name = docker_volume.data.name
    container_path = "/var/jenkins_home"
  }
  volumes{
    volume_name = docker_volume.certs.name
    container_path = "/certs/client"
  }

}

resource "docker_container" "dind" {
  name  = "dind"
  image = "docker:dind"
  privileged = true
  networks_advanced {
    name = docker_network.my_network.name
    aliases = ["docker"]
  }

  env = [     
    "DOCKER_TLS_CERTDIR=/certs",   
    ]

  ports {
    internal = 3000
    external = 3000
  }
  ports {
    internal = 5000
    external = 5000
  }
  ports {
    internal = 2376
    external = 2376
  }
  volumes{
    volume_name = docker_volume.data.name
    container_path = "/var/jenkins_home"
  }
  volumes{
    volume_name = docker_volume.certs.name
    container_path = "/certs/client"
  }



}
```

Este fichero presenta la estructura necesaria para el despliegue del entorno, abarcando la configuración del proveedor Docker, la red "My network", los volúmenes "certs"y "data" y, por último, los contenedores (dos en este caso, uno para Jenkins y otro para Docker-in-Docker) que nos hacen falta según la especificación de la práctica.


Habiendo preparado el fichero, ejecutamos un terminal en la dirección en la que se encuentra el mismo para ejecutar los dos siguientes comandos, los cuales sirven para iniciar y aplicar Terraform respectivamente (deben ejecutarse en el orden en que se presentan):

```
terraform init
terraform apply
```

Cabe destacar que cuando ejecutemos el comando terraform apply, se nos va a pedir que confirmemos los cambios que se van a efectuar mediante el término "yes", introducido por teclado.

Una vez haya terminado la ejecución del comando terraform apply, podemos pasar a la siguiente fase de esta práctica.


## Paso 3 - Preparación de Jenkinsfile

En primer lugar, antes de comenzar con la configuración de Jenkins, debemos crear un fichero Jenkinsfile en el que tendremos un pipeline como el siguiente:

``` groovy

pipeline {
    agent none
    stages {
        stage('Build') {
            agent {
                docker {
                    image 'python:2-alpine'
                }
            }
            steps {
                sh 'python -m py_compile sources/add2vals.py sources/calc.py'
            }
        }
        stage('Test') {
            agent {
                docker {
                    image 'qnib/pytest'
                }
            }
            steps {
                sh 'py.test --verbose --junit-xml test-reports/results.xml sources/test_calc.py'
            }
            post {
                always {
                    junit 'test-reports/results.xml'
                }
            }
        }
        stage('Deliver') {
            agent {
                docker {
                    image 'cdrx/pyinstaller-linux:python2'
                }
            }
            steps {
                sh 'pyinstaller --onefile sources/add2vals.py'
            }
            post {
                success {
                    archiveArtifacts 'dist/add2vals'
                }
            }
        }
    }
}

```

En este fichero distinguimos tres etapas principales que se encargan de gestionar una parte del despliegue de la aplicación de Python que se nos pide, de tal forma que en cada una se van indicando en pantalla los pasos que se están dando dentro de cada etapa.


Con este fichero creado, el cual tendremos en nuestro repositorio Git, dentro de una carpeta llamada Jenkins, podemos pasar a la configuración del mismo.


## Paso 4 - Configuración de Jenkins y despliegue

En esta última fase de la práctica nos dirigiremos al navegador que estemos usando y accederemos a la configuración de Jenkins mediante la dirección "localhost:8080" en nuestro caso. Una vez entremos se nos pedirá que introduzcamos una clave, la cual podemos obtener ejecutando en consola el siguiente comando:

```
docker logs jenkins
```

Una vez obtenemos la clave, la introducimos y Jenkins nos pedirá que elijamos una de las dos opciones, instalar Jenkins con los plugins recomendados o no. Selecciona la opción que sí instala los plugins recomendados.

Cuando la instalación ha terminado, dentro del panel de control de Jenkins hacemos clic en "Create a job", que nos encontraremos en el centro de la pantalla. Una vez dentro, escribimos el nombre de la tarea y marcamos como pipeline la misma, pulsamos OK y automáticamente se nos dirigirá a la configuración del pipeline. En esta parte tenemos que dirigirnos a la sección "Definition", en la que seleccionaremos la opción "Pipeline script for SCM", después "Git" en la sección que nos aparece y finalmente introduciremos la URL del Jenkinsfile que hemos preparado en el paso 3 en la sección "Repository URL". Hacemos clic en "Guardar" al final de la página y se te redirigirá automáticamente al panel de control. En la sección de la izquierda encontraremos "Open Blue Ocean", haremos clic ahí y se nos redirigirá a una pantalla en la que sale el mensaje "This job has not been run", Pulsamos en "Iniciar" justo debajo y vemos como se van sucediento las distintas etapas marcadas en el Jenkinsfile. Si tras el despliegue, aparece un tick en la sección superior izquierda junto al nombre del pipeline, significa que habremos realizado correctamente la práctica.

