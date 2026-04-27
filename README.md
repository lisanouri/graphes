# graphes

```mermaid
graph TD
    subgraph "Navigation Stack (Nav2)"
        BT[BT Navigator] --> |Request Plan| LP[LaneletPlanner Plugin]
        LP --> |nav_msgs/srv/GetPlan| LMS
    end

    subgraph "Sémantique & Cartographie"
        LMS[lanelet_map_server] --> |Load| OSM[.osm File]
        LMS --> |Publish| VM[visualization_msgs/MarkerArray]
        LMS --> |Transform| UTM[UTM Projection Engine]
    end

    subgraph "Simulation (Gazebo)"
        LRM[lanelet_random_manager] --> |Request Random Pose| LMS
        LRM --> |ros_gz_interfaces/srv/SetEntityPose| GZ((Gazebo Engine))
    end

    VM -.-> RViz[RViz Visualizer]
    LP --> |nav_msgs/msg/Path| Controller[Local Controller]
```


```mermaid
sequenceDiagram
    participant Config as YAML/Launch
    participant LRM as lanelet_random_manager
    participant LMS as lanelet_map_server
    participant GZ as Gazebo (Bridge)

    Config->>LRM: Spawn Parameter (n_agents=5)
    loop Pour chaque agent
        LRM->>LMS: Service: get_random_lanelet_pose
        LMS-->>LRM: Pose (x, y, yaw) sur centerline
        LRM->>GZ: Service: SetEntityPose
    end
    loop Update Loop
        LRM->>LRM: Calcul cinématique (vitesse constante)
        LRM->>GZ: Update Entity Pose
    end
```

```mermaid
graph LR
    subgraph "Perception"
        Cam[Camera Feed] --> YOLO[YOLOv8 Detector]
        YOLO --> Det[Detected Traffic Light]
    end

    subgraph "Map Validation"
        LMS[lanelet_map_server] --> |Current Lane| RE[Regulatory Element ID]
        RE --> Match{ID Match?}
    end

    Det --> Match
    Match -->|Yes| Control[Action: Stop/Go]
    Match -->|No| Ignore[Ignore: Secondary Light]
```

```mermaid
graph TD
    subgraph Couche_Semantique [Couche Semantique - Regles]
        RE[Regulatory Element - ex. Traffic Light / Stop / Speed Limit]
    end

    subgraph Couche_Topologique [Couche Topologique - Voies]
        LL[Lanelet - Portion de voie navigable]
    end

    subgraph Couche_Geometrique [Couche Geometrique - Lignes]
        LS_L[Linestring - Bord gauche]
        LS_R[Linestring - Bord droit]
        LS_C[Linestring - Centerline calculee]
    end

    subgraph Couche_Physique [Couche Physique - Points]
        P[Points 3D - ID / Latitude / Longitude / Elevation]
    end

    %% Relations
    P --> LS_L
    P --> LS_R
    LS_L --> LL
    LS_R --> LL
    LS_C -.-> LL
    LL <--> RE
```

```mermaid
graph TD
    subgraph "1. LA DEMANDE (Manager)"
        A[Constructeur du Manager] -->|Boucle pour chaque agent| B(Création requête vide)
        B -->|Appel Service asynchrone| C{get_random_lanelet_pose}
    end

    subgraph "2. LE CALCUL (Map Server)"
        C --> D[Scan de la carte Lanelet2]
        D --> E[Sélection Lanelet aléatoire]
        E --> F[Calcul X, Y, Yaw sur centerline]
        F -->|Réponse PoseStamped| G(Callback de réception)
    end

    subgraph "3. LA RÉCEPTION (Manager)"
        G --> H[Stockage dans agent_start_poses_]
        H --> I[Log: Position générée pour agent_X]
    end

    subgraph "4. L'UTILISATION (Update Loop)"
        I --> J{update_step}
        J -->|Si pose présente| K[Téléportation Gazebo + Calcul Path]
        J -->|Si pose absente| L[Attente / Skip]
    end

    style C fill:#f9f,stroke:#333,stroke-width:2px
    style G fill:#bbf,stroke:#333,stroke-width:2px
```


```mermaid
graph LR
    A[<b>Manager</b><br/>Le Cerveau] -- 1. Demande position --> B[<b>Map Server</b><br/>La Carte]
    B -- 2. Envoie coordonnées --> A
    A -- 3. Note la position --> C[(Mémoire)]
    C -- 4. Déplace le robot --> D[<b>Gazebo</b>]
    C -- 4. Affiche le cube --> E[<b>RViz</b>]

    style A fill:#e1f5fe,stroke:#01579b
    style B fill:#fff3e0,stroke:#e65100
    style D fill:#e8f5e9,stroke:#2e7d32
```

```mermaid
graph TD
    A[<b>Départ</b><br/>Besoin d'un Goal] --> B{Parcourir la Carte}
    B --> C[Liste des routes valides (Lanelets)]
    C --> D[<b>Tirage au sort</b><br/>std::mt19937</b>]
    D --> E[Route choisie au hasard]
    E --> F[Point au milieu de la ligne (index size/2)]
    F --> G[Calculer l'angle de la route]
    G --> H[<b>Goal Validé</b><br/>X, Y, Yaw</b>]

    style D fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style H fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
```