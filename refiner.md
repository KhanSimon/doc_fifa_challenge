# Field Converter

**ground truth :** 

Poses

```python
"betas": betas, # np.array of (N, 10). Utile pour finetuner sam3dBody
"global_orients": global_orients, # np.array of (N, T, 3) Rotation du corps dans les coordonnées World
"body_poses": body_poses, # np.array of (N, T, 69). Utile pour notre modèle à 2 tête
"transl": transl, # np.array of (N, T, 3) .
```

Cameras

```jsx
"K": K, # np.array of shape (T, 3, 3), intrinsic matrix = coordonnées pixel| coordonnées camera
"R": R, # np.array of shape (T, 3, 3), rotation matrix = ou regarde la caméra
"t": t, # np.array of shape (T, 3), translation vector = ou est la caméra
"k": k, # np.array of shape (T, 2), distortion coefficients (k1, k2) = deformation de l'image
```

### Opérations que subissent les points 3d :  (issu du code)

- Pour la gt : (fichier create_bounding_box.py). $X_{monde}$

Les 4 paramètres de la GT Pose sont passé dans le modèle SMPL (déterministe, pas un NN). Il en ressort ?? keypoints. On garde ensuite que 25 de ces keypoints (basé sur un mapping de OpenSim25). ??Demander à TJ comment il a fait pour avoir ce mapping. 

Ces keypoints sont pelvocentrés avant de rentrer dans le data vizu. 

- Pour l’estimation : (fichiers mhr_head.py, sam3d_body.py, sam_3d_body_estimator.py, preprocess_seq.py) $X_{sam3dbody}$

Inversion des axes y et z, on garde que les 70 keypoints puis les 25. On remplace le point du mid hip par la moyenne des 2 points des 2 hips. 

Le mapping semble globalement correct. Ça peut être une limitation. Notamment pour la loss $L_{rel}$ dans la tête 1 (expliquée plus tard). 

### Rappel de notation :

X est un ensemble de point. Par exemple : X.shape = (25, 3)

**En indice** : le repère. Par exemple : 

$X_{world}$ (origine dans un coin du terrain de foot)

$X_{cam}$  (origine à l’emplacement de la caméra)

**En exposant** : une transformation ou une propriété : 

$X^{rel}$ = X - X[pelvis]

$X^{pred}$

$X^{GT}$

Sam3DBody renvoie des poses 3D dont on ignore le repère : $X^{rel}_{?}$
**Objectif du refiner :** 

On obtient en sortie de sam3dbody (entre autres) les keypoints 3D : $X^{transla : sam3dbody}_{rota : sam3dbody}$. Ils sont dans un repère de rotation et de translation complètement stochastique. De plus, l’axe y et z sont inversés et l’on passe de 308 keypoints à 70 puis 25. On pelvo centre cela avec :

$X^{rel}_{sam3dbody} = X^{transla : sam3dbody}_{rota : sam3dbody} - X^{transla : sam3dbody}_{rota : sam3dbody}[pelvis]$

- **Tête 1  : Pose relative** : Apprendre à passer de $X^{rel}_{sam3dbody}$(centré pelvis, orienté selon sam3dbody)à $X^{rel}_{cam}$ (centré pelvis, orienté selon la caméra)(ML supervisé) :
    - $X^{sam3Dbody}$, $X^{sam3Dbody}_{image(2D)}$, et autres inputs→ [MLP ou Transfo] → delta, avec 2 loss :
        - $L_{rel} = ||X^{rel pred}_{cam} - X^{rel  gt}_{cam}||$
        - forcer : $L_{proj}^{TF} = ||Π(X_{cam}^{rel}+root^{GT}_{cam})-X^{GT}_{image(2D)}||$ (loss teacher- forced)
- **Tête 2 : Translation globale** : Estimer root_cam = $X^{pelvis}_{cam}$ pour faire : $X_{cam}$ = $X^{rel}_{cam}$ + $root_{cam}$ (ML supervisé) :
    - $X^{sam}$, $X^{sam}_{image(2D)}$, param_cam, et autres input → [MLP ou Transfo ? ]→ root_cam = (3,)
    - $L_{root}=||root_{cam}−root^{gt}_{cam}||$

