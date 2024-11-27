# kubernetes-manual
El presente documento pretende ser una introducción al manejo de Kubernetes, para permitir a los programadores con experiencia en Docker poder aprovechar sus características para desplegar aplicaciones.

## Conceptos
- **Kubernetes:** Un clúster de Kubernetes es un conjunto de nodos, que pueden ser máquinas físicas o virtuales, donde se ejecutan las aplicaciones en contenedores. Este clúster permite no sólo desplegar aplicaciones de manera eficiente, sino también asegurar su escalabilidad y disponibilidad.
- **Namespace:** Provee un espacio para agrupar recursos y administrarlos de forma conjunta.
- **Image Registry:** Provee una ubicación centralizada para el almacenamiento de las imágenes que usan los contenedores.
- **Ingress:** Es un servicio del cluster que permite acceder a las aplicaciones desde fuera del cluster, el cual es configurable para redirigir el tráfico acorde al dominio y ruta de la URL de acceso.
- **Service:** Por defecto, los pods corren de forma aislada dentro de un cluster de Kubernetes, al configurarles un servicio, permitimos el acceso a los mismos de forma controlada, configurando los los puertos correspondientes en el mismo. Cada vez que se crea un servicio, una entrada DNS dentro del cluster es creada para poder utilizarla con el siguiente formato:
   <service-name>.<namespace>.svc.cluster.local.
También se puede utilizar un servicio para permitir el acceso a un pod desde fuera del cluster, pero habitualmente no es utilizado para ello, ya que Ingress provee mejores y más formas de hacer lo mismo.
- **Deployment:** Es un mecanismo de Kubernetes para ejecutar aplicaciones stateless, proveyendo mecanismos para el manejo de su ciclo de vida y su escalamiento horizontal.
Una de sus principales características esta relacionada con el método en el que se hace el rollout de una nueva versión, para lo cual se inician primero los PODs con la nueva versión de la misma, se redirige el tráfico de los pods de versión previa y recién se dan de baja, una vez que ya no tienen tráfico accediendo a los mismos.
- **StatefulSet:** Es un mecanismo de Kubernetes para ejecutar aplicaciones stateless, su principal diferencia con un Deployment es que cuando se hace un rollout de una nueva versión, se bajan primero los pods del mismo y una vez que se terminaron de apagar, recién se levantan los pods con la nueva versión.
- **Persistent Volumes:** Es un mecanismo de Kubernetes para aprovisionar almacenamiento persistente a los PODs, lo cual puede permitir (dependiendo de la tecnología del backend) que varios pods lean y escriban en el mismo espacio de almacenamiento, así como por ejemplo que un pod escriba y los demás lean, o bloquear el mismo espacio para ser accedido por solo un pod a la vez.
Los mismos se definen sobre StorageClasses y Drivers que reflejan las características del almacenamiento de backend.
- **Persistent Volume Claim:** Es una solicitud de acceso a un PV por parte de un recurso de Kubernetes, los mismos se pueden listar para saber que recursos están accediendo a un mismo PV o PVs.
- **Config Maps:** Es un mecanismo de Kubernetes que permite administrar archivos de configuración de forma separada y montarlos sobre los pods.
Por ejemplo, se puede tener un config map en desarrollo que indique al apache a responder a un nombre de dominio para ambientes bajos y luego otro en producción para indicarle a la aplicación que responda al nombre de dominio de producción. De esa forma no hay necesidad de indicarle al código de la aplicación en que ambiente está trabajando.
- **Secrets:** Básicamente su funcionamiento es similar a los Config Maps, con la excepción de que su contenido está oculto a simple vista y debe ser codificado y decodificado (en Base64) para almacenarlo o leerlo administrativamente. Pero al montarlo, es accesible de forma transparente para la aplicación. Puede ser utilizado, por ejemplo, para indicarle a la aplicación en un archivo el acceso a la base de datos, incluyendo nombre de dominio del servidor, usuario y clave.


