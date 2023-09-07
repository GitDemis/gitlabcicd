# Gitlab CI/CD

En este documento se indicaran pasos básicos para la configuración de un pipeline para el deploy de microservicios en openshift Anses utilizando la función de CI/CD disponible en Gitlab


## Conceptos

Para la siguiente implementación vamos a necesitar entender el funcionamiento de la pipeline y como se conecta a distintos servicios para realizar las tareas asignadas en el código.

 - El pipeline se va a correr desde un gitlab runner, el cual debe estar
   bien configurado y asignado a un proyecto de gitlab
- El gitlab runner debe tener acceso a la api de Openshift y a Quay
- Debe haber un repositorio de quay en anses.lab asignado al proyecto.
- El repositorio de quay debe tener asignada una cuenta robot para hacer push y pull de imágenes.
- Acceso a la api de Openshift desde una maquina para la solicitud de Tokens

El flujo de la pipeline funciona de la siguiente manera:
El pipeline es ejecutado desde Gitlab
Gitlab le deriva la ejecución de la tarea a un Gitlab Runner
El Gitlab runner es el encargado de (en orden):
- Clonar el repositorio
- Buildear la imagen utilizando Docker
- Taguear la imagen para subirla a Quay
- Pushear la imagen a quay
A partir de este momento se ejecutan comandos OC para comunicarse con Openshift y realizar las siguientes tareas (en orden):
- Parar la ejecucion de deploys en openshift
- Vincular la imagen en quay con un Image-stream en Openshift
- Actualizar tag "latest"
- Reanudar el deploy del CD
Lo que lleva a hacer la actualización del DC y el pull de la nueva imagen

## 

# Instrucciones para replicar
## Tokens

Necesitaremos algunos tokens para utilizar en el pipeline y en Openshift

1. Openshift pipeline token ($OC_PIPELINE):
Dentro de Openshift nos ubicamos en el proyecto que queremos integrar
Dentro de la consola de *Administrator* nos vamos a *User Management* -> *ServiceAccounts* y nos cercioramos de la existencia de una cuenta llamada *pipeline*
Abrimos una consola (windows o linux) que tenga acceso a la api de Openshift y nos logueamos, nos ubicamos en el proyecto correspondiente y ejecutamos:
`oc create token pipeline`
Lo que nos devuelve un token que utilizaremos mas adelante, lo llamaremos ***OC_PIPELINE***


2. Quay Robot Account ($QUAY_USER, $QUAY_PASSWD y quay-login):
Dentro de Quay (Anses) nos ubicamos dentro de la organizacion *anses.lab*
Hacemos clic en el icono de *Robot Accounts*
Haciendo clic sobre la cuenta de servicio que tenga acceso a nuestra imagen (en este caso *anses.lab+gestionservidores*) , vamos a guardar 3 valores:
En la pestaña *Robot Account* guardamos los *Username & Robot Account* que llamaremos ***QUAY_USER*** y ***QUAY_PASSWD***
Y en la pestaña *Docker Configuration* vamos al link (azul) *view [...].json* y guardamos el contenido que llamaremos ***quay_login***

## Quay
En quay tomamos nota del nombre de la imagen y la llamaremos ***quay_image***

## OpenShift
En openshift vamos a realizar la creacion de un nuevo secret, de tipo *Image pull secret* 
Decidimos el nombre, en este caso vamos a utilizar *quayLogin*
En el desplegable seleccionar *Upload configuration file* y en el campo inferior pegar el contenido de nuestro ***quay_ login***

Tomamos nota del proyecto, en este caso lo llamaremos ***OC_PROJ***

En la consola de *administrador* nos ubicamos en *Builds* y luego en *ImageStreams*, tomamos nota del Image-Stream asignado a nuestro proyecto, lo llamaremos **ImageStream**

En *Workloads* nos ubicamos en *DeploymenConfigs* y tomamos nota del nombre del deployment config y lo llamaremos **dc_name**

Desde una consola cmd o bash ejecutar el siguiente comando:

    oc secrets link default quayLogin --for=pull

## Gitlab
### Variables
Nos ubicamos en la configuracion CI/CD del repositorio de Gitlab y deplegamos la seccion de *Variables*

Vamos a crear 3 variables:
**OC_PIPELINE:** Pegamos nuestro token *OC_PIPELINE*
**QUAY_USER:** Pegamos nuestro *QUAY_USER*
**QUAY_PASSWD:** Pegamos nuestro *QUAY_PASSWD*

### Pipeline
Vamos a crear nuestro pipeline en gitlab y lo completaremos con el siguiente contenido reemplazando los valores entre llaves **{}**

```yaml
stages:
  - login
  - build
  - push
  - deploy 
  - clean

Build:
  stage: build
  script:
    - docker build -t {quay_image} .

Login:
  stage: login
  script:
    - docker login -u="$QUAY_LOGIN" -p="$QUAY_PASSWD" quay-enterprise-quay-quay-enterprise.apps.ocp.anses.gov.ar

Push:
  stage: push
  script:
    - oc login --token=$OC_PIPELINE --server=https://api.ocpd.anses.gov.ar:6443 --insecure-skip-tls-verify=true
    - oc project {OC_PROJ}
    - docker image tag quay_image quay-enterprise-quay-quay-enterprise.apps.ocp.anses.gov.ar/anses.lab/{quay_image}:$CI_COMMIT_SHORT_SHA
    - docker push quay-enterprise-quay-quay-enterprise.apps.ocp.anses.gov.ar/anses.lab/{quay_image}:$CI_COMMIT_SHORT_SHA
    - oc create imagestream {ImageStream}|| echo "image stream ya presente"
    - oc import-image {quay_image}:$CI_COMMIT_SHORT_SHA --from=quay-enterprise-quay-quay-enterprise.apps.ocp.anses.gov.ar/anses.lab/{quay_image}:$CI_COMMIT_SHORT_SHA --confirm
    - oc tag {quay_image}:$CI_COMMIT_SHORT_SHA {quay_image}:latest
    - oc secrets link default {quayLogin} --for=pull

deploy:
  stage: deploy
  script:
    - oc login --token=$OC_PIPELINE --server=https://api.ocpd.anses.gov.ar:6443 --insecure-skip-tls-verify=true
    - oc project {OC_PROJ}
    - oc rollout pause dc/{dc_name} || true
    - oc set image dc/{dc_name} {ImageStream}={OC_PROJ}:$CI_COMMIT_SHORT_SHA --source=imagestreamtag
    - oc rollout resume dc/{dc_name}|| true
    - oc rollout status dc/{dc_name}

clean:
  stage: clean
  script:
    - docker logout quay-enterprise-quay-quay-enterprise.apps.ocp.anses.gov.ar
    - docker system prune -af