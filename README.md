# taller-segundo-corte
# Taller Segundo Corte – PyBullet + ESP32

## 1. Descripción

Este proyecto integra simulación robótica en **PyBullet** con un **ESP32** como dispositivo de mando.

El taller se divide en:

- **Parte A:** control de múltiples drones con `gym-pybullet-drones`, desplazándolos desde una posición inicial hacia los puntos A, B y C. El ESP32 envía las órdenes de alto nivel.
- **Parte B:** consola de mando para el robot Baxter en PyBullet. El ESP32 permite modificar la posición cartesiana del efector final y abrir/cerrar la pinza. El programa implementa cinemática inversa y un mecanismo de agarre de un objeto.
- **Parte C:** el enunciado recibido solamente proporciona el enlace de referencia del repositorio `pybullet_robots`, pero no contiene una descripción de la actividad. Por ello no se inventa un requisito que no aparece en el taller suministrado.

> **Nota:** se entrega la documentación y el funcionamiento sin video, tal como se solicitó.

---

## 2. Objetivos

### Objetivo general

Desarrollar una arquitectura de control robótico en la que un ESP32 actúe como consola física de alto nivel y un computador ejecute la simulación en PyBullet.

### Objetivos específicos

1. Controlar varios drones simultáneamente en un entorno de simulación.
2. Implementar desplazamientos hacia puntos de referencia A, B y C.
3. Utilizar comunicación Wi-Fi entre ESP32 y Python.
4. Controlar el efector final del robot Baxter mediante coordenadas cartesianas.
5. Aplicar cinemática inversa para transformar una posición cartesiana en posiciones articulares.
6. Implementar apertura, cierre y agarre de un objeto.
7. Documentar la arquitectura, instalación, funcionamiento y análisis del sistema.

---

# 3. Arquitectura general

```text
                 ┌─────────────────────┐
                 │        ESP32         │
                 │ Consola de mando    │
                 └──────────┬──────────┘
                            │
                       Wi-Fi / TCP
                            │
              ┌─────────────┴─────────────┐
              │                            │
              ▼                            ▼
     ┌─────────────────┐         ┌──────────────────┐
     │ Python - Parte A│         │ Python - Parte B │
     │ Control drones  │         │ Control Baxter   │
     └────────┬────────┘         └────────┬─────────┘
              │                           │
              ▼                           ▼
     ┌─────────────────┐         ┌──────────────────┐
     │ gym-pybullet-   │         │ PyBullet         │
     │ drones           │         │ Baxter + objeto  │
     └─────────────────┘         └──────────────────┘
```

La comunicación se plantea mediante **TCP sobre Wi-Fi**. El ESP32 no ejecuta la física de la simulación: envía órdenes de alto nivel. El computador recibe dichas órdenes y ejecuta el modelo físico.

---

# 4. Parte A – Movimiento de los drones

## 4.1 Análisis

La librería `gym-pybullet-drones` proporciona entornos PyBullet para control de cuadricópteros y soporta escenarios multiagente. Para este taller se utiliza `VelocityAviary`, que está diseñado para planificación de alto nivel y admite varios drones.

La acción tiene cuatro componentes:

```text
[X, Y, Z, velocidad]
```

Las tres primeras componentes indican la dirección de movimiento y la cuarta representa la magnitud relativa de la velocidad.

El control utilizado es de alto nivel:

1. Se define la posición objetivo.
2. Se calcula el vector desde la posición actual hasta el objetivo.
3. Se normaliza el vector.
4. Se envía la dirección a `VelocityAviary`.
5. El controlador PID interno transforma la orden de velocidad en RPM de los motores.
6. El ciclo continúa hasta que el dron queda dentro de una tolerancia del objetivo.

La documentación actual del repositorio muestra que `VelocityAviary` integra `DSLPIDControl` y convierte el vector de velocidad deseado en RPM. citeturn3view0

## 4.2 Puntos de navegación

