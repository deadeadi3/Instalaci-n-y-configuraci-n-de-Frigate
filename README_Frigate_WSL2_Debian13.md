# Frigate NVR con WSL2 + Debian 13 + Docker

Guía práctica para instalar y utilizar [Frigate NVR](https://frigate.video/) en Windows mediante WSL2, usando Debian 13 y Docker.

> **Caso de referencia:** Windows 10 Enterprise LTSC 2021, WSL2, Debian 13 (Trixie), Docker Engine y Docker Compose.  
> Esta guía está pensada para un laboratorio/prueba en un PC Windows y posteriormente conectar una cámara IP o un teléfono Android como cámara.

---

## 1. Arquitectura del proyecto

```text
Windows 10/11
└── WSL2
    └── Debian 13 (Trixie)
        └── Docker Engine
            └── Frigate NVR
                ├── Detección de objetos
                ├── Grabación
                ├── RTSP / go2rtc
                └── Interfaz web
```

La idea es mantener Frigate aislado dentro de un contenedor Docker, mientras Debian proporciona el entorno Linux.

---

# PARTE I — Preparar WSL2

## 2. Comprobar la versión de Windows

Abrir **PowerShell como administrador**:

```powershell
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, OsBuildNumber
```

En el caso de referencia se obtuvo:

```text
Windows 10 Enterprise LTSC 2021
WindowsVersion  2009
OsBuildNumber   19044
```

Microsoft indica que los comandos actuales de instalación de WSL funcionan en Windows 10, versión 2004 o posterior, compilación 19041 o posterior, y Windows 11. En equipos donde WSL ya está instalado también pueden usarse los comandos específicos de instalación/configuración. [Microsoft Learn](https://learn.microsoft.com/windows/wsl/install)

---

## 3. Comprobar las características necesarias

En PowerShell como administrador:

```powershell
Get-WindowsOptionalFeature -Online |
  Where-Object {$_.FeatureName -match "Linux|VirtualMachine"} |
  Select-Object FeatureName, State
```

Las dos características importantes son:

```text
Microsoft-Windows-Subsystem-Linux
VirtualMachinePlatform
```

Si aparecen como `Disabled`, habilitarlas:

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

```powershell
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

Reiniciar Windows después.

---

## 4. Configurar WSL2 como versión predeterminada

En PowerShell:

```powershell
wsl --set-default-version 2
```

Comprobar:

```powershell
wsl --status
```

Debe aparecer:

```text
Default Version: 2
```

Si aparece:

```text
The WSL 2 kernel file is not found.
```

actualizar el kernel:

```powershell
wsl --update
```

Después volver a comprobar:

```powershell
wsl --status
```

---

## 5. Ver distribuciones disponibles

```powershell
wsl --list --online
```

Para este proyecto se usará:

```text
Debian
```

Microsoft documenta `wsl --list --online` para consultar las distribuciones disponibles y `wsl --install -d <Distro>` para instalar una distribución concreta. [Microsoft Learn](https://learn.microsoft.com/windows/wsl/install)

---

# PARTE II — Instalar Debian 13 en WSL2

## 6. Instalar Debian

En PowerShell:

```powershell
wsl --install -d Debian
```

Al finalizar, abrir Debian y crear:

- Usuario Linux
- Contraseña Linux

Ejemplo:

```text
Enter new UNIX username:
diego
```

La contraseña no se muestra mientras se escribe; es normal.

---

## 7. Verificar la versión de Debian

Dentro de Debian:

```bash
cat /etc/os-release
```

En el caso de referencia:

```text
PRETTY_NAME="Debian GNU/Linux 13 (trixie)"
VERSION_ID="13"
VERSION="13 (trixie)"
```

Comprobar también el usuario y directorio:

```bash
whoami
pwd
```

Ejemplo:

```text
diego
/home/diego
```

---

## 8. Actualizar Debian

```bash
sudo apt update
sudo apt upgrade -y
```

---

# PARTE III — Instalar Docker Engine

## 9. Instalar requisitos

```bash
sudo apt update
sudo apt install -y ca-certificates curl
```

Docker soporta oficialmente Debian 13 (Trixie) para Docker Engine. [Docker Docs](https://docs.docker.com/engine/install/debian/)

---

## 10. Agregar la clave de Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

```bash
sudo curl -fsSL https://download.docker.com/linux/debian/gpg \
  -o /etc/apt/keyrings/docker.asc
```

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

---

## 11. Agregar el repositorio oficial

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Actualizar índices:

```bash
sudo apt update
```

---

## 12. Instalar Docker Engine y Docker Compose

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

Docker documenta estos paquetes para la instalación desde su repositorio APT oficial. [Docker Docs](https://docs.docker.com/engine/install/debian/)

---

## 13. Comprobar Docker

```bash
sudo docker --version
```

```bash
sudo docker compose version
```

Prueba funcional:

```bash
sudo docker run hello-world
```

Debe aparecer:

```text
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

---

## 14. Permitir Docker sin sudo

```bash
sudo usermod -aG docker $USER
```

Cerrar Debian:

```bash
exit
```

Volver a abrir:

```powershell
wsl -d Debian
```

Probar:

```bash
docker run hello-world
```

Si funciona sin `sudo`, el usuario ya tiene acceso al socket de Docker.

---

# PARTE IV — Preparar Frigate

## 15. Crear estructura de directorios

En Debian:

```bash
mkdir -p ~/frigate/config
mkdir -p ~/frigate/storage
cd ~/frigate
```

Comprobar:

```bash
pwd
```

Resultado esperado:

```text
/home/diego/frigate
```

La estructura mínima de Frigate incluye el archivo de Compose, `config/` y `storage/`. [Frigate Docs](https://docs.frigate.video/guides/getting_started/)

---

## 16. Comprobar Docker Compose

```bash
docker compose version
```

---

## 17. Crear docker-compose.yml

Crear/editar:

```bash
nano docker-compose.yml
```

Configuración inicial de laboratorio:

```yaml
services:
  frigate:
    container_name: frigate
    restart: unless-stopped
    stop_grace_period: 30s

    image: ghcr.io/blakeblackshear/frigate:stable

    shm_size: "256mb"

    volumes:
      - ./config:/config
      - ./storage:/media/frigate
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 500000000

    ports:
      - "8971:8971"
      - "8554:8554"
      - "8555:8555/tcp"
      - "8555:8555/udp"

    environment:
      TZ: "America/Lima"
```

> `shm_size` depende de la cantidad de cámaras y de las resoluciones usadas para detección. En este laboratorio se aumentó a 256 MB después de que Frigate indicara que 128 MB era insuficiente.

La documentación de Frigate muestra un Compose inicial con `ghcr.io/blakeblackshear/frigate:stable`, `/config`, `/media/frigate`, el puerto 8971 y RTSP en 8554. [Frigate Docs](https://docs.frigate.video/guides/getting_started/)

---

## 18. Descargar la imagen de Frigate

```bash
docker compose pull
```

---

## 19. Iniciar Frigate

```bash
docker compose up -d
```

Comprobar:

```bash
docker compose ps
```

Ver logs:

```bash
docker compose logs --tail=50
```

Un arranque correcto mostrará procesos como:

```text
Recording process started
Review process started
go2rtc process pid
Output process started
Starting FastAPI app
FastAPI started
```

La imagen de Frigate se descarga desde GitHub Container Registry.

---

# PARTE V — Entrar a Frigate

## 20. Abrir la interfaz

Desde Windows:

```text
https://localhost:8971
```

> El puerto 8971 usa HTTPS en la interfaz autenticada actual. Si se utiliza `http://localhost:8971`, puede aparecer:
>
> `400 Bad Request`
>
> `The plain HTTP request was sent to HTTPS port`

Frigate documenta el acceso a la interfaz mediante `https://server_ip:8971`. [Frigate Docs](https://docs.frigate.video/guides/getting_started/)

---

## 21. Primer usuario

Durante el primer arranque Frigate puede crear un usuario administrador y mostrar una contraseña inicial en los logs:

```bash
docker logs frigate
```

No publicar esa contraseña en un repositorio GitHub.

Después de entrar, cambiar la contraseña desde la interfaz.

---

# PARTE VI — Agregar una cámara

## 22. Cámara RTSP

Frigate puede usar cámaras que proporcionen una URL RTSP.

Para agregar una cámara:

```text
Settings
→ Global configuration
→ Camera management
→ Add Camera
```

Frigate incluye un asistente para añadir cámaras. [Frigate Docs](https://docs.frigate.video/guides/getting_started/)

Ejemplo genérico de URL:

```text
rtsp://usuario:contraseña@192.168.1.100:554/stream
```

Los datos reales dependen de la cámara.

---

# PARTE VII — Usar un teléfono Android como cámara

Un teléfono Android puede actuar como cámara IP mediante una aplicación que entregue un stream compatible.

Ejemplo del laboratorio:

```text
Redmi Note 12 Pro+
Android 14
HyperOS 2.x
```

La aplicación debe proporcionar una dirección RTSP accesible desde la red local.

Arquitectura:

```text
📱 Android
    │
    │ Wi-Fi / RTSP
    ▼
Windows
    │
    ▼
WSL2
    │
    ▼
Docker
    │
    ▼
Frigate
```

Antes de configurar Frigate, comprobar desde la red que el teléfono está transmitiendo y obtener la URL RTSP.

---

# PARTE VIII — Detección de personas y perros

## 23. Activar objetos

En:

```text
Settings
→ Global configuration
→ Objects
```

activar:

```text
person
dog
```

Opcionalmente:

```text
cat
```

Frigate rastrea objetos y permite configurar qué clases seguir desde la interfaz o mediante YAML. [Frigate Docs](https://docs.frigate.video/configuration/objects/)

---

## 24. Filtros de objeto

Para comenzar una prueba, evitar filtros demasiado restrictivos.

Los filtros pueden controlar:

- Área mínima
- Área máxima
- Relación de aspecto
- Umbral de confianza
- Confianza mínima

No conviene empezar con umbrales altos porque pueden eliminar detecciones válidas.

---

# PARTE IX — Movimiento, grabación y Review

## 25. Movimiento vs detección

Frigate usa movimiento para identificar regiones que merecen procesarse y después realiza detección de objetos.

Conceptualmente:

```text
Movimiento
   ↓
Procesamiento
   ↓
Detector de objetos
   ↓
Person / Dog / Cat
```

Por eso:

```text
Motion != Person
```

Que exista movimiento no significa automáticamente que exista una persona.

---

## 26. Habilitar grabación

Para usar Review y conservar eventos de una cámara, habilitar la grabación de esa cámara.

En la configuración de la cámara:

```text
Recording
→ Enable recording
```

Frigate documenta que Review depende de los datos/eventos de una cámara con grabación habilitada. [Frigate Docs](https://docs.frigate.video/configuration/record/)

---

# PARTE X — Resolución y FPS recomendados para una cámara Full HD

Para una cámara:

```text
1920x1080
```

se puede separar:

```text
Grabación:
1920x1080

Detección:
1280x720
5 FPS
```

Ventaja:

```text
Grabación = máxima calidad
Detección = menor consumo de CPU
```

La resolución usada para detectar debe ser suficiente para que los objetos tengan tamaño útil en la imagen. Frigate recomienda prestar atención especialmente a la resolución de detección y normalmente 5 FPS es suficiente para detección. [Frigate Docs](https://docs.frigate.video/frigate/camera_setup/)

---

# PARTE XI — Detector

## 27. Detector por defecto

Frigate puede iniciar usando:

```text
OpenVINO
CPU
```

Es una opción válida para una prueba.

La documentación de Frigate indica que, por defecto, puede utilizar un detector OpenVINO sobre CPU. También recomienda aprovechar aceleración por hardware cuando el equipo la soporte. [Frigate Docs](https://docs.frigate.video/guides/getting_started/)

---

## 28. Hardware acceleration

La aceleración para decodificar vídeo es diferente de la detección de objetos.

Conceptualmente:

```text
Decodificación de vídeo
        ↓
GPU / iGPU
        ↓
menos carga de CPU
```

y:

```text
Detección de objetos
        ↓
CPU / GPU / acelerador
```

En un Intel compatible, Frigate documenta el uso de `/dev/dri/renderD128` y un preset VAAPI como ejemplo de aceleración. Debe adaptarse al hardware real. [Frigate Docs](https://docs.frigate.video/guides/getting_started/)

---

# PARTE XII — Comandos útiles

## Ver estado

```bash
docker compose ps
```

## Ver logs

```bash
docker compose logs --tail=100
```

## Seguir logs en vivo

```bash
docker compose logs -f
```

## Reiniciar

```bash
docker compose restart
```

## Detener

```bash
docker compose down
```

## Iniciar

```bash
docker compose up -d
```

## Descargar nueva imagen

```bash
docker compose pull
```

## Aplicar una nueva versión después de actualizar la imagen

```bash
docker compose up -d
```

---

# PARTE XIII — Problemas frecuentes del laboratorio

## `The WSL 2 kernel file is not found`

Ejecutar:

```powershell
wsl --update
```

y volver a comprobar:

```powershell
wsl --status
```

---

## `400 Bad Request — plain HTTP request was sent to HTTPS port`

Usar:

```text
https://localhost:8971
```

en lugar de:

```text
http://localhost:8971
```

---

## `/dev/shm ... must be increased`

Aumentar `shm_size` en `docker-compose.yml`.

Ejemplo:

```yaml
shm_size: "256mb"
```

El valor necesario depende de la resolución, FPS y cantidad de cámaras.

---

## Review indica que la grabación no está habilitada

Activar:

```text
Camera → Recording → Enable recording
```

---

## Frigate consume demasiada CPU

Revisar primero:

1. Resolución de `detect`
2. FPS de `detect`
3. Número de cámaras
4. Aceleración de decodificación
5. Detector de objetos

Una estrategia inicial razonable para una cámara Full HD es:

```text
record: 1920x1080
detect: 1280x720
detect FPS: 5
```

