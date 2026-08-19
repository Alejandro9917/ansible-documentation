# Sesión 4 — Módulos y ejecución de tareas

## Objetivo

Practicar cómo Ansible ejecuta tareas mediante módulos sobre los dispositivos del laboratorio.

En esta sesión se trabajará con `ping`, `setup`, `group`, `user`, `file`, `copy`, `apt`, `package`, `service`, `command` y `shell`, prestando especial atención a la **idempotencia** y a la interpretación de resultados.

> Ejecutar los comandos desde la raíz del proyecto.

---

## 1. Validar el laboratorio

```bash
docker ps
ansible-inventory -i inventory/lab.ini --graph
ansible devices -i inventory/lab.ini -m ping
```

---

## 2. Consultar información con `setup`

```bash
ansible devices -i inventory/lab.ini -m setup
```

Filtrar información de la distribución:

```bash
ansible devices -i inventory/lab.ini -m setup -a "filter=ansible_distribution*"
```

Consultar memoria:

```bash
ansible devices -i inventory/lab.ini -m setup -a "filter=ansible_memtotal_mb"
```

---

## 3. Crear un grupo

```bash
ansible devices -i inventory/lab.ini -b -m group -a "name=devops state=present"
```

Ejecutar nuevamente el mismo comando y comparar el resultado.

---

## 4. Crear un usuario

```bash
ansible devices -i inventory/lab.ini -b -m user -a "name=operador group=devops shell=/bin/bash state=present create_home=yes"
```

Validar:

```bash
ansible devices -i inventory/lab.ini -m command -a "id operador"
```

Volver a ejecutar la creación del usuario y observar si Ansible reporta cambios.

---

## 5. Administrar directorios con `file`

```bash
ansible devices -i inventory/lab.ini -b -m file -a "path=/opt/ansible-demo state=directory owner=operador group=devops mode=0755"
```

Validar:

```bash
ansible devices -i inventory/lab.ini -m command -a "ls -ld /opt/ansible-demo"
```

---

## 6. Copiar archivos con `copy`

Crear un archivo en el nodo de control:

```bash
echo "Archivo administrado por Ansible" > mensaje.txt
```

Copiarlo:

```bash
ansible devices -i inventory/lab.ini -b -m copy -a "src=mensaje.txt dest=/opt/ansible-demo/mensaje.txt owner=operador group=devops mode=0644"
```

Validar:

```bash
ansible devices -i inventory/lab.ini -m command -a "cat /opt/ansible-demo/mensaje.txt"
```

Ejecutar nuevamente `copy` y observar la idempotencia.

Modificar el archivo local:

```bash
echo "Contenido actualizado desde Ansible" > mensaje.txt
```

Volver a copiarlo y observar qué hosts reportan `changed`.

---

## 7. Instalar paquetes con `apt`

Actualizar el índice:

```bash
ansible devices -i inventory/lab.ini -b -m apt -a "update_cache=yes"
```

Instalar `cowsay`:

```bash
ansible devices -i inventory/lab.ini -b -m apt -a "name=cowsay state=present"
```

Validar:

```bash
ansible devices -i inventory/lab.ini -m command -a "/usr/games/cowsay Ansible"
```

Ejecutar nuevamente la instalación y comparar el resultado.

---

## 8. Utilizar `package`

```bash
ansible devices -i inventory/lab.ini -b -m package -a "name=curl state=present"
```

Validar:

```bash
ansible devices -i inventory/lab.ini -m command -a "curl --version"
```

---

## 9. Administrar SSH con `service`

```bash
ansible devices -i inventory/lab.ini -b -m service -a "name=ssh state=started"
```

Ejecutar nuevamente y observar el resultado.

> Los contenedores Ubuntu del laboratorio pueden no utilizar `systemd`. Por ello, `systemctl` no es requisito para completar este ejercicio.

---

## 10. Módulo `command`

```bash
ansible devices -i inventory/lab.ini -m command -a "hostname"
```

```bash
ansible devices -i inventory/lab.ini -m command -a "uptime"
```

`command` ejecuta directamente el programa indicado sin pasar por un shell.

---

## 11. Módulo `shell`

Ejecutar una tubería:

```bash
ansible devices -i inventory/lab.ini -m shell -a "ps aux | grep sshd"
```

