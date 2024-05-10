# ST0263: Tópicos Especiales en Telemática, 2024-1

# Estudiantes: Diego Alejandro Vanegas, Sebastián Marín.

# Correos: davanegasg@eafit.edu.co, smaring2@eafit.edu.co.

# Profesor: Juan Carlos Montoya, jcmontoy@eafit.edu.co.


# Reto 4 - Kubernetes
En el reto 3, se implementó una versión dockerizada de un CMS con un balanceador de carga básico y dependencias en una base de datos y un sistema de archivos distribuido. En el reto 4, se desplegará la misma aplicación en un clúster Kubernetes en la nube, utilizando un servicio administrado como EKS de AWS. Los requisitos del reto 3, como nombre de dominio, HTTPS, balanceador de carga, base de datos externa y sistema de archivos independiente de la capa de servicios, también deben ser cumplidos. Los clústeres Kubernetes están ganando popularidad debido a su flexibilidad y capacidad de escalar vertical y horizontalmente con facilidad, utilizando herramientas como Manifiestos y lenguajes declarativos para gestionar eficazmente el crecimiento de la infraestructura y el despliegue de aplicaciones contenerizadas.

# Compilación y ejecución
## Paso 1
Dentro de EKS, crear un cluster con la siguiente configuración
### Configure cluster
- Name: [EKS_Cluster]
- Kubernetes version: default
- Cluster Service Role: LabRole
### Crear un security group para el EKS
Dentro de VPC > Security Groups > Create security group, colocar la siguiente configuración,
- Security group name: eks cluster
- Description: Open to all
- VPC: default
- Inbound rules: 
  - Type: All traffic
  - Protocol: All
  - Port range: All
  - Source: Anywhere IPv4
### Specify networking
- VPC: default
- subnets: 
  - us-east-1a
  - us-east-1b
  - us-east-1c
- Security groups: eks cluster
- Cluster endpoint access: Public
- Amazon VPC CNI Version: Latest
- CoreDNS Version: Latest
- kube-proxy Version: Latest

## Paso 2
Dentro del cluster, en la parte de "Compute" crear un Node Group con la siguiente configuración, 
### Configurar Node Group
- Name: EC2
- Node IAM Role: LabRole
- Kubernetes labels: 
  - worker: ec2
### Establecer configuración computador y escalamiento 
- AMI type: Amazon Linux 2 (x86_64)
- Capacity type: On-Demand
- Instance types: t3.medium
- Disk size: 10GiB
- Node Group scaling configuration
  - Minimum size: 2 nodes
  - Maximum size: 4 nodes
  - Desired size: 2 nodes

## Paso 3
Crear un environment en Cloud9, donde se manejarán todos los archivos, se ejecutarán los comandos y se configurarán los pods de acuerdo a lo requerido. Para esto, crearlo con la siguiente configuración
- Name: [c9_enviroment]
- Environment type: New EC2 Instance
- Instance type: t2.micro
- Platform: Amazon Linux 2023/Amazon Linux 2
- Timeout: 30 minutes
- Connection: (SSH)
- VPC settings: default-vpc
Finalmente, dentro del EC2 asociado al Cloud9, modificar el rol IAM a LabInstanceProfile

## Paso 4
Crear una consola de cloud9 y ejecutar el comando `aws configure`, el cual pedirá ingresar lo siguiente,
- aws_access_key: ASIAYS2NSONZZDRF5AMO (configuracion del cliente de su AWS academy)
- aws_secret_access_key: dY4U8RUihYpYtO/Xa8R/jdbe+arXn+A9rS7pp8A1 (configuracion del cliente de su AWS academy)
- Default region name: us-east-1
- Default output format: json

## Paso 5 - Instalar kubectl
Dentro de la consola de cloud9, ejecutar los siguientes comandos

`curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.29.3/2024-04-19/bin/linux/amd64/kubectl`

`chmod +x ./kubectl`

`mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH`

`echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc`

