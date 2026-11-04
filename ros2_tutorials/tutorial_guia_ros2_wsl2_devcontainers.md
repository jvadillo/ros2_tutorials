# Guía Definitiva: Desarrollo con ROS 2, Dev Containers, WSL2 y Simulación con Gazebo en Windows

Esta guía aborda los conceptos teóricos, las arquitecturas de red y el despliegue práctico paso a paso para configurar un entorno de desarrollo profesional con **ROS 2**, **WSL2**, **Docker (Dev Containers)** y simulación 3D con **Gazebo** (TurtleBot 4) desde un ordenador con **Windows**.

---

## PARTE 1: Conceptos y Arquitectura General

### 1. Tendencia Actual en ROS 2: Desarrollo vs. Producción
En el ecosistema profesional de ROS 2 no se suele elegir de forma excluyente entre instalar ROS 2 nativo o usar Docker:
* **Para el Desarrollo Diario:** Se utiliza **Desarrollo Asistido por Contenedores (VS Code Dev Containers)**. Garantiza un entorno exacto, aislado y reproducible, evitando conflictos de dependencias entre proyectos sin perder la comodidad de programar desde un editor moderno.
* **Para Producción y Despliegue en Robots Reales:** Se utiliza **Docker y Docker Compose**. Todos los nodos de ROS 2 del robot se ejecutan dentro de contenedores aislados en el ordenador de abordo (ej. Raspberry Pi 4). Esto permite actualizaciones sencillas en remoto (OTA) e integración continua (CI/CD).

### 2. La "Foto Final" de la Arquitectura de Despliegue
Cuando trabajas con un robot (simulado o real), la distribución de tareas se organiza de la siguiente forma:

```text
+-------------------------------------------------------------+
|               ROBOT REAL O SIMULADO (TURTLEBOT 4)           |
|                                                             |
|  [Sensores / Motores] <---> [Cerebro / SBC con ROS 2]       |
|                                     |                       |
|                             (Corre en Docker)               |
+-------------------------------------|-----------------------+
                                      |
                           Comunicaciones ROS 2 (DDS)
                                      |
                                      v
+-------------------------------------------------------------+
|                 ESTACIÓN DE TRABAJO (PORTÁTIL)              |
|                                                             |
|  [VS Code / Dev Container]  <---> [RViz2 / Gazebo / Herram.]|
+-------------------------------------------------------------+
```

1. **Robot / Simulador:** Ejecuta los nodos principales de ROS 2 (motores, LIDAR, cámara, control y algoritmos de navegación).
2. **Estación de Trabajo:** Se conecta vía Wi-Fi/Red local mediante el middleware **DDS** para enviar comandos, visualizar el mapa o depurar mediante herramientas como RViz2.

### 3. Exposición a Internet vía API (`rosbridge`)
Para exponer datos de sensores a aplicaciones web o móviles fuera de la red local:
* Se ejecuta **`rosbridge_server`** dentro del robot para traducir los *topics* binarios de ROS 2 a mensajes **JSON** vía WebSockets.
* Para conectar con clientes en internet se utilizan **Túneles Seguros (como Ngrok o Cloudflare Tunnels)** o redes **VPN en malla (como Tailscale)**.

### 4. ¿Cómo interactúan Windows, WSL2, Docker y el sistema de archivos?
* **Bind Mount (Sin duplicación):** Docker no "copia" tus archivos al contenedor; genera una ventana directa al disco de WSL2. Modificar un archivo en VS Code actualiza al instante la lectura en el contenedor.
* **Ubicación del Proyecto (WSL2 vs Windows):** El proyecto debe guardarse físicamente en la ruta nativa de Linux en WSL2 (`/home/usuario/...`), no en discos de Windows (`C:\...`). Al estar dentro del sistema de archivos de Linux, la velocidad de compilación de ROS 2 (`colcon build`) es máxima, evitando la penalización de traducción de discos cruzados.

---

## PARTE 2: Tutorial Paso a Paso

### Paso 1: Configurar WSL2 y el Modo Espejo de Red

1. Abre **PowerShell como Administrador** en Windows e instala WSL2:
   ```powershell
   wsl --install
   ```
   *(Reinicia el equipo si el sistema lo solicita).*

