# Cowrie Honeypot — Analisis de intentos de Ataque a un Servidor

Proyecto de despliegue de un honeypot SSH/Telnet en AWS para capturar,
almacenar y analizar tráfico de ataque real proveniente de internet: intentos
de fuerza bruta, comandos ejecutados por atacantes/bots tras el login, y
malware distribuido durante los ataques.

## ¿Qué es un honeypot?

Un honeypot es un sistema señuelo diseñado para parecer un servidor real
vulnerable, esta "vulnerabilidad", se deja adrede con el fin de atraer
intentos de ataque por parte de personas / bots, registrando toda su actividad sin que el sistema real corra riesgo
— ya que todo ocurre dentro de un entorno aislado.

Este proyecto usa [Cowrie](https://github.com/cowrie/cowrie), dado que es capaz de:

- Simular un shell Linux completo (sistema de archivos falso, comandos falsos)
- Registrar credenciales probadas, comandos ejecutados y archivos que el
  atacante intenta descargar
- Grabar sesiones completas, reproducibles como una grabación de terminal

## Arquitectura

```
Internet
   │
   ▼
[ AWS Security Group ]  puertos 22 / 23 abiertos a 0.0.0.0/0
   │
   ▼
[ iptables PREROUTING/REDIRECT ]
   │  22 → 2222     23 → 2223
   ▼
[ Cowrie (venv, usuario sin privilegios) ]
   │  simula shell SSH/Telnet
   ▼
[ MySQL local (solo localhost) ]
   │  tablas: sessions, auth, input, downloads, clients, keyfingerprints...
   ▼
[ Análisis: consultas SQL + Python (pandas, geolocalización) ]
```

El acceso administrativo real a la instancia se movió del puerto 22 a un
puerto no estándar, quedando reservado exclusivamente a mi IP — todo el
tráfico al puerto 22/23 "público" es interceptado por Cowrie, nunca llega
al SSH real del sistema.

## Stack tecnológico

| Componente | Uso |
|---|---|
| AWS EC2 (Ubuntu) | Instancia donde corre el honeypot |
| Cowrie | Honeypot SSH/Telnet, corre en entorno virtual de Python |
| iptables | Redirección de puertos 22/23 → Cowrie |
| MySQL | Almacenamiento estructurado de toda la actividad capturada |
| Python (pandas, requests, folium) | Análisis y visualización de los datos |
| ip-api.com | Geolocalización de IPs de origen |

## Datos capturados

| Tabla | Contenido |
|---|---|
| `sessions` | IP de origen, timestamps de inicio/fin de cada conexión |
| `auth` | Combinaciones usuario/contraseña probadas |
| `input` | Comandos ejecutados por el atacante tras el login |
| `downloads` | URLs y hashes de archivos que el atacante intentó descargar |
| `clients` | Fingerprint del cliente SSH usado por el atacante |
| `keyfingerprints` | Claves públicas SSH usadas en intentos de autenticación |

## Hallazgos destacados

Durante el análisis manual de los comandos capturados (tabla `input`) se
identificaron patrones bien documentados en la literatura de seguridad:

- **Dropper multi-protocolo tipo Mirai/Gafgyt**: scripts que prueban
  descargar el mismo payload por `wget`, `busybox wget` y `tftp` en secuencia,
  adaptándose a las herramientas disponibles en el dispositivo comprometido.
- **Barrido de directorios escribibles ("shotgun")**: intentos automatizados
  de escritura en más de una decena de rutas (`/tmp`, `/var`, `/dev/shm`,
  `/etc`, `/root`, etc.) para detectar dónde el sistema de archivos permite
  escribir — común en dispositivos IoT con firmware de solo lectura.
- **Firma `dvrHelper`**: uso del nombre de archivo `dvrHelper` (copiando el
  binario `/bin/echo` y renombrándolo), una firma ampliamente asociada a la
  familia de malware **Mirai**, originalmente diseñada para infectar DVRs y
  cámaras IP.
- **Fingerprinting de hardware orientado a cryptomining**: un script mucho
  más sofisticado que recolecta CPU, núcleos, y específicamente **presencia
  de GPU NVIDIA** — comportamiento atípico para bots IoT tradicionales y más
  asociado a campañas de minería de criptomonedas o reventa de accesos.
- **Detección activa de honeypots**: el mismo script anterior ejecuta
  comandos inexistentes a propósito (`./xxxxxx`) y compara el mensaje de
  error exacto contra el de un shell Linux real, además de crear y ejecutar
  un script de prueba (`echo "xxxxxx"`) para confirmar que el entorno
  ejecuta comandos de verdad y no está siendo simulado.

Ver el detalle completo, con los comandos exactos capturados y su análisis
línea por línea, en [`docs/findings.md`](docs/findings.md).

## Estructura del repositorio

```
cowrie-honeypot-analysis/
├── README.md
├── docs/
│   ├── setup.md          # cómo se desplegó el honeypot paso a paso
│   └── findings.md       # hallazgos técnicos detallados
├── data/                 # exports de la base de datos (IPs, auth, input, downloads)
├── scripts/              # scripts de análisis (geolocalización, etc.)
└── analysis/             # notebooks y resultados de análisis (en progreso)
```

## Próximos pasos

- [ ] Geolocalización completa de todas las IPs capturadas + mapa interactivo
- [ ] Clasificación automática de comandos por fase de ataque (reconocimiento,
      evasión, descarga, ejecución, limpieza de huellas)
- [ ] Cruce de hashes de `downloads` contra VirusTotal para identificar
      familias de malware con certeza
- [ ] Dashboard con métricas agregadas (top países, top credenciales,
      duración de sesiones, tendencia temporal)

## Créditos

Construido sobre [Cowrie](https://github.com/cowrie/cowrie), proyecto
open-source mantenido por la comunidad de seguridad.
