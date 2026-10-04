# CDR2026 Simulator

Simulateur du robot **Wall-A** pour la Coupe de France de Robotique 2026, basé sur [Webots](https://cyberbotics.com/) et **ROS 2** (via [`webots_ros2`](https://github.com/cyberbotics/webots_ros2)).

Il reproduit la table de jeu (3 m × 2 m), les éléments de jeu (tuiles bleues/jaunes, grenier, curseurs) et le robot, piloté comme le vrai : `cmd_vel` en entrée, odométrie, IMU, télémètre et caméras en sortie.

## Contenu du dépôt

| Dossier | Type | Rôle |
|---|---|---|
| [simulateur_robot_2016/](simulateur_robot_2016/) | package `ament_python` | Monde Webots, modèle du robot, launch files, nœuds Python |
| [robot_wall_a_msgs/](robot_wall_a_msgs/) | package `ament_cmake` | Messages ROS 2 personnalisés (`ColorRecognition`, `ColorRecognitionArray`) |

```
simulateur_robot_2016/
├── launch/
│   ├── robot_launch.py              # Lance Webots + le robot + ros2_control + TF
│   └── view_robot_rviz_launch.py    # Affiche l'URDF dans RViz (sans Webots)
├── resource/
│   ├── World_CDR_2026/              # Monde de la Coupe 2026
│   │   ├── World_CDR_2026.wbt
│   │   ├── proto/                   # Table, tuiles, grenier, curseurs
│   │   └── meshes/
│   ├── Robot/                       # Modèle du robot
│   │   ├── urdf/                    # URDF brut et URDF Webots (plugins + ros2_control)
│   │   ├── proto/                   # PROTO Webots (Assemblage_Carcasse, capteur de distance)
│   │   ├── meshes/{visual,collision}/
│   │   └── ros2control.yml          # Paramètres des contrôleurs
│   └── view.rviz                    # Configuration RViz
└── simulateur_robot_2016/           # Code Python
    ├── robot_spawner.py             # Fait apparaître le robot dans le monde
    ├── color_recognition.py         # Cameras -> couleur des tuiles
    ├── read_robot_position_in_tf_map.py
    └── plugin_example.py            # Exemple de plugin Webots (copie du modèle Cyberbotics)
```

## Prérequis

- **ROS 2** avec `colcon`. Le launch gère les distributions où `diff_drive_controller` attend un `TwistStamped` (`rolling`, `jazzy`, `kilted`) et les autres.
- **Webots R2025a** (version des fichiers `.wbt` / `.proto`).
- Paquets ROS 2 : `webots_ros2_driver`, `webots_ros2_control`, `webots_ros2_msgs`, `diff_drive_controller`, `joint_state_broadcaster`, `position_controllers`, `controller_manager`, `robot_state_publisher`, `tf2_ros`, `rviz2`.
- Pour `read_robot_position_in_tf_map.py` : `tf_transformations`.
- Pour `view_robot_rviz_launch.py` avec `use_joint_state_pub:=True` : `joint_state_publisher_gui`.
- Une connexion internet au premier lancement : le monde charge `Floor`, `TexturedBackground` et `TexturedBackgroundLight` depuis le GitHub de Webots.

## Compilation

Le dossier `Similator` peut servir de workspace colcon.

```bash
cd Similator
colcon build --packages-select robot_wall_a_msgs simulateur_robot_2016
```

Puis sourcer le workspace :

```bash
source install/setup.bash
```

## Lancer la simulation

```bash
ros2 launch simulateur_robot_2016 robot_launch.py
```

Arguments :

| Argument | Défaut | Description |
|---|---|---|
| `world` | `World_CDR_2026.wbt` | Fichier monde, cherché dans `resource/World_CDR_2026/` |
| `mode` | `realtime` | Mode de démarrage de Webots (`realtime`, `fast`, `pause`) |

### Choisir le côté (jaune / bleu)

Le côté n'est pas un argument du launch : modifier `ROBOT_COLOR` en tête de [robot_launch.py](simulateur_robot_2016/launch/robot_launch.py) (`"YELLOW"` ou `"BLUE"`), puis recompiler.

| Côté | Position de départ (x, y) | Orientation |
|---|---|---|
| `YELLOW` | 0.3 m, 1.775 m | −1.57 rad |
| `BLUE` | 2.7 m, 1.775 m | −1.57 rad |

Le même choix règle la transformation statique `map → odom`, donc la pose du robot dans `map` est directement celle sur la table.

### Ce que lance `robot_launch.py`

1. Webots avec le monde choisi et le `Ros2Supervisor`.
2. `robot_spawner` : appelle `/Ros2Supervisor/spawn_node_from_string` pour créer le robot à sa position de départ.
3. Les TF statiques `map → odom` et `base_link → base_footprint`.
4. Le contrôleur Webots du robot (`WebotsController`) avec l'URDF `Assemblage_Carcasse_webots.urdf` et `ros2control.yml`.
5. Les contrôleurs ros2_control : `diffdrive_controller`, `joint_state_broadcaster`, `position_controller`, démarrés une fois le contrôleur Webots connecté.

La fermeture de Webots arrête tous les nœuds.

## Le robot dans ROS 2

| Élément | Interface | Détails |
|---|---|---|
| Déplacement | `/cmd_vel` | Robot différentiel : roues de 32,5 mm de rayon, entraxe 215 mm, vitesse linéaire limitée à 0,2 m/s |
| Odométrie | `/odom` + TF `odom → base_link` | Publiée par `diff_drive_controller` à 50 Hz |
| Bras (pinces) | `/position_controller/commands` | Commande en position de `arm_left_joint` et `arm_right_joint` |
| Articulations | `/joint_states` | Publiées par `joint_state_broadcaster` |
| IMU | `/imu` | Frame `imu_link` |
| Télémètre | `/range_meter` | Capteur de distance `range_meter_1`, 1 rayon |
| Caméras | `/camera_1` … `/camera_4` | 4 caméras de 10 × 10 px, champ de vision 0,01 rad, alignées à l'avant du robot (espacées de 5 cm). Reconnaissance d'objets activée, portée 10 cm |

Exemple, faire avancer le robot (distributions qui n'utilisent pas `TwistStamped`) :

```bash
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.1}}"
```

## La table de jeu

Le monde [World_CDR_2026.wbt](simulateur_robot_2016/resource/World_CDR_2026/World_CDR_2026.wbt) contient :

- la table (`arena_cdr2026`) de 3 m × 2 m avec ses bordures et l'image de fond ;
- le grenier (`granary`) ;
- 8 groupes de 4 tuiles (`tiles_group`) : 2 bleues et 2 jaunes. **L'ordre des tuiles dans chaque groupe est tiré au hasard à chaque démarrage** ;
- 2 curseurs (`cursor_1`, `cursor_2`).

## Détection de la couleur des tuiles

`color_recognition.py` écoute `/camera_N/recognitions/webots` (N = 1 à 4), convertit la couleur de l'objet reconnu (bleu `(0,0,1)`, jaune `(1,1,0)`) et publie le résultat sur `tiles_color` (`robot_wall_a_msgs/ColorRecognitionArray`, une entrée par caméra).

Message `ColorRecognition` :

| Champ | Type | Valeurs |
|---|---|---|
| `object_id` | `int32` | Identifiant Webots de l'objet reconnu |
| `color` | `int8` | `UNKNOWN = 0`, `YELLOW = 1`, `BLUE = 2` |

```bash
python3 simulateur_robot_2016/simulateur_robot_2016/color_recognition.py
```

## Outils

**Position du robot dans `map`** : `read_robot_position_in_tf_map.py` affiche 10 fois par seconde `x`, `y` et `theta` de `base_link` dans `map`.

**Visualiser le robot dans RViz** (sans Webots) :

```bash
ros2 launch simulateur_robot_2016 view_robot_rviz_launch.py
```

Arguments : `rviz_config_file`, `urdf_file`, `use_rviz` (`True`), `use_robot_state_pub` (`True`), `use_joint_state_pub` (`False`).

## Modifier le modèle du robot

La chaîne SolidWorks → STL / URDF → PROTO Webots, avec les retouches manuelles à faire à chaque étape (échelle des maillages, géométries de collision, capteurs de distance), est décrite dans [simulateur_robot_2016/README.md](simulateur_robot_2016/README.md).

## Limitations connues

- `package.xml` de `simulateur_robot_2016` déclare une dépendance `simulateur_robot_2016_msgs` alors que le package de messages s'appelle `robot_wall_a_msgs`. `color_recognition.py` utilise ce dernier sans qu'il soit déclaré.
- Seul `robot_spawner` est enregistré comme exécutable dans `setup.py` : `color_recognition.py` et `read_robot_position_in_tf_map.py` ne se lancent pas avec `ros2 run`.
- Le côté du robot est codé en dur dans `robot_launch.py` (voir plus haut).
- La description de l'argument `world` dans `robot_launch.py` mentionne un dossier `resource/worlds` qui n'existe pas.

## Licence

Apache-2.0 (voir [simulateur_robot_2016/LICENSE](simulateur_robot_2016/LICENSE)). `plugin_example.py` provient de Cyberbotics, sous la même licence.
