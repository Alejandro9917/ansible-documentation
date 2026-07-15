# Sesión 0 — Requirements del laboratorio

## Objetivo

Preparar el entorno local necesario para ejecutar los primeros laboratorios de Ansible utilizando contenedores Docker que simulan servidores Ubuntu Server.

En esta primera etapa no se instalarán servicios como Redis ni software adicional del caso de estudio. El objetivo inicial es contar con varios servidores Linux disponibles para practicar instalación de dependencias, validación de conectividad y ejecución de comandos con Ansible.

---

## Enfoque del laboratorio

Para evitar el uso de máquinas virtuales completas, utilizaremos contenedores Docker como pequeños servidores Ubuntu.

```text
Máquina host
Ansible Control Node
        |
        | SSH hacia puertos locales
        |
+----------------+
| Ubuntu Server  |
| device01:2221  |
+----------------+

+----------------+
| Ubuntu Server  |
| device02:2222  |
+----------------+

+----------------+
| Ubuntu Server  |
| device03:2223  |
+----------------+
```

---

## Herramientas requeridas

Antes de iniciar los laboratorios, cada participante debe tener instalado:

- Git
- Docker
- Docker Compose
- Python 3
- pip
- Ansible
- Editor de código, recomendado Visual Studio Code
- Terminal Linux, macOS o WSL2 en Windows

---

## Requisitos mínimos recomendados

| Recurso | Recomendado |
|---|---|
| Memoria RAM | 8 GB |
| CPU | 2 cores o más |
| Espacio libre | 5 GB |
| Sistema operativo | Linux, macOS o Windows con WSL2 |
| Internet | Requerido para descargar imágenes y dependencias |

---

## Instalación de componentes

Antes de validar el entorno, cada participante debe instalar los componentes base que se utilizarán durante el laboratorio.

### Instalar Python 3, pip y venv

#### Ubuntu o Debian

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv
```

#### macOS

```bash
brew install python
```

#### Windows con WSL2

Ejecutar dentro de la distribución Linux instalada en WSL2:

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv
```

### Instalar Ansible en un virtual environment

Ansible se instalará dentro de un entorno virtual para evitar dependencias globales en el sistema operativo.

Crear el entorno virtual:

```bash
python3 -m venv .venv-ansible
```

Activar el entorno virtual:

```bash
source .venv-ansible/bin/activate
```

Actualizar `pip` e instalar Ansible:

```bash
python -m pip install --upgrade pip
python -m pip install ansible
```

Validar la instalación:

```bash
ansible --version
```

Cuando se termine de trabajar, el entorno virtual se puede desactivar con:

```bash
deactivate
```

### Instalar Docker y Docker Compose

