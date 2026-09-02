# Sesión 6 — Roles y organización de proyectos

## Objetivo

Reorganizar el laboratorio de las sesiones anteriores mediante **Roles de Ansible**, separando responsabilidades y haciendo reutilizables Tasks, Handlers, Variables, Files y Templates.

Durante la práctica se trabajará con:

- `tasks/`
- `handlers/`
- `files/`
- `templates/`
- `defaults/`
- `vars/`
- Templates Jinja2
- `site.yml`
- `ansible-galaxy role init`

> Ejecutar los comandos desde la raíz de `ansible-cap/`.

---

## 1. Punto de partida

Actualmente tenemos una estructura similar a:

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

En esta sesión moveremos la implementación de `setup_devices.yml` a un Role llamado `device_agent`.

---

## 2. Crear el Role

```bash
mkdir -p roles/device_agent/{tasks,handlers,files,templates,defaults,vars}
```

Crear los archivos principales:

```bash
touch roles/device_agent/tasks/main.yml
touch roles/device_agent/handlers/main.yml
touch roles/device_agent/defaults/main.yml
touch roles/device_agent/vars/main.yml
```

Estructura:

```text
roles/
└── device_agent/
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── files/
    ├── templates/
    ├── defaults/
    │   └── main.yml
    └── vars/
        └── main.yml
```

---

## 3. Definir valores por defecto

Crear `roles/device_agent/defaults/main.yml`:

```yaml
---
agent_user: operador
agent_group: devops
agent_directory: /opt/ansible-demo

agent_packages:
  - curl
  - cowsay

heartbeat_interval: 30
```

Estos valores representan configuraciones que quien consume el Role puede sobrescribir.

---

## 4. Mover las Tasks al Role

Crear `roles/device_agent/tasks/main.yml`:

```yaml
---
- name: Crear grupo del agente
  ansible.builtin.group:
    name: "{{ agent_group }}"
    state: present

- name: Crear usuario del agente
  ansible.builtin.user:
    name: "{{ agent_user }}"
    group: "{{ agent_group }}"
    shell: /bin/bash
    create_home: true
    state: present

- name: Actualizar cache
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600

- name: Instalar dependencias
  ansible.builtin.apt:
    name: "{{ agent_packages }}"
    state: present

- name: Crear directorio del agente
  ansible.builtin.file:
    path: "{{ agent_directory }}"
    state: directory
    owner: "{{ agent_user }}"
    group: "{{ agent_group }}"
    mode: "0755"
```

---

## 5. Consumir el Role

Simplificar `playbooks/setup_devices.yml`:

```yaml
---
- name: Preparar dispositivos
  hosts: devices
  become: true

  roles:
    - device_agent
```

Validar:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml --syntax-check
```

Ejecutar:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml
```

Ejecutar una segunda vez y comparar el `PLAY RECAP`.

---

## 6. Trabajar con `files/`

Si existe:

```text
playbooks/files/app.conf
```

moverlo:

```bash
mv playbooks/files/app.conf roles/device_agent/files/app.conf
```

Agregar a `tasks/main.yml`:

```yaml
- name: Copiar archivo estático
  ansible.builtin.copy:
    src: app.conf
    dest: "{{ agent_directory }}/app.conf"
    owner: "{{ agent_user }}"
    group: "{{ agent_group }}"
    mode: "0644"
```

Dentro del Role, Ansible busca automáticamente `app.conf` en `files/`.

---

## 7. Crear un Template Jinja2

Crear:

```text
roles/device_agent/templates/agent.conf.j2
```

Contenido:

```jinja2
# Archivo generado por Ansible

DEVICE_NAME={{ inventory_hostname }}
DEVICE_USER={{ agent_user }}
DEVICE_GROUP={{ agent_group }}
ENVIRONMENT={{ environment | default('laboratorio') }}
HEARTBEAT_INTERVAL={{ heartbeat_interval }}
```

Agregar a `tasks/main.yml`:

```yaml
- name: Generar configuración del agente
  ansible.builtin.template:
    src: agent.conf.j2
    dest: "{{ agent_directory }}/agent.conf"
    owner: "{{ agent_user }}"
    group: "{{ agent_group }}"
    mode: "0644"
  notify: Registrar cambio de configuración
```

---

## 8. Mover el Handler al Role

Crear `roles/device_agent/handlers/main.yml`:

```yaml
---
- name: Registrar cambio de configuración
  ansible.builtin.command:
    cmd: touch /tmp/agent_configuration_changed
```

Ejecutar:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml
```

Validar la configuración generada:

```bash
ansible devices -i inventory/lab.ini -m command -a "cat /opt/ansible-demo/agent.conf"
```

---

## 9. Probar el Handler

Eliminar el archivo utilizado como evidencia:

```bash
ansible devices -i inventory/lab.ini -b -m file -a "path=/tmp/agent_configuration_changed state=absent"
```

Ejecutar sin cambiar el Template:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml
```

El Handler no debería ejecutarse.

Ahora cambiar en `defaults/main.yml`:

```yaml
heartbeat_interval: 60
```

Ejecutar nuevamente.

El flujo esperado es:

```text
Template cambia
      ↓
    notify
      ↓
   Handler
```

---

## 10. Sobrescribir `defaults`

En `inventory/host_vars/device01.yml` agregar:

```yaml
heartbeat_interval: 10
```

