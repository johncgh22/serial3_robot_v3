# serial3_robot_v3

Paquete de ROS Noetic (Ubuntu 20.04) con la descripción y la configuración de un **robot serial de 3 GDL**: modelo URDF/Xacro, mallas 3D, controladores de `ros_control` y archivos de lanzamiento para visualizarlo en RViz y simularlo en Gazebo.

El robot tiene tres juntas rotacionales, cada una con un rango de ±90° (±1.5708 rad):

| Junta | Une | Esfuerzo máx. | Velocidad máx. |
|---|---|---|---|
| `base_to_e1_joint` | `base_link` → `e1_link` | 77 N·m | 1.43 rad/s |
| `e1_to_e2_joint` | `e1_link` → `e2_link` | 133 N·m | 1.29 rad/s |
| `e2_to_e3_joint` | `e2_link` → `e3_link` | 66 N·m | 2.13 rad/s |

`e3_link` se une al `end_effector` con una junta fija.

---

## 1. Requisitos

- Ubuntu 20.04
- ROS Noetic (versión `desktop-full`, que ya incluye RViz y Gazebo):

  ~~~bash
  sudo apt install ros-noetic-desktop-full
  ~~~

  Para más detalles, consulta la [Wiki de Instalación de ROS](http://wiki.ros.org/noetic/Installation/Ubuntu).

- Paquetes de control para la simulación:

  ~~~bash
  sudo apt install ros-noetic-ros-control ros-noetic-ros-controllers \
                   ros-noetic-gazebo-ros-control ros-noetic-joint-state-publisher-gui \
                   ros-noetic-xacro
  ~~~

## 2. Instalación

1. Crea un workspace de catkin (si aún no tienes uno):

   ~~~bash
   mkdir -p ~/catkin_ws/src
   cd ~/catkin_ws/src
   ~~~

2. Clona este repositorio dentro de `src`. La carpeta se tiene que llamar `serial3_robot_v3`, igual que el paquete:

   ~~~bash
   git clone https://github.com/johncgh22/serial3_robot_v3.git
   ~~~

3. Compila el workspace y carga el entorno:

   ~~~bash
   cd ~/catkin_ws
   catkin_make
   source devel/setup.bash
   ~~~

   Para no repetir el `source` en cada terminal nueva, agrégalo a tu `~/.bashrc`:

   ~~~bash
   echo "source ~/catkin_ws/devel/setup.bash" >> ~/.bashrc
   ~~~

4. Comprueba que ROS encuentra el paquete:

   ~~~bash
   rospack find serial3_robot_v3
   ~~~

## 3. Visualizar el robot en RViz

~~~bash
roslaunch serial3_robot_v3 display.launch
~~~

Se abren dos ventanas:
- **RViz**, con el modelo del robot ya configurado.
- **Joint State Publisher GUI**, con un deslizador por junta. Al moverlos, el robot se mueve en RViz.

Esto sirve para revisar el modelo y sus rangos de movimiento sin simular física.

## 4. Simular el robot en Gazebo

1. Lanza la simulación:

   ~~~bash
   roslaunch serial3_robot_v3 serial3v3_simulation.launch
   ~~~

   Este archivo:
   - abre Gazebo con un mundo vacío,
   - coloca el robot con todas las juntas en 0 rad,
   - carga un controlador de posición por esfuerzo (`effort_controllers/JointPositionController`) para cada junta,
   - publica el estado de las juntas en `/serial3v3/joint_states` y las transformaciones en `/tf`.

   Argumentos opcionales:

   | Argumento | Valor por defecto | Descripción |
   |---|---|---|
   | `gui` | `true` | Muestra la ventana de Gazebo |
   | `paused` | `false` | Inicia la simulación en pausa |
   | `x`, `y`, `z` | `0.0` | Posición inicial del robot |

   Ejemplo sin interfaz gráfica: `roslaunch serial3_robot_v3 serial3v3_simulation.launch gui:=false`

2. En otra terminal, comprueba que los controladores estén activos:

   ~~~bash
   rosservice call /serial3v3/controller_manager/list_controllers
   ~~~

   Deben aparecer `joint_state_controller` y los tres `*_joint_position_controller` en estado `running`.

3. Mueve una junta publicando la posición deseada (en radianes, entre -1.5708 y 1.5708):

   ~~~bash
   rostopic pub -1 /serial3v3/base_to_e1_joint_position_controller/command std_msgs/Float64 "data: 0.5"
   rostopic pub -1 /serial3v3/e1_to_e2_joint_position_controller/command std_msgs/Float64 "data: -0.8"
   rostopic pub -1 /serial3v3/e2_to_e3_joint_position_controller/command std_msgs/Float64 "data: 1.0"
   ~~~

4. Observa la respuesta de las juntas:

   ~~~bash
   rostopic echo /serial3v3/joint_states
   ~~~

   También puedes graficarla con `rqt_plot /serial3v3/joint_states/position[0]`.

### Ajustar las ganancias PID

Las ganancias de cada controlador están en [config/serial3v3_controllers.yaml](config/serial3v3_controllers.yaml) (campo `pid` de cada `*_joint_position_controller`). Para probarlas en vivo sin reiniciar la simulación:

~~~bash
rosrun rqt_reconfigure rqt_reconfigure
~~~

y selecciona `serial3v3/<controlador>/pid`. Cuando encuentres valores adecuados, cópialos al archivo YAML.

## 5. Estructura del paquete

~~~
serial3_robot_v3/
├── config/
│   ├── serial3v3_controllers.yaml   # Controladores de ros_control y sus ganancias PID
│   └── gazebo_pid.yaml              # Ganancias PID de gazebo_ros_control
├── launch/
│   ├── display.launch               # RViz + deslizadores de juntas
│   ├── serial3v3_simulation.launch  # Gazebo + controladores
│   ├── serial3v3_control.launch     # Solo carga controladores (Gazebo ya abierto)
│   ├── serial3v3_visualize.launch   # RViz con un modelo indicado por argumento
│   ├── gazebo.launch                # Gazebo con el modelo, sin controladores
│   ├── serial3v3_hardware.launch    # Robot físico (ver sección 6)
│   └── serial3v3_test_trajectory.launch
├── meshes/                          # Mallas STL (visual y colisión)
├── urdf/
│   ├── serial3_robot_v3.urdf        # Modelo que usan RViz y Gazebo
│   ├── serial3_robot_v3.xacro       # Modelo modular (usado por el hardware)
│   ├── serial3v3.gazebo.xacro       # Plugin de ros_control y materiales de Gazebo
│   ├── serial3v3_transmission.xacro # Transmisiones de los motores
│   └── materials.xacro              # Colores
└── world/serial3_w1                 # Mundo de Gazebo (SDF)
~~~

### Modificar el modelo

Las simulaciones cargan `urdf/serial3_robot_v3.urdf`. Si editas el Xacro, regenera el URDF y valídalo:

~~~bash
cd $(rospack find serial3_robot_v3)/urdf
xacro serial3_robot_v3.xacro > serial3_robot_v3.urdf
check_urdf serial3_robot_v3.urdf
~~~

## 6. Robot físico (opcional)

`serial3v3_hardware.launch` y `serial3v3_test_trajectory.launch` controlan el robot real. Usan una interfaz de hardware (`serial3v3_hw_main`) y un nodo de prueba (`test_trajectory`) del paquete [ros_control_boilerplate](https://github.com/PickNikRobotics/ros_control_boilerplate). **Ese código no forma parte de este repositorio**, así que hay que instalarlo y configurarlo por separado antes de usar estos archivos.

## 7. Solución de problemas

| Problema | Posible causa / solución |
|---|---|
| `Resource not found: serial3_robot_v3` | No hiciste `source devel/setup.bash`, o la carpeta no se llama `serial3_robot_v3`. |
| `Could not load controller ...` | Faltan los paquetes `ros-noetic-ros-controllers` o `ros-noetic-gazebo-ros-control`. |
| El robot se cae o tiembla en Gazebo | Las ganancias PID no son adecuadas. Ajústalas como se explica en la sección 4. |
| RViz no muestra el robot | En *Global Options*, pon `Fixed Frame` en `base_link`. |