Loss qui combine les 2 : $L_{proj} = ||Π(X_{cam}^{rel}+root_{cam})-X^{GT}_{image(2D)}||$

Autre loss qui combine les 2 : 

- **Tête 3 : Supervision orientation : exploite la GT**  `global_orients (N, T, 3)` . On la convertie en matrice de rotation $R_{body}^{world}$ avec : $K = \begin{pmatrix}0 & -u_z & u_y \\u_z & 0 & -u_x \\-u_y & u_x & 0\end{pmatrix}$

et $R=I+sin(θ)K+(1−cos(θ))K^2$, puis : $R_{\text{body}}^{cam} = R_{\text{cam}} \cdot R_{body}^{world}$

$R_{\text{body}}^{cam}$ : orientation du corps vu depuis la caméra

$R_{\text{cam}}$ : rotation de la caméra

- $L_{\text{orient}} 
= \theta
=\arccos\left(\frac{\operatorname{Tr}\!\left(R_{\text{pred}}^{\top} R_{\text{gt}}\right) - 1}{2}\right)$Intuition : plus le produit scalaire entre les 2 angles est faible, plus les rotations sont similaires. $\theta$ est l’angle de différence.  On veut entrainer le modèle à prédire la rotation du joueur dans le référentiel caméra.
- $~~L_{\text{joints}} = \left\| R_{\text{pred}}^{cam} \cdot X_{\text{rel}}^{cam} - R_{\text{gt}}^{cam} \cdot X_{\text{rel,gt}}^{cam} \right\|$ Intuition : est-ce que les joints finaux sont corrects dans le repère de la caméra ?~~ UPDATE : ne marche pas.

L’output de cette tête n’est pas utilisé pour la reconstruction. Elle sert juste à superviser l’apprentissage. 

On peut aussi utiliser des loss de velocity ou de longueur d’os. 

On peut alors passer dans le monde réel avec : $X_{world} = R^T(X_{cam} - t)$

Avec :

- $R^T$  la transposée de la matrice rotation R.
- t le vecteur de translation monde / caméra.

(attention aux conventions de multiplications)

**Architecture : 1 seul modèle avec 3 têtes :** 

**en entrée : (**Ne pas tout mettre dès le départ. Faire des ablations. Tester. Eviter l’overfitting. Penser à normaliser**)**

sorties de sam3dbody : 

- X3Dsam (l’indice joueur est cohérent dans le temps, idem pour la GT)
- X2Dsam
- pred_cam_t [3] : translation caméra pour passer du 3D  repère “sam3dbody” au repère caméra. On ne prend pas ça directe pour notre World Convertor car on veut entrainer un modèle avec les vraies données caméra.
- focal_length (scalaire) : $f_x$
- pred_pose_raw [266] : concaténation de la rot globale 6D et la pose “cont” à 260 paramètres. Utilisé pour l’initialisation.
- body_pose_params [133] : pose du corps en paramètres MHR
- shape_params [45] : coeffs en PCA/latents pour le MHR
- scale_params [28] : coeffs PCA
- global_rot [3] : rotation globale du corps
- pred_global_rots [127,3,3] (lourd) : rotation globale des joints
- mhr_model_params [204]

autres entrées : 

box

```jsx
cx = (xmin + xmax) / 2 / W
cy = (ymin + ymax) / 2 / H
w  = (xmax - xmin) / W
h  = (ymax - ymin) / H
a  = w / h
```

cam_feat

Rajouter du bruit au train car on aura pas des super estimations caméras en entrée car estimées géométriquement. 

bruit rotation
bruit translation
bruit focal
bruit centre optique

**Comment obtenir les ground truth :** 

$X_{world}^{gt}$ 

Faire un mapping clair à N joints :

```python
GT model 69 joints → N joints 
SAM3D 70 joints → N joints 
Les indices doivent être les mêmes
```

$root^{gt}_{cam}$

