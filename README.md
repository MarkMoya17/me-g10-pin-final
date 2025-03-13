# Proyecto integrador DevOps Grupo 10

## Integrantes

- Marquinho Moya
- Carlos Navarro
- Gabriel Barrera
- Federico Pascarella
- Julio Sejas

---

## Requisitos

- [**aws cli**](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [**kubectl**](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)
- [**eksctl**](https://eksctl.io/installation/)
- [**helm**](https://helm.sh/docs/intro/install/)

## Planificación del Cluster EKS

- Numero de nodos: 3
- Tipo de instancia: t3.small
- Region: us-east-1
- Zonas:
    - us-east-1a
    - us-east-1b
    - us-east-1c

---

## Configuración del Entorno Local para Desplegar un Cluster de EKS en AWS

> OS: Linux, Ubuntu 24.04.1 LTS
> 

### Instalar los paquete necesarios en requisitos

- aws cli
    
    Se ejecuto lo siguiente
    
    ```bash
    sudo apt install unzip -y 
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    sudo ./aws/install
    ```
    
    al finalizar se debería ver lo siguiente
    
    ![image.png](attachment:b89111d4-1266-4459-b19f-936d01d842ea:image.png)
    
    Y para verificar ejecutamos
    
    ```bash
    aws --version
    ```
    
    Debería imprimir los siguiente
    
    ![image.png](attachment:40c2aa43-47de-4dcc-ad43-ca662b699298:image.png)
    
- Kubectl
    
    Se ejecuto lo siguiente
    
    ```bash
    curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.32.0/2024-12-20/bin/linux/amd64/kubectl
    chmod +x ./kubectl
    mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
    echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
    ```
    
    Ahora ejecutamos lo siguiente para verificar que esta todo correcto
    
    ```bash
    kubectl version --client
    ```
    
    Debería imprimir lo siguiente
    
    ![image.png](attachment:60e7763f-616c-484b-bffe-623b00a6ece6:image.png)
    
- eksctl
    
    Se ejecuto lo siguiente
    
    ```bash
    # for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
    ARCH=amd64
    PLATFORM=$(uname -s)_$ARCH
    
    curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
    
    # (Optional) Verify checksum
    curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check
    
    tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
    
    sudo mv /tmp/eksctl /usr/local/bin
    ```
    
    Ahora ejecutamos lo siguiente para verificar que esta todo correcto
    
    ```bash
    eksctl info
    ```
    
    Debería imprimir lo siguiente
    
    ![image.png](attachment:1443940c-e4fb-4880-adbb-91285556572d:image.png)
    
- Helm
    
    Se ejecuto lo siguiente
    
    ```bash
    curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
    sudo apt-get install apt-transport-https --yes
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
    sudo apt-get update
    sudo apt-get install helm -y
    ```
    
    Ahora ejecutamos lo siguiente para verificar que esta todo correcto
    
    ```bash
    helm version
    ```
    
    Debería imprimir lo siguiente
    
    ![image.png](attachment:5a5d7f7b-74ce-4dcb-85d5-4939befc2851:image.png)
    

### Configurar AWS

Ahora vamos a configurar el AWS cli con las credenciales. Para mas información consulte

[Setting up the AWS CLI - AWS Command Line Interface](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html)

Una vez que tenemos el KEY ID y el ACCESS KEY, lo configuramos de la siguiente manera

![Por seguridad pintamos las keys](attachment:47e2e66b-a338-4d9c-bed9-6a10769c59ab:image.png)

Por seguridad pintamos las keys

Una vez que tenemos configurado el AWS cli podemos verificar con el siguiente comando

```bash
aws sts get-caller-identity
```

Debería imprimir

![image.png](attachment:ef80c8bf-e8c4-410a-ae64-10e79e9afcd7:image.png)

Con esto tenemos todo configurado para continuar

---

## Crear el Cluster EKS

### Creando Key Pair

Antes de crear en cluster necesitamos crear un ssh-key y subirlo a AWS.
Creamos una carpeta en $HOME y generamos la key
Se ejecuto lo siguiente

```bash
mkdir $HOME/eks && cd $HOME/eks && ssh-keygen -t rsa -b 4096 -C "eks-ssh" -f ./eks-ssh -N ""
```

Una ves finalizado debería imprimir los siguiente y con **`ls`** verificamos que este

![image.png](attachment:2159b2b3-81dd-40b6-add9-a2f99bda292f:image.png)

### Iniciar Cluster

Para la creación del cluster eks usamos este comando

```bash
eksctl create cluster \
--name eks-tp-m \
--region us-east-1 \
--node-type t3.small \
--nodes 3 \
--with-oidc \
--ssh-access \
--ssh-public-key $HOME/eks/eks-ssh.pub \
--managed \
--full-ecr-access \
--zones us-east-1a,us-east-1b,us-east-1c
```

Si todo esta correcto deberíamos tener lo siguiente a la espera que se termine de ejecutar la creación. 

![Esto podría tardar entre 15m-40m](attachment:7a692eab-51b3-4fcb-a0cc-9ef94ebaebb0:image.png)

Esto podría tardar entre 15m-40m

Una vez finalizado deberíamos ver lo siguiente

![Con `kubectl get nodes` podemos ver los nodos disponibles](attachment:df096a14-c626-409c-aea1-1bbd692cbcb8:image.png)

Con `kubectl get nodes` podemos ver los nodos disponibles

---

## Levantar Servicios

Ahora vamos a levantar varios servicios

### Nginx

Para levantar nginx creamos un archivo `nginx.yaml` con el siguiente contenido

```yaml
---
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx  
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:
  externalTrafficPolicy: Local
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: LoadBalancer
```

Ahora levantamos con

```bash
kubectl apply -f nginx.yaml
```

Una vez levantado podemos acceder desde el dns de elb. Para traer los dns de los elb podemos ejecutar el siguiente comando

```bash
kubectl get svc -n default
```

Vamos a tener dos entramos al EXTERNAL-IP de nginx

![image.png](attachment:8f530b7e-4588-4bf8-b5f2-777c012a392b:image.png)

accedemos y listo ya tenemos nginx
