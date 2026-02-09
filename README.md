# kubernetes-manual
El presente documento pretende ser una introducción al manejo de Kubernetes, para permitir a los programadores con experiencia en Docker aprovechar sus funciones para desplegar aplicaciones.
No pretende reemplazar la documentación con los detalles técnicos de kubernetes, pero si permitir una introducción más sencilla a este mundo.

## Tabla de contenido

- [Conceptos](#Conceptos)
- [Prerequisitos](#Prerequisitos)
- [Manejo del Cluster](#Manejo-del-Cluster)
  * [Comando kubectl](#kubectl)
    * [Configuracion de acceso para kubectl](#Configuracion-de-acceso-para-kubectl)
  * [Namespaces](#Namespaces)
    * [Creacion de un namespace](#Creacion-de-un-namespace)
    * [Eliminacion de un namespace](#Eliminacion-de-un-namespace)
  * [Manejo de usuarios y cuentas de servicio](#Manejo-de-usuarios-y-cuentas-de-servicio)
    * [Service Account](#Service-Account)
    * [Listar los usuarios](#Listar-los-usuarios)
    * [Creacion de un usuario](#Creacion-de-un-usuario)
    * [Creacion de un rol](#Creacion-de-un-rol)
    * [Role binding](#Role-binding)
    * [Borrar un usuario](#Borrar-un-usuario)
    * [Borrar una cuenta de servicio](#Borrar-una-cuenta-de-servicio)
  * [Deployment](#Deployment)
    * [Creacion de un deployment](#Creacion-de-un-deployment)
    * [Servicio](#Servicio)
    * [Ingress](#Ingress)
- [Comandos utiles](#Comandos-utiles)
    * [Pods](#Pods)
    * [Deployment](#Deployment)
    * [Namespace](#Namespace)
    * [Port forward](#Port-forward)
    * [Worker Nodes](#Worker-Nodes)
- [Velero](#Velero)
    * [Backup de una vez](#Backup-de-una-vez)
    * [Backup periodico](#Backup-periodico)
    * [Restauracion](#Restauracion)

---

## Conceptos

- **Kubernetes:** Un clúster de Kubernetes es un conjunto de nodos, que pueden ser máquinas físicas o virtuales, donde se ejecutan las aplicaciones en contenedores. Este clúster permite no sólo desplegar aplicaciones de manera eficiente, sino también asegurar su escalabilidad y disponibilidad.
- **Namespace:** Provee un espacio para agrupar recursos y administrarlos de forma conjunta.
- **Image Registry:** Provee una ubicación centralizada para el almacenamiento de las imágenes que usan los contenedores.
- **Ingress:** Es un servicio del cluster que permite acceder a las aplicaciones desde fuera del cluster, el cual es configurable para redirigir el tráfico acorde al dominio y ruta de la URL de acceso.
- **Service:** Por defecto, los pods corren de forma aislada dentro de un cluster de Kubernetes, al configurarles un servicio, permitimos el acceso a los mismos de forma controlada, configurando los puertos correspondientes en el mismo. Cada vez que se crea un servicio, una entrada DNS dentro del cluster es creada para poder utilizarla con el siguiente formato:
   `<service-name>.<namespace>.svc.cluster.local`.
También se puede utilizar un servicio para permitir el acceso a un pod desde fuera del cluster, pero habitualmente no es utilizado para ello, ya que Ingress provee otras formas mejores de hacerlo.
- **Deployment:** Es un mecanismo de Kubernetes para ejecutar aplicaciones stateless, proveyendo mecanismos para el manejo de su ciclo de vida y su escalamiento horizontal.
Una de sus principales características esta relacionada con el método en el que se hace el rollout de una nueva versión, para lo cual se inician primero los PODs con la nueva versión de la misma, se redirige el tráfico de los pods de versión previa y recién se dan de baja, una vez que ya no tienen tráfico accediendo a los mismos.
- **StatefulSet:** Es un mecanismo de Kubernetes para ejecutar aplicaciones stateless, su principal diferencia con un Deployment es que cuando se hace un rollout de una nueva versión, se bajan primero los pods del mismo y una vez que se terminaron de apagar, recién se levantan los pods con la nueva versión.
- **Persistent Volumes:** Es un mecanismo de Kubernetes para aprovisionar almacenamiento persistente a los PODs, lo cual puede permitir (dependiendo de la tecnología del backend) que varios pods lean y escriban en el mismo espacio de almacenamiento, así como por ejemplo que un pod escriba y los demás lean, o bloquear el mismo espacio para ser accedido por solo un pod a la vez.
Los mismos se definen sobre StorageClasses y Drivers que reflejan las características del almacenamiento de backend.
- **Persistent Volume Claim:** Es una solicitud de acceso a un PV por parte de un recurso de Kubernetes, los mismos se pueden listar para saber que recursos están accediendo a un mismo PV o PVs.
- **Config Maps:** Es un mecanismo de Kubernetes que permite administrar archivos de configuración de forma separada para montarlos en los pods o mandarles la información como variables de entorno.
Por ejemplo, se puede tener un config map en desarrollo que indique al apache a responder a un nombre de dominio para ambientes bajos y luego otro en producción para indicarle a la aplicación que responda al nombre de dominio de producción. De esa forma no hay necesidad de indicarle al código de la aplicación en que ambiente está trabajando.
- **Secrets:** Básicamente su funcionamiento es similar a los Config Maps, con la excepción de que su contenido está oculto a simple vista y debe ser codificado y decodificado (en Base64) para almacenarlo o leerlo administrativamente. Pero al montarlo o pasarlo como variable de entorno, es accesible de forma transparente para la aplicación. Puede ser utilizado, por ejemplo, para indicarle a la aplicación en un archivo el acceso a la base de datos, incluyendo nombre de dominio del servidor, usuario y clave.

---

## Prerequisitos:

### Kubectl:
Bajar e instalar _kubectl_ desde el siguiente enlace:
[https://kubernetes.io/docs/tasks/tools/](https://kubernetes.io/docs/tasks/tools/)

**Nota:**
Muchos de los comandos mencionados en este documento necesitan bash, por lo que es recomendable utilizarlos con el shell de bash provisto provisto por Git, CygWin o WSL.
Si si están accediendo desde Linux no necesitan bajar el cliente de git con el shell de bash.

La mejor opción, desde mi punto de vista, es instalar Ubuntu (u otra de su preferencia) con WSL2.
[Install Ubuntu on WSL2](https://documentation.ubuntu.com/wsl/en/latest/guides/install-ubuntu-wsl2/)
---

## Manejo del Cluster

### kubectl
Es el comando oficial de Kubernetes para administrar los clusters.

El formato de uso es el siguiente:
```bash
kubectl [command] [TYPE] [NAME] [flags]
```
Para obtener ayuda se agrega el flag “-h” al final.
Ejemplo para obtener ayuda sobre como aplicar un archivo de configuracion:
```bash
kubectl apply -h
```

#### Configuracion de acceso para kubectl
El comando kubectl lee la configuración de acceso al cluster desde el archivo `config`, sin extension, ubicado en:

##Windows##
`C:\Users\<nombre-de-usuario>\.kube\`

##Linux##
`/home/<nombre-de-usuario>/.kube`

**Nota:** Si bien con kubectl es posible crear algunos recursos directamente desde la linea de comandos, en las instrucciones de este documento se va a recomendar siempre crear el archivo yaml y luego aplicarlo, teniendo en cuenta las buenas practicas para tener los recursos documentados en un git y preparando todo para manejar el ciclo de vida de los recursos de la aplicacion utilizando IaC.

##### Estructura del archivo de configuracion de kubectl
La siguiente estructura de configuración permite manejar más de una conexión a un cluster utilizando el archivo de configuración por defecto de _kubectl_.

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: XXXXXXXXXXXX # El certificado de CA del Cluster;
    certificate-authority: /home/user1/.kube/cluster-1-ca.crt # O la ruta del archivo que contiene el CA del Cluster.
    server: https://192.168.10.190:6443
  name: cluster-1
- cluster:
    certificate-authority-data: XXXXXXXXXXXX # El certificado de CA del Cluster;
    certificate-authority: /home/user1/.kube/cluster-2-ca.crt # O la ruta del archivo que contiene el CA del Cluster.
  name: cluster-2
contexts:
- context:
    cluster: cluster-1
    namespace: app1 # Se puede configurar este valor si se desea cambiar el namespace por defecto al usar kubectl en este contexto.
    user: kubernetes-admin-1
  name: cluster-1
- context:
    cluster: cluster-2
    user: kubernetes-admin-2
  name: cluster-2
current-context: cluster-1 # El conexto o servidor actual.
kind: Config
preferences: {}
users:
- name: test-user
  user:
    client-certificate-data: XXXXXXXXXXXXXXX # El certificado del usuario;
    client-certificate: /home/.kube/test-user.crt # O la ruta al archivo del certificado del usuario.
    client-key-data: XXXXXXXXXXXXXXX # El key del usuario;
    client-key: /home/.kube/test-user.key # O la ruta al archivo del key del usuario.
- name: dev-user
  user:
    client-certificate-data: XXXXXXXXXXXXXXX # El certificado del usuario;
    client-certificate: /home/.kube/test-user-2.crt # O la ruta al archivo del certificado del usuario.
    client-key-data: XXXXXXXXXXXXXXX # El key del usuario;
    client-key: /home/.kube/test-user-2.key # O la ruta al archivo del key del usuario.
```

##### Cambio de contextos (o servidores)
```bash
# Listar los contextos configurados:
kubectl config get-contexts

# Para saber el contexto (o servidor actual):
kubectl config current-context

# Para cambiar a otro contexto (o servidor):
kubectl config use-context cluster-2

# y para volver al otro cluster:
kubectl config use-context cluster-1
```
### Namespaces
Para aplicaciones pequeñas se utiliza un solo namespace para agrupar todos sus recursos. Para aplicaciones más grandes se pueden utilizar más de un namespace, como por ejemplo _systema-backend_ y _systema-frontend_.
Los nombres soportados son los mismos que para los nombres de dominio, solo pueden contener letras mayusculas, minusculas, numeros y guiones medios.
Comando:
```bash
kubectl create namespace NAME [--dry-run=server|client|none]
```
Ejemplo:
```bash
kubectl create namespace nuevo-namespace
```

#### Creacion de un namespace
Para crear un namespace llamado _namespace1_ por ejemplo, debemos ejecutar el siguiente comando:

```bash
 kubectl create namespace namspace1
```

#### Eliminacion de un namespace
Para eliminar un namespace llamado _namespace1_ por ejemplo, debemos ejecutar el siguiente comando:

```bash
kubectl delete namespace namspace1
```


#### Para setear el namespace actual para trabajar
```bash
kubectl config set-context --current --namespace=namespace-name
```
Ejemplo para situarse en el namespace _aplicacion1_
```bash
kubectl config set-context --current --namespace=aplicacion1
```

### Manejo de usuarios y cuentas de servicio

#### Service Account
Se utilizan para la automatización de tareas. Lo más común es crear al menos una por namespace para permitir a los pipelines ejecutar tareas sobre el namespace.
Siguiendo las buenas practicas de la administracion por IaC se puede hacer de la siguiente forma.

##### Listar las service accounts
```bash
kubectl get sa -n <nombre del namespace>
```

##### Creacion de una service account

Crear un archivo para la definicion de la service account:
_service-account-cicd-namespace1.yaml_
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd
  namespace: namespace1
```

Luego otro archivo para la definicion del secret donde se va a guardar el token:
_secret-service-account-cicd-namespace1.yaml_
```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: cicd-sa-token
  namespace: namespace1
  annotations:
    kubernetes.io/service-account.name: "cicd"
```

Luego ejecutamos los siguientes comandos para crear los nuevos recursos:

```bash
# Creamos la service account
kubectl apply -f service-account-cicd-namespace1.yaml

# Creamos el secret para guardar el token
kubectl apply -f secret-service-account-cicd-namespace1.yaml
```

Para obtener el token debemos ejecutar el siguiente comando en bash:

```bash
kubectl get secret $(kubectl get secret | grep secret-name | awk '{print $1}') -o jsonpath='{.data.token}' | base64 --decode
```

Ejemplo para ver el token en el secret _cicd-sa-token_:
```bash
kubectl get secret $(kubectl get secret | grep cicd-sa-token | awk '{print $1}') -o jsonpath='{.data.token}' | base64 --decode
```

Ejemplo de uso del token del service account con el comando kubectl
```bash
kubectl --insecure-skip-tls-verify --kubeconfig="/dev/null" --server=https://<direccion web de la administracion del cluster> --token=<token de la SA> get deployments -n namespace1
```

**Nota:** La cuenta de servicio debera ser asignada a un rol para poder operar sobre el namespace _(role_binding)_

#### Creacion de un usuario
**Nota**: Si tienes Windows debes utilizar un Linux con WSL para poder ejecutar los siguientes comandos sin problemas.

Para poder crear un usuario hay que realizar los siguientes pasos:
El ejemplo sirve para crear un usuario llamado _juan_

##### Creación de la clave privada para el nuevo usuario:

```bash
openssl genrsa -out juan.key
```

##### Crear una CSR (Certificate Signing Request)
```bash
openssl req -new -key juan.key -out juan.csr -subj "/CN=juan"
```

##### Crear un CertificateSigningRequest para el cluster

```bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: juan-csr
spec:
  request: $(cat juan.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF
```

##### Verficiar que la solicitud haya ingresado y este pendiente de aprobacion:

```bash
kubectl get certificatesigningrequests

# Y deberiamos obtener una salida similar a la siguiente:
NAME       AGE   SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
juan       34s   kubernetes.io/kube-apiserver-client   kubernetes-admin   10d                 Pending
```

##### Aprobar la solicitud del certificado en el cluster y obtener el certificado

```bash
# Aprobar la solicitud de certificado
kubectl certificate approve juan-csr

# Obtener el certificado
kubectl get csr juan-csr -o jsonpath='{.status.certificate}' | base64 -d > juan.crt
```

##### Obtener datos para generar un archivo kubeconfig

Obtener el nombre del cluster:
```bash
kubectl config get-clusters
```

Obtener el CA root del cluster:
```bash
kubectl get cm kube-root-ca.crt -o jsonpath="{['data']['ca\.crt']}"
```

##### Crear el archivo de configuracion:
```bash
kubectl config set-cluster <Nombre-del-Cluster> --server=https://<Cluster-IP-Management-API>:<Port> --certificate-authority=./ca.crt --embed-certs=true --kubeconfig=juan.conf
```

##### Agregar la información del login para el usuario Juan
```bash
kubectl config set-credentials juan --client-key=juan.key --client-certificate=juan.crt --embed-certs=true --kubeconfig=juan.conf
```

##### Configurar el contexto:
```bash
kubectl config set-context <Nombre-del-Cluster> --cluster=<Nombre-del-Cluster> --namespace=default --user=juan --kubeconfig=iraguet.conf
```

##### Configurar el contexto por defecto:
```bash
kubectl config use-context <Nombre-del-contexto> --kubeconfig=juan.conf
```

**Notas:**
- El archivo resultante puede ser copiado a la carpeta home del usuario, dentro de la carpeta _./kube_ con el nombre _config_ para permitirle al mismo operar sobre el cluster. O agregar el contenido al archivo _config_ preexistente para adicionar el acceso al cluster.
- El usuario debera ser asignado a un rol para poder operar sobre un cluster _(Role Binding)_

#### Creacion de un rol:
La configuración de roles de Kubernetes permite crear un sinfín de combinaciones para permitir ciertas acciones y negar otras.
A continuación, solo se brindará un ejemplo para crear un rol de admin para un Namespace, que luego podrá ser utilizado para una Service Account utilizado para los pipelines para mantener el ciclo de vida de la aplicación o un programador que deba tener acceso de admin a todos los recursos del mismo.

Primero vamos a crear un archivo para definir el rol para un namespace llamado _aplicaciones-juan_

_aplicaciones-juan-admin.yaml_
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: aplicaciones-juan-admin
  namespace: aplicaciones-juan
rules:
  - apiGroups: ["", "apps", "batch", "extensions"]
    resources: ["deployments", "services", "replicasets", "pods", "jobs", "cronjobs"]
    verbs: ["*"]
```

Luego ejecutamos el siguiente comando para aplicar la configuración del archivo y crear el rol:

```bash
kubectl apply -f aplicaciones-juan-admin.yaml
```

#### Role Binding
Una vez creados los roles, estos deben ser asignados a los usuarios o cuentas de servicio para que las mismas puedan obtener los permisos definidos.

##### Ejemplo de asignacion de rol a usuario
Primero creamos el archivo con la configuracion que deseamos definir, en este caso darle permisos de administrador al usuario _juan_ sobre el namespace _aplicaciones-juan_

_aplicaciones-juan-admin-juan-binding.yaml_
```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: aplicaciones-juan-admin-juan-binding
  namespace: aplicaciones-juan
subjects:
  - kind: user
    name: juan
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: aplicaciones-juan-admin
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f aplicaciones-juan-admin-juan-binding.yaml
```

##### Ejemplo de asignacion de rol a cuenta de servicio

Primero creamos el archivo de configuracion que deseamos definir, en este caso darle permisos de administrador a la cuenta de servicio _cicd_ sobre el namespace _aplicaciones-juan_

_aplicaciones-juan-admin-cicd-binding.yaml_
```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: aplicaciones-juan-admin-cicd-binding
  namespace: aplicaciones-juan
subjects:
  - kind: ServiceAccount
    name: cicd
    namespace: aplicaciones-juan
roleRef:
  kind: Role
  name: aplicaciones-juan-admin
  apiGroup: rbac.authorization.k8s.io
```

#### Borrar un usuario
Para borrar un usuario simplemente debemos ejecutar el comando _kubectl config delete-user <nombre-de-usuario>_.

En el siguiente ejemplo veremos la eliminacion del usuario juan:

```bash
kubectl config delete-user juan
```

#### Borrar una cuenta de servicio
Si deseamos eliminar una cuenta de servicio al que le hayamos creado un secret (como en el ejemplo anterior), primero debemos eliminar el secret y luego la cuenta de servicio.

**Nota:** En los siguientes ejemplos tenemos una cuenta de servicio llamada _cicd_ en el namespace _namespace1_ con un secret llamado _cicd-sa-token_.

Ejemplo:
```bash
kubectl delete secret cicd-sa-token -n namespace1
```

Luego podemos proceder a eliminar la cuenta de servicio con el siguiente comando:
```bash
kubectl delete serviceaccount -n namespace1 cicd
```

### Deployment

#### Creacion de un deployment:

A continuación se presenta un ejemplo extremadamente simple de como crear un Deployment, se recomienda profundizar más sobre todas las opciones que brindan los deployments para poder aprovecharlos al maximo.

Creamos un archivo para definir el deployment:

deplyment-ejemplo.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
<ins>Descripción del Archivo YAML:</ins>

- **apiVersion:** Define la versión de la API que se está utilizando. En este caso, es apps/v1.
 
- **kind:** Especifica el tipo de recurso, que en este caso es un Deployment.
 
- **metadata:** Contiene información sobre el deployment, como su name y labels.
 
- **spec:** Define la especificación del deployment:

   - **replicas:** Indica el número de réplicas de pods que se desea.
 
   - **selector:** Define cómo seleccionar los pods que este deployment gestionará.
 
   - **template:** Especifica la plantilla para los pods:
 
      - **metadata:** Define las etiquetas de los pods.
 
      - **spec:** Define la especificación de los contenedores dentro de los pods:
 
        - **containers:** Lista de contenedores a ejecutar en cada pod.
 
        - **name:** Nombre del contenedor.
 
        - **image:** Imagen del contenedor que se usará.
 
        - **ports:** Puertos expuestos por el contenedor.

Luego para crear el deployment todo lo que tenemos que hacer es aplicar el archivo yaml con kubectl:

```bash
kubectl apply -f deplyment-ejemplo.yaml -n namespace1
```

#### Servicio
Ahora necesitamos exponer el puerto del contenedor al cluster y debemos hacerlo con un servicio como el definido a continuacion:

servicio-deployment-ejemplo.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```
<ins>Descripción del Archivo YAML:</ins>
- **apiVersion:** Define la versión de la API que se está utilizando, en este caso v1.

- **kind:** Especifica el tipo de recurso, que en este caso es un Service.

- **metadata:** Contiene información sobre el servicio, como su name.

- **spec:** Define la especificación del servicio:

    - **selector:** Indica qué pods serán gestionados por este servicio basándose en las etiquetas. En este caso, selecciona los pods con la etiqueta app: nginx.

    - **ports:** Lista de puertos que el servicio expone y el puerto correspondiente en los pods:

        - **protocol:** El protocolo utilizado, aquí es TCP.

        - **port:** El puerto en el cual el servicio estará disponible.

        - **targetPort:** El puerto en los pods al cual el tráfico será redirigido.

    - **type:** Especifica el tipo de servicio. _ClusterIP_ expone el servicio dentro del clúster de Kubernetes utilizando una dirección IP interna. Esto significa que el servicio solo es accesible desde otros recursos dentro del mismo clúster.

Luego solo tenemos que crear el servicio aplicando el archivo de configuración
```bash
kubectl apply -f servicio-deployment-ejemplo.yaml -n namespace1
```

y podremos acceder al servicio del pod utilizando el siguiente nombre de dominio dentro del cluster: `nginx-service.namespace1.svc.cluster.local`

#### Ingress
Por ultimo, para poder exponer el servicio al mundo exterior debemos configurarle un ingress:

ingress-deployment-ejemplo.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80

```

<ins>Descripción del Archivo YAML:</ins>
- **apiVersion:** Define la versión de la API que se está utilizando, en este caso networking.k8s.io/v1.

- **kind:** Especifica el tipo de recurso, que en este caso es un Ingress.

- **metadata:** Contiene información sobre el Ingress, como su name.

- **spec:** Define la especificación del Ingress:

    - **rules:** Lista de reglas de enrutamiento. Cada regla tiene un host y una configuración HTTP.

        - **host:** Define el dominio para el cual se aplica la regla.

        - **http:** Define las configuraciones HTTP:

            - **paths:** Lista de rutas y cómo deben ser gestionadas.

                - **path:** Especifica el camino al cual se aplica la regla (/ en este caso).

                - **pathType:** Especifica el tipo de coincidencia del camino (Prefix en este caso).

                - **backend:** Define el servicio backend y puerto al cual se debe redirigir el tráfico:

                    - **service:**

                        - **name:** Nombre del servicio al que se dirigirá el tráfico (nginx-service).

                        - **port:**

                            - **number:** Puerto del servicio al cual se redirigirá el tráfico (80 en este caso).

Luego creanmos el servicio aplicando el archivo de configuracion:
```bash
kubectl apply -f ingress-deployment-ejemplo.yaml -n namespace1
```

### Comandos utiles
A continuación se presentan los comandos más utilizados para ver el estado de los recursos y analizar problemas si fuera necesario:

#### Pods

##### Listar los pods
```bash
#Ver el listado de los pods de un namespace llamado namespace1
kubectl get pods -n namespace1

#Ver el listado de los pods de un namespace llamado namespace1 con información extendida
kubectl get pods -n namespace1 -o wide
```

##### Logs
Ver los logs de un **pod** llamado _aplicacion1-abc1234_ de un **deployment** llamado _aplicacion1_ en un **namespace** llamado _namespace1_
```bash

# Comando para obtener information del estado y chequear si los pods del deployment funcionan correctamente
kubectl describe pods aplicacion1 -n namespace1

# Comando a ejecutar para ver si un pod inicio y esta funcionando correctamente
kubectl describe pods aplicacion1-abc1234 -n namespace1

# Ver los logs de un pod
kubectl logs aplicacion1-abc1234 -n namespace1

# Ver los logs de todos los pods corriendo dentro de un deployment
kubectl logs deployment/aplicacion1
```

##### Ejecturar comandos dentro de un pod
Si es necesario entrar a un pod para ejecutar comandos se puede hacer de la siguiente forma.

El siguiente ejemplo se muestra teniendo en cuenta un **pod** llamado _aplicacion1-abc1234_ en un **namespace** llamado _namespace1_ y permite abrir una sesión interactiva de shell.

```bash
kubectl exec -it aplicacion1-abc1234 -n namespace1 -- /bin/bash
```
##### Metricas de performance
Para ver las metricas de performance de un **pod** llamado _aplicacion1-abc1234_ en un **namespace** llamado _namespace1_ podemos ejecutar el siguiente comando.
```bash
kubectl top pod aplicacion1-abc1234 -n namespace1
```

#### Deployment

##### Rollout
Esto permite reiniciar los pods para levantar una nueva imagen con una nueva version. En un Deployment, primero se levantan los pods con la nueva version de la aplicacion, luego se transfiere el trafico hacia los nuevos pods y al ultimo se detienen los pods con la version anterior de la aplicacion.

###### Actualizar la version de una aplicacion
Ejemplo de rollout de un **deployment** llamado _aplicacion1_ en un **namespace** llamado _namespace1_. Teniendo en cuenta que la version de la imagen utilizada es _latest_.
```bash
kubectl rollout restart deployment/aplicacion1 -n namespace1
```

Ejemplo de rollout de:
un **deployment** llamado _aplicacion1_
con un **container** llamado _app1_
que utiliza una **imagen** llamada _appimage_
en un **namespace** llamado _namespace1_.
Teniendo en cuenta que la version de la imagen utilizada esta especificada y el tag de la **nueva version** es _v2_

```bash
kubectl set image deployments/aplicacion1 appimage=docker.io/app1:v2 -n namespace1
```

###### Ver informacion de los rollouts
Para ver el estado de un rollout de un **deployment** llamado _aplicacion1_ en un **namespace** llamado _namespace1_ debemos ejecutar el siguiente comando.
```bash
kubectl rollout status deployments/aplicacion1 -n namespace1
```

Ver el historial de rollout de un **deployment** llamado _aplicacion1_ en un **namespace** llamado _namespace1_.
```bash
kubectl rollout history deployment/aplicacion1 -n namespace1
```

###### Volver atrás
Para volver atras un rollout y restaurar la version anterior de la aplicacion se pueden utilizar los siguientes comandos.

Volver a la version anterior de la imagen utilizada de un **deployment** llamado _aplicacion1_ en un **namespace** llamado _namespace1_.
```bash
kubectl rollout undo deployment/aplicacion1 -n namespace1
```

Para volver atras teniendo en cuenta un rollout especifico, que se puede observar con el _kubectl rollout history_,  de un **deployment** llamado _aplicacion1_ en un **namespace** llamado _namespace1_.
```bash
kubectl rollout undo deployment/aplicacion1 --to-revision=2 -n namespace1
```

Para volver atrás especificando la versión de la imagen, el ejemplo tiene en cuenta lo siguiente:
un **deployment** llamado _aplicacion1_
con un **container** llamado _app1_
que utiliza una **imagen** llamada _appimage_
en un **namespace** llamado _namespace1_.
Teniendo en cuenta que la version de la imagen utilizada esta especificada y el tag de la **version anterior es** es _v1_

```bash
kubectl set image deployments/aplicacion1 appimage=docker.io/app1:v1 -n namespace1
```

#### Namespace

##### Eventos
En los eventos se puede encontrar informacion reciente sobre lo que esta pasando en un namespace.

Para ver los eventos de un **namespace** llamado _namespace1_ debemos ejecutar.
```bash
kubectl get events -n namespace1
```

Para ver los eventos de todos los namespaces.
```bash
kubectl get events --all-namespaces
```

#### Port forward
Con los siguientes comandos se puede acceder directamente a un pod o servicio desde la computadora local.

Para el siguiente ejemplo tenemos en cuenta lo siguiente,
un **pod* llamado _aplicacion1-abc1234_ que brinda una interface web por el **puerto** _80_
al cual se puede conectar desde un **servicio** llamado _servicio-aplicacion1_ a traves del **puerto** _80_
en un **namespace** llamado _namespace1_

Para acceder al **puerto** _80_ del pod desde nuestra workstation con la dirección _http://127.0.0.1:4080_ ejecutamos el siguiente comando.
```bash
kubectl port-forward pods/aplicacion1-abc1234 4080:80 -n namespace1
```

Para acceder al **puerto** _80_ del servicio desde nuestra workstation con la dirección _http://127.0.0.1:5080_ ejecutamos el siguiente comando.
```bash
kubectl port-forward service/servicio-aplicacion 5080:80 -n namespace1
```

Si queremos exponer el puerto redirigido a todas las IPs de la estación de trabajo donde lo ejecutamos (util si se usa WSL por ejemplo, para poder acceder desde el host).
```bash
kubectl port-forward --address 0.0.0.0 service/servicio-aplicacion 5080:80 -n namespace1
```

Luego, simpelmente abrimos nuestro navegador y ponemos la dirección del localhost con los dos puntos para indicar el puerto.

#### Worker Nodes

Obtener informacion basica de los nodos:
```bash
#Para listar los nodos:
kubectl get nodes

#Para obtener una lista de los nodos con informacion extendida:
kubectl get nodes -o wide
```

Para obtener los detalles completos de un **nodo* llamado _node01_:
```bash
kubectl describe node node01
```

Para poder observer el uso de recursos de los nodos debemos ejecutar:
```bash
kubectl top nodes
```

Para ver los eventos recientes de los nodos debemos ejecutar:
```bash
kubectl get events --field-selector involvedObject.kind=Node
```

### Velero
Para su instalación, visitar el siguient link: https://velero.io/docs/v1.15/basic-install/
A continuacion se detallan los comandos basicos para realizar copias de resguardo con Velero de los namespaces.

#### Backup de una vez
Esto es util para cuando se esta por realizar un cambio en un namespace y se desea tener una copia de seguridad del momento previo a aplicar el cambio.

Para realizar una copia de resguardo de un namespace llamado _namespace1_ y ponerle de nombre _2024-12-05-resguardo-de-namespace1_ podemos ejecutar el siguiente commando:
```bash
velero backup create 2024-12-05-resguardo-de-namespace1 --include-namespaces namespace1
```

Para ver el estado y los detalles de la copia de resguardo llamada _2024-12-05-resguardo-de-namespace1_:
```bash
velero backup describe 2024-12-05-resguardo-de-namespace1 --details
```

Para listar todas las copias de resguardo:
```bash
velero backup get
```

Para eliminar una copia de resguardo de unica vez llamada _2024-12-05-resguardo-de-namespace1_:
```bash
velero backup delete 2024-12-05-resguardo-de-namespace1
```

#### Backup periódico
Mediante los siguientes comandos podemos establecer copias de resguardo periodicas y establecer un tiempo de retencion para las mismas:

Para crear una configuracion de resguardo periodica llamada _backup-semanal-namespace1_ de lunes a viernes a las 5 AM de un namespace llamado _namespace1_ y mantener las ultimas dos semanas podemos ejecutar el siguiente comando:
```bash
velero create schedule backup-semanal-namespace1 --schedule “0 5 * * 1-5” --ttl 336h00m00s --include-namespaces namespace1
```
**Notas**:
--schedule: Especifica cuando se van a realizar las copias de resguardo y utiliza el formato _Cron_
--ttl: Especifica el tiempo de retencion, en el formato _XXhXXmXXs_, para la cantidad de dias simplemente tenemos que multiplicarlo por 24, por ejemplo, para 2 dias el valor seria _48h00m00s_

Para revisar el estado y los detalles de la configuracion de resguardo periodica llamada _backup-semanal-namespace1_ debejemos ejecutar el siguiente comando:
```bash
velero schedule describe backup-semana-namespace1
```

Para listar todas las configuraciones de resguardo periodicas podemos ejecutar:
```bash
velero get schedule
```

**Nota:**
Los backups individuales generados por las configuraciones de backup periodicas se pueden listar utilizando el comando anteriormente mencionado:
```bash
velero get backup
```

#### Restauracion
Velero permite restaurar el namespace completo o un recurso del mismo. A continuacion veremos ambos ejemplos

Primero debemos obtener la lista de los backups, para saber cual debemos restaura, lo cual podemos hacer con el comando anteriormente mencionado.

Restaurar un namespace completo de un backup llamado _backup-semanal-namespace1-20241207050003_ y nombrar la tarea como _recuperacion-de-namespace1_:
```bash
velero restore create recuperacion-de-namespace1 --from-backup backup-semanal-namespace1-20241207050003
```

A continuacion damos un ejemplo sobre como restaurar un solo de los deployments de un namespace, teniendo en cuenta lo siguiente:
**Nombre de la copia de resguardo:** backup-semanal-namespace1-20241207050003
**Etiqueta utilizada para filtrar el deployment que queremos recuperar:** app
**Valor de la etiqueta utilizada para filtrar el deployment que queremos recuperar:** nginx
**Nombre que queremos ponerle a la tarea de restauracion:** recuperacion-deployment-nginx-namespace1

```bash
velero restore create recuperacion-deployment-nginx-namespace1 --include-resources deployments --selector app=nginx --from-backup backup-semanal-namespace1-20241207050003
```

**Nota:**
Para mas inforamcion sobre como filtrar los recursos en una restauracion se pueden fijar en el siguiente link de la documentacion oficial:
[https://velero.io/docs/v1.9/resource-filtering](https://velero.io/docs/v1.9/resource-filtering)


Para ver el estado y los detealles de la tarea de restauracion llamada _recuperacion-de-namespace1_:
```bash
velero describe restore recuperacion-de-namespace1 --details
```
