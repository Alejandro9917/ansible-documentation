# Sesión 1 — Fundamentos de Infraestructura como Código

## Objetivo

En esta primera práctica **no utilizaremos Ansible**. El objetivo es experimentar cómo se administran servidores de forma manual para comprender posteriormente el valor de la Infraestructura como Código.

Al finalizar la sesión el estudiante será capaz de:

- Levantar el laboratorio local.
- Identificar los servidores Ubuntu.
- Acceder a cada servidor.
- Ejecutar tareas básicas de administración.
- Reflexionar sobre las limitaciones de la administración manual.

---

## 1. Levantar el laboratorio

```bash
docker compose up -d
```

Verificar los contenedores:

```bash
docker ps
```

---

## 2. Acceder al nodo de control

```bash
docker exec -it ansible-control bash
```

```bash
cat /etc/os-release
hostname
hostname -I
```

---

## 3. Explorar un servidor Ubuntu

```bash
docker exec -it device01 bash
```

```bash
hostname
cat /etc/passwd
free -h
lscpu
df -h
ip addr
ps aux
```

---

## 4. Administración manual

```bash
apt update
apt install -y curl
curl --version
apt install -y tree
tree --version
```

---

## 5. Crear un usuario

```bash
useradd laboratorio
passwd laboratorio
id laboratorio
```

---

## 6. Crear archivos

```bash
mkdir -p /opt/demo
touch /opt/demo/app.conf
nano /opt/demo/app.conf
cat /opt/demo/app.conf
```

---

## 7. Revisar SSH

```bash
service ssh status
```

Si la imagen lo soporta:

```bash
systemctl status ssh
```

---

## Preguntas para discusión

- ¿Cuántos comandos ejecutamos para preparar un servidor?
- ¿Qué ocurriría si tuviéramos que repetir estos pasos en 100 servidores?
- ¿Qué pasa si olvidamos un comando?
- ¿Cómo garantizaríamos que todos los servidores queden configurados exactamente igual?
- ¿Cómo documentaríamos este procedimiento?

---

## Conclusión

Todo lo realizado en esta sesión será automatizado progresivamente durante el resto del curso utilizando Ansible e Infraestructura como Código.

---

## Reto opcional

- Equipo 1: instalar curl.
- Equipo 2: instalar tree.
- Equipo 3: crear un usuario y un directorio.