2. Habilita el **modo espejo de red** para que WSL2 comparta la misma interfaz de red de tu equipo (vital para el descubrimiento de nodos DDS por Wi-Fi). Ve a tu carpeta de usuario en Windows (`C:\Users\TuUsuario\`) y crea o edita el archivo `.wslconfig`:
   ```ini
   [wsl2]
   networkingMode=mirrored
   ```

3. Reinicia WSL para aplicar los cambios:
   ```powershell
   wsl --shutdown
   ```

---

### Paso 2: Instalar Docker Desktop

1. Descarga e instala **Docker Desktop para Windows**.
2. En los ajustes de Docker Desktop, verifica lo siguiente:
   * **General:** Que esté marcada la opción **"Use the WSL 2 based engine"**.
   * **Resources > WSL Integration:** Asegúrate de activar la integración con tu distribución (ej. *Ubuntu*).

---

### Paso 3: Crear el Workspace dentro de WSL2 y Abrir VS Code

1. Abre tu terminal de **Ubuntu (WSL2)** desde el menú Inicio de Windows.
2. Crea la estructura de tu espacio de trabajo de ROS 2:
   ```bash
   mkdir -p ~/ros2_ws/src
   cd ~/ros2_ws
   ```
3. Abre VS Code directamente conectado a esta ruta interna de Linux:
   ```bash
   code .
   ```

---

### Paso 4: Configurar el Dev Container en VS Code

1. Instala las extensiones **WSL** y **Dev Containers** en VS Code desde el apartado de extensiones (`Ctrl + Shift + X`).
2. Dentro de la carpeta `ros2_ws` recién abierta en VS Code, crea una carpeta llamada `.devcontainer` y dentro un archivo `devcontainer.json`:

```json
{
  "name": "ROS 2 Humble Simulation",
  "image": "osrf/ros:humble-desktop",
  "runArgs": [
    "--net=host",
    "--ipc=host"
  ],
  "containerEnv": {
    "DISPLAY": ":0"
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-iot.vscode-ros"
      ]
    }
  },
  "postCreateCommand": "apt-get update && apt-get install -y ros-humble-turtlebot4-simulator"
}
```

#### Desglose de Parámetros Clave:
* **`image`**: Usa la imagen base oficial de ROS 2 Humble sobre Ubuntu 22.04.
* **`runArgs` (`--net=host`, `--ipc=host`)**: Permite que los nodos dentro del contenedor compartan la red y la memoria rápida del sistema host sin restricciones aisladas de Docker.
* **`containerEnv` (`DISPLAY: :0`)**: Redirige la salida gráfica hacia **WSLg** para mostrar las ventanas de Gazebo o RViz2 flotando en Windows.
* **`postCreateCommand`**: Descarga e instala automáticamente el paquete de simulación oficial del **TurtleBot 4** al crear el entorno.

3. Pulsa el botón verde en la esquina inferior izquierda de VS Code y selecciona **"Reopen in Container"**.
4. Espera a que Docker construya y configure el contenedor.

---

### Paso 5: Lanzar la Simulación del TurtleBot 4 con Gazebo

1. Dentro de VS Code, abre una terminal integrada (`Ctrl + Shift + ~`). Verás que estás operando dentro del contenedor (el prompt muestra la ruta `/workspaces/ros2_ws`).
2. Carga las variables de entorno de ROS 2:
   ```bash
   source /opt/ros/humble/setup.bash
   ```
3. Lanza el simulador Gazebo con el entorno y el robot TurtleBot 4:
   ```bash
   ros2 launch turtlebot4_ignition_bringup turtlebot4_ignition.launch.py
   ```
4. Se abrirá automáticamente la ventana gráfica de **Gazebo** en tu escritorio de Windows gracias a WSLg, mostrando el robot listo para ser programado o controlado mediante ROS 2.

---

## PARTE 3: Variables de Entorno de Red para DDS

Si conectas tu portátil a un TurtleBot 4 real a través de la Wi-Fi local, asegúrate de configurar estas variables tanto en tu terminal como en el robot:

* **Mismo Canal de Comunicación:**
  ```bash
  export ROS_DOMAIN_ID=42
  ```
* **Permitir Comunicación por Red Física:**
  ```bash
  export ROS_LOCALHOST_ONLY=0
  ```