Se definen tres posiciones:

```text
A = posición inicial
B = destino 1
C = destino 2
```

Para demostrar el funcionamiento multiagente se utilizan cuatro drones:

```text
Dron 0 → B
Dron 1 → B
Dron 2 → C
Dron 3 → C
```

También se permiten órdenes para enviar todos los drones a A, B o C.

## 4.3 Comunicación

El servidor Python escucha:

```text
Puerto: 5000
Protocolo: TCP
```

Comandos:

| Comando | Acción |
|---|---|
| `A` | Todos los drones regresan a A |
| `B` | Todos los drones van a B |
| `C` | Todos los drones van a C |
| `M` | Drones 0-1 → B y drones 2-3 → C |
| `STOP` | Detiene el desplazamiento |

El comando `M` es el modo principal de la actividad porque permite demostrar simultáneamente los dos destinos.

## 4.4 Código

Archivo:

```text
parte_a_drones.py
```

El código completo está incluido en este repositorio.

---

# 5. Parte B – Consola de mando del Baxter

## 5.1 Análisis

El repositorio de Erwin Coumans contiene ejemplos de robots para PyBullet, entre ellos Baxter. El ejemplo `baxter_ik_demo.py` carga el modelo Baxter y utiliza el efector final izquierdo con ID 48. Además, implementa `calculateInverseKinematics()` y posteriormente utiliza `POSITION_CONTROL` para llevar las articulaciones a las posiciones calculadas. citeturn1view0turn1view1

La arquitectura desarrollada en este taller conserva ese principio, pero lo adapta para:

- recibir órdenes desde el ESP32;
- controlar X, Y y Z del efector;
- abrir/cerrar la pinza;
- acercarse a un objeto;
- crear una restricción de agarre cuando la pinza se encuentra sobre el objeto;
- transportar el objeto hacia otra posición.

## 5.2 Cinemática inversa

La entrada del controlador es:

```text
P = [X, Y, Z]
```

PyBullet calcula los ángulos articulares:

```text
q = IK(P)
```

donde:

- `P` es la posición cartesiana deseada.
- `q` contiene las posiciones articulares necesarias.

Después:

```text
q → POSITION_CONTROL → articulaciones Baxter
```

El ejemplo de referencia utiliza precisamente `p.calculateInverseKinematics(...)` y posteriormente `p.setJointMotorControl2(..., controlMode=p.POSITION_CONTROL, ...)`. citeturn1view1turn1view2

## 5.3 Comandos de la consola

| Comando | Función |
|---|---|
| `XP` | X aumenta |
| `XM` | X disminuye |
| `YP` | Y aumenta |
| `YM` | Y disminuye |
| `ZP` | Z aumenta |
| `ZM` | Z disminuye |
| `OPEN` | Abrir pinza |
| `CLOSE` | Cerrar pinza |
| `GRASP` | Cerrar y tomar objeto |
| `RELEASE` | Soltar objeto |
| `HOME` | Volver a posición inicial |
| `STOP` | Detener movimiento |

El desplazamiento es incremental para que la consola permita posicionamiento fino.

## 5.4 Agarre

Para la simulación se utiliza una restricción de PyBullet entre el objeto y el efector final cuando se ejecuta `GRASP`.

Esto permite comprobar funcionalmente:

```text
Baxter → acercamiento → cierre → agarre → desplazamiento → liberación
```

No se presenta como un modelo físico detallado de la transmisión mecánica de la pinza; es una estrategia de simulación para verificar la lógica de manipulación.

---

# 6. Parte C

El material suministrado incluye como referencia el repositorio:

`https://github.com/erwincoumans/pybullet_robots/tree/master`

Sin embargo, en el enunciado recibido no aparece una instrucción concreta para la Parte C.

Por rigor académico, esta documentación no inventa una actividad que no haya sido solicitada. El repositorio de referencia contiene varios modelos y ejemplos de robótica, incluyendo Baxter, Sawyer, Panda, Dobot, Atlas y otros. citeturn0view1