Ejecutar:

```bash
ansible-playbook -i inventory/lab.ini playbooks/setup_devices.yml
```

Comparar:

```bash
ansible device01 -i inventory/lab.ini -m command -a "cat /opt/ansible-demo/agent.conf"
```

```bash
ansible device02 -i inventory/lab.ini -m command -a "cat /opt/ansible-demo/agent.conf"
```

El mismo Role debe generar configuraciones distintas.

---

## 11. `defaults` vs `vars`

Crear `roles/device_agent/vars/main.yml`:

```yaml
---
agent_config_filename: agent.conf
```

Modificar la Task del Template:

```yaml
- name: Generar configuración del agente
  ansible.builtin.template:
    src: agent.conf.j2
    dest: "{{ agent_directory }}/{{ agent_config_filename }}"
    owner: "{{ agent_user }}"
    group: "{{ agent_group }}"
    mode: "0644"
  notify: Registrar cambio de configuración
```

Regla práctica:

```text
defaults/
├── Valores configurables.
├── Valores que pueden cambiar por ambiente.
└── Prioridad baja.

vars/
├── Variables internas del Role.
├── Valores que normalmente no se modifican.
└── Mayor prioridad que defaults.
```

---

## 12. Ejecutar desde `site.yml`

`playbooks/site.yml`:

```yaml
---
- import_playbook: setup_devices.yml
```

Ejecutar:

```bash
ansible-playbook -i inventory/lab.ini playbooks/site.yml
```

La organización queda:

```text
site.yml
   │
   ▼
setup_devices.yml
   │
   ▼
device_agent
   │
   ├── defaults
   ├── vars
   ├── tasks
   ├── templates
   ├── files
   └── handlers
```

---

## 13. Preparar los siguientes Roles

Crear las estructuras que posteriormente utilizará el caso de estudio:

```bash
mkdir -p roles/redis/{tasks,handlers,files,templates,defaults,vars}
mkdir -p roles/monitoring/{tasks,handlers,files,templates,defaults,vars}
```

No es necesario implementar todavía Redis o Monitoring.

La estructura objetivo será:

```text
roles/
├── device_agent/
├── redis/
└── monitoring/
```

---

## 14. Crear un Role con Ansible Galaxy

Después de construir el primer Role manualmente, comparar con el generador de Ansible:

```bash
ansible-galaxy role init roles/demo_role
```

Visualizar:

```bash
tree roles/demo_role
```

Después de revisar:

```bash
rm -rf roles/demo_role
```

---

# Ejercicio integrador

Reorganizar completamente la automatización de los dispositivos.

El resultado debe cumplir:

1. `setup_devices.yml` consume `device_agent`.
2. Las Tasks están en `roles/device_agent/tasks/main.yml`.
3. Los Handlers están en `handlers/main.yml`.
4. Los valores configurables están en `defaults/main.yml`.
5. Las variables internas están en `vars/main.yml`.
6. La configuración se genera con un Template Jinja2.
7. `device01` utiliza un `heartbeat_interval` diferente.
8. Un cambio en el Template ejecuta el Handler.
9. Una segunda ejecución sin cambios demuestra idempotencia.
10. `site.yml` continúa siendo el punto de entrada.

Ejecutar:

```bash
ansible-playbook -i inventory/lab.ini playbooks/site.yml
```

---

# Reto

Agregar en `defaults/main.yml`:

```yaml
agent_log_level: INFO
```

Agregar a `agent.conf.j2`:

```jinja2
LOG_LEVEL={{ agent_log_level }}
```

Configurar solamente `device03` con:

```yaml
agent_log_level: DEBUG
```

Ejecutar el proyecto y validar:

```bash
ansible devices -i inventory/lab.ini -m command -a "cat /opt/ansible-demo/agent.conf"
```

El mismo Role debe producir configuraciones distintas dependiendo de las variables de cada host.

---

## Estructura final esperada

```text
ansible-cap/
├── docker/
├── inventory/
│   ├── lab.ini
│   ├── group_vars/
│   └── host_vars/
├── playbooks/
│   ├── setup_devices.yml
│   └── site.yml
├── roles/
│   ├── device_agent/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── handlers/
│   │   │   └── main.yml
│   │   ├── files/
│   │   ├── templates/
│   │   │   └── agent.conf.j2
│   │   ├── defaults/
│   │   │   └── main.yml
│   │   └── vars/
│   │       └── main.yml
│   ├── redis/
│   └── monitoring/
└── ssh/
```

---

## Preguntas de cierre

1. ¿Qué es un Role?
2. ¿Qué problema resuelven los Roles?
3. ¿Dónde se almacenan las Tasks?
4. ¿Dónde se almacenan los Handlers?
5. ¿Cuál es la diferencia entre `files/` y `templates/`?
6. ¿Cuál es la diferencia entre `defaults/` y `vars/`?
7. ¿Qué ventaja ofrece Jinja2?
8. ¿Cómo puede el mismo Role producir configuraciones distintas?
9. ¿Qué función cumple `site.yml`?
10. ¿Qué ventajas ofrece reutilizar Roles?

---

## Resultado esperado

Al finalizar, el proyecto habrá evolucionado de Playbooks con múltiples responsabilidades a una estructura modular basada en Roles, preparada para incorporar posteriormente los Roles `redis` y `monitoring`.
