
# info de la materia: ST0263 TOPICOS ESPEC. EN TELEMATICA
#
# Estudiante(s): Carla Sofía Rendón Baliero,Victor Manuel Botero Gómez,Maria Paulina Lopez Salazar 
#
# Profesor: Alvaro Enrique Ospina Sanjuan
#
#  Reto 4 Kubernetes
#
# 1. Descripción de la actividad
#
El presente proyecto tiene como objetivo en desplegar una aplicación Dockerizada (wordpress) en un clúster de alta disponibilidad en Kubernetes, utilizando servicios administrados en la nube como Google Kubernetes Engine (GKE) en Google Cloud Platform (GCP). Este despliegue implica implementar un entorno altamente escalable y resiliente, capaz de manejar cargas de trabajo variables y garantizar la disponibilidad de la aplicación
## 1.1. Que aspectos cumplió o desarrolló de la actividad propuesta por el profesor

- **Despliegue en Kubernetes**: Se implementó la aplicación en un clúster de Kubernetes, como se solicitó en el reto.
- **Alta Disponibilidad**: Se configuró el clúster para garantizar la alta disponibilidad en todas las capas del sistema, incluyendo la aplicación, la base de datos y el almacenamiento persistente.
- **Servicios Administrados en la Nube**: Se utilizó Google Kubernetes Engine (GKE) como servicio administrado en la nube para alojar el clúster Kubernetes.
- **Servicios Adicionales**: Se integraron servicios adicionales como Cloud SQL MySQL para la base de datos y NFS Server para el almacenamiento persistente.
- **Gestión de Certificados SSL**: Se implementó cert-manager para gestionar automáticamente los certificados SSL y habilitar el acceso seguro a la aplicación a través de HTTPS.
- **Ingress y Balanceo de Carga**: Se configuró Ingress con balanceo de carga para servir como punto de entrada único al clúster desde el mundo exterior, tal como se requirió en los requisitos del reto.

## 1.2. Que aspectos NO cumplió o desarrolló de la actividad propuesta por el profesor 
- **Autoescalado del Clúster**: Aunque se implementó el autoescalado de los pods dentro del clúster utilizando Horizontal Pod Autoscaler (HPA), no se configuró el autoescalado del propio clúster Kubernetes. Esto significa que el clúster no se ajusta automáticamente en función de la carga de trabajo, lo que puede afectar la capacidad de manejar picos de tráfico inesperados de manera eficiente.


# 2. información general de diseño de alto nivel, arquitectura, patrones, mejores prácticas utilizadas.

## Arquitectura del Sistema
- **Contenedores Docker**: La aplicación WordPress y otros servicios se encapsulan en contenedores Docker, lo que proporciona portabilidad, consistencia y facilidad de implementación en cualquier entorno compatible con Docker.
- **Kubernetes (GKE)**: Se utiliza Google Kubernetes Engine (GKE) como plataforma de orquestación de contenedores, permitiendo la administración automatizada de los contenedores y proporcionando características como el autoescalado, la alta disponibilidad y la gestión de recursos.
- **Servicios Administrados en la Nube**: Se optó por utilizar servicios administrados en la nube como Cloud SQL MySQL y almacenamiento persistente mediante Persistent Disk y NFS Server, lo que reduce la carga operativa y simplifica la gestión de recursos.
- **Ingress y Balanceador de Carga**: Se configura Ingress con un controlador de balanceador de carga para dirigir el tráfico externo al clúster Kubernetes y distribuirlo de manera eficiente entre los pods de la aplicación.
- **Cert-manager y Let's Encrypt**: Cert-manager se utiliza para la gestión automatizada de certificados SSL a través de Let's Encrypt, garantizando el acceso seguro a la aplicación a través de HTTPS.
- **Escalabilidad y Resiliencia**: Se implementan técnicas de autoescalado horizontal (HPA) y vertical (VPA) para garantizar que el sistema pueda manejar cargas variables y optimizar el uso de recursos en función de la demanda.
- **Despliegue Declarativo**: Se adopta un enfoque declarativo para el despliegue de recursos en Kubernetes, utilizando archivos YAML para describir el estado deseado del sistema y permitir la gestión eficiente de la infraestructura como código.
  
