# Enjambre de 3 carritos con algoritmo de hormigas (ACO) en ESP32 y gemelo digital en PyBullet
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

