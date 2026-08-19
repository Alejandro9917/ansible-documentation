
# Sesión 2 — Primeros pasos con Ansible

## Objetivo

Al finalizar esta sesión el estudiante será capaz de:

- Comprender la arquitectura de Ansible.
- Configurar un inventario básico.
- Entender cómo Ansible establece conexión con los dispositivos.
- Ejecutar comandos Ad-Hoc.
- Administrar archivos y usuarios.
- Instalar software en múltiples servidores.
- Comprender el concepto de idempotencia.

---

# Arquitectura del laboratorio

```text
                 Host (Ansible Control)

                         │
                     Inventario
                         │
                     Conexión SSH
                         │
        ┌────────────┬────────────┬────────────┐
        │            │            │
    device01     device02     device03
```

Todos los dispositivos corresponden a servidores Ubuntu ejecutándose en Docker.

---

# ¿Cómo trabaja Ansible?

```text
Administrador
      │
ansible all -i inventory/lab.ini -m command
      │
Lee el Inventory
      │
Establece conexión SSH
      │
Copia temporalmente el módulo Python
      │
Ejecuta el módulo remoto
      │
Obtiene el resultado
      │
Elimina archivos temporales
      │
Muestra la salida
```

Aunque solo ejecutamos un comando, Ansible realiza automáticamente todos estos pasos.

---

# Crear el Inventory

