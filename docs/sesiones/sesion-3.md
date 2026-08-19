# Sesión 3 — Inventarios y variables en Ansible

## Objetivo de la sesión

Integrar al proyecto del laboratorio los ejercicios relacionados con inventarios estáticos, grupos de hosts, variables de conexión, variables por grupo, variables por host y validación del inventario.

Al finalizar, el proyecto permitirá administrar los dispositivos del laboratorio sin repetir los parámetros de conexión en cada comando.

---

## 1. Estructura esperada del proyecto

```text
ansible-cap/
├── docker/
│   └── docker-compose.yml
├── inventory/
│   ├── lab.ini
│   ├── group_vars/
│   │   └── devices.yml
│   └── host_vars/
│       ├── device01.yml
│       ├── device02.yml
│       └── device03.yml
├── ssh/
│   ├── ansible_lab
│   └── ansible_lab.pub
├── mensaje.txt
├── sesion-1.md
├── sesion-2.md
└── sesion-3.md
```

Crear las carpetas necesarias:

```bash
mkdir -p inventory/group_vars
mkdir -p inventory/host_vars
```

---

## 2. Crear el inventario principal

Crear el archivo `inventory/lab.ini`:

```ini
[devices]
device01 ansible_host=127.0.0.1 ansible_port=2201
device02 ansible_host=127.0.0.1 ansible_port=2202
device03 ansible_host=127.0.0.1 ansible_port=2203
```

> Los puertos deben coincidir con los publicados en `docker/docker-compose.yml`.

Validar el inventario:

```bash
ansible-inventory -i inventory/lab.ini --graph
```

Resultado esperado:

```text
@all:
  |--@ungrouped:
  |--@devices:
  |  |--device01
  |  |--device02
  |  |--device03
```

---

## 3. Agregar variables de conexión al grupo

Crear el archivo `inventory/group_vars/devices.yml`:

```yaml
ansible_user: ansible
ansible_ssh_private_key_file: ssh/ansible_lab
ansible_python_interpreter: /usr/bin/python3
ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
```

Estas variables se aplicarán automáticamente a todos los hosts del grupo `devices`.

> La ruta de la llave privada se interpreta desde la raíz del proyecto. Ejecuta los comandos desde `ansible-cap/`.

---

## 4. Probar la conectividad

```bash
ansible devices -i inventory/lab.ini -m ping
```

Resultado esperado:

```text
device01 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Si ocurre un error, ejecutar la prueba en modo detallado:

```bash
ansible devices -i inventory/lab.ini -m ping -vvv
```

---

## 5. Consultar las variables del inventario

Mostrar las variables aplicadas a un host:

```bash
ansible-inventory -i inventory/lab.ini --host device01
```

Mostrar todo el inventario en formato YAML:

```bash
ansible-inventory -i inventory/lab.ini --list --yaml
```

---

## 6. Crear variables específicas por host

Crear `inventory/host_vars/device01.yml`:

```yaml
device_name: dispositivo_principal
device_role: pruebas
environment: laboratorio
```

Crear `inventory/host_vars/device02.yml`:

```yaml
device_name: dispositivo_secundario
device_role: desarrollo
environment: laboratorio
```

Crear `inventory/host_vars/device03.yml`:

```yaml
device_name: dispositivo_respaldo
device_role: validacion
environment: laboratorio
```

Validar las variables:

```bash
ansible-inventory -i inventory/lab.ini --host device02
```

---

## 7. Utilizar variables con el módulo `debug`

Mostrar una variable específica:

```bash
ansible devices \
  -i inventory/lab.ini \
  -m debug \
  -a "var=device_name"
```

Mostrar un mensaje construido con varias variables:

```bash
ansible devices \
  -i inventory/lab.ini \
  -m debug \
  -a 'msg="El host {{ inventory_hostname }} se llama {{ device_name }}, tiene el rol {{ device_role }} y pertenece al ambiente {{ environment }}"'
```

---

## 8. Ejercicio: crear un archivo usando variables

Crear un archivo personalizado en cada dispositivo:

```bash
ansible devices \
  -i inventory/lab.ini \
  -b \
  -m copy \
  -a 'content="Host: {{ inventory_hostname }}
Nombre: {{ device_name }}
Rol: {{ device_role }}
Ambiente: {{ environment }}
" dest=/tmp/informacion_dispositivo.txt owner=ansible group=ansible mode=0644'
```

Validar el contenido:

```bash
ansible devices \
  -i inventory/lab.ini \
  -m command \
  -a "cat /tmp/informacion_dispositivo.txt"
```

---

## 9. Agregar grupos de hosts

Modificar `inventory/lab.ini`:

```ini
[devices]
device01 ansible_host=127.0.0.1 ansible_port=2201
device02 ansible_host=127.0.0.1 ansible_port=2202
device03 ansible_host=127.0.0.1 ansible_port=2203

[production]
device01

[development]
device02
device03
```

Validar la estructura:

```bash
ansible-inventory -i inventory/lab.ini --graph
```

Probar únicamente el grupo `development`:

```bash
ansible development -i inventory/lab.ini -m ping
```

Ejecutar un comando sobre `production`:

```bash
ansible production -i inventory/lab.ini -m command -a "hostname"
```

---

## 10. Usar patrones del inventario

Ejecutar sobre todos los dispositivos:

```bash
ansible devices -i inventory/lab.ini -m command -a "hostname"
```

Ejecutar sobre un host:

```bash
ansible device01 -i inventory/lab.ini -m command -a "hostname"
```

Ejecutar sobre dos hosts:

```bash
ansible 'device01:device02' -i inventory/lab.ini -m ping
```

Excluir un host:

```bash
ansible 'devices:!device03' -i inventory/lab.ini -m ping
```

---

## 11. Crear un grupo con grupos hijos

Dejar el inventario de la siguiente manera:

```ini
[production]
device01 ansible_host=127.0.0.1 ansible_port=2201

[development]
device02 ansible_host=127.0.0.1 ansible_port=2202
device03 ansible_host=127.0.0.1 ansible_port=2203

[devices:children]
production
development
```

Validar:

```bash
ansible-inventory -i inventory/lab.ini --graph
```

Las variables comunes continuarán en:

```text
inventory/group_vars/devices.yml
```

---

## 12. Validación final

```bash
ansible-inventory -i inventory/lab.ini --graph
```

```bash
ansible devices -i inventory/lab.ini -m ping
```

```bash
ansible devices -i inventory/lab.ini -m debug -a "var=device_name"
```

```bash
ansible development -i inventory/lab.ini -m command -a "hostname"
```

```bash
ansible production -i inventory/lab.ini -m command -a "hostname"
```

---

## 13. Resultado esperado

Al finalizar la sesión, el proyecto debe permitir:

- Administrar los tres dispositivos desde un solo inventario.
- Separar hosts por grupos.
- Reutilizar variables de conexión.
- Definir variables específicas para cada dispositivo.
- Ejecutar comandos por host, grupo o patrón.
- Consultar las variables efectivas de cada host.
- Crear archivos personalizados con información del inventario.

---

## 14. Preguntas de cierre

1. ¿Qué diferencia existe entre una variable de grupo y una variable de host?
2. ¿Qué sucede si una variable está definida en ambos lugares?
3. ¿Qué ventaja ofrece separar las variables del archivo `lab.ini`?
4. ¿Cuándo sería conveniente utilizar un inventario dinámico?
5. ¿Qué patrón permite ejecutar una tarea en todos los hosts excepto uno?