Si el docente tiene una instrucción adicional para C, debe agregarse al README y al código antes de cerrar la entrega.

---

# 7. Instalación

## 7.1 Python

Se recomienda utilizar Python 3.10–3.12.

Para la Parte A:

```bash
git clone https://github.com/learnsyslab/gym-pybullet-drones.git
cd gym-pybullet-drones
pip install -e .
```

El repositorio actual documenta la instalación mediante `pip install -e .` y sus ejemplos de control en la carpeta `examples`. citeturn0view0

Luego copiar `parte_a_drones.py` en una ubicación desde la que Python pueda importar:

```text
gym_pybullet_drones
```

También puede colocarse dentro de la carpeta `examples`.

## 7.2 Parte B

Clonar:

```bash
git clone https://github.com/erwincoumans/pybullet_robots.git
cd pybullet_robots
pip install pybullet numpy
```

El repositorio indica `pip install pybullet` como instalación de PyBullet. citeturn0view1

Colocar `parte_b_baxter.py` en la carpeta raíz del repositorio.

---

# 8. Ejecución

## Parte A

Primero se ejecuta:

```bash
python parte_a_drones.py
```

Aparecerá la simulación.

Después se conecta el ESP32 a la misma red Wi-Fi y se configura en su código:

```cpp
const char* SERVER_IP = "IP_DEL_PC";
const uint16_t SERVER_PORT = 5000;
```

Ejemplo:

```cpp
const char* SERVER_IP = "192.168.1.100";
```

El PC y el ESP32 deben estar en la misma red.

## Parte B

Desde la raíz de `pybullet_robots`:

```bash
python parte_b_baxter.py
```

El ESP32 se conecta al mismo PC utilizando el puerto:

```text
5001
```

---

# 9. ESP32 – Control Parte A

Archivo:

```text
esp32_parte_a.ino
```

Conexiones propuestas:

| GPIO ESP32 | Función |
|---:|---|
| 18 | A |
| 19 | B |
| 21 | C |
| 22 | M |
| 23 | STOP |

Cada pulsador se conecta entre el GPIO y GND. El programa utiliza `INPUT_PULLUP`.

Funcionamiento:

```text
Botón A → "A"
Botón B → "B"
Botón C → "C"
Botón M → "M"
Botón STOP → "STOP"
```

---

# 10. ESP32 – Consola Parte B

Archivo:

```text
esp32_parte_b.ino
```

Se utiliza un teclado matricial 4×4.

Mapa:

```text
┌─────┬─────┬─────┬─────┐
│  1  │  2  │  3  │  A  │
├─────┼─────┼─────┼─────┤
│  4  │  5  │  6  │  B  │
├─────┼─────┼─────┼─────┤
│  7  │  8  │  9  │  C  │
├─────┼─────┼─────┼─────┤
│  *  │  0  │  #  │  D  │
└─────┴─────┴─────┴─────┘
```

Asignación:

```text
1 → XP
2 → XM
4 → YP
5 → YM
7 → ZP
8 → ZM
A → OPEN
B → CLOSE
C → GRASP
D → RELEASE
* → HOME
# → STOP
```

Cada pulsación genera un comando TCP.

---

# 11. Pruebas de validación

## 11.1 Parte A

### Prueba 1 – Retorno a A

Enviar:

```text
A
```

Resultado esperado:

Todos los drones deben desplazarse hacia la posición A y detenerse dentro de la tolerancia.

### Prueba 2 – Destino B

Enviar:

```text
B
```

Resultado esperado:

Los cuatro drones deben desplazarse hacia B.

### Prueba 3 – Destino C

Enviar:

```text
C
```

Resultado esperado:

Los cuatro drones deben desplazarse hacia C.

### Prueba 4 – Dos destinos

Enviar:

```text
M
```

Resultado esperado:

```text
Drone 0 → B
Drone 1 → B
Drone 2 → C
Drone 3 → C
```

Esta es la prueba principal.

---

# 12. Pruebas de validación Parte B

### Prueba 1 – Movimiento X

Enviar:

```text
XP
```

El efector debe aumentar su coordenada X.

### Prueba 2 – Movimiento Y

Enviar:

```text
YP
```

El efector debe aumentar su coordenada Y.

### Prueba 3 – Movimiento Z

Enviar:

```text
ZM
```

El efector debe descender.

### Prueba 4 – Agarre

1. Ubicar el efector sobre el objeto.
2. Ejecutar `GRASP`.
3. Comprobar que el objeto queda unido al efector.
4. Mover X, Y o Z.
5. Verificar que el objeto se desplaza junto al Baxter.
6. Ejecutar `RELEASE`.

---

# 13. Análisis de resultados

El sistema propuesto separa el control de alto nivel de la simulación física. Esto permite que el ESP32 funcione como interfaz de usuario mientras el computador ejecuta las operaciones matemáticas y físicas.

En la Parte A, el uso de `VelocityAviary` permite trabajar con varios drones y delegar la conversión de velocidad a las acciones de los motores en el controlador integrado. La documentación del proyecto identifica explícitamente esta clase como un entorno de planificación de alto nivel y muestra su uso con múltiples drones. citeturn3view0

En la Parte B, la cinemática inversa evita que el operador tenga que calcular manualmente cada ángulo articular. El operador especifica una posición cartesiana y PyBullet obtiene una configuración articular compatible con el objetivo. Este principio coincide con el ejemplo de Baxter suministrado como referencia. citeturn1view1

La principal ventaja de la arquitectura es la modularidad:

```text
ESP32
  ↓
Comunicación
  ↓
Controlador Python
  ↓
Modelo robótico
  ↓
PyBullet
```

Si posteriormente se reemplaza PyBullet por un robot físico, la capa de interfaz puede conservarse y solamente debe sustituirse la capa encargada de convertir las órdenes en señales para los actuadores.

---

# 14. Limitaciones

1. La simulación no representa todas las imperfecciones de un sistema físico.
2. La comunicación Wi-Fi introduce una latencia variable.
3. El agarre de la Parte B se implementa mediante una restricción de simulación y no representa completamente la dinámica de una pinza física.
4. La posición alcanzable del efector está limitada por la geometría y las articulaciones del Baxter.
5. La Parte A utiliza un controlador de alto nivel; no se está diseñando desde cero el controlador de bajo nivel de los cuatro motores.

---

# 15. Conclusiones

1. Se diseñó una arquitectura de control distribuido en la que el ESP32 funciona como interfaz de mando y Python/PyBullet como plataforma de simulación y control robótico.

2. La simulación de drones permite establecer diferentes puntos de navegación y dirigir varios agentes hacia destinos definidos, demostrando el funcionamiento multiagente requerido.

3. La consola del Baxter permite controlar el efector final mediante coordenadas cartesianas, utilizando cinemática inversa para determinar las posiciones articulares correspondientes.

4. La incorporación de las funciones de agarre y liberación permite extender el control de posicionamiento hacia una tarea básica de manipulación de objetos.

5. La arquitectura modular facilita una futura migración desde la simulación hacia una implementación física, debido a la separación entre interfaz, comunicación, control y planta robótica.

---

# 16. Codigo parte b 

/*
  PARTE B - Consola ESP32 para Baxter

  Teclado matricial 4x4:

  1 -> XP
  2 -> XM
  4 -> YP
  5 -> YM
  7 -> ZP
  8 -> ZM
  A -> OPEN
  B -> CLOSE
  C -> GRASP
  D -> RELEASE
  * -> HOME
  # -> STOP
*/

#include <WiFi.h>
#include <Keypad.h>

const char* SSID = "TU_WIFI";
const char* PASSWORD = "TU_PASSWORD";

const char* SERVER_IP = "192.168.1.100";
const uint16_t SERVER_PORT = 5001;

WiFiClient client;

const byte ROWS = 4;
const byte COLS = 4;

char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};

// Cambiar estos GPIO si el montaje físico usa otros pines.
byte rowPins[ROWS] = {13, 12, 14, 27};
byte colPins[COLS] = {26, 25, 33, 32};

Keypad keypad = Keypad(
  makeKeymap(keys),
  rowPins,
  colPins,
  ROWS,
  COLS
);

void sendCommand(const char* command) {
  if (!client.connected()) {
    Serial.println("Conectando a Baxter...");

    if (!client.connect(SERVER_IP, SERVER_PORT)) {
      Serial.println("No se pudo conectar.");
      return;
    }
  }

  client.print(command);
  client.print("\n");

  Serial.print("Enviado: ");
  Serial.println(command);
}

void setup() {
  Serial.begin(115200);

  WiFi.begin(SSID, PASSWORD);

  Serial.print("Conectando WiFi");

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println();
  Serial.print("ESP32 IP: ");
  Serial.println(WiFi.localIP());

  Serial.println("Consola Baxter lista.");
}

void loop() {
  char key = keypad.getKey();

  if (!key) {
    return;
  }

  switch (key) {
    case '1': sendCommand("XP"); break;
    case '2': sendCommand("XM"); break;

    case '4': sendCommand("YP"); break;
    case '5': sendCommand("YM"); break;

    case '7': sendCommand("ZP"); break;
    case '8': sendCommand("ZM"); break;

    case 'A': sendCommand("OPEN"); break;
    case 'B': sendCommand("CLOSE"); break;
    case 'C': sendCommand("GRASP"); break;
    case 'D': sendCommand("RELEASE"); break;

    case '*': sendCommand("HOME"); break;
    case '#': sendCommand("STOP"); break;

    default:
      break;
  }

  delay(100);
}

# 17. Codigo parte a drones

"""
PARTE A - Taller Segundo Corte
Control multi-dron con ESP32 + Wi-Fi + TCP + gym-pybullet-drones.

Comandos recibidos:
A    -> todos a A
B    -> todos a B
C    -> todos a C
M    -> drones 0-1 a B, drones 2-3 a C
STOP -> detener
"""

import socket
import threading
import time
import numpy as np

from gym_pybullet_drones.envs.VelocityAviary import VelocityAviary
from gym_pybullet_drones.utils.enums import DroneModel, Physics
from gym_pybullet_drones.utils.utils import sync


HOST = "0.0.0.0"
PORT = 5000

NUM_DRONES = 4
SIM_FREQ = 240
CTRL_FREQ = 48

# A, B y C: coordenadas en metros
POINTS = {
    "A": np.array([0.0, 0.0, 1.0]),
    "B": np.array([1.8, 1.0, 1.0]),
    "C": np.array([-1.8, 1.0, 1.0]),
}

INITIAL_XYZS = np.array([
    [-0.45, -0.45, 1.0],
    [ 0.45, -0.45, 1.0],
    [-0.45,  0.45, 1.0],
    [ 0.45,  0.45, 1.0],
])

INITIAL_RPYS = np.zeros((NUM_DRONES, 3))

TOLERANCE = 0.12
SPEED_FRACTION = 0.70

command_lock = threading.Lock()
command = "A"
running = True


def set_command(new_command: str):
    global command
    new_command = new_command.strip().upper()

    valid = {"A", "B", "C", "M", "STOP"}
    if new_command not in valid:
        return

    with command_lock:
        command = new_command

    print(f"[TCP] Comando recibido: {new_command}")


def get_command():
    with command_lock:
        return command