```ini
[devices]
device01 ansible_host=127.0.0.1 ansible_port=2201
device02 ansible_host=127.0.0.1 ansible_port=2202
device03 ansible_host=127.0.0.1 ansible_port=2203

[devices:vars]
ansible_user=ansible
ansible_ssh_private_key_file=ssh/ansible_lab
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

## Explicación de las variables

- **ansible_user**: usuario utilizado para conectarse por SSH.
- **ansible_ssh_private_key_file**: llave privada usada para autenticarse.
- **ansible_python_interpreter**: ruta de Python en el servidor remoto; la mayoría de módulos dependen de él.
- **ansible_ssh_common_args**: argumentos adicionales para SSH. En el laboratorio deshabilitamos la validación interactiva de la huella del servidor.

---

# Validar el Inventory

```bash
ansible-inventory -i inventory/lab.ini --list
ansible-inventory -i inventory/lab.ini --graph
ansible all -i inventory/lab.ini -m ping
```

---

# Módulo: command

```bash
ansible all -i inventory/lab.ini -m command -a "hostname"
```

## ¿Qué hace?

Ejecuta un comando directamente en el sistema remoto sin utilizar un shell.

### Flujo

```text
Host → SSH → command → Servidor → Resultado
```

### ¿Qué ocurre detrás de escena?

1. Lee el inventario.
2. Abre la conexión SSH.
3. Copia el módulo `command`.
4. Ejecuta el comando.
5. Devuelve la salida y elimina los archivos temporales.

Otros ejemplos:

```bash
ansible all -i inventory/lab.ini -m command -a "date"
ansible all -i inventory/lab.ini -m command -a "uname -a"
ansible all -i inventory/lab.ini -m command -a "free -h"
ansible all -i inventory/lab.ini -m command -a "lscpu"
ansible all -i inventory/lab.ini -m command -a "df -h"
ansible all -i inventory/lab.ini -m command -a "hostname -I"
```

---

# Módulo: file

```bash
ansible all -i inventory/lab.ini -b -m file -a "path=/tmp/demo state=directory"
```

## Explicación de los parámetros

| Parámetro | Función |
|---|---|
| `ansible` | Ejecuta la herramienta de línea de comandos de Ansible. |
| `all` | Indica que el comando se ejecutará sobre todos los hosts definidos en el inventario. |
| `-i inventory/lab.ini` | Define el archivo de inventario que contiene los hosts administrados y sus variables de conexión. |
| `-b` | Activa `become`, es decir, ejecuta la tarea con privilegios elevados en el host remoto. |
| `-m file` | Indica que se usará el módulo `file`, utilizado para administrar archivos, directorios y enlaces. |
| `-a "..."` | Envía argumentos al módulo seleccionado. |
| `path=/tmp/demo` | Define la ruta que se desea administrar en los hosts remotos. |
| `state=directory` | Declara el estado deseado: la ruta debe existir como directorio. |

En conjunto, este comando indica a Ansible que cree el directorio `/tmp/demo` en todos los devices del inventario. Si el directorio ya existe, Ansible no lo vuelve a crear y reporta `ok`.

## ¿Qué hace?

Permite crear, eliminar o modificar archivos y directorios.

### Flujo

```text
Host → SSH → file → Sistema de archivos Linux
```

### ¿Qué ocurre detrás de escena?

El módulo verifica el estado actual antes de actuar.

- Si el directorio existe: `ok`
- Si debe crearlo: `changed`

---

# Módulo: copy

Crear un archivo local en la máquina host:

```bash
echo "Mensaje creado desde Ansible" > mensaje.txt
```

```bash
ansible all -i inventory/lab.ini -m copy -a "src=mensaje.txt dest=/tmp/demo/"
```

## ¿Qué hace?

Copia un archivo desde el nodo de control hacia todos los servidores.

### Flujo

```text
Archivo local → SSH → copy → Servidores
```

### ¿Qué ocurre detrás de escena?

- Comprueba que el archivo exista.
- Calcula su checksum.
- Solo lo reemplaza si el contenido cambió.

---

# Módulo: apt

```bash
ansible all -i inventory/lab.ini -b -m apt -a "name=tree state=present"
```

## ¿Qué hace?

Instala, actualiza o elimina paquetes en distribuciones Debian/Ubuntu.

### Flujo

```text
Host → SSH → apt → Gestor de paquetes → Repositorios
```

### ¿Qué ocurre detrás de escena?

Antes de instalar, verifica si el paquete ya existe.

```bash
ansible all -i inventory/lab.ini -b -m apt -a "update_cache=yes"
ansible all -i inventory/lab.ini -b -m apt -a "name=tree state=present"
ansible all -i inventory/lab.ini -b -m apt -a "name=curl state=present"
ansible all -i inventory/lab.ini -b -m apt -a "name=htop state=present"
ansible all -i inventory/lab.ini -b -m apt -a "name=cowsay state=present"
ansible all -i inventory/lab.ini -m shell -a "/usr/games/cowsay Bienvenidos al curso"
```

---

# Módulo: user

```bash
ansible all -i inventory/lab.ini -b -m user -a "name=devops state=present"
```

## ¿Qué hace?

Administra usuarios del sistema operativo.

### Flujo

```text
Host → SSH → user → Base de usuarios Linux
```

### ¿Qué ocurre detrás de escena?

Comprueba si el usuario existe antes de crearlo o eliminarlo.

```bash
ansible all -i inventory/lab.ini -b -m user -a "name=devops state=present"
ansible all -i inventory/lab.ini -b -m user -a "name=devops state=absent remove=yes"
```

---

# ¿Cómo sabe Ansible si debe realizar cambios?

Cada módulo compara el **estado actual** con el **estado deseado**.

| Módulo | ¿Qué verifica? |
|--------|----------------|
| apt | Si el paquete está instalado |
| copy | Si cambió el contenido del archivo |
| file | Si existe el archivo o directorio |
| user | Si el usuario existe |

Solo cuando ambos estados son diferentes, Ansible realiza modificaciones.

---

# Idempotencia

Ejecutar la instalación de `tree` una primera vez:

```bash
ansible all -i inventory/lab.ini -b -m apt -a "name=tree state=present"
```

Ejecutar el mismo comando una segunda vez:

```bash
ansible all -i inventory/lab.ini -b -m apt -a "name=tree state=present"
```

En la primera ejecución Ansible puede reportar `changed`, porque instala el paquete si no existe. En la segunda ejecución debe reportar `ok`, porque el paquete ya está instalado y el estado deseado ya se cumple.

Para observar el cambio de estado nuevamente, remover el paquete y volver a instalarlo:

```bash
ansible all -i inventory/lab.ini -b -m apt -a "name=tree state=absent"
ansible all -i inventory/lab.ini -b -m apt -a "name=tree state=present"
ansible all -i inventory/lab.ini -b -m apt -a "name=tree state=present"
```

Resultado esperado:

```text
Primera instalación: changed
Segunda instalación: ok
Remoción: changed
Instalación después de remover: changed
Nueva ejecución después de instalar: ok
```

---

# Resumen de módulos

| Módulo | Propósito |
|---------|-----------|
| ping | Verificar conectividad |
| command | Ejecutar comandos |
| shell | Ejecutar comandos usando un shell |
| file | Administrar archivos y directorios |
| copy | Copiar archivos |
| apt | Administrar paquetes |
| user | Administrar usuarios |
