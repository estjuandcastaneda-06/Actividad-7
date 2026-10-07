# Enjambre de 3 carritos con algoritmo de hormigas (ACO) en ESP32 y gemelo digital en PyBullet

Tres nodos ESP32 buscan la ruta óptima desde un punto de partida (**S**) hasta una meta (**G**) en un laberinto.
Cada ESP32 ejecuta un algoritmo de optimización por colonia de hormigas (ACO), los nodos intercambian feromonas
por WiFi y una simulación en PyBullet (dentro de Docker) muestra 3 robots virtuales que replican el comportamiento.

## Arquitectura

```
 MUNDO FÍSICO                 PARTE DE RED                    PARTE VIRTUAL
 ┌──────────────┐        ┌──────────────────────┐        ┌───────────────────────┐
 │ 3 x ESP32    │  UDP   │ ESP32 nodo 0         │  UDP   │ Docker: twin.py       │
 │ (ACO local)  │◄──────►│ en modo AP           │───────►│ PyBullet + visor web  │
 └──────────────┘        │ WiFi "ACO_SWARM"     │        │ http://localhost:8000 │
                         └──────────────────────┘        └───────────────────────┘
```

| Bloque del diagrama | Qué lo implementa |
|---|---|
| Mundo físico | `firmware/aco_swarm/aco_swarm.ino` cargado en 3 ESP32 |
| Parte de red (ESP modo AP) | El ESP32 con `NODE_ID 0` crea el WiFi y los otros dos se conectan a él |
| Parte virtual (Docker) | `sim/twin.py` dentro del contenedor definido por `sim/Dockerfile` y `docker-compose.yml` |

## Estructura del repositorio

```
aco-swarm/
├── README.md
├── .gitignore
├── docker-compose.yml
├── firmware/
│   └── aco_swarm/
│       └── aco_swarm.ino
└── sim/
    ├── Dockerfile
    ├── requirements.txt
    ├── maze.py
    ├── aco.py
    ├── emulator.py
    └── twin.py
```

## Explicación de cada archivo

### `firmware/aco_swarm/aco_swarm.ino`
Programa que corre en cada ESP32. Hace tres cosas:
1. **Red:** si `NODE_ID` es 0, crea el punto de acceso WiFi `ACO_SWARM` (modo AP). Si es 1 o 2, se conecta a ese WiFi. Todos escuchan y envían por UDP en el puerto 4210 usando broadcast (`192.168.4.255`).
2. **ACO local:** cada 500 ms lanza 10 hormigas desde S hacia G. Cada hormiga elige su siguiente celda según la feromona de cada camino y la cercanía a la meta, sin repetir celdas. Luego evapora feromona y refuerza la mejor ruta de esa iteración.
3. **Intercambio de feromonas:** envía su mejor ruta a los demás nodos. Cuando recibe la ruta de otro nodo, deposita feromona sobre ella.

La función `onBestPath()` es el punto donde se conectan los motores del carrito físico. Por ahora solo imprime la ruta por el monitor serial.

Mensaje UDP: `P;<id_nodo>;<iteración>;<pasos>;c0,c1,c2,...`, donde cada celda es `fila*10 + columna`.

### `sim/maze.py`
Define el laberinto de 10x10 (`#` pared, `.` libre, `S` inicio, `G` meta) y funciones auxiliares (vecinos de una celda, conversión celda a fila/columna). Lo usan `aco.py`, `emulator.py` y `twin.py`. El mismo laberinto está copiado en el `.ino`.

### `sim/aco.py`
Implementación en Python del mismo ACO del firmware (clase `ACO`): construcción de rutas por hormigas, depósito y evaporación de feromona. Lo usa el emulador.

### `sim/emulator.py`
Simula 3 ESP32 en tu computador. Corre 3 instancias de `ACO`, se pasan feromonas entre ellas y envían sus rutas por UDP con el mismo formato que el firmware. Sirve para probar el gemelo digital sin tener las placas.