![imagen](https://github.com/csofia1408/Reto4TopicosTelem-tica/assets/72955238/6a733ed8-60f2-4a4f-ad6f-0a9b8439b1dc)

- En el diagrama anterior, el clúster se encuentra en su estado inicial, cuando se despliega por primera vez sin ningún tráfico/carga, y consta de un número mínimo de pods y un único nodo.Cuando tenga trafico,

![imagen](https://github.com/csofia1408/Reto4TopicosTelem-tica/assets/72955238/e4295eae-78d6-41df-a62a-2907ad7811e8)

- En el diagrama anterior, el clúster se encuentra en estado autoescalado. Representa un estado en el que cumple las condiciones de utilización de CPU/Memoria definidas en nuestro HPA y ha generado WordPress adicionales y 2 pods Ingress adicionales . Cuando el clúster detecte que se necesitan más recursos para los pods, se añadirán más nodos al clúster.

# 3.Detalles del Desarrollo
### Provisionar el clúster de GKE con el autoscaler de clúster y VPC nativa

```
gcloud container clusters create wp-k8s \
  --zone europe-west4-a \
  --machine-type e2-small \
  --disk-size 10 \
  --enable-autoscaling \
  --enable-vertical-pod-autoscaling \
  --num-nodes 1 \
  --min-nodes 1 \
  --max-nodes 2 \
  --enable-ip-alias \
  --create-subnetwork name="" \
  --no-enable-cloud-logging
```
### Cloud SQL (MySQL base de datos de WordPress)
* Creamos una instancia de Cloud SQL MySQL desde la consola de GCP.
* Configuramos la instancia con las especificaciones deseadas.
* Creamos la base de datos de WordPress.
* Proporcionamos permisos de acceso a la base de datos.

![imagen](https://github.com/csofia1408/Reto4TopicosTelem-tica/assets/72955238/45d1a86b-fded-40c0-84c6-692a6b37899f)


### NFS y almacenamiento persistente
* Crea un disco persistente en GKE
```
gcloud compute disks create --size=10GB --zone=europe-west4-a wp-nfs-disk
```
### WordPress
* Configurar el archivo Kustomization
```
kind: Kustomization
secretGenerator:
- name: mysql-root-pass
  literals:
  - password=REDACTED
- name: wp-db-host
  literals:
    - host=REDACTED
- name: wp-db-user
  literals:
    - password=REDACTED
- name: mysql-db-pass
  literals: 
    - password=REDACTED
- name: wp-db-name
  literals:
    - password=REDACTED
resources: 
  - metallb.yaml
  - nfs-synology.yaml
  - wordpress-deployment.yaml
  - hpa.yaml
```
* Despliegamos WordPress, NFS server, Ingress, VPA y HPA a través de los yaml ejecutando:
```
kubectl apply -k ./
```
### Ingress
* Agregamos el repositorio de Helm de ingress-nginx:
  ```
  helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  ```
* Instalamos el chart de ingress-nginx:
  ```
  helm install ingress ingress-nginx/ingress-nginx --set 'ingress.annotations.nginx.ingress.kubernetes.io/client-max-body-size=40m'
  ```
* Espera a que se exponga la IP externa del controlador de ingress:
  ```
  kubectl get services -w ingress-ingress-nginx-controller
  ```
* Actualiza el registro DNS con la IP externa expuesta.

### Cert-manager (Let's Encrypt)

* Agregamos el repositorio de Helm de cert-manager:
  ```
  helm repo add jetstack https://charts.jetstack.io
  ```
* Creamos un espacio de nombres para cert-manager:
  ```
  kubectl create namespace cert-manager
  ```
* Instalamos cert-manager:
  ```
  helm install cert-manager jetstack/cert-manager --namespace cert-manager --set installCRDs=true
  ```
* Creamos los emisores de certificado Let's Encrypt.
    
# 4. Descripción del ambiente de desarrollo y técnico: lenguaje de programación, librerias, paquetes, etc, con sus numeros de versiones.
El ambiente de desarrollo y técnico utilizado para este proyecto está compuesto por las siguientes tecnologías y herramientas:
- **Contenedores y Orquestación**: Se emplea Docker para la creación y gestión de contenedores, permitiendo la encapsulación de la aplicación y sus dependencias. Kubernetes se utiliza para la orquestación de contenedores, proporcionando una plataforma robusta y escalable para la ejecución de la aplicación en clústeres.
- **Servicios en la Nube**: Se hace uso de servicios administrados en la nube como Google Kubernetes Engine (GKE) para la gestión de clústeres Kubernetes, Cloud SQL para la base de datos MySQL y Persistent Disk para el almacenamiento persistente.
- **Herramientas de Automatización**: Se utilizan herramientas de automatización como Helm para la gestión de paquetes de Kubernetes, permitiendo la configuración y despliegue simplificados de recursos en el clúster.
- **Cert-manager y Let's Encrypt**: Cert-manager se emplea para la gestión automatizada de certificados SSL a través de Let's Encrypt, garantizando la seguridad de las comunicaciones mediante HTTPS.
- **Monitoreo y Métricas**: Se integra el monitoreo y la recolección de métricas mediante herramientas como Prometheus y Grafana, permitiendo la observabilidad y el análisis del rendimiento del sistema.
- **Control de Versiones**: Git se utiliza como sistema de control de versiones para el seguimiento de los cambios en el código fuente y la colaboración entre los miembros del equipo de desarrollo.
- **Infraestructura como Código (IaC)**: Se adopta la práctica de infraestructura como código, utilizando archivos YAML y scripts de automatización para describir y gestionar la infraestructura de manera programática.

# 5. otra información que considere relevante para esta actividad.
### Glosario de Terminos
* Kubernetes (K8s): Plataforma de código abierto diseñada para automatizar la implementación, escalado y administración de aplicaciones en contenedores.
* Google Kubernetes Engine (GKE): Servicio de Google Cloud Platform que permite ejecutar cargas de trabajo de contenedores en la infraestructura de Google, gestionado por Kubernetes.
* Cloud SQL: Servicio de base de datos relacional de Google Cloud Platform, que permite implementar, mantener y administrar bases de datos MySQL, PostgreSQL y SQL Server en la nube.
* Persistent Volume (PV): Recurso en Kubernetes que representa un volumen de almacenamiento persistente en el clúster, independiente del ciclo de vida de los pods.
* Persistent Volume Claim (PVC): Solicitud de almacenamiento persistente en Kubernetes, que permite a los pods acceder a un volumen de almacenamiento persistente mediante una abstracción declarativa.
* NFS (Network File System): Protocolo de sistema de archivos distribuido que permite a los clientes acceder y compartir archivos almacenados en un servidor a través de una red.
* Ingress: Recurso en Kubernetes que gestiona el acceso externo a los servicios en un clúster, mediante la exposición de rutas HTTP y HTTPS.
* Horizontal Pod Autoscaler (HPA): Característica en Kubernetes que automáticamente escala el número de pods en función de la carga de trabajo, ajustando la cantidad de réplicas para satisfacer la demanda.
* Vertical Pod Autoscaler (VPA): Componente en Kubernetes que ajusta los límites de recursos de los contenedores en función de la utilización de recursos observada, optimizando la eficiencia y el rendimiento.
* Cert-manager: Controlador en Kubernetes que automatiza la solicitud, emisión y renovación de certificados TLS, incluyendo los proporcionados por Let's Encrypt.
* Let's Encrypt: Autoridad de certificación que ofrece certificados TLS gratuitos y automatizados, utilizados para habilitar conexiones seguras HTTPS en aplicaciones web.
  
Referencias:
- Kubernetes: [Documentación oficial de Kubernetes](https://kubernetes.io/docs/home/)
- Google Kubernetes Engine (GKE): [Documentación oficial de GKE](https://cloud.google.com/kubernetes-engine/docs)
- Cloud SQL: [Documentación oficial de Cloud SQL](https://cloud.google.com/sql/docs)
- NFS (Network File System): [Documentación oficial de NFS](https://help.ubuntu.com/community/SettingUpNFSHowTo)
- Autoscaling en Kubernetes: [Documentación oficial de Kubernetes sobre Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- Ingress en Kubernetes: [Documentación oficial de Kubernetes sobre Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- Let's Encrypt: [Sitio web oficial de Let's Encrypt](https://letsencrypt.org/)
- Cert-manager: [Documentación oficial de cert-manager](https://cert-manager.io/docs/)
- Vertical Pod Autoscaler (VPA): [Documentación oficial de VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- Horizontal Pod Autoscaler (HPA): [Documentación oficial de HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- Metallb: [Documentación oficial de Metallb](https://metallb.universe.tf/)


