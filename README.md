# kubernetes-manual
El presente documento pretende ser una introducción al manejo de Kubernetes, para permitir a los programadores con experiencia en Docker aprovechar sus funciones para desplegar aplicaciones.
No pretende reemplazar la documentación con los detalles técnicos de kubernetes, pero si permitir una introducción más sencilla a este mundo.

## Tabla de contenido

- [Conceptos](#Conceptos)
- [Prerequisitos](#Prerequisitos)
- [Manejo del Cluster](#Manejo-del-Cluster)
  * [Comando kubectl](#kubectl)
  * [Namespaces](#Namespaces)
    * [Creacion de un namespace](#Creacion-de-un-namespace)
    * [Eliminacion de un namespace](#Eliminacion-de-un-namespace)
  * [Manejo de usuarios y cuentas de servicio](#Manejo-de-usuarios-y-cuentas-de-servicio)
    * [Service Account](#Service-Account)
    * [Creacion de un usuario](#Creacion-de-un-usuario)
    * [Creacion de un rol](#Creacion-de-un-rol)
    * [Role binding](#Role-binding)
    * [Borrar un usuario](#Borrar-un-usuario)
    * [Borrar una cuenta de servicio](#Borrar-una-cuenta-de-servicio)
  * [Deployment](#Deployment)
    * [Creacion de un deployment](#Creacion-de-un-deployment)
    * [Servicio](#Servicio)
    * [Ingress](#Ingress)
- [Resolucion de Problemas](#Resolucion-de-Problemas)

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

La opcion mas sencilla podria ser descargarse el cliente offical de GiT e instalarlo con la opcion _"Git Bash"_ habilitada:
[https://git-scm.com/downloads/win]([https://kubernetes.io/docs/tasks/tools/](https://git-scm.com/downloads/win)

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

#### Configuracioin de acceso para kubectl
El comando kubectl lee la configuración de acceso al cluster desde el archivo _config_ (también llamado kubeconfig en otras documentaciones) ubicado en:

##Windows##
C:\Users\<nombre-de-usuario>\.kube\

##Linux##
/home/<nombre-de-usuario>/.kube

**Nota:** Si bien con kubectl es posible crear algunos recursos directamente desde la linea de comandos, en las instrucciones de este documento se va a recomendar siempre crear el archivo yaml y luego aplicarlo, teniendo en cuenta las buenas practicas para tener los recursos documentados en un git y preparando todo para manejar el ciclo de vida de los recursos de la aplicacion utilizando IaC.

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

**Nota:** La cuenta de servicio debera ser asignada a un rol para poder operar sobre el namespace _(role_binding)_

#### Creacion de un usuario

Para poder crear un usuario hay que realizar los siguientes pasos:
El ejemplo sirve para crear un usuario llamado _juan_

##### Creación de la clave privada para el nuevo usuario:

```bash
openssl genrsa -out juan.pem
```

##### Crear una CSR (Certificate Signing Request)
```bash
openssl req -new -key new-user.pem -out juan.csr -subj "/CN=juan"
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
kubectl get csr USER-NAME-csr -o jsonpath='{.status.certificate}' | base64 -d > juan.crt
```

##### Conectarse sin generar un archivo kubeconfig

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

**Notas:**
- El archivo resultante puede ser copiado a la carpeta home del usuario, dentro de la carpeta _./kube_ con el nombre _config_ para permitirle al mismo operar sobre el cluster. O agregar el contenido al archivo _config_ preexistente para adicionar el acceso al cluster.
- El usuario debera ser asignado a un rol para poder operar sobre un cluster _(Role Binding)_

#### Creacion de un rol:
La configuracion de roles de Kubernetes permite crear un sinfin de combinaciones para permitir ciertas acciones y negar otras.
A continuacion solo se brindara un ejemplo para crear un rol de admin para un Namespace, que luego podra ser utilizado para una Service Account utilizado para los pipelines para mantener el ciclo de vida de la aplicacion o un programador que deba tener acceso de admin a todos los recursos del mismo.

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

Luego ejecutamos el siguiente comando para aplicar la configuracion del archivo yc rear el rol:

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

### Resolucion de Problemas
A continuación se presentan comandos que pueden ayudar a identificar y arreglar problemas.