#### Ubuntu o Debian

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo systemctl enable --now docker
```

Agregar el usuario actual al grupo `docker`:

```bash
sudo usermod -aG docker "$USER"
```

Cerrar la sesión y volver a entrar para que el cambio de grupo tenga efecto.

#### macOS

Instalar Docker Desktop desde el sitio oficial de Docker. Docker Compose ya viene incluido con Docker Desktop.

#### Windows

Instalar Docker Desktop con soporte para WSL2 y habilitar la integración con la distribución Linux que se utilizará para el laboratorio. Docker Compose ya viene incluido con Docker Desktop.

---

## Validar instalaciones

```bash
git --version
docker --version
docker compose version
python3 --version
pip3 --version
python3 -m venv --help
source .venv-ansible/bin/activate
ansible --version
```

Si alguno de estos comandos falla, se debe corregir la instalación antes de continuar.

---

## Estructura sugerida del repositorio

```text
ansible-agent-lab/
├── docs/
│   ├── sesion-0.md
│   ├── sesion-1.md
│   └── sesion-2.md
│
├── docker/
│   ├── Dockerfile.ubuntu
│   └── docker-compose.yml
│
├── inventory/
│   └── lab.ini
│
├── group_vars/
├── host_vars/
├── playbooks/
├── roles/
├── ansible.cfg
├── site.yml
└── README.md
```

---

## Contenedores iniciales

Para los primeros laboratorios únicamente se utilizarán servidores Ubuntu:

| Contenedor | Rol |
|---|---|
| device01 | Servidor Ubuntu administrado |
| device02 | Servidor Ubuntu administrado |
| device03 | Servidor Ubuntu administrado |

Más adelante se podrán agregar otros servicios como Redis, monitoreo o software agente, pero no forman parte de esta sesión inicial.

---

## Objetivo técnico del laboratorio inicial

Al finalizar esta preparación, se espera contar con:

- La máquina host funcionando como nodo de control con Ansible instalado.
- Tres contenedores Ubuntu Server accesibles por SSH.
- Un inventario básico de Ansible.
- Conectividad validada entre el nodo de control y los hosts administrados.
- Ambiente listo para probar instalación de dependencias con módulos de Ansible.

---

## Flujo esperado

```text
1. Crear la estructura del laboratorio
2. Crear la llave SSH del laboratorio
3. Crear el archivo docker-compose.yml
4. Levantar los contenedores Ubuntu
5. Validar que los contenedores estén activos
6. Probar conectividad SSH desde la máquina host
7. Crear o validar el inventory de Ansible
8. Ejecutar el primer ping de Ansible
```

---

## Docker Compose del laboratorio

Antes de crear o ejecutar el `docker-compose.yml`, se debe preparar la llave SSH del laboratorio dentro de la carpeta de la capacitación.

Desde la raíz del proyecto del laboratorio:

```bash
mkdir -p ssh
ssh-keygen -t rsa -b 4096 -f ssh/ansible_lab -N ""
chmod 600 ssh/ansible_lab
ls -l ssh/ansible_lab ssh/ansible_lab.pub
```

El archivo que Docker Compose necesita encontrar es:

```text
ssh/ansible_lab.pub
```

Ese archivo será montado dentro de cada contenedor mediante un bind mount. Si `ssh/ansible_lab.pub` no existe antes de levantar los contenedores, los devices no podrán copiar la llave pública a `authorized_keys` y la conexión SSH desde Ansible fallará.

Crear el archivo `docker-compose.yml` en la raíz del laboratorio con el siguiente contenido:

```yaml
services:
  device01:
    image: ubuntu:24.04
    container_name: device01
    hostname: device01
    command: >
      bash -lc "
      apt-get update &&
      apt-get install -y openssh-server python3 sudo &&
      useradd -m -s /bin/bash ansible || true &&
      echo 'ansible ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/ansible &&
      mkdir -p /var/run/sshd /home/ansible/.ssh &&
      cp /tmp/ansible_lab.pub /home/ansible/.ssh/authorized_keys &&
      chown -R ansible:ansible /home/ansible/.ssh &&
      chmod 700 /home/ansible/.ssh &&
      chmod 600 /home/ansible/.ssh/authorized_keys &&
      /usr/sbin/sshd -D
      "
    ports:
      - "2221:22"
    volumes:
      - ./ssh/ansible_lab.pub:/tmp/ansible_lab.pub:ro
    networks:
      - ansible-lab

  device02:
    image: ubuntu:24.04
    container_name: device02
    hostname: device02
    command: >
      bash -lc "
      apt-get update &&
      apt-get install -y openssh-server python3 sudo &&
      useradd -m -s /bin/bash ansible || true &&
      echo 'ansible ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/ansible &&
      mkdir -p /var/run/sshd /home/ansible/.ssh &&
      cp /tmp/ansible_lab.pub /home/ansible/.ssh/authorized_keys &&
      chown -R ansible:ansible /home/ansible/.ssh &&
      chmod 700 /home/ansible/.ssh &&
      chmod 600 /home/ansible/.ssh/authorized_keys &&
      /usr/sbin/sshd -D
      "
    ports:
      - "2222:22"
    volumes:
      - ./ssh/ansible_lab.pub:/tmp/ansible_lab.pub:ro
    networks:
      - ansible-lab

  device03:
    image: ubuntu:24.04
    container_name: device03
    hostname: device03
    command: >
      bash -lc "
      apt-get update &&
      apt-get install -y openssh-server python3 sudo &&
      useradd -m -s /bin/bash ansible || true &&
      echo 'ansible ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/ansible &&
      mkdir -p /var/run/sshd /home/ansible/.ssh &&
      cp /tmp/ansible_lab.pub /home/ansible/.ssh/authorized_keys &&
      chown -R ansible:ansible /home/ansible/.ssh &&
      chmod 700 /home/ansible/.ssh &&
      chmod 600 /home/ansible/.ssh/authorized_keys &&
      /usr/sbin/sshd -D
      "
    ports:
      - "2223:22"
    volumes:
      - ./ssh/ansible_lab.pub:/tmp/ansible_lab.pub:ro
    networks:
      - ansible-lab

