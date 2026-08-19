# Sesión 5 — Playbooks

## Objetivo

Convertir las tareas ejecutadas con comandos ad hoc en Playbooks organizados, declarativos y reutilizables.

La sesión cubre Playbooks, Plays, Tasks, variables, `notify`, Handlers, `--limit` y la orquestación mediante `site.yml`.

> En esta etapa se implementará `setup_devices.yml`. Los Playbooks de Redis, agente y monitoreo quedarán preparados en la estructura para incorporarlos cuando esos componentes estén disponibles.

---

## 1. Preparar la estructura

```bash
mkdir -p playbooks/files
```

```text
ansible-cap/
├── docker/
├── inventory/
├── playbooks/
│   ├── files/
│   ├── setup_devices.yml
│   └── site.yml
└── ssh/
```

---

## 2. Primer Playbook

Crear `playbooks/setup_devices.yml`:

```yaml
---
- name: Preparar dispositivos
  hosts: devices
  become: true

  tasks:
    - name: Crear grupo DevOps
      ansible.builtin.group:
        name: devops
        state: present

    - name: Crear usuario operador
      ansible.builtin.user:
        name: operador
        group: devops
        shell: /bin/bash
        create_home: true
        state: present

    - name: Crear directorio
      ansible.builtin.file:
        path: /opt/ansible-demo
        state: directory
        owner: operador
        group: devops
        mode: "0755"
```

Validar:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml --syntax-check
```

Ejecutar:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml
```

Ejecutarlo una segunda vez y comparar el `PLAY RECAP`.

---

## 3. Entender la estructura de una Task

En un Playbook, `tasks:` contiene la lista de acciones que Ansible ejecutará sobre los hosts definidos en `hosts`.

Ejemplo:

```yaml
  tasks:
    - name: Crear directorio
      ansible.builtin.file:
        path: /opt/ansible-demo
        state: directory
        owner: operador
        group: devops
        mode: "0755"
```

Cada bloque dentro de `tasks:` representa una tarea.

| Elemento | Significado |
|---|---|
| `tasks:` | Lista de tareas que se ejecutarán dentro del Play. |
| `- name:` | Nombre descriptivo de la tarea. Aparece en la salida de Ansible y ayuda a entender qué está ocurriendo. |
| `ansible.builtin.file:` | Módulo que ejecutará la acción. En este caso administra archivos, directorios y permisos. |
| `path:` | Ruta del archivo o directorio que se desea administrar. |
| `state:` | Estado deseado del recurso. Por ejemplo: `directory`, `file`, `absent` o `present`, según el módulo. |
| `owner:` | Usuario propietario que debe tener el archivo o directorio. |
| `group:` | Grupo propietario que debe tener el archivo o directorio. |
| `mode:` | Permisos esperados en formato octal. Se escribe entre comillas para evitar interpretaciones incorrectas de YAML. |

### Qué significa `ansible.builtin`

`ansible.builtin` es la colección oficial incluida con Ansible. Al escribir el nombre completo del módulo, por ejemplo `ansible.builtin.apt`, se indica explícitamente que se usará el módulo `apt` incluido en Ansible.

Esto evita ambigüedades cuando existen módulos con nombres similares en otras colecciones.

| Módulo | Propósito |
|---|---|
| `ansible.builtin.group` | Crea, modifica o elimina grupos del sistema operativo. |
| `ansible.builtin.user` | Crea, modifica o elimina usuarios del sistema operativo. |
| `ansible.builtin.file` | Administra archivos, directorios, enlaces, propietarios y permisos. |
| `ansible.builtin.copy` | Copia archivos o contenido desde el nodo de control hacia los hosts administrados. |
| `ansible.builtin.apt` | Administra paquetes en sistemas basados en Debian o Ubuntu usando APT. |
| `ansible.builtin.command` | Ejecuta comandos directamente en el host remoto sin pasar por un shell. |

### Ejemplo con `ansible.builtin.apt`

```yaml
    - name: Instalar dependencias
      ansible.builtin.apt:
        name:
          - curl
          - cowsay
        state: present
```

En esta tarea:

