Hola que tal amigos, en esta ocasion vamos a hablar acerca de los usuarios y roles en Kubernetes

Una vez hayamos creado nuestro cluster de kubernetes vamos a tener un archivo de configuración el cual tiene las credenciales de administrador o super usuario de nuestro cluster, entonces si queremos que nuestro cluster lo administren varias personas, compartir esas credenciales no es una buena practica de seguridad, ya que con esta cuenta se podrían ocasionar cambios a gran escala y hasta destructivos para nuestro cluster debido como les comente cuenta con permisos de super usuario.

Entonces para solucionar este problema vamos a crear usuarios adicionales que se autenticaran desde sus respectivas computadoras o clientes locales.

Para eso crearemos certificados SSL y TLS seguros. que les mostrare a continuación:

## Paso 1

Probar el acceso a nuestro cluster, ejecutamos el siguiente comando:

```
kubectl cluster-info
```

## Paso 2

Creamos una carpeta llamada rbac e ingresamos a ella

```
$ mkdir certs
$ cd certs/
```

## Paso 3

Creamos un private key para el usuario que vamos a usar utilizando la herramienta openssl

```
openssl genrsa -out kevo.key 4096
```

## Paso 4

Creamos un Manifiesto llamado  **kevo.csr.cnf**

```
code kevo.csr.cnf
```

con el siguiente contenido:

```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
[ dn ]
CN = kevo
O = developers
[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
```

el CN o Common Name sera para el Nombre y O de Organizacion sera para agregar o especificar el grupo al que pertenece

**extendedKeyUsage=serverAuth,clientAuth** permitirá que los usuarios autentiquen sus clientes locales con el clúster usando el certificado una vez que se haya firmado.



**PASO 5**

A continuancion vamos a crear la solicitud de firma de certificados para el usuario

```
openssl req -config kevo.csr.cnf -new -key kevo.key -nodes -out kevo.csr
```

Con `-config` se le permite especificar el archivo de configuración para la CSR, y `-new` indica que creará una nueva CSR para la clave especificada por `-key`.

Podemos ver la solicitud de firma de certificados ejecutando el siguiente comando:

```
openssl req -in kevo.csr -noout -text
```

**PASO 6**

Con el siguiente comando enviaremos la solicitud para ser evaluada y posteriormente aprobada

```
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: kevo-authentication
spec:
  groups:
  - system:authenticated
  request: $(cat ~/your path/kevo.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
  - client auth
EOF
```

Con esto habremos enviado la solicitud de firma de certificado

Para verificarlo ejecutamos el siguiente comando:

```
kubectl get csr
```

## Paso 7

Para aprobar la solicitud ejecutamos el siguiente comando:

```
kubectl certificate approve kevo-authentication
```

## Paso 8

Una vez que se haya  aprobado el CSR, Puedoo descargarla a la maquina local ejecutando lo siguiente

```
kubectl get csr kevo-authentication -o jsonpath='{.status.certificate}' | base64 --decode > kevo.crt
```

## Paso 9

A continuación, creará un archivo **kubeconfig** específico para el usuario **kevo**. Esto le proporcionará más control sobre el acceso del usuario a su clúster.

```
cp ~/.kube/config ~/.kube/config-kevo
```

Y luego modificaremos el archivo de configuracion creado 

```
code ~/.kube/config-kevo
```

Una vez listo podriamos probar el acceso al cluster con lo siguiente:

```
kubectl --kubeconfig=/home/your local user/.kube/config-kevo cluster-info
```



## Paso 10

Como vemos no tendremos permisos, eso se debe a que este usuario aun no se le ha asignado ningun rol por lo que ahora vamos a proceder a ello, En este caso yo quiero que ese usuario solo tenga acceso a un NS en especifico asi que lo voy a crear:



Creamos un nuevo Namespace llamado **example**

```
kubectl create ns example
```

y ademas voy a crear un deployment de prueba en el NS creado anteriormente:

```
kubectl -n example create deployment nginx --image=nginx:alpine
```

```
kubectl -n example get pods
```

## Paso 11

Creare una carpeta llamada rbac:

```
mkdir rbac
```

Vamos a crear un rol que nos permita listar los pods de ese NS usando un archivo llamado **role.yaml**

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: example
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

Lo creamos con el siguiente comando:

```
kubectl create -f role.yaml
```

Y para asiganar dicho rol al usuario Studiante debemos crear un archivo llamado **rolebinding.yaml**

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-read-access
  namespace: example
subjects:
- kind: User
  name: kevo
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Y lo ejecutamos con el siguiente comando:

```
kubectl create -f rolebinding.yaml 
```

una vez ya aplicado el role veremos si ahora tenemos permisos:

```
kubectl --kubeconfig=/home/Your local user/.kube/config-kevo -n example get pods
```

## Mi pagina web:

[kevorojas.com](https://kevorojas.com)

## Link video de Youtube

[usuarios y  roles kubernetes]()
