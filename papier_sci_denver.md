# Papier et code 3d ball trajectory Denver 2026
Plusieurs étapes (MODIFIABLE) : 

- Découper la trajectoire en segments entre des pivots points
- Tester un modèle sol et fly pour chaque segment
- Optimiser les paramètres physiques pour que la trajectoire 3d simulée se reprojette mieux sur les observations 2D.

Les entrées sont chargées dans estimate_trajectory.py (voir le DATA.md pour plus de détails)

- `detection/ball_detection*.csv` : détections balle 2D.
- `dev/df_merged_ball_player*.csv` : balle lissée + homographie + contacts joueurs. Contient plusieurs infos sur le contact entre la balle et le joueur. Mais aussi la traj de la balle smoothée et les homographies.
- `track/ball_pivot_point-gt*.csv` ou `track/ball_pivot_point*.csv` : surement pour evaluate
- `camera_smooth.csv` : caméra par frame.
- `pitch_geom/calibrate_camera_dict.pickle` : calibration terrain/caméra.
- `sequence_metadata.json` : fps, largeur, hauteur.

### STEP 1 : Prétraitement : preprocess_trajectory.py

La fonction `preprocess_ball` fait 4 sous-étapes (une fonction par sous-étape) : 

1. Choisit la meilleure détection balle par frame. Il y a plusieurs détections, prend celle qui a le meilleur score. 
2. Fusionne les tracks proches de detection. Les tracks qui sont séparés de quelques frames sont mergés en un seul track (une seule trajectoire). 
3. Lisse la position 2D avec un filtre de Kalman
4. Détecte les contacte ball-joueur et projete sur le sol grâce à l’homographie. TOMODIFY 

Détail de la fonction 4 : detect_ball_player_contacts : 

- garde seulement les frames ou il y a un overlap entre bouding box joueur et bounding box balle.
- stocke la postition de la balle au niveau du sol.
- stocke la hauteur relative de la balle.

Renvoie `df_merged_ball_player*.csv` qui est les intersections balle player projettés au sol. Les pivots ne sont pas save, alors que la doc l’indique (pull request?). Finalement dans le projet, les pivot points sont données en entrée. On a besoin de ces pivot points. Il faudra donc annoter à la main pour notre cas ou récupérer les vidéos. 

Normalement, on a les pivots (4d : position + temps) à l’issu de cette partie. 

### STEP 2 : Estimation 3D : src/ball_estimation/functions.py

La fonction principale : 

1. Filtre la plage temporelle start_sec/end_sec pour garder la traj entre les 2 pivots. 
2. Instancie l’estimateur Hydra configuré
3. Crée des `track_id` artificiels si `use_multiple_balls=false` . Un track ID est un ???
4. Lance l’estimation par piste, éventuellement en parallèle avec `n_jobs`. (détaillé ci dessous)
5. Remplit quelques prédictions manquantes (pivots).
6. Recolle la sortie sur toutes les frames (concaténation).
7. Corrige certains “sauts” de trajectoire. Un saut est une position de balle abérrante. On supprime ces valeurs abbérantes et les remplace par des valeurs qui fit linéairement. 
8. Calcule vitesses et accélérations.

#### Traitement des pistes (étape 4) : src/ball_estimation/trajectory/estimator.py

Pour chaque segment :

1. Si le segment est trop court, il est marqué `too_short_track`.
2. S’il est trop long, il est marqué `too_long_track`.
3. Sinon, le code ajuste d’abord un modèle droit au sol.
4. Si l’erreur du modèle droit dépasse `good_fit_threshold`, il essaie un modèle aérien.
5. Il garde le modèle avec la plus petite erreur.
6. Si l’erreur finale dépasse `allowed_detection_error`, le segment est rejeté comme `not_fitted`.

Il existe différents modèles physiques. 

### STEP 3 : Evaluation :

Compare `track/ball_3d.<version>.csv` à `track/ball_3d-gt.csv` . Enlève les pauses (surement les trajectoires not fitted). Calcule l’erreur 3d moyenne, l’erreur pour chaque axe et d’autres métriques avancées. 

### Pour run :

configuration avec Hydra.