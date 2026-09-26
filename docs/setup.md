# Setup — Cómo se desplegó el honeypot

Resumen del proceso completo de despliegue, en orden cronológico. Los valores
sensibles (IPs propias, passwords) se muestran como placeholders.

## 1. Instancia base

- AWS EC2, Ubuntu.
- Puertos abiertos en el Security Group:
  - `22` — cebo del honeypot (Cowrie SSH), origen `0.0.0.0/0`
  - `23` — cebo del honeypot (Cowrie Telnet), origen `0.0.0.0/0`
  - `<puerto-admin>` — SSH real de administración, origen restringido a mi IP

## 2. Dependencias del sistema

```bash
sudo apt update
sudo apt install -y python3-pip python3-venv libssl-dev libffi-dev \
    build-essential libpython3-dev python3-minimal authbind
```

## 3. Instalación de Cowrie en entorno virtual

```bash
python3 -m venv cowrie-env
source cowrie-env/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

## 4. Migración del SSH real de administración

Antes de redirigir el puerto 22 hacia Cowrie, el SSH real del sistema se
migró a un puerto no estándar, para no perder acceso administrativo:

1. En `/etc/ssh/sshd_config`, se agregó el nuevo puerto **sin quitar el 22**
   todavía (`Port 22` + `Port <nuevo>`), validando el acceso al nuevo puerto
   en una sesión aparte antes de continuar.
2. Confirmado el acceso al nuevo puerto, se removió `Port 22` del archivo,
   dejando el SSH real escuchando únicamente en el puerto de administración.
3. Se validó la sintaxis (`sudo sshd -t`) antes de cada `systemctl restart
   ssh`, y siempre se mantuvo una sesión activa como red de seguridad
   mientras se probaba el nuevo puerto desde una terminal nueva.

## 5. Redirección de puertos con iptables

Con el SSH real ya migrado y confirmado, se aplicó la redirección de los
puertos "cebo" hacia donde escucha Cowrie:

```bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2222
sudo iptables -t nat -A PREROUTING -p tcp --dport 23 -j REDIRECT --to-port 2223
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

## 6. Configuración de Cowrie (`etc/cowrie.cfg`)

- SSH habilitado en `2222`, Telnet habilitado en `2223` (`interface=0.0.0.0`).

## 7. Salida a base de datos (MySQL)

```bash
sudo apt install -y mysql-server
sudo mysql
```

```sql
CREATE DATABASE cowrie;
CREATE USER 'cowrie'@'localhost' IDENTIFIED BY '<password>';
GRANT SELECT, INSERT, UPDATE ON cowrie.* TO 'cowrie'@'localhost';
FLUSH PRIVILEGES;
```

El esquema de tablas (`docs/sql/mysql.sql` del repo oficial de Cowrie) se
cargó con un usuario con privilegios de `CREATE` (root de MySQL), y luego se
recortaron los privilegios del usuario `cowrie` a solo lo necesario en
tiempo de ejecución (`SELECT`, `INSERT`, `UPDATE`) — sin permisos de
`CREATE`/`DROP`/`ALTER`, como medida de seguridad.

En `etc/cowrie.cfg`:

```ini
[output_mysql]
enabled = true
host = localhost
database = cowrie
username = cowrie
password = <password>
port = 3306
```

MySQL se dejó escuchando **únicamente en localhost** — nunca se expuso el
puerto 3306 al Security Group ni a internet.

## 8. Usuario de solo lectura para análisis

Para trabajar los datos sin riesgo de modificar accidentalmente la base
"viva" que Cowrie sigue escribiendo, se creó un usuario adicional restringido
a lectura:

```sql
CREATE USER 'analista'@'localhost' IDENTIFIED BY '<otro-password>';
GRANT SELECT ON cowrie.* TO 'analista'@'localhost';
FLUSH PRIVILEGES;
```

## 9. Acceso remoto a la base de datos (para análisis desde mi equipo)

En vez de exponer MySQL a internet, el acceso desde mi máquina personal se
hace mediante un **túnel SSH**, reutilizando el puerto de administración:

```bash
ssh -i clave.pem -p <puerto-admin> -L 3306:localhost:3306 usuario@<ip-instancia>
```

Con el túnel activo, cualquier cliente MySQL en mi equipo (CLI, DBeaver,
scripts de Python) se conecta a `127.0.0.1:3306` como si la base estuviera
local, sin que el puerto 3306 esté nunca expuesto públicamente.

## Principios de seguridad aplicados

- El acceso administrativo real nunca quedó expuesto sin verificar el nuevo
  puerto primero (evitando quedar fuera de la instancia).
- MySQL nunca se expuso a internet; todo acceso remoto pasa por túnel SSH.
- El usuario de la base usado para análisis tiene permisos de solo lectura.
- El usuario `cowrie` (runtime) no tiene privilegios de `CREATE`/`ALTER`/`DROP`.