```jsx
root_cam_gt = X_cam_gt[pelvis] #choix prioritaire pour l'instant
ou bien : root_cam_gt=R⋅transl+t #pas forcément adapté car pas forcément la translation du pelvis
```

$X^{rel  gt}_{cam}$

```jsx
X_cam_gt = R @ X_world_gt + t
root_cam_gt = X_cam_gt[pelvis]
X_rel_cam_gt = X_cam_gt - root_cam_gt
```

$X^{gt}_{image(2D)}$  

```python

def project_world_to_image(X_world, K, R, t, k):
    """
    X_world: (J, 3) world coordinates
    K: (3, 3)
    R: (3, 3)
    t: (3,)
    k: (2,) = [k1, k2]

    returns:
        X_img: (J, 2)
        valid: (J,) bool
    """
    # 1 World -> camera
    X_cam = X_world @ R.T + t  # (J, 3)

    Z = X_cam[:, 2]
    valid = Z > 1e-6
    
		# 2 Passage de la 3D cam à la 2D cam (perspective)
    X_img = np.full((X_world.shape[0], 2), np.nan, dtype=np.float32)
    if not np.any(valid):
        return X_img, valid

    x = X_cam[valid, 0] / Z[valid]
    y = X_cam[valid, 1] / Z[valid]

    # 3 Radial distortion
    k1, k2 = k
    r2 = x**2 + y**2
    factor = 1.0 + k1 * r2 + k2 * r2**2
    x_d = x * factor
    y_d = y * factor

    # 4 Conversion en pixels avec K 
    fx, fy = K[0, 0], K[1, 1]
    cx, cy = K[0, 2], K[1, 2]

    u = fx * x_d + cx
    v = fy * y_d + cy

    X_img[valid, 0] = u
    X_img[valid, 1] = v
    return X_img, valid
```

1. Pour passer du repère monde au repère caméra, on multiplie par R et on ajoute t : $X_{cam} = R X_{world} + t$
2. Pour passer de la 3D caméra à la 2D caméra, on fait :

$(x_{cam}, y_{cam}) = ({X_{cam}}/Z_{cam} , {Y_{cam}}/Z_{cam})$

Sans cette division, un point à 10m de la cam et un point à 100m aurait la même taille projettée. 

1. On applique ensuite la distorsion radiale (on a seulement k1 et k2) :

 $(x_d, y_d) = (x, y) \cdot (1 + k_1 r^2 + k_2 r^4)$

1. On passe en pixel avec K : $u=f_{x}x_d+c_{x},v=f_{y}y_d+c_{y}$

### Pour obtenir les bounding boxs

```python
def boxes_from_2d_joints(joints_2d, valid, margin=0.1):
    """
    joints_2d: (J,2)
    valid: (J,)
    """
    pts = joints_2d[valid]
    if len(pts) == 0:
        return np.array([np.nan, np.nan, np.nan, np.nan], dtype=np.float32)

    x1, y1 = pts.min(axis=0)
    x2, y2 = pts.max(axis=0)

    return np.array([x1, y1, x2, y2], dtype=np.float32)
Rajouter du margin et du bruit. 
```

### Petit rappel sur les paramètres caméra

- K : projection en Pixels u et v
    
    $K =\begin{pmatrix}f_x & 0 & c_x \\0 & f_y & c_y \\0 & 0 & 1\end{pmatrix}$
    

$u=f_{x}x+c_{x},v=f_{y}y+c_{y}$

- R : rotation → orientation de la caméra

- k : distorsion. Les points sur les côtés de l’image prise par la caméra sont déformés. Cette déformation est mesurée avec k1 et k2.

- t : translation → position relative du monde

### Limites du Word Converter :

La conversion HMR → 25keypoints (estimation) peut donner des keypoints différents de la conversion SMPL → 25 keypoints (ground truth). Ce qui altèrerait la 1ère tête notament avec la loss : $L_{rel} = ||X^{rel pred}_{cam} - X^{rel  gt}_{cam}||$

Entrainé uniquement sur des les matchs de la FIFA (même matchs, mêmes caméras, mêmes angles), il peut y avoir sur sur-apprentissage : un biais en inférence sur d’autres types de matchs