networks:
  ansible-lab:
    driver: bridge
```

Este archivo crea tres servidores Ubuntu administrados con SSH y Python 3. La máquina host ejecutará Ansible y se conectará a los contenedores por los puertos locales `2221`, `2222` y `2223`.

### Por qué se utilizan estos comandos

El bloque `command` permite preparar cada contenedor cuando inicia. Como estamos usando la imagen base `ubuntu:24.04`, el contenedor arranca con un sistema mínimo y necesita instalar las herramientas requeridas para el laboratorio.

En `device01`, `device02` y `device03` se ejecutan estos pasos:

| Comando | Propósito |
|---|---|
| `apt-get update` | Actualiza el índice de paquetes disponibles dentro del contenedor. |
| `apt-get install -y openssh-server python3 sudo` | Instala servidor SSH, Python para los módulos de Ansible y sudo. |
| `useradd -m -s /bin/bash ansible \|\| true` | Crea el usuario remoto que Ansible utilizará para conectarse. |
| `echo 'ansible ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/ansible` | Permite que el usuario `ansible` ejecute tareas con privilegios sin pedir contraseña. |
| `mkdir -p /var/run/sshd /home/ansible/.ssh` | Crea los directorios necesarios para SSH y las llaves del usuario. |
| `cp /tmp/ansible_lab.pub .../authorized_keys` | Autoriza la llave pública generada en la máquina host para permitir acceso SSH. |
| `/usr/sbin/sshd -D` | Inicia el servidor SSH en primer plano para mantener el contenedor activo. |

Cada contenedor también publica su puerto SSH interno `22` hacia un puerto distinto de la máquina host:

| Contenedor | Puerto del contenedor | Puerto en la máquina host |
|---|---:|---:|
| `device01` | `22` | `2221` |
| `device02` | `22` | `2222` |
| `device03` | `22` | `2223` |

Esto permite que Ansible se ejecute desde la máquina host y se conecte a cada servidor usando `127.0.0.1` con un puerto diferente.

### Por qué se usa una network

La sección `networks` crea una red Docker llamada `ansible-lab`:

```yaml
networks:
  ansible-lab:
    driver: bridge
```

Esta red permite que los contenedores se comuniquen entre sí de forma aislada del resto del equipo. Todos los servicios están conectados a esa red:

```yaml
networks:
  - ansible-lab
```

Docker también proporciona resolución DNS interna dentro de la red. Aunque Ansible se ejecutará desde la máquina host usando puertos publicados, mantener una red explícita permite que los contenedores queden agrupados y aislados como parte del mismo laboratorio.

El beneficio principal es que el laboratorio queda reproducible y separado de otros contenedores que puedan existir en el equipo.

### Por qué se monta la llave pública

Cada device monta la llave pública generada en la máquina host antes de iniciar los contenedores:

```yaml
volumes:
  - ./ssh/ansible_lab.pub:/tmp/ansible_lab.pub:ro
```

Este bind mount toma el archivo local `./ssh/ansible_lab.pub` y lo presenta dentro del contenedor como `/tmp/ansible_lab.pub` en modo solo lectura.

En este laboratorio se usa para instalar la llave pública dentro de cada servidor:

- La máquina host genera `ssh/ansible_lab` y `ssh/ansible_lab.pub`.
- Docker Compose monta `ssh/ansible_lab.pub` como `/tmp/ansible_lab.pub` en cada device.
- Los servidores `device01`, `device02` y `device03` leen `/tmp/ansible_lab.pub` al iniciar.
- Cada servidor copia esa llave pública a `authorized_keys`.

Esto permite que la máquina host se conecte por SSH a los servidores sin contraseñas usando la llave privada `ssh/ansible_lab`.

---

## Comandos base del laboratorio

### Crear la llave SSH del laboratorio

Antes de levantar los contenedores, crear una llave RSA que usará la máquina host para conectarse a los devices por SSH.

Estos comandos deben ejecutarse desde la raíz de la carpeta de la capacitación, en el mismo nivel donde estará el archivo `docker-compose.yml`.

Crear el directorio donde se guardarán las llaves:

```bash
mkdir -p ssh
```

Generar la llave RSA del laboratorio:

```bash
ssh-keygen -t rsa -b 4096 -f ssh/ansible_lab -N ""
```

Ajustar permisos de la llave privada:

```bash
chmod 600 ssh/ansible_lab
```

Validar que se crearon los dos archivos:

