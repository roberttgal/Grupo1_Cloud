# Reto 4: Docker Swarm en AWS con Terraform

Despliegue de una web estática de Fórmula 1 en un clúster Docker Swarm
provisionado con Terraform sobre AWS `us-east-1`.

## Descripción general

La infraestructura está formada por cuatro instancias EC2:

| Nodo | Cantidad | Tipo | Función |
| --- | ---: | --- | --- |
| Manager | 1 | `t3.medium` | Gestiona el clúster y programa los servicios |
| Worker | 3 | `t3.small` | Ejecutan las réplicas de la aplicación |

El servicio web se publica mediante el routing mesh de Swarm en el puerto
`80`. La aplicación se despliega como cuatro réplicas sobre la red overlay
externa `red-g1`, con MTU `1450`.

> El estado actual del repositorio usa `ec2-ubuntu/` para Terraform,
> `web/` para los archivos de la aplicación, `nginx/` para la configuración
> del servidor y `probar-balanceo.sh` para comprobar el balanceo. 
## Estructura del repositorio

```text
.
├── .devcontainer/       # Entorno VS Code con AWS CLI y Terraform
├── ec2-ubuntu/           # Terraform: EC2, security group, variables y outputs
├── nginx/                # Configuración de Nginx y cabecera de réplica
├── web/                  # Sitio estático de F1 Bilbao
├── Dockerfile            # Imagen Nginx de la aplicación
├── docker-compose.yml    # Servicio Swarm f1-bilbao_web
└── probar-balanceo.sh    # Comprueba las réplicas mediante X-Container-Hostname
```

Terraform crea:

- una instancia manager con hostname `manager-g1`;
- tres instancias worker con hostnames `worker1-g1` a
  `worker3-g1`;
- un security group con SSH, HTTP y los puertos necesarios para Swarm:
  TCP `2377`, TCP/UDP `7946` y UDP `4789`.

## Imágenes publicadas en Docker Hub

- [F1 Bilbao `1.0`](https://hub.docker.com/r/lopiasir/f1-bilbao/tags?name=1.0)
- [F1 Bilbao `2.0`](https://hub.docker.com/r/lopiasir/f1-bilbao/tags?name=2.0)

Para construir y publicar ambas versiones desde la raíz del repositorio:

```bash
docker login
docker build --build-arg IMAGE_VERSION=1.0 -t lopiasir/f1-bilbao:1.0 .
docker push lopiasir/f1-bilbao:1.0

docker build --build-arg IMAGE_VERSION=2.0 -t lopiasir/f1-bilbao:2.0 .
docker push lopiasir/f1-bilbao:2.0
```

## Despliegue rápido

### 1. Provisionar AWS con Terraform

Configura tus credenciales de AWS y asegúrate de tener disponible el par de
claves indicado por `clave` (por defecto, `grupo1-aws`):

```bash
cd ec2-ubuntu
terraform init
terraform plan
terraform apply
terraform output manager_public_ip
terraform output manager_private_ip
terraform output workers_public_ips
terraform output workers_private_ips
```

La región por defecto es `us-east-1`. Para usar otro grupo o clave:

```bash
terraform apply \
  -var='grupo=1' \
  -var='clave=grupo1-aws'
```

### 2. Inicializar el Swarm

Conéctate al manager usando su IP pública y anuncia su IP privada:

```bash
ssh -i ~/.ssh/grupo1-aws ubuntu@<IP_PUBLICA_MANAGER>
docker swarm init --advertise-addr <IP_PRIVADA_MANAGER>
docker swarm join-token worker
```

Ejecuta el comando `docker swarm join` mostrado por el manager en cada uno
de los tres workers:

```bash
docker swarm join --token <TOKEN_WORKER> <IP_PRIVADA_MANAGER>:2377
```

Verifica el clúster desde el manager:

```bash
docker node ls
```

### 3. Crear la red overlay y desplegar la aplicación

El archivo Compose debe estar disponible en el manager. Desde tu equipo,
cópialo después de unir los nodos:

```bash
scp -i ~/.ssh/grupo1-aws docker-compose.yml \
  ubuntu@<IP_PUBLICA_MANAGER>:/home/ubuntu/docker-compose.yml
```

Desde el manager, crea la red externa con la MTU requerida y despliega el
stack:

```bash
docker login
docker network create \
  --driver overlay \
  --attachable \
  --opt com.docker.network.driver.mtu=1450 \
  red-g1

docker stack deploy --with-registry-auth \
  --compose-file docker-compose.yml \
  f1-bilbao

docker service ls
docker service ps f1-bilbao_web
```

El servicio debe mostrar `4/4` réplicas. Si `red-g1` ya existe, no es
necesario crearla de nuevo.

### 4. Verificar el acceso

El routing mesh permite acceder al servicio desde la IP pública de cualquier
nodo:

```bash
for ip in <IP_MANAGER> <IP_WORKER_1> <IP_WORKER_2> <IP_WORKER_3>; do
  echo "== $ip =="
  curl --fail --silent --show-error "http://$ip/"
  echo
done
```

Para observar el balanceo entre réplicas:

```bash
chmod +x probar-balanceo.sh
./probar-balanceo.sh http://<IP_MANAGER>/
```

La respuesta incluye la cabecera `X-Container-Hostname`, que permite
identificar el contenedor que atendió cada petición.

## Actualización y rollback

La imagen inicial del stack es `lopiasir/f1-bilbao:1.0`. Para actualizarla
sin detener el servicio y conservar la autenticación del registro:

```bash
docker service update \
  --image lopiasir/f1-bilbao:2.0 \
  --with-registry-auth \
  --update-parallelism 1 \
  --update-delay 10s \
  --update-order start-first \
  --update-failure-action rollback \
  f1-bilbao_web
```

Comprueba la versión y, si es necesario, vuelve a la anterior:

```bash
curl http://<IP_MANAGER>/version.txt
docker service ps f1-bilbao_web
docker service rollback f1-bilbao_web
```

## Comandos útiles

Escalar temporalmente el servicio:

```bash
docker service scale f1-bilbao_web=8
docker service scale f1-bilbao_web=12
docker service scale f1-bilbao_web=2
```

Poner un worker en mantenimiento y reactivarlo:

```bash
docker node update --availability drain <NODO_WORKER>
docker service ps f1-bilbao_web
docker node update --availability active <NODO_WORKER>
```

Consultar de nuevo las IP después de detener o iniciar instancias:

```bash
cd ec2-ubuntu
terraform refresh
terraform output manager_public_ip
terraform output workers_public_ips
```

