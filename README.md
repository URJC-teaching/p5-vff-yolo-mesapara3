[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/7dvYcpAO)
# p5_vff_yolo_follow_person

En esta práctica haremos que el robot busque y siga a una persona.

Para ello crearemos una máquina de estados con 2 estados: 
- Buscando: El robot girará sobre si mismo para buscar una persona. Si se detecta una persona, se pasa al estado Siguiendo.
- Siguiendo: El robot seguirá a la persona más grande que se detecte. Para ello se publicará un vector atractivo hacia esa persona que el robot seguirá. Si no se detecta a nadie durante 1 segundo, se pasa al estado Buscando.

Utiliza las instrucciones de la wiki de [asr-clase](https://github.com/URJC-teaching/asr-clase/tree/py) para lanzar yolo, y los nodos vff_control/vff_controller_node, vff_control/yolo_class_detector_node_2d.py y camera/yolo_detection_node.

Escribe un launch para lanzar todos los nodos necesarios con el comando ros2 launch.

## 🧠 Arquitectura del Sistema (VFF + YOLO)

Este paquete implementa el algoritmo **Virtual Force Field (VFF)**, una técnica de navegación reactiva que utiliza fuerzas vectoriales para el guiado del robot. El sistema combina visión artificial (YOLO) para la atracción y sensores de distancia (Láser) para la repulsión.

### 📋 Funcionamiento de los Nodos

1.  **Adaptador de YOLO ([yolo_detection_node.py](sensors/camera/camera/yolo_detection_node.py))**: Actúa como puente. Traduce los mensajes `DetectionArray` (propios de este nodo YOLO) al estándar de ROS 2 `vision_msgs/Detection2DArray`.
2.  **Atracción 2D ([yolo_class_detector_node_2d.py](vff_control/vff_control/yolo_class_detector_node_2d.py))**: 
    *   Busca el objeto objetivo (`target_class`).
    *   Usa la posición del píxel y los parámetros intrínsecos de la cámara (`camera_info`) para calcular el ángulo del objeto.
    *   Publica un **Vector Atractivo** (`/attractive_vector`) direccionado hacia el objetivo.
3.  **Repulsión ([obstacle_detector_node.py](vff_control/vff_control/obstacle_detector_node.py))**: Procesa los datos del láser para generar un **Vector Repulsivo** (`/repulsive_vector`) que aleja al robot de los obstáculos cercanos.
4.  **Controlador VFF ([vff_controller_node.py](vff_control/vff_control/vff_controller_node.py))**: Es el núcleo del control. Suma ambos vectores (Atracción + Repulsión) para obtener una resultante que se traduce en comandos de velocidad (`cmd_vel`).

---

## 🚀 Guía de Ejecución (Orden de Configuración)

Para que el sistema funcione, los nodos deben lanzarse en el siguiente orden para asegurar que los tópicos estén disponibles:

### 1. Cámara Física
Lanza el driver de tu webcam o cámara RGBD:
```bash
ros2 run v4l2_camera v4l2_camera_node --ros-args -p video_device:="/dev/video0" -p image_size:="[640,480]"
```

### 2. Inferencia YOLO (en el entorno adecuado)
Es fundamental activar el entorno virtual donde residen PyTorch y Ultralytics:
```bash
source ~/Documents/ROBOTICA/mp3_ws/venv_asr/bin/activate
ros2 launch yolo_bringup yolo.launch.py model:=yolov8n.pt input_image_topic:=/image_raw use_3d:=False use_tracking:=False device:=cpu
```

### 3. Adaptador de Mensajes y Control VFF
Lanza el sistema de navegación:
```bash
ros2 launch vff_control vff_2d.launch.py
```
*Nota: Asegúrate de que en el launch el `target_class` esté configurado como `'person'` y los remapeos coincidan con `/image_raw`.*

---

## 💡 Estrategia de Seguimiento 2D

Al no tener profundidad (3D), el robot utiliza el **área del Bounding Box** para estimar la cercanía:
*   **Lógica**: `bbox.size_x * bbox.size_y`. La caja más grande corresponde a la persona más cercana.
*   **Velocidades**: Para monitorizar el comportamiento, se recomienda cambiar el nivel de log en `vff_controller_node.py:99` de `debug` a `info` para ver las velocidades en tiempo real en la terminal.

---

## 🛠 Comandos Útiles de Depuración
*   **Ver imagen de la cámara**: `ros2 run rqt_image_view rqt_image_view`
*   **Ver detecciones**: `ros2 topic echo /yolo/detections`
*   **Ver vectores VFF**: `ros2 topic echo /attractive_vector`
