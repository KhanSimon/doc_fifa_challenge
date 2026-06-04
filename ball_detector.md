

# Pipeline - Méthode hybride

## Données disponibles

Pour chaque séquence (~40 secondes) :

* vidéo broadcast (50 FPS, 1920×1080),
* paramètres caméra par frame :

  * (K(T,3,3))
  * (R(T,3,3))
  * (t(T,3))
  * (k(T,2))
* positions des keypoints des joueurs :

  * (X(N,T,25,3))
* points clés du terrain en coordonnées monde (`pitchpoints`)
  (((0,0,0)) au centre du rond central).

---

# Étape 1 - Tracking 2D de la balle

Effectuer le tracking de la balle dans l’image :

$$
    (u_t,v_t,w_t)
$$

avec :

* (u_t,v_t) : coordonnées pixels,
* (w_t) : confiance du tracking.

---

# Étape 2 - Génération des rayons 3D

Construire pour chaque frame le rayon monde associé :

$$
\mathrm{B_t=o_t+\lambda_t d_t}
$$

avec :

* (o_t) : origine du rayon (centre caméra),
* (d_t) : vecteur unitaire de direction,
* (\lambda_t) : profondeur inconnue.

---

# Étape 3 - Découpage en segments (détection des impacts)

Calcul d’un score de rupture :

$$
S_{break}=
C_{ray}
-\mathrm{\lambda D_{player}}
$$

ou :

$$
S_{break}=
C_{ray}+
\lambda
\exp\left(-\frac{D_{player}}{\sigma}\right)
$$

avec :

$$
C_{ray}=
|d_{t+1}-2d_t+d_{t-1}|
$$

(courbure locale du rayon)

et :

$$
D_{player}=
\min_j dist(ray_t,X_j)
$$

où la distance est principalement calculée sur :

* jambes,
* tête,
* poitrine.

Ca peut être plus précis de calculer la distance à un segment plutôt que la distance à un keypoint. 

Le résultat est une plage de :

```text
impact_start → impact_end
```

Si le découpage est insuffisamment précis (cas critiques comme les jeux de tête), correction manuelle possible.

---

# Étape 4 - Classification des impacts

Pour chaque impact détecté :

déterminer s'il est :

```text
impact_player
ou
impact_ground
```

Règle :

* impact joueur :
  intersection balle ↔ pied / genou / tête,
* impact sol :
  sinon.

On obtient alors :

$$
P_{impact}
$$

position monde estimée de l’impact.

---

# Étape 5 - Classification de la trajectoire

Pour chaque segment entre deux impacts :

tester 2 hypothèses :

```text
ROLLING
ou
FLYING
```

---

## Hypothèse ROLLING

Supposer :

$$
z=r_{ball}
$$

Calcul :

$$
B_{ground}=
\mathrm{ray\_intersect\_z(ray,z=r_{ball})}
$$

Calcul du score :

$$
score_{rolling}=
smoothness +
linearity +
\mathrm{speed\_valid} +
\mathrm{field\_valid}
$$

avec :

### Smoothness

Trajectoire lisse, décélération raisonnable.

### Linearity

Ajustement d’une droite 2D :

$$
\mathrm{line\_error}=
dist(B,\hat B)
$$

### Speed valid

La vitesse reste plausible.

### Field valid

La balle reste sur le terrain.

Si le score est satisfaisant :

```text
ROLLING
```

---

## Hypothèse FLYING

Supposer une trajectoire balistique :

$$
position(t)=
position_0+
vitesse_0 t+
\frac12gt^2
$$

Variables :

* (b_t=(u_t,v_t)) : position de la balle en coordonnées pixels. 2 dimensions,
* (o_t) : origine du rayon, position de la caméra en coordonnées monde. 3 dimensions,
* (d_t) : vecteur unitaire ||d_t||=1. Indique dans quel direction la caméra regarde. 3 dimensions,
* (B_t) : position de la balle en coordonnées monde. origine + direction * un scalaire. 3 dimensions.

$$
\mathrm{B_t=o_t+\lambda_t d_t}
$$

---

### Fonction coût

$$
L_{ray}=
\sum_t
w_t
\left|
(I-d_td_t^T)
(B(t)-o_t)
\right|^2
$$

Tout vecteur v peut être composée en une composante perpendiculaire et une composante parralèle à une autre vecteur d. 

$$
(I-d_td_t^T)
$$

retire la composante parallèle au rayon.

Le coût mesure donc :

distance perpendiculaire entre trajectoire reconstruite et rayon observé.

Score :

$$
score_{flying}=
\mathrm{ray\_error+}
\mathrm{speed\_valid}
$$

Si le score est satisfaisant :

```text
FLYING
```

---

# Étape 6 - Reconstruction analytique

Une fois le type de segment identifié :

reconstruire analytiquement la trajectoire.

---

## Cas FLYING

Pour chaque paire :

$$
(i',j')
$$

avec :

```text
impact_start = i'
impact_end = j'
```

Calcul :

$$
\Delta t=
\frac{j'-i'}{fps}
$$

Vitesse initiale :

$$
V_0=
\frac{
P_1-P_0-\frac12 g\Delta t^2
}{
\Delta t
}
$$

Reconstruction :

$$
B(\tau)=
P_0+
V_0\tau+
\frac12g\tau^2
$$

Cette equation se retrouve en intégrant l'equation : v(t)=dB/dt <=> v(t)dt = dB sur le temps 0 à t
Conserver :

la paire qui minimise :

$$
L_{ray}=
\sum_t
w_t
\left|
(I-d_td_t^T)
(B(t)-o_t)
\right|^2
$$

---

## Cas ROLLING

Pour chaque paire :

$$
(i',j')
$$

Calcul :

$$
P_0=
ray_{i'}
\cap z=r_{ball}
$$

$$
P_1=
ray_{j'}
\cap z=r_{ball}
$$

Temps :

$$
\Delta t=
\frac{j'-i'}{fps}
$$

Vitesse :

$$
V_0=
\frac{
P_1-P_0
}{
\Delta t
}
$$

Modèle simple :

$$
B(\tau)=
P_0+V_0\tau
$$

ou modèle avec décélération :

$$
B(\tau)=
P_0+
V_0\tau+
\frac12A\tau^2
$$

Conserver :

la paire qui minimise :

$$
score=
linearity+
smoothness+
\mathrm{speed\_valid+}
\mathrm{field\_valid+}
(z=r_{ball})
$$

---

# Étape 7 - Raffinement global

Optimisation finale sous contraintes :

* vitesse maximale,
* cohérence physique,
* contrainte sol,
* proximité joueurs,
* continuité des trajectoires.

Objectif :

corriger localement les impacts et obtenir une trajectoire monde finale cohérente.
