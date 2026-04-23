# Oscar-work
Este repositorio abarca la entrega realizada para el profesor Oscar para la asignatura: TE3003B.501

# Exploración autónoma de Robotino en entorno interior usando SLAM, RRT y LiDAR
- Video de la simulación en funcionamiento: https://youtu.be/iwGmthLcTfo
  
## Integrantes
- Daniel Alejandro De La Peña Rosales A01658540
- Natalia Ameyali Zarco Heras A01663337
- Gael González Arbesú A01611800
- Rebecca Flores Mora A01654281

---

## Descripción general

En este proyecto se implementó un sistema de exploración autónoma para un robot móvil tipo **Robotino** en un entorno interior simulado, utilizando **ROS 2**, **Webots**, **SLAM**, planeación de trayectoria mediante **RRT (Rapidly-exploring Random Tree)**, seguimiento de trayectoria y detección de obstáculos locales con **LiDAR**.

El sistema permite que el robot:

- construya un mapa del entorno,
- identifique regiones no exploradas,
- seleccione metas de exploración de manera autónoma,
- calcule trayectorias seguras,
- siga dichas trayectorias,
- detecte obstáculos pequeños o difíciles de representar en el mapa global,
- y finalmente regrese a su posición inicial.

---

## Objetivos

Los objetivos principales de la implementación fueron:

1. Elaborar un mapa de ocupación del entorno.
2. Explorar de forma autónoma regiones no visitadas.
3. Estimar trayectorias usando un algoritmo de planeación.
4. Seguir dichas trayectorias con control de movimiento.
5. Detectar patas de sillas, mesas u obstáculos pequeños a partir del LiDAR.
6. Integrar todo en una arquitectura funcional dentro del repositorio del equipo.

---

## Herramientas utilizadas

- **ROS 2 Jazzy**
- **Webots**
- **Python**
- **slam_toolbox**
- **RViz2**
- **Robotino Webots package**
- **nav2_map_server** para guardado del mapa

---

## Arquitectura general

La solución se construyó a partir de una infraestructura base de simulación y mapeo, más una capa de nodos desarrollados por el equipo para la exploración y navegación.

### Infraestructura base utilizada

Se utilizó el launch:

- `slam_mapping.launch.py`

Este launch se encarga de:

- abrir el entorno en Webots,
- inicializar el controlador del Robotino,
- publicar odometría y transformaciones,
- ejecutar `slam_toolbox`,
- mostrar la visualización en RViz.

### Nodos implementados por el equipo

Los nodos principales desarrollados para la solución fueron:

- `exploration_manager`
- `rrt_planner`
- `path_follower`
- `leg_detector`

---

## Flujo general de funcionamiento

El sistema opera de la siguiente manera:

1. **SLAM** genera un mapa global de ocupación del entorno.
2. El nodo **`exploration_manager`** analiza ese mapa y selecciona una meta de exploración a partir de las fronteras entre espacio conocido y desconocido.
3. El nodo **`rrt_planner`** recibe la meta y calcula una trayectoria libre de colisión mediante **RRT**.
4. El nodo **`path_follower`** sigue dicha trayectoria generando comandos de velocidad.
5. El nodo **`leg_detector`** analiza el LiDAR para detectar obstáculos pequeños, como patas de muebles, y genera un mapa local de obstáculos.
6. Si una meta resulta no navegable, el sistema la descarta y selecciona una nueva.
7. Cuando ya no quedan fronteras válidas, el robot regresa a su posición inicial.

---

## Descripción de los nodos implementados

## 1. `exploration_manager`

Este nodo funciona como el **administrador de alto nivel** de la exploración.

### Función
Selecciona la siguiente meta hacia la cual debe desplazarse el robot para continuar explorando el entorno.

### Funcionamiento conceptual
- Lee el mapa global.
- Detecta **fronteras**, es decir, zonas donde una celda libre es vecina de una celda desconocida.
- Evalúa cuáles fronteras son utilizables.
- Selecciona una meta de exploración.
- Supervisa si esa meta fue alcanzada, falló o debe descartarse.
- Controla estados globales como:
  - `EXPLORE`
  - `RETURN_HOME`
  - `CHARGING`

### Entradas
- `/map`
- `/path_follower/status`
- `/rrt_planner/status`
- transformaciones `TF`

### Salidas
- `/exploration_goal`
- `/goal_pose`
- `/exploration_state`

### Rol dentro del sistema
Define **qué región del entorno conviene explorar a continuación**.

---

## 2. `rrt_planner`

Este nodo es el **planeador de trayectoria**.

### Función
Calcular una trayectoria segura desde la posición actual del robot hasta la meta seleccionada.

### Funcionamiento conceptual
Se implementó el algoritmo **RRT (Rapidly-exploring Random Tree)**, que construye un árbol dentro del espacio libre del mapa hasta conectar con la meta.

Además, el nodo:
- valida la posición inicial del robot,
- corrige metas o inicios que caen en celdas inválidas,
- verifica colisiones usando el mapa global y el mapa local,
- suaviza la trayectoria final,
- y replantea si el robot se bloquea.

### Entradas
- `/map`
- `/local_obstacle_map`
- `/exploration_goal`
- `/exploration_state`
- `/path_follower/status`
- transformaciones `TF`

### Salidas
- `/planned_path`
- `/rrt_planner/status`

### Rol dentro del sistema
Define **cómo llegar a la meta sin colisionar**.

---

## 3. `path_follower`

Este nodo es el **seguidor de trayectoria**.

### Función
Convertir una trayectoria planeada en velocidades lineales y angulares para mover al robot.

### Funcionamiento conceptual
- Recibe una trayectoria como una secuencia de puntos.
- Selecciona un punto adelantado (*lookahead*) sobre la trayectoria.
- Calcula el error angular y la distancia hacia ese punto.
- Genera comandos de movimiento.
- Usa el LiDAR para desacelerar, evitar o detenerse ante obstáculos cercanos.

### Entradas
- `/planned_path`
- `/scan`
- `/exploration_state`
- transformaciones `TF`

### Salidas
- `/cmd_vel`
- `/path_follower/status`

### Rol dentro del sistema
Convierte la trayectoria en **movimiento real del robot**.

---

## 4. `leg_detector`

Este nodo corresponde a la **percepción local de obstáculos**.

### Función
Detectar patas de muebles y otros obstáculos pequeños con el LiDAR, para representarlos como obstáculos locales adicionales.

### Funcionamiento conceptual
- Convierte la lectura del LiDAR a puntos 2D.
- Agrupa puntos cercanos en clusters.
- Evalúa el tamaño geométrico de cada cluster.
- Identifica clusters compatibles con patas de sillas o mesas.
- Da persistencia temporal a las detecciones para reducir falsos positivos.
- Agrupa patas cercanas para inferir muebles más grandes.
- Genera un mapa local de obstáculos inflado.

### Entradas
- `/scan`
- transformaciones `TF`

### Salidas
- `/local_obstacle_map`

### Rol dentro del sistema
Permite que el planeador y el seguidor consideren **obstáculos pequeños o locales** que el mapa global puede no representar con suficiente precisión.

---

## Algoritmo utilizado

Para cumplir con el requisito de implementar uno de los algoritmos propuestos, se utilizó:

## **RRT (Rapidly-exploring Random Tree)**

Este algoritmo fue implementado en el nodo `rrt_planner`.

### Motivo de elección
Se eligió RRT porque:
- permite planeación en espacios complejos,
- funciona adecuadamente en mapas de ocupación,
- no necesita una red fija de caminos,
- y es robusto para entornos interiores con obstáculos.

---

## Detección de patas de silla o mesa

Para cubrir este requisito, se desarrolló el nodo `leg_detector`.

### Estrategia implementada
1. Adquisición del `LaserScan`.
2. Conversión a puntos 2D.
3. Agrupamiento de puntos en clusters.
4. Evaluación geométrica del tamaño del cluster.
5. Filtrado de candidatos compatibles con patas.
6. Confirmación temporal mediante seguimiento.
7. Agrupación de patas cercanas para inferir muebles.
8. Publicación en un mapa local de obstáculos.

### Importancia
Esto mejora la navegación en entornos interiores, ya que algunos obstáculos delgados o pequeños pueden no quedar bien reflejados en el mapa global generado por SLAM.

---

## Topics principales

### Subscripciones
- `/map`
- `/scan`
- `/exploration_goal`
- `/planned_path`
- `/path_follower/status`
- `/rrt_planner/status`
- `/exploration_state`

### Publicaciones
- `/exploration_goal`
- `/goal_pose`
- `/planned_path`
- `/cmd_vel`
- `/local_obstacle_map`
- `/path_follower/status`
- `/rrt_planner/status`
- `/exploration_state`

---

## Estados del sistema

La lógica de exploración contempla los siguientes estados:

### `EXPLORE`
El robot busca y visita regiones no exploradas.

### `RETURN_HOME`
Cuando ya no se detectan fronteras válidas o se activa la lógica de retorno, el robot vuelve a la posición inicial.

### `CHARGING`
Estado lógico posterior al retorno, utilizado como parte de la administración de la misión.

---

## Ejecución del sistema

### 1. Lanzar simulación y SLAM
```bash
ros2 launch robotino_webots slam_mapping.launch.py

2. Ejecutar detector de obstáculos locales
ros2 run robot_movement leg_detector
3. Ejecutar planeador RRT
ros2 run robot_movement rrt_planner
4. Ejecutar seguidor de trayectoria
ros2 run robot_movement path_follower
5. Ejecutar administrador de exploración
ros2 run robot_movement exploration_manager
Guardado del mapa

Una vez finalizada la exploración y con el sistema aún en ejecución, el mapa puede guardarse con:

ros2 run nav2_map_server map_saver_cli -f ~/mapa_robotino

Esto genera normalmente:

~/mapa_robotino.yaml
~/mapa_robotino.pgm
Resultados obtenidos

Durante las pruebas en simulación se logró que el robot:

construyera un mapa del entorno,
identificara regiones no exploradas,
seleccionara metas de forma autónoma,
generara trayectorias con RRT,
siguiera dichas trayectorias,
evitara obstáculos cercanos con el LiDAR,
detectara obstáculos pequeños mediante un mapa local,
y regresara a la posición inicial al finalizar la exploración.
Problemas encontrados y mejoras realizadas

Durante el desarrollo se identificaron varios problemas prácticos, entre ellos:

metas de exploración no navegables,
poses iniciales inválidas para el planeador,
zonas estrechas aparentemente transitables en el mapa,
atascos entre muebles,
y regiones residuales sin explorar.

Para mejorar el sistema se realizaron ajustes como:

aumento del margen de seguridad alrededor de obstáculos,
corrección automática del inicio y la meta en el planeador,
blacklist de regiones problemáticas,
seguimiento reactivo con LiDAR,
y generación de un mapa local de obstáculos.

Conclusión

Se desarrolló un sistema modular de exploración autónoma para un robot móvil en entorno interior, integrando percepción, planeación y control.

La solución combina:

mapeo por SLAM,
selección autónoma de metas,
planeación de trayectorias con RRT,
seguimiento de trayectoria,
y detección local de obstáculos con LiDAR.

De esta manera se cumple con los requisitos principales de la actividad y se demuestra una arquitectura robótica funcional orientada a exploración autónoma en interiores.