def tcp_server():
    """Servidor TCP para recibir órdenes desde el ESP32."""
    global running

    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind((HOST, PORT))
    server.listen(1)

    print(f"[TCP] Servidor escuchando en {HOST}:{PORT}")

    while running:
        try:
            server.settimeout(1.0)
            conn, addr = server.accept()
            print(f"[TCP] ESP32 conectado: {addr}")

            with conn:
                conn.settimeout(1.0)

                while running:
                    try:
                        data = conn.recv(128)
                        if not data:
                            break

                        text = data.decode("utf-8", errors="ignore")
                        for item in text.replace("\r", "\n").split("\n"):
                            item = item.strip()
                            if item:
                                set_command(item)

                        conn.sendall(b"OK\n")

                    except socket.timeout:
                        continue
                    except ConnectionResetError:
                        break

            print("[TCP] Cliente desconectado.")

        except socket.timeout:
            continue
        except OSError:
            break

    server.close()


def targets_for_command(cmd):
    """Devuelve un objetivo por dron."""
    if cmd == "A":
        return np.tile(POINTS["A"], (NUM_DRONES, 1))

    if cmd == "B":
        return np.tile(POINTS["B"], (NUM_DRONES, 1))

    if cmd == "C":
        return np.tile(POINTS["C"], (NUM_DRONES, 1))

    if cmd == "M":
        # Modo solicitado: mitad a B y mitad a C
        return np.array([
            POINTS["B"],
            POINTS["B"],
            POINTS["C"],
            POINTS["C"],
        ])

    return None


def build_action(obs, targets):
    """Construye la orden de velocidad para cada dron."""
    action = np.zeros((NUM_DRONES, 4), dtype=np.float32)

    for i in range(NUM_DRONES):
        current = obs[i, 0:3]
        error = targets[i] - current
        distance = np.linalg.norm(error)

        if distance > TOLERANCE:
            direction = error / distance

            # X, Y, Z + magnitud relativa de velocidad
            action[i, 0:3] = direction
            action[i, 3] = SPEED_FRACTION

    return action


def main():
    global running

    env = VelocityAviary(
        drone_model=DroneModel.CF2X,
        num_drones=NUM_DRONES,
        initial_xyzs=INITIAL_XYZS,
        initial_rpys=INITIAL_RPYS,
        physics=Physics.PYB,
        pyb_freq=SIM_FREQ,
        ctrl_freq=CTRL_FREQ,
        gui=True,
        record=False,
        obstacles=False,
        user_debug_gui=False,
    )

    thread = threading.Thread(target=tcp_server, daemon=True)
    thread.start()

    print()
    print("=== PARTE A ===")
    print("Comandos locales: A, B, C, M, STOP")
    print("Puerto TCP:", PORT)
    print()

    # Iniciar en A
    set_command("A")

    start = time.time()

    try:
        for step in range(int(300 * env.CTRL_FREQ)):
            cmd = get_command()

            if cmd == "STOP":
                action = np.zeros((NUM_DRONES, 4), dtype=np.float32)
            else:
                targets = targets_for_command(cmd)
                action = build_action(env._getDroneStateVector(0)[None, :].repeat(NUM_DRONES, axis=0), targets)

                # Utilizar la observación real de todos los drones.
                obs = env._computeObs()
                action = build_action(obs, targets)

            obs, _, _, _, _ = env.step(action)
            env.render()

            # Sin grabar video: solo sincronización de la GUI.
            sync(step, start, env.CTRL_TIMESTEP)

    except KeyboardInterrupt:
        print("\n[INFO] Simulación interrumpida por el usuario.")

    finally:
        running = False
        env.close()


if __name__ == "__main__":
    main()


# 17. Codigo parte b baxter



"""
PARTE B - Baxter + ESP32 + PyBullet

Control:
XP, XM, YP, YM, ZP, ZM
OPEN, CLOSE
GRASP, RELEASE
HOME, STOP

El programa utiliza cinemática inversa para posicionar el efector
izquierdo y una restricción fija para simular el agarre del objeto.
"""