```bash
ls -l ssh/ansible_lab ssh/ansible_lab.pub
```

Los archivos generados tienen propósitos distintos:

| Archivo | Uso |
|---|---|
| `ssh/ansible_lab` | Llave privada. La usa Ansible desde la máquina host para conectarse por SSH. |
| `ssh/ansible_lab.pub` | Llave pública. Docker Compose la monta dentro de cada device y la copia a `authorized_keys`. |

El archivo `ssh/ansible_lab.pub` será montado dentro de cada contenedor y agregado a `authorized_keys`. La llave privada `ssh/ansible_lab` nunca debe copiarse dentro de los contenedores ni subirse al repositorio.

### Levantar contenedores

```bash
docker compose up -d
```

### Verificar contenedores activos

```bash
docker ps
```

### Probar conectividad SSH desde la máquina host

Validar que la máquina host puede conectarse a cada device usando la llave privada del laboratorio:

```bash
ssh -i ssh/ansible_lab -p 2221 ansible@127.0.0.1 hostname
ssh -i ssh/ansible_lab -p 2222 ansible@127.0.0.1 hostname
ssh -i ssh/ansible_lab -p 2223 ansible@127.0.0.1 hostname
```

### Validar Ansible desde la máquina host

```bash
source .venv-ansible/bin/activate
ansible --version
```

### Crear el inventory inicial

Desde la máquina host, crear el directorio `inventory`:

```bash
mkdir -p inventory
```

Crear el archivo `inventory/lab.ini` con el siguiente contenido:

```ini
[devices]
device01 ansible_host=127.0.0.1 ansible_port=2221
device02 ansible_host=127.0.0.1 ansible_port=2222
device03 ansible_host=127.0.0.1 ansible_port=2223

[all:vars]
ansible_user=ansible
ansible_ssh_private_key_file=ssh/ansible_lab
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

### Cómo se configuran las IPs

En este laboratorio no se configurarán IPs fijas manualmente. Docker Compose crea la red `ansible-lab` y asigna una IP privada a cada contenedor dentro de esa red.

Como Ansible se ejecuta desde la máquina host, la conexión se hará usando `127.0.0.1` y los puertos publicados por Docker:

| Host en Ansible | Contenedor Docker | Dirección usada por Ansible |
|---|---|---|
| device01 | device01 | `127.0.0.1:2221` |
| device02 | device02 | `127.0.0.1:2222` |
| device03 | device03 | `127.0.0.1:2223` |

Cada puerto local redirige al puerto `22` del contenedor correspondiente.

Para consultar la IP asignada por Docker a un contenedor:

```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' device01
```

No se recomienda depender de esas IPs para el laboratorio, porque pueden cambiar si los contenedores se eliminan y se vuelven a crear. La referencia estable para Ansible será `127.0.0.1` más el puerto publicado de cada device.

### Ejecutar prueba de conectividad

```bash
ansible -i inventory/lab.ini all -m ping
```

---

## Resultado esperado

La salida esperada del comando `ansible -i inventory/lab.ini all -m ping` debe indicar que todos los hosts responden correctamente:

```text
device01 | SUCCESS
device02 | SUCCESS
device03 | SUCCESS
```

Esto confirma que el entorno está listo para continuar con los laboratorios de instalación de paquetes, configuración de servicios y ejecución de tareas automatizadas.

---

## Troubleshooting básico

### Docker no está ejecutándose

Validar que Docker Desktop o el servicio Docker estén activos.

```bash
docker ps
```

### Los contenedores no aparecen

Ejecutar nuevamente:

```bash
docker compose up -d
```

### Ansible no conecta por SSH

Revisar:

- Que los contenedores estén activos.
- Que el usuario SSH exista.
- Que las llaves SSH estén configuradas.
- Que el inventory tenga los nombres correctos.
- Que los puertos `2221`, `2222` y `2223` estén publicados correctamente.
- Que no exista otro proceso usando esos puertos en la máquina host.

### Error de Python en los hosts

Verificar que Python 3 esté instalado en los contenedores Ubuntu:

```bash
python3 --version
```

Ansible necesita Python en los hosts administrados para ejecutar la mayoría de módulos.

---

## Nota para el curso

Docker se utilizará únicamente como mecanismo para simular servidores Linux. El objetivo de esta capacitación no es profundizar en Docker, sino utilizarlo como base ligera para practicar Ansible de forma local y reproducible.