Utilizar una redirección:

```bash
ansible devices -i inventory/lab.ini -b -m shell -a "echo 'Creado mediante shell' > /tmp/shell-demo.txt"
```

Validar:

```bash
ansible devices -i inventory/lab.ini -m command -a "cat /tmp/shell-demo.txt"
```

Utilizar `shell` cuando se necesiten características como:

```text
|   >   >>   &&   *
```

En caso contrario, preferir `command` o un módulo específico.

---

## 12. Ejercicio de idempotencia: `command` vs `file`

Crear un directorio utilizando `command`:

```bash
ansible devices -i inventory/lab.ini -b -m command -a "mkdir /tmp/demo-command"
```

Ejecutar el mismo comando una segunda vez.

El comando puede fallar porque el directorio ya existe.

Eliminarlo:

```bash
ansible devices -i inventory/lab.ini -b -m file -a "path=/tmp/demo-command state=absent"
```

Ahora crearlo con `file`:

```bash
ansible devices -i inventory/lab.ini -b -m file -a "path=/tmp/demo-command state=directory"
```

Ejecutar exactamente el mismo comando una segunda vez.

Comparar:

```text
command → solicita ejecutar una instrucción.
file    → solicita alcanzar un estado.
```

---

## 13. Interpretar resultados

| Estado | Significado |
|---|---|
| `ok` | El recurso ya se encuentra en el estado esperado. |
| `changed` | Ansible realizó una modificación. |
| `failed` | La tarea se ejecutó, pero terminó con error. |
| `unreachable` | Ansible no pudo conectarse al host. |
| `skipped` | La tarea fue omitida por una condición. |

En comandos ad hoc veremos principalmente `SUCCESS`, `CHANGED`, `FAILED` y `UNREACHABLE`. `skipped` y `PLAY RECAP` serán más visibles al trabajar con Playbooks.

### Provocar un error controlado

```bash
ansible devices -i inventory/lab.ini -m command -a "cat /archivo-que-no-existe"
```

Identificar el código de retorno y el mensaje del error.

### Provocar `unreachable`

```bash
ansible device01 -i inventory/lab.ini -m ping -e ansible_port=2299
```

Comparar `FAILED` con `UNREACHABLE`.

---

# Ejercicio integrador

Preparar todos los dispositivos utilizando comandos ad hoc.

Cada dispositivo debe tener:

1. Grupo `devops`.
2. Usuario `operador`.
3. Directorio `/opt/ansible-demo`.
4. Paquete `cowsay`.
5. Archivo `/opt/ansible-demo/mensaje.txt`.
6. Servicio SSH disponible.

Después de completar la preparación, volver a ejecutar todos los comandos.

## Resultado esperado

Las operaciones idempotentes deberían reportar principalmente `ok` cuando el estado deseado ya se cumple.

---

# Reto: detectar y corregir un cambio manual

Eliminar manualmente el archivo únicamente en `device01`:

```bash
docker exec device01 rm /opt/ansible-demo/mensaje.txt
```

Validar:

```bash
ansible device01 -i inventory/lab.ini -m command -a "ls -l /opt/ansible-demo/"
```

Volver a ejecutar:

```bash
ansible devices -i inventory/lab.ini -b -m copy -a "src=mensaje.txt dest=/opt/ansible-demo/mensaje.txt owner=operador group=devops mode=0644"
```

Comparar el resultado de `device01` con `device02` y `device03`.

---

## Preguntas de cierre

1. ¿Qué diferencia existe entre `ok` y `changed`?
2. ¿Por qué `file` puede ser preferible a ejecutar `mkdir` mediante `command`?
3. ¿Cuándo utilizaríamos `shell` en lugar de `command`?
4. ¿Qué significa que una operación sea idempotente?
5. ¿Qué diferencia existe entre `failed` y `unreachable`?
6. ¿Qué ocurrió al modificar manualmente uno de los dispositivos?
7. ¿Qué ventaja tendría agrupar estos comandos en un Playbook?

---

## Resultado de la sesión

Al finalizar, el participante habrá ejecutado tareas reales utilizando módulos de Ansible, comprobado la idempotencia e interpretado los principales estados de ejecución.

La siguiente evolución será agrupar estas operaciones como tareas declarativas reutilizables mediante Playbooks.