### `sim/twin.py`
Gemelo digital:
- Un hilo escucha UDP en el puerto 4210 y guarda la mejor ruta de cada nodo.
- Con PyBullet construye el laberinto, el punto de partida, la meta y 3 carritos virtuales (rojo, azul y verde, uno por nodo).
- Cada carrito recorre la mejor ruta de su nodo y, al llegar, vuelve a empezar con la mejor ruta más reciente. Las baldosas se colorean más intensamente donde hay más feromona.
- Genera imágenes con la cámara de PyBullet y las publica como video en `http://localhost:8000`, por eso funciona dentro de Docker sin ventana.
- Con `--gui` abre la ventana normal de PyBullet (fuera de Docker).

### `sim/requirements.txt`
Dependencias de Python: `pybullet`, `numpy`, `pillow`.

### `sim/Dockerfile`
Construye la imagen del gemelo digital: parte de Python 3.11, instala el compilador (PyBullet se compila al instalarse), instala las dependencias y ejecuta `twin.py`. La primera construcción tarda varios minutos.

### `docker-compose.yml`
Define dos servicios:
- `twin`: el gemelo digital. Expone el puerto 8000 (visor web) y el 4210/UDP (datos de los ESP32).
- `emulator`: 3 ESP32 virtuales. Solo se inicia con `--profile emulator`.

### `.gitignore`
Evita subir archivos temporales (`__pycache__`, etc.).

## Cómo usarlo

### Opción A: sin hardware (emulador)
```bash
docker compose --profile emulator up --build
```
Abre http://localhost:8000. Verás los 3 carritos recorriendo el laberinto y la mejor ruta de cada nodo bajando hasta 18 pasos.

### Opción B: con los 3 ESP32
1. En Arduino IDE instala el paquete **esp32 by Espressif** y selecciona la placa `ESP32 Dev Module`.
2. Abre `aco_swarm.ino`, deja `#define NODE_ID 0` y carga la primera placa. Repite con `NODE_ID 1` y `NODE_ID 2` en las otras dos.
3. Enciende primero el nodo 0 (crea el WiFi). Los nodos 1 y 2 se conectan solos.
4. Conecta tu PC al WiFi `ACO_SWARM` (contraseña `aco12345`).
5. Ejecuta:
   ```bash
   docker compose up --build
   ```
6. Abre http://localhost:8000. Permite el puerto UDP 4210 en el firewall de tu PC.

### Opción C: sin Docker (ventana de PyBullet)
```bash
cd sim
pip install -r requirements.txt
python twin.py --gui
python emulator.py
```
Ejecuta `emulator.py` en otra terminal.

## Parámetros del ACO
Están al inicio de `aco.py` y en `aco_swarm.ino` (deben coincidir):

| Parámetro | Valor | Significado |
|---|---|---|
| `ALPHA` | 1.0 | Peso de la feromona |
| `BETA` | 1.0 | Peso de la cercanía a la meta |
| `RHO` | 0.10 | Evaporación por iteración |
| `Q` | 10 | Cantidad de feromona depositada (Q / largo de la ruta) |
| `ANTS` | 10 | Hormigas por iteración |

Para cambiar el laberinto, edita `MAZE` en `maze.py` y el mismo arreglo (junto con `R` y `C`) en el `.ino`.

## Solución de problemas
- **El visor dice "Esperando datos":** el contenedor no recibe UDP. Docker Desktop a veces no reenvía broadcast; ejecuta `python twin.py` directamente en el PC, o en Linux usa `network_mode: host` en `docker-compose.yml`.
- **Un ESP32 no se conecta:** revisa que el nodo 0 esté encendido y que SSID y contraseña coincidan en los tres.
- **La vista aparece rotada:** en `twin.py` cambia el yaw `0` por `90` en `computeViewMatrixFromYawPitchRoll`.