import argparse
import math
import os
import socket
import threading
import time

import numpy as np
import pybullet as p
import pybullet_data


HOST = "0.0.0.0"
PORT = 5001

END_EFFECTOR_ID = 48

STEP = 0.04
TOLERANCE = 0.05

command_lock = threading.Lock()
commands = []
running = True


def push_command(cmd):
    cmd = cmd.strip().upper()
    if not cmd:
        return

    with command_lock:
        commands.append(cmd)

    print("[TCP]", cmd)


def pop_commands():
    with command_lock:
        data = commands.copy()
        commands.clear()
        return data


def tcp_server():
    global running

    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind((HOST, PORT))
    server.listen(1)

    print(f"[TCP] Baxter escuchando en puerto {PORT}")

    while running:
        try:
            server.settimeout(1.0)
            conn, addr = server.accept()
            print("[TCP] ESP32 conectado:", addr)

            with conn:
                conn.settimeout(1.0)

                while running:
                    try:
                        data = conn.recv(256)

                        if not data:
                            break

                        text = data.decode("utf-8", errors="ignore")

                        for line in text.replace("\r", "\n").split("\n"):
                            if line.strip():
                                push_command(line)

                        conn.sendall(b"OK\n")

                    except socket.timeout:
                        continue
                    except ConnectionResetError:
                        break

        except socket.timeout:
            continue
        except OSError:
            break

    server.close()


def load_baxter(robot_root):
    urdf = os.path.join(
        robot_root,
        "baxter_common",
        "baxter_description",
        "urdf",
        "toms_baxter.urdf",
    )

    if not os.path.isfile(urdf):
        raise FileNotFoundError(
            "No se encontró el URDF de Baxter:\n" + urdf +
            "\nEjecute el programa desde el repositorio pybullet_robots "
            "o indique --robot_root."
        )

    baxter_id = p.loadURDF(urdf, useFixedBase=True)

    p.resetBasePositionAndOrientation(
        baxter_id,
        [0.5, -0.8, 0.0],
        [0.0, 0.0, -1.0, -1.0],
    )

    return baxter_id


def set_joint_targets(body_id, joint_poses):
    """Envía las posiciones articulares usando POSITION_CONTROL."""
    num_joints = p.getNumJoints(body_id)

    for i in range(num_joints):
        info = p.getJointInfo(body_id, i)
        q_index = info[3]

        if q_index > -1 and q_index - 7 < len(joint_poses):
            p.setJointMotorControl2(
                bodyIndex=body_id,
                jointIndex=i,
                controlMode=p.POSITION_CONTROL,
                targetPosition=joint_poses[q_index - 7],
                force=500,
            )


def inverse_kinematics(baxter_id, target):
    """Calcula las posiciones articulares para el efector izquierdo."""
    joint_poses = p.calculateInverseKinematics(
        baxter_id,
        END_EFFECTOR_ID,
        target,
        maxNumIterations=100,
        residualThreshold=1e-4,
    )

    set_joint_targets(baxter_id, joint_poses)
    return joint_poses


def create_object():
    collision = p.createCollisionShape(
        p.GEOM_BOX,
        halfExtents=[0.07, 0.07, 0.07],
    )

    visual = p.createVisualShape(
        p.GEOM_BOX,
        halfExtents=[0.07, 0.07, 0.07],
        rgbaColor=[0.2, 0.7, 0.2, 1.0],
    )

    obj = p.createMultiBody(
        baseMass=0.5,
        baseCollisionShapeIndex=collision,
        baseVisualShapeIndex=visual,
        basePosition=[0.55, 0.0, 0.0],
    )

    return obj


def distance(a, b):
    return float(np.linalg.norm(np.array(a) - np.array(b)))


def get_ee_position(baxter_id):
    return np.array(p.getLinkState(baxter_id, END_EFFECTOR_ID)[4])