| Elemento | Significado |
|---|---|
| `- name: Instalar dependencias` | Describe la tarea que aparecerá en la salida del Playbook. |
| `ansible.builtin.apt:` | Usa el módulo `apt` para administrar paquetes en Ubuntu. |
| `name:` | Lista de paquetes que deben administrarse. |
| `curl` y `cowsay` | Paquetes que se instalarán en los dispositivos. |
| `state: present` | Indica que los paquetes deben estar instalados. Si ya existen, Ansible reportará `ok`. |

La idea central es que una task no describe necesariamente un comando, sino un estado deseado. Ansible compara el estado actual del host con ese estado deseado y solo realiza cambios cuando es necesario.

---

## 4. Instalar dependencias

Agregar a `tasks`:

```yaml
    - name: Actualizar cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Instalar dependencias
      ansible.builtin.apt:
        name:
          - curl
          - cowsay
        state: present
```

Ejecutar dos veces para comprobar la idempotencia:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml
```

---

## 5. Reutilizar variables

Agregar al Play:

```yaml
  vars:
    app_directory: /opt/ansible-demo
    app_user: operador
    app_group: devops
```

Utilizarlas:

```yaml
    - name: Crear grupo
      ansible.builtin.group:
        name: "{{ app_group }}"
        state: present

    - name: Crear usuario
      ansible.builtin.user:
        name: "{{ app_user }}"
        group: "{{ app_group }}"
        shell: /bin/bash
        create_home: true
        state: present

    - name: Crear directorio
      ansible.builtin.file:
        path: "{{ app_directory }}"
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0755"
```

---

## 6. Copiar una configuración

Crear:

```bash
echo "Configuración administrada por Ansible" > playbooks/files/app.conf
```

Agregar:

```yaml
    - name: Copiar configuración
      ansible.builtin.copy:
        src: files/app.conf
        dest: "{{ app_directory }}/app.conf"
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0644"
```

Ejecutar:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml
```

---

## 7. `notify` y Handlers

Un **Handler** es una tarea especial que se ejecuta únicamente cuando otra tarea lo notifica mediante `notify`.

Se utiliza para acciones que no deben ejecutarse siempre, sino solo cuando hubo un cambio real. Por ejemplo:

- Reiniciar un servicio cuando cambió su archivo de configuración.
- Recargar una aplicación después de desplegar una nueva versión.
- Registrar que una configuración fue modificada.

La diferencia principal entre una task normal y un handler es el momento de ejecución:

| Elemento | Comportamiento |
|---|---|
| Task normal | Se ejecuta cuando Ansible llega a esa tarea. |
| Handler | Se ejecuta al final del Play, pero solo si fue notificado por una task que produjo `changed`. |

`notify` es la instrucción que conecta una task con un handler. Si la task termina en `ok`, el handler no se ejecuta. Si la task termina en `changed`, Ansible deja el handler pendiente y lo ejecuta al final del Play.

En esta sesión no reiniciaremos un servicio real. Para observar el comportamiento, el handler creará el archivo `/tmp/configuration_changed` cuando el archivo de configuración cambie.

Modificar la tarea anterior:

```yaml
    - name: Copiar configuración
      ansible.builtin.copy:
        src: files/app.conf
        dest: "{{ app_directory }}/app.conf"
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0644"
      notify: Registrar cambio de configuración
```

Agregar al final del Play:

```yaml
  handlers:
    - name: Registrar cambio de configuración
      ansible.builtin.command:
        cmd: touch /tmp/configuration_changed
```

Ejecutar:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml
```

Validar:

```bash
ansible devices -i inventory/lab.ini -m command -a "ls -l /tmp/configuration_changed"
```

---

## 8. Comprobar cuándo se ejecuta un Handler

Eliminar el archivo de prueba:

```bash
ansible devices -i inventory/lab.ini -b -m file -a "path=/tmp/configuration_changed state=absent"
```

Ejecutar nuevamente el Playbook sin modificar `app.conf`:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml
```

El Handler no debería ejecutarse porque `copy` no produjo cambios.

Ahora modificar la configuración:

```bash
echo "Configuración modificada" > playbooks/files/app.conf
```

Ejecutar nuevamente:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml
```

Observar `changed` y posteriormente `RUNNING HANDLER`.

---

## 9. Ejecutar con `--limit`

Solo `device01`:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml --limit device01
```