Reiniciar la consola de Cloud9

`kubectl version --client`

`aws eks update-kubeconfig --region us-east-1 --name [eks_name]`

## Paso 6 - Wordpress
Primero, se creará el despliegue con la imagen de wordpress, para ello, ejecutar el comando `kubectl create deployment [name] --image=wordpress`. Luego podremos hacerlo escalable al ejecutar `kubectl scale deployment [name] --replicas=[n_replicas]`.

Para verificar que todo haya quedado configurado correctamente, podremo ver los pods corriendo con el comando `kubectl get pods`. 

Luego, podremos exponer los puertos de los pods, para poder crear una conexión exitosa tanto del cliente como de la base de dato, para esto, el comando `kubectl expose deployment [name] --port=80 --type=LoadBalancer`.

*Nota:* Si se quiere hacer una aplicación wordpress con certificado SSL, es necesario exponer también el puerto 443 con los comandos 

- `kubectl expose deployment [load-balancer-123] --port=80 --target-port=80 --type=LoadBalancer` 
- `kubectl patch svc [name] -p '{"spec": {"ports": [{"port": 443, "targetPort": 80, "protocol": "TCP"}]}}'`

Una vez expuesto el puerto, podremos verificar que todo haya quedado correcto al ejecutar `kubectl get svc`, donde saldrá algo así

```
NAME         TYPE           CLUSTER-IP EXTERNAL-IP                        PORT(S)
wp-app       LoadBalancer (load)   10.0.0.0   name.us-east-1.elb.amazonaws.com   80:30120/TCP
```

## Paso 7 - MySQL
Dentro de RDS, crear una base de datos con el método "Standard create" y las siguientes configuraciones,
- Engine options: MySQL
- Engine Version: MySQL 8.0.35
- Templates: Free tier
- DB instance identifier: [db-reto4]
- Credentials Settings:
  - Master username: admin
  - Credentials management: Self managed
  - Master password: [12341234]
  - Confirm master password: [12341234]
- DB instance class: db.t3.micro
- Storage:
  - Storage type: gp2
  - Allocated storage: 20 GiB
  - Storage autoscalling: 
    - Enable storage autoscaling: true
    - Maximum storage threshold: 1000
- Compute resource:
  - Don't connect to an EC2 compute resource
  - VPC: default
  - DB subnet group: default
  - Public access: Yes
  - VPC security group (firewall): Choose existing
  - Existing VPC security groups: default
  - Availability Zone: No preference
  - Database port: 3306
- Database authentication: password authentication

## Paso 8 - WordPress Database
Desde una consola linux que tenga instalado previamente mysql, hacer una conexión remota a RDS para crear la base de datos `mysql -h [endpoint] -u admin -p`, al ejecutar el comando, pedirá ingresar la contraseña ingresada al crear el endpoint en RDS. Una vez dentro de mysql, crear una base de datos para configurarla dentro de wordpress, para ello `CREATE DATABASE [database-name];`; y para verificar que todo haya sido creado exitosamente, podemos ver las bases de datos creadas con `SHOW DATABASES;`.

## Paso 9 - Configurar WordPress
Para acceder al wordpress, podemos obtener el dns del cluster con el comando `kubectl get vpc`, el cual tendrá un external-ip que se verá así 'name.us-east-1.elb.amazonaws.com', con este podemos acceder dentro del navegador a la instancia de wordpress y configurar la conexión con la base de datos. 

En primer instancia, wordpress pedirá ingresar los siguientes valores,
- Database name: [database-name]
- Username: admin
- Password: [password]
- Database host: [endpoint]
- Table Prefix: wp_

## Paso 10 - EFS
Dentro de la sección de EFS de AWS Console, crear dos EFS
- WordPress
- MySql

Ubicar ambos EFS dentro del mismo VPC, en mi caso, vpc-default, luego verificar que ambos VPCs tengan los mismos subnets de la descripción de vpc-default.

Nota: Anotar los File system ID, ya que luego se usarán.
