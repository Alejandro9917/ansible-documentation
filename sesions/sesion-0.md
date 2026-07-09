# Sesión 0 — Requirements del laboratorio

## Objetivo

Preparar el entorno local necesario para ejecutar los primeros laboratorios de Ansible utilizando contenedores Docker que simulan servidores Ubuntu Server.

En esta primera etapa no se instalarán servicios como Redis ni software adicional del caso de estudio. El objetivo inicial es contar con varios servidores Linux disponibles para practicar instalación de dependencias, validación de conectividad y ejecución de comandos con Ansible.

---

## Enfoque del laboratorio

Para evitar el uso de máquinas virtuales completas, utilizaremos contenedores Docker como pequeños servidores Ubuntu.

```text
Ansible Control Node
        |
        | SSH
        |
+----------------+
| Ubuntu Server  |
| device01       |
+----------------+

+----------------+
| Ubuntu Server  |
| device02       |
+----------------+

+----------------+
| Ubuntu Server  |
| device03       |
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

### Instalar Python 3 y pip

#### Ubuntu o Debian

```bash
sudo apt update
sudo apt install -y python3 python3-pip
```

#### macOS

```bash
brew install python
```

#### Windows con WSL2

Ejecutar dentro de la distribución Linux instalada en WSL2:

```bash
sudo apt update
sudo apt install -y python3 python3-pip
```

### Instalar Ansible

Ansible se instalará con `pip` para mantener una instalación simple y reproducible:

```bash
python3 -m pip install --user ansible
```

Validar que el directorio local de binarios de Python esté disponible en el `PATH`:

```bash
python3 -m site --user-base
```

Si `ansible` no queda disponible después de la instalación, agregar el directorio `bin` del usuario al `PATH`. En Linux o macOS normalmente es:

```bash
export PATH="$HOME/.local/bin:$PATH"
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
| ansible-control | Nodo de control para ejecutar Ansible |
| device01 | Servidor Ubuntu administrado |
| device02 | Servidor Ubuntu administrado |
| device03 | Servidor Ubuntu administrado |

Más adelante se podrán agregar otros servicios como Redis, monitoreo o software agente, pero no forman parte de esta sesión inicial.

---

## Objetivo técnico del laboratorio inicial

Al finalizar esta preparación, se espera contar con:

- Un nodo de control con Ansible instalado.
- Tres contenedores Ubuntu Server accesibles por SSH.
- Un inventario básico de Ansible.
- Conectividad validada entre el nodo de control y los hosts administrados.
- Ambiente listo para probar instalación de dependencias con módulos de Ansible.

---

## Flujo esperado

```text
1. Crear la estructura del laboratorio
2. Crear el archivo docker-compose.yml
3. Levantar los contenedores Ubuntu
4. Validar que los contenedores estén activos
5. Probar conectividad SSH
6. Crear o validar el inventory de Ansible
7. Ejecutar el primer ping de Ansible
```

---

## Docker Compose del laboratorio

Crear el archivo `docker-compose.yml` en la raíz del laboratorio con el siguiente contenido:

```yaml
services:
  ansible-control:
    image: ubuntu:24.04
    container_name: ansible-control
    hostname: ansible-control
    command: >
      bash -lc "
      apt-get update &&
      apt-get install -y ansible openssh-client python3 sudo &&
      useradd -m -s /bin/bash ansible || true &&
      mkdir -p /home/ansible/.ssh /shared &&
      if [ ! -f /shared/id_rsa ]; then ssh-keygen -t rsa -b 4096 -f /shared/id_rsa -N ''; fi &&
      cp /shared/id_rsa /home/ansible/.ssh/id_rsa &&
      cp /shared/id_rsa.pub /home/ansible/.ssh/id_rsa.pub &&
      chown -R ansible:ansible /home/ansible/.ssh &&
      chmod 700 /home/ansible/.ssh &&
      chmod 600 /home/ansible/.ssh/id_rsa &&
      tail -f /dev/null
      "
    volumes:
      - ansible-ssh:/shared
    networks:
      - ansible-lab

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
      until [ -f /shared/id_rsa.pub ]; do sleep 1; done &&
      cp /shared/id_rsa.pub /home/ansible/.ssh/authorized_keys &&
      chown -R ansible:ansible /home/ansible/.ssh &&
      chmod 700 /home/ansible/.ssh &&
      chmod 600 /home/ansible/.ssh/authorized_keys &&
      /usr/sbin/sshd -D
      "
    volumes:
      - ansible-ssh:/shared
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
      until [ -f /shared/id_rsa.pub ]; do sleep 1; done &&
      cp /shared/id_rsa.pub /home/ansible/.ssh/authorized_keys &&
      chown -R ansible:ansible /home/ansible/.ssh &&
      chmod 700 /home/ansible/.ssh &&
      chmod 600 /home/ansible/.ssh/authorized_keys &&
      /usr/sbin/sshd -D
      "
    volumes:
      - ansible-ssh:/shared
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
      until [ -f /shared/id_rsa.pub ]; do sleep 1; done &&
      cp /shared/id_rsa.pub /home/ansible/.ssh/authorized_keys &&
      chown -R ansible:ansible /home/ansible/.ssh &&
      chmod 700 /home/ansible/.ssh &&
      chmod 600 /home/ansible/.ssh/authorized_keys &&
      /usr/sbin/sshd -D
      "
    volumes:
      - ansible-ssh:/shared
    networks:
      - ansible-lab

volumes:
  ansible-ssh:

networks:
  ansible-lab:
    driver: bridge
```

Este archivo crea un nodo de control con Ansible instalado y tres servidores Ubuntu administrados con SSH y Python 3.

---

## Comandos base del laboratorio

### Levantar contenedores

```bash
docker compose up -d
```

### Verificar contenedores activos

```bash
docker ps
```

### Acceder al nodo de control

```bash
docker exec -it ansible-control bash
```

### Validar Ansible desde el nodo de control

```bash
ansible --version
```

### Crear el inventory inicial

Dentro del contenedor `ansible-control`, crear el directorio `inventory`:

```bash
mkdir -p inventory
```

Crear el archivo `inventory/lab.ini` con el siguiente contenido:

```ini
[devices]
device01
device02
device03

[all:vars]
ansible_user=ansible
ansible_ssh_private_key_file=/home/ansible/.ssh/id_rsa
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

### Cómo se configuran las IPs

En este laboratorio no se configurarán IPs fijas manualmente. Docker Compose crea la red `ansible-lab` y asigna una IP privada a cada contenedor dentro de esa red.

Los hosts se declaran en el inventory usando los nombres de los contenedores:

| Host en Ansible | Contenedor Docker | Resolución dentro de la red |
|---|---|---|
| device01 | device01 | Docker DNS |
| device02 | device02 | Docker DNS |
| device03 | device03 | Docker DNS |

Cuando Ansible se ejecuta desde `ansible-control`, los nombres `device01`, `device02` y `device03` se resuelven automáticamente a sus IPs internas porque todos los contenedores están conectados a la misma red Docker.

Para consultar la IP asignada por Docker a un contenedor:

```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' device01
```

No se recomienda depender de esas IPs para el laboratorio, porque pueden cambiar si los contenedores se eliminan y se vuelven a crear. La referencia estable será el nombre del contenedor.

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
- Que todos los contenedores estén en la misma red Docker.

### Error de Python en los hosts

Verificar que Python 3 esté instalado en los contenedores Ubuntu:

```bash
python3 --version
```

Ansible necesita Python en los hosts administrados para ejecutar la mayoría de módulos.

---

## Nota para el curso

Docker se utilizará únicamente como mecanismo para simular servidores Linux. El objetivo de esta capacitación no es profundizar en Docker, sino utilizarlo como base ligera para practicar Ansible de forma local y reproducible.