def grasp_object(baxter_id, obj_id, current_constraint):
    ee = get_ee_position(baxter_id)
    obj = np.array(p.getBasePositionAndOrientation(obj_id)[0])

    if distance(ee, obj) > 0.20:
        print("[GRASP] El objeto está demasiado lejos del efector.")
        print("[GRASP] Distancia:", round(distance(ee, obj), 3), "m")
        return current_constraint

    if current_constraint is not None:
        return current_constraint

    cid = p.createConstraint(
        parentBodyUniqueId=baxter_id,
        parentLinkIndex=END_EFFECTOR_ID,
        childBodyUniqueId=obj_id,
        childLinkIndex=-1,
        jointType=p.JOINT_FIXED,
        jointAxis=[0, 0, 0],
        parentFramePosition=[0, 0, 0],
        childFramePosition=[0, 0, 0],
    )

    print("[GRASP] Objeto tomado.")
    return cid


def release_object(current_constraint):
    if current_constraint is not None:
        p.removeConstraint(current_constraint)
        print("[RELEASE] Objeto liberado.")

    return None


def apply_command(cmd, target, baxter_id, obj_id, constraint):
    cmd = cmd.upper()

    if cmd == "XP":
        target[0] += STEP
    elif cmd == "XM":
        target[0] -= STEP
    elif cmd == "YP":
        target[1] += STEP
    elif cmd == "YM":
        target[1] -= STEP
    elif cmd == "ZP":
        target[2] += STEP
    elif cmd == "ZM":
        target[2] -= STEP
    elif cmd == "HOME":
        target[:] = [0.20, 0.00, 0.35]
    elif cmd == "OPEN":
        print("[GRIPPER] OPEN")
    elif cmd == "CLOSE":
        print("[GRIPPER] CLOSE")
    elif cmd == "GRASP":
        # Para que el agarre sea realista, primero hay que posicionarse
        # sobre el objeto.
        constraint = grasp_object(baxter_id, obj_id, constraint)
    elif cmd == "RELEASE":
        constraint = release_object(constraint)
    elif cmd == "STOP":
        # Mantener la posición actual equivale a detener el movimiento.
        target[:] = get_ee_position(baxter_id)

    return target, constraint


def main():
    global running

    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--robot_root",
        default=".",
        help="Ruta al repositorio pybullet_robots",
    )
    args = parser.parse_args()

    p.connect(p.GUI)
    p.setAdditionalSearchPath(pybullet_data.getDataPath())

    p.resetSimulation()
    p.setGravity(0, 0, -9.81)

    p.loadURDF("plane.urdf", [0, 0, -1], useFixedBase=True)

    baxter_id = load_baxter(args.robot_root)
    obj_id = create_object()

    target = np.array([0.20, 0.00, 0.35])
    constraint = None

    p.addUserDebugText(
        "TARGET",
        target,
        textColorRGB=[1, 0, 0],
        textSize=1.5,
    )

    thread = threading.Thread(target=tcp_server, daemon=True)
    thread.start()

    print()
    print("=== PARTE B ===")
    print("Puerto TCP:", PORT)
    print("Comandos: XP XM YP YM ZP ZM OPEN CLOSE GRASP RELEASE HOME STOP")
    print()

    start = time.time()

    try:
        for _ in range(200000):
            for cmd in pop_commands():
                target, constraint = apply_command(
                    cmd,
                    target,
                    baxter_id,
                    obj_id,
                    constraint,
                )

            target[0] = np.clip(target[0], -0.3, 1.1)
            target[1] = np.clip(target[1], -1.2, 0.8)
            target[2] = np.clip(target[2], 0.05, 1.1)

            inverse_kinematics(baxter_id, target)

            p.stepSimulation()

            if int(time.time() * 10) % 10 == 0:
                pass

            time.sleep(1 / 240)

    except KeyboardInterrupt:
        print("\n[INFO] Simulación detenida.")

    finally:
        running = False
        p.disconnect()



    main()