Solo el grupo `development`:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml --limit development
```

Dos hosts:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml --limit 'device01:device02'
```

`--limit` permite probar, mantener o desplegar gradualmente sin ejecutar el Playbook sobre todo el Inventory.

---

## 10. Ejercicio de despliegue gradual

Modificar:

```bash
echo "Nueva versión de configuración" > playbooks/files/app.conf
```

Desplegar primero en un dispositivo:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml --limit device01
```

Validar:

```bash
ansible device01 -i inventory/lab.ini -m command -a "cat /opt/ansible-demo/app.conf"
```

Después desplegar sobre todos:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml
```

---

## 11. Crear `site.yml`

Crear `playbooks/site.yml`:

```yaml
---
- import_playbook: setup_devices.yml
```

Ejecutar:

```bash
ansible-playbook -i inventory/lab.ini playbooks/site.yml
```

`site.yml` funcionará como punto de entrada del laboratorio.

---

## 12. Estructura objetivo del caso de estudio

```text
playbooks/
├── files/
│   └── app.conf
├── setup_devices.yml
├── setup_redis.yml
├── setup_monitoring.yml
└── site.yml
```

Posteriormente `site.yml` podrá evolucionar a:

```yaml
---
- import_playbook: setup_devices.yml
- import_playbook: setup_redis.yml
- import_playbook: setup_monitoring.yml
```

---

## 13. Playbook final de la sesión

```yaml
---
- name: Preparar dispositivos
  hosts: devices
  become: true

  vars:
    app_directory: /opt/ansible-demo
    app_user: operador
    app_group: devops

  tasks:
    - name: Crear grupo
      ansible.builtin.group:
        name: "{{ app_group }}"
        state: present

    - name: Crear usuario
      ansible.builtin.user:
        name: "{{ app_user }}"
        group: "{{ app_group }}"
        shell: /bin/bash
        create_home: true
        state: present

    - name: Actualizar cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Instalar dependencias
      ansible.builtin.apt:
        name:
          - curl
          - cowsay
        state: present

    - name: Crear directorio
      ansible.builtin.file:
        path: "{{ app_directory }}"
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0755"

    - name: Copiar configuración
      ansible.builtin.copy:
        src: files/app.conf
        dest: "{{ app_directory }}/app.conf"
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0644"
      notify: Registrar cambio de configuración

  handlers:
    - name: Registrar cambio de configuración
      ansible.builtin.command:
        cmd: touch /tmp/configuration_changed
```

---

# Ejercicio integrador

1. Ejecutar `site.yml`.
2. Preparar los dispositivos.
3. Validar usuario, grupo, paquetes y directorio.
4. Ejecutar nuevamente y comprobar idempotencia.
5. Modificar `app.conf`.
6. Desplegar únicamente en `device01` mediante `--limit`.
7. Comprobar la ejecución del Handler.
8. Validar el cambio.
9. Desplegar finalmente sobre todos los dispositivos.

### Validaciones

```bash
ansible devices -i inventory/lab.ini -m command -a "id operador"
```

```bash
ansible devices -i inventory/lab.ini -m command -a "ls -la /opt/ansible-demo"
```

```bash
ansible devices -i inventory/lab.ini -m command -a "cat /opt/ansible-demo/app.conf"
```

---

## Preguntas de cierre

1. ¿Qué es un Playbook?
2. ¿Cuál es la diferencia entre Play y Task?
3. ¿Qué ventaja aporta reutilizar variables?
4. ¿Para qué sirve `notify`?
5. ¿Qué hace un Handler?
6. ¿Cuándo se ejecuta un Handler?
7. ¿Qué ocurre si la Task que contiene `notify` retorna `ok`?
8. ¿Para qué sirve `--limit`?
9. ¿Cómo podemos utilizar `--limit` para un despliegue gradual?
10. ¿Qué función cumple `site.yml`?

---

## Resultado esperado

Al finalizar, el participante será capaz de construir Playbooks, reutilizar variables, utilizar Handlers, ejecutar despliegues parciales con `--limit` y utilizar `site.yml` como punto de entrada para la futura orquestación completa del laboratorio.
