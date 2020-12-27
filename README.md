![Nestor Tech Tips](https://storage.googleapis.com/nestortechtips.online/cover.png)
# Provisionador dinámico de volumenes de Kubernetes con servidor NFS

El motivo por el que escribo esta publicación es el final de soporte del chart de Helm que utilizaba para crear el provisionador dinámico de volúmenes en Kubernetes. Después de una búsqueda rápida por Internet descubrí que los tutoriales/repositorios existentes no son muy claros al momento de explicar como migrar de Helm o, como lo haré en este caso, cómo crear los objetos necesarios para configurar el provisionador.

# Requisitos
* Acceso al clúster de Kubernetes con *kubectl*
* Acceso al servidor NFS desde cada nodo del clúster de Kubernetes (este req    uisito se puede satisfacer instalando el paquete *nfs-common*)

# Procedimiento

Para poder configurar el provisionador necesitamos contar con los siguientes datos del servidor NFS
* Dirección IP o hostname
* Ruta del directorio compartido por NFS

## Creación de roles
En las versiones modernas de Kubernetes es necesario crear los Roles, ClusterRoleBindings y ServiceAccounts que nos permitan crear el *StorageClass* dentro del clúster.

En tu editor de texto favorito crea el archivo rbac.yaml y agrega el siguiente contenido, no olvides reemplazar **nfs-nas** por el valor del Namespace donde deseas crear el provisionador:

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # Reemplaza con el nombre del Namespace donde se creará el provisionador
  namespace: nfs-nas
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # Reemplaza con el nombre del Namespace donde se creará el provisionador
    namespace: nfs-nas
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # Reemplaza con el nombre del Namespace donde se creará el provisionador
  namespace: nfs-nas
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # Reemplaza con el nombre del Namespace donde se creará el provisionador
  namespace: nfs-nas
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # Reemplaza con el nombre del Namespace donde se creará el provisionador
    namespace: nfs-nas
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

Una vez que el archivo *rbac.yaml* se encuentre configurado de forma correcta podemos crear los roles con el siguiente comando:
```
kubectl apply -f rbac.yaml
```
## Creación de Deployment
En tu editor de texto favorito crea el archivo *nfs-provisioner-deployment.yaml* y agrega el siguiente contenido:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nas-provisioner
  namespace: nfs-nas
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
      release: nas-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
        release: nas-provisioner
    spec:
      containers:
      - env:
        - name: PROVISIONER_NAME
          value: nfs-nas
        - name: NFS_SERVER
          value: 192.168.4.19
        - name: NFS_PATH
          value: /volume1/storageclass/
        image: quay.io/external_storage/nfs-client-provisioner
        imagePullPolicy: IfNotPresent
        name: nfs-client-provisioner
        volumeMounts:
        - mountPath: /persistentvolumes
          name: nfs-client-root
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      serviceAccount: nfs-client-provisioner
      serviceAccountName: nfs-client-provisioner
      volumes:
      - name: nfs-client-root
        nfs:
          path: /volume1/storageclass/
          server: 192.168.4.19
```
En el archivo anterior debemos modificar un par de valores de acuerdo a los datos de nuestro servidor NFS.

En las variables de ambiente debemos de cambiar el valor de:
* PROVISIONER_NAME - Nombre que queremos asignar al provisionador, este valor será el que se utilice en el siguiente paso
* NFS_SERVER - Dirección IP o hostname del servidor NFS
* NFS_PATH - Ruta de la carpeta compartida en el servidor NFS
* image - En caso de utilizar una Raspberry cambiar la imagen por *quay.io/external_storage/nfs-client-provisioner-arm*
* namespace - Nombre del namespace configurado en el paso anterior

Una vez configurados los valores en el archvio creamos el Deployment con el siguiente comando:
```
kubectl apply -f nfs-provisioner-deployment.yaml
```

## Creación de  StorageClass

Finalmente crearemos el StorageClass en Kubernetes.

Para hacerlo debemos crear el archivo *provisioner-class.yaml* con el siguiente contenido:

```bash
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  labels:
    app: nfs-client-provisioner
    release: nas-provisioner
  name: nfs-nas
parameters:
  archiveOnDelete: "true"
provisioner: nfs-nas
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

En el archivo anterior no olvidemos cambiar los valores siguientes:
* name - Nombre del Provisionador que definimos en el paso anterior
* provisioner - Nombre que tendrá el Provisionador en nuestro clúster de Kubernetes. Se recomienda que sea el mismo valor que asignamos en *name*

Para crear la clase es suficiente con ejecutar el siguiente comando:
```bash
kubectl apply -f provisioner-class.yaml
```

Una vez realizados los pasos anteriores podremos crear PVC para que nuestras Workloads pueden disponer de un almacenamiento no volátil

## Contribuciones
Puedes contribuir utilizando **Pull Requests** en el repositorio correspondiente. De no existir crea un archivo *contrib.txt*, de existir modifica el archivo para agregar tu nombre de usuario o correo. Por ejemplo:
* `fulano.detal@correo.com`
* `perengano`

## Créditos
El contenido de este repositorio corresponde a la publicación homónima en [nestortechtips.online](https://nestortechtips.online/).