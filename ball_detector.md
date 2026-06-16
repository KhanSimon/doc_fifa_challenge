


# Pipeline - Méthode hybride

## Données disponibles

Pour chaque séquence (~40 secondes) :

* vidéo broadcast (50 FPS, 1920×1080),
* paramètres caméra par frame :

  * $K(T,3,3)$
  * $R(T,3,3)$
  * $t(T,3)$
  * $k(T,2)$
* positions des keypoints des joueurs (coordonnées monde) :

  * $X(N,T,25,3)$
* points clés du terrain en coordonnées monde (`pitchpoints`), $(0,0,0)$ au centre du rond central.

---

# Étape 1 - Tracking 2D de la balle

Effectuer le tracking de la balle dans l’image :

$$
    (u_t,v_t,w_t)
$$

avec :

* $u_t,v_t$ : coordonnées pixels,
* $w_t$ : confiance du tracking.

---

# Étape 2 - Génération des rayons 3D

Construire pour chaque frame le rayon monde associé :

$$
\mathrm{B_t=o_t+\lambda_t d_t}
$$

avec :

* $o_t$ : origine du rayon (centre caméra),
* $d_t$ : vecteur unitaire de direction,
* $\lambda_t$ : profondeur inconnue.

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

(courbure locale du rayon, en coordonnées monde)

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

Calculer la position de la balle en coordonnées monde si elle était au sol :

$$
\mathrm{
B_{ground}=
ray\_intersect\_z(ray,z=r_{ball})
}
$$

Calcul du score :

$$
\mathrm{
score_{rolling}=
smoothness +
linearity +
speed\_valid +
field\_valid
}
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
B(t)=
position_0+
vitesse_0 t+
\frac12gt^2
$$

Variables :

* $position_0$ : position monde de la balle lors du premier impact. 3 dimensions
* $vitesse_0$: vitesse initiale. 3 dimensions.
*  $B_t$ : position de la balle en coordonnées monde.
* $o_t$ : origine du rayon, position de la caméra en coordonnées monde. 3 dimensions
* $d_t$ : vecteur unitaire ||d_t||=1. Indique dans quel direction la caméra regarde. 3 dimensions

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

retire la composante parallèle au rayon. Le coût mesure donc la distance perpendiculaire entre trajectoire reconstruite et rayon observé. 

**Si $position_0$ est mal estimée à l'étape 4 (se vérifie à la main) :** on veut trouver $position_0$ et $vitesse_0$ qui minimise la distance entre B et le rayon de la caméra pour chaque instant t de la trajectoire.

**Si $position_0$ est bien estimée à l'étape 4 :** on a pas besoin d'optimiser $vitesse_0$, on l'a analytiquement : 
$$
V_0=
\frac{
P_1-P_0-\frac12g\Delta t^2
}{
\Delta t
}
$$

Une fois qu'on a calculé la fonction coût, on calcule un score :

$$
score_{flying}=
ray_{error} +
speed_{valid}
$$

$ray_{error}$ vient de $L_{ray}$ : 

$$
ray_{error}=
\frac{
L_{ray}
}{
\sum_t w_t
}
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

On a  :

$$
B(\tau)=
P_0+
V_0\tau+
\frac12g\tau^2
$$

Cette equation se retrouve en intégrant l'equation : v(t)=dB/dt <=> v(t)dt = dB sur le temps 0 à t
On en déduit la vitesse initiale au temps $\Delta t$ :

$$
V_0=
\frac{
P_1-P_0-\frac12 g\Delta t^2
}{
\Delta t
}
$$

Si on a un doute sur le temps des impacts, 
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

On pourrait s'arrêter là mais $P_0$ et $P_1$ ne sont souvent pas parfaits (bruités). 
On optimise donc $P_0$, $P_1$ et $V_0$ avec une loss qui combine : 

La distance perpendiculaire au rayon  :
$$
L_{ray}=
\sum_t
w_t
\left|
(I-d_td_t^T)
(B(\tau_t)-o_t)
\right|^2
$$

Deux contraintes souples (pour empêcher l'optimisation de trop s'éloigner des positions d'impacts détectés) :

$$
L_{impact0}=
|B(0)-P_{impact,0}|^2
$$

$$
L_{impact1}=
|B(\Delta t)-P_{impact,1}|^2
$$
Une contrainte sur la vitesse : 
$$
L_{speed}=
\max(0,|V_0|-V_{max})^2
$$
Et une contrainte qui pénalise les positions trop éloignées du terrain : $L_{field}$
La loss totale se note $L_{flying}$ : 
$$
L_{flying}=
L_{ray}
+
\alpha L_{impact0}
+
\beta L_{impact1}
+
\gamma L_{speed}
+
\eta L_{field}
$$
Une fois l'optimisation faite, on conserve :

```text
P0_final
V0_final
B_t pour toutes les frames du segment
ray_error
speed_score
field_score
```

avec :

$$
B_t=
P_0^{final}
+
V_0^{final}\tau_t
+
\frac12g\tau_t^2
$$

## Cas ROLLING

Pour chaque paire :

$$
(i',j')
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

ou modèle avec décélération constante :

$$
B(\tau)=
P_0+
V_0\tau+
\frac12A\tau^2
$$
Cette équation vient de la double intégration également. 

Si on a un doute sur le temps des impacts, conserver la paire qui minimise :

$$
score=
linearity+
smoothness+
\mathrm{speed\_valid+}
\mathrm{field\_valid+}
(z=r_{ball})
$$

On pourrait s'arrêter là mais $P_0$ et $P_1$ ne sont souvent pas parfaits (bruités).

On optimise donc $P_0$, $P_1$, $V_0$ et $A$ avec une loss qui combine :

La distance perpendiculaire au rayon :

$$  
L_{ray}=  
\sum_t  
w_t  
\left|  
(I-d_td_t^T)  
(B(\tau_t)-o_t)  
\right|^2  
$$

Deux contraintes souples (pour empêcher l'optimisation de trop s'éloigner des positions d'impacts détectés) :

$$  
L_{impact0}=  
|B(0)-P_{impact,0}|^2  
$$

$$  
L_{impact1}=  
|B(\Delta t)-P_{impact,1}|^2  
$$

Une contrainte sur la vitesse :

$$  
L_{speed}=   
\max(0,|V_0|-V_{max})^2  
$$

Une contrainte qui force l’accélération $A$ à être proche de l'accélération théorique sur pelouse :

$$
L_{friction}=
\sum_t 
\left|
A_t+\mu g\frac{V_t}{|V_t|+\epsilon}
\right|^2
$$
car : 
$$
A_{friction}=
-\mu g\frac{V}{|V|}
$$

Une contrainte qui pénalise les positions trop éloignées du terrain :

$$  
L_{field}  
$$

La contrainte de contact au sol est imposée directement :

$$  
B_z(\tau)=r_{ball}  
$$

La loss totale se note :

$$  
L_{rolling}=  
L_{ray}  
+  
\alpha L_{impact0}  
+  
\beta L_{impact1}  
+  
\gamma L_{speed}  
+  
\eta L_{field}  
+  
\mu L_{friction}  
$$

Une fois l’optimisation faite, on conserve :

```text
P0_final
V0_final
A_final
B_t pour toutes les frames du segment
ray_error
speed_score
field_score
acc_score
```

avec :

$$  
B_t=  
P_0^{final}  
+  
V_0^{final}\tau_t  
+  
\frac12A^{final}\tau_t^2  
$$

et :

$$  
B_z=r_{ball}  
$$

pour toutes les frames du segment.