# 3DHPE -WorldPose Dataset - Challenge FIFA

# DATA

5 dossiers principaux. Dans chaque dossiers, des éléments time stampés HHMMSS (Heure Heure Minute Minute Seconde Seconde)

- 1 dossier FIFA 2026 avec 20 MP4 non labélisés. Telechargé depuis le sharepoint microsoft FIFA. (Accès au sharepoint reçu par mail)
- 1 dossier FIFA 2025 avec 5 MOV non labélisés. Telechargé depuis le sharepoint microsoft FIFA. (Accès au sharepoint reçu par mail)
- 1 dossier world pose avec 89 MP4 labélisés (cf dossier 3DnCAM) compressés 50Mo, 1 min par fichier. Il y a aussi les raw en format MOV mais trop volumineux : 1Giga/video donc 130 Giga. Telechargé depuis le sharepoint microsoft FIFA. (Accès au sharepoint reçu par mail)
- 1 dossier avec des 2D 3D camera et box. Téléchargé depuis [la page Hugging Face du challenge](https://huggingface.co/datasets/tijiang13/FIFA-Skeletal-Tracking-Light-2026). Les 2D et 3D poses du dossiers sont les données qu’on est censé obtenir après run [preprocess.py](http://preprocess.py) du "Starter Kit" du challenge, disponible sur [GitHub](https://github.com/FIFA-Skeletal-Light-Tracking-Challenge/FIFA-Skeletal-Tracking-Starter-Kit-2026). Plus d’info sur ce Starter Kit dans la suite du README. *Nb : le starter kit n'utilise pas la GT SMPL. C’est donc une solution un peu naïve du challenge*.
- 1 dossier 3DnCAM de 2 sous-dossiers (3D poses et calibrations cam) avec chacun 89 npz qui fit avec les vidéos du dossier précédent. Détail d’un élément npz :

Soit N le nombre de joueur et T le nombre de frames 

Camera :

```jsx
"K": K, # np.array of shape (T, 3, 3), intrinsic matrix
"R": R, # np.array of shape (T, 3, 3), rotation matrix
"t": t, # np.array of shape (T, 3), translation vector //dispo que sur la frame 1
"k": k, # np.array of shape (T, 5), distortion coefficients (k1, k2, p1, p2, k3)
//idem
```

3D poses :

```jsx
"betas": betas,                   # np.array of (N, 10)
"global_orients": global_orients, # np.array of (N, T, 3)
"body_poses": body_poses,         # np.array of (N, T, 69)
"transl": transl,                 # np.array of (N, T, 3)
```

# REPO GITHUB : Starter Kit

Le repo github est une base / référence pour le challenge FIFA. Il n’utilise pas la GT SMPL, il fait juste du 2DHPE et 3DHPE. 

## preprocess.py

camera / image / bounding boxes `(NUM_FRAMES, NUM_PERSONS, 4)` → preprocess.py → 2DPoses `(T, N, 25, 2)` / 3DPoses `(T, N, 25, 3)`

- Approche Top Down, image par image : pas time aware. Le modèle prend : `IMG (H,W,3), BBOXES (N, 4) et CAM_INTR (3, 3)`  et renvoie une paire : `Skels_2D (N, 70, 2)` et `Skels_3D (N, 70, 3)`
- Conversion 70 indices vers 25 keypoints.

NB : On pourrait utiliser la GT SMPL pour obtenir les 15 points 3D par supervision ou bien faire de la projection.

## main.py

camera / images / bounding boxes / 2DPoses / 3DPoses → main.py → prédictions finales par séquence : `(N, T, 15, 3)` dans un npz. 

génère les poses 3D dans le référentiel monde. 

**Estimation des paramètres caméras :** 

On a toutes les infos caméras de la 1st frame. Pas celles des autres frames. Projection des pitchs points 3D (lignes du terrain) dans l'image de la 1st frame. Création d’un masque pour le sol et d’une plage HSV du gazon. Le masque isole le terrain sur les frames suivantes. 
****Pour chaque frame :

??? 

rafinement périodique ? 

(détection des lignes et optical flow)
Avec detection et mask, on connait les lignes dans le monde 2D. 

**Conversion 3DPose Camera→3DPose World**

Pour chaque personne dans la frame: 

- On récupère le membre le plus bas du skel2D (on suppose qu'il est au sol)
- On trace un rayon de la caméra vers ce point : on associe ce point à une droite dans le monde réel.
- Pour obtenir le point dans le monde 3D, on fait l’intersection entre cette droite et le plan z=0.
- On reconstruit le squelette 3D en faisant :
    - rotation par `R` via `skel_3d = skel_3d @ R` (obtenue lors de l’estimation des param cam)
    - translation globale via `skel_3d = skel_3d - skel_3d[IDX] + intersection` (obtenue avec l’intersection droite plan)

**Raffinement du paramètre de translation : finetune_translation()**

Optimisation d’un vecteur 3D t rajouté aux points 3D avec minimize_reprojection_error(). La loss est une MSE entre la 2D estimé dans preprocess et la projection de la 3D estimée plus haut. On travaille uniquement sur les points de la hanche. 

loss(t) = MSE( proj( skel_3D + t ), skel_2D )

**Smoothing de la 3DHPE : smoothen()**

## prepare_submission.py

Pour SUBMIT, utiliser prepare_submission.py :
Avant de lancer le script : faire la prédiction puis l'enregistrer dans submission_full.npz
Ne pas toucher aux fichiers txt sequences test et sequences val de data. C’est dans ces fichiers qu’est définit les sample sur lequel le modèle candidat est testé / évalué. 

### Avoir une meilleure accuracy :