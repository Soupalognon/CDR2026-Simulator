## Conversion SolidWorks vers urdf
### Installation
https://wiki.ros.org/sw_urdf_exporter

### Tuto
https://www.youtube.com/watch?v=H6YPkXmkdPg

Après avoir récupéré le dossier avec l'urdf et les mesh
Ranger l'urdf dans simulateur_robot_2016/resource/urdf
Ranger les mesh dans simulateur_robot_2016/resource/meshes/visual



## Conversion de Solidworks vers stl pour le visuel
### Etape 1: Modifier le scaling du stl
Quand on exporte un fichier stl sur Solidworks l'unité est le mm alors que webots utilise le mètre.
Il faut exporter son fichier sur Solidworks et l'importer dans Meshlab.
Ensuite il faut aller dans Filter -> Normals,Curves and Orientations -> Transform: Scale,Normalize
Mettre pour les 3 axes 0.001 et cliquer sur Apply.
https://www.universal-robots.com/articles/ur/actinav/actinav-how-to-use-meshlab-to-resize-your-part-models/

### Etape 2: Modifier l'origine du stl
On veut que l'origine du stl soir au centre.
On aller dans Filter -> Normals, Curvatures and Orientation -> Transform: Translate, Center, Set Origin
Dans la liste déroulante "Transformation" --> Center on Layer BBox
Cliquer Apply.
https://revthat.com/updating-origin-meshes-meshlab/

### Etape 3: Exporter
Ensuite il faut aller dans File -> Export mesh as...
Maintenant le fichier est importable dans Webots

## Conversion urdf vers proto (webots)
### Installation
https://github.com/cyberbotics/urdf2webots/tree/master

### Tuto
Déjà il y a un git qui utilise webots et qui est indispensable
https://github.com/cyberbotics/webots_ros2

J'ai beaucoup utilisé turtlebot, e-puck et husarion

La ligne que j'utilise dans mon projet est:

```bash
python3 -m urdf2webots.importer --input=Assemblage_Carcasse.urdf --output=../proto/Assemblage_Carcasse.proto
```



## Créer un stl simplifié
### Installation
Installer meshlab

### Tuto
Voici un tuto pour comprendre et résoudre une partie du problème
https://adamconkey.medium.com/collision-meshes-for-simulating-robots-in-gazebo-ccc647d8b87d

Ranger tous les stl simplifié créés dans le dossier `simulateur_robot_2016/resource/meshes/collision`





## Spécificité pour ce projet

### URDF
Après avoir converti le Solidworks en urdf il faut maintenant modifier l'urdf
Voici la liste des changements:
- Modifier les chemins des stl visuel, mettre

  ```xml
  <visual>
      <geometry>
          <mesh filename="package://simulateur_robot_2016/resource/meshes/visual/[mesh_name].stl" />
      </geometry>
  </visual>
  ```

- Modifier les chemins des stl collision, mettre

  ```xml
  <collision>
      <geometry>
          <mesh filename="package://simulateur_robot_2016/resource/meshes/collision/[mesh_name].stl" />
      </geometry>
  </collision>
  ```

- Créer un stl simplifié pour chaque objet qui ne touche pas le sol
  - référer vous à la section "créer un stl simplifié"

- Modifier toutes les géométries des objets qui sont en contact avec le sol par des géométries simple. Exemple:
  - tous les caster (roulements à bille)
    - Dans le link/visual/geometry et link/collision/geometry la géométrie de l'objet doit être une sphère simple

      ```xml
      <link name="caster_link">
          <visual>
              <geometry>
              <sphere radius="0.005"/>
              </geometry>
          </visual>
          <collision>
              <geometry>
              <sphere radius="0.005"/>
              </geometry>
          </collision>
      </link>
      ```

  - les roues sont des cylindres, modifier comme pour les caster

    ```xml
    <link name="wheel_link">
        <visual>
            <origin
                xyz="0 0 0"
                rpy="1.57 0 0" />
            <geometry>
            <cylinder length="0.030" radius="0.0325"/>
            </geometry>
        </visual>
        <collision>
            <origin
                xyz="0 0 0"
                rpy="1.57 0 0" />
            <geometry>
            <cylinder length="0.030" radius="0.0325"/>
            </geometry>
        </collision>
    </link>
    ```

  - les bras sont des box, modifier: (Attention si les pinces ont bougées dans le solidwork alors la box de collision sera à ajuster...)
    (Le mieux serai de modifier le arm_link pour être uniquement la plaque et non le support et la plaque)

    ```xml
    <link name="arm_left_link">
        <collision>
          <origin xyz="-0.0255 0.044 -0.005" rpy="0 0 -0.44"/>
          <geometry>
            <box size="0.12 0.006 0.08"/>
          </geometry>
        </collision>
    </link>

    <link name="arm_right_link">
        <collision>
            <origin xyz="-0.0255 -0.044 -0.005" rpy="0 0 0.435"/>
            <geometry>
                <box size="0.12 0.006 0.08"/>
            </geometry>
        </collision>
    </link>
    ```

- Si vous voulez un imu --> Mettre à la fin du fichier

  ```xml
  <gazebo>
      <plugin name="imu_plugin" filename="libgazebo_ros_imu.so">
          <topicName>imu</topicName>
          <gaussianNoise>0.0</gaussianNoise>
      </plugin>
  </gazebo>
  ```

### PROTO
L'imu se créer tout seul mais pas les distance mètre...
Il faut donc les rajouter à la main une fois le fichier généré:
- Au début du fichier, rajouter l'import de l'objet

  ```
  EXTERNPROTO "distance_sensor.proto"
  ```

  ça ressemble en gros à ça à la fin:

  ```
  #Extracted ...
  EXTERNPROTO "distance_sensor.proto"
  PROTO Assemblage_Carcasse
  ...
  ```

- Dans chaque Solid nommé "distance_sensorX_link" (il y aura un numéro à la place du X), ajouter au dessus du Shape l'objet:

  ```
  DEF EPUCK_PS0 distance_sensor {
  translation 0 0 0
  rotation 0 0 0 0
  name "range_meter_1"
  numberOfRays 1
  }
  ```

  ça ressemblera à ça:

  ```
  Solid {
      ...
      children [
          DEF EPUCK_PS0 distance_sensor {
          translation 0 0 0
          rotation 0 0 0 0
          name "range_meter_1"
          numberOfRays 1
          }
          Shape ...
      ]
      name ...
  }
  ```
