# DBZ Super Saiya Densetsu (SNES) — Traduction FR  
# Crash de Freezer forme finale — Cause et correction

**Date :** 2026-07-06  
**ROM concernée :** `Dragon_Ball_Z_-_Super_Saiyan_Densetsu_Translated_FR.smc`  
**Taille :** 1 573 376 octets  
**Format :** ROM LoROM étendue à 1,5 MB avec header copier de 512 octets  
**Références utilisées :** ROM japonaise originale et traduction anglaise Klepto v1.02  
**Symptôme :** le jeu se bloque pendant le combat final, au moment où Freezer passe en 4e forme / forme finale.

---

## 1. Résumé simple

Le crash vient de la **table des noms des unités de combat** dans la ROM française.

Dans le jeu, chaque personnage ou ennemi utilisé en combat possède une entrée dans une table.  
Cette table indique où le jeu doit aller chercher le nom à afficher à l’écran.

Dans la version française, certaines entrées de cette table sont corrompues.  
Elles ne pointent pas vers le début d’un vrai nom, mais vers le milieu d’autres noms comme :

- `Gokou`
- `Dendé`

À cause de ça, le jeu lit de mauvaises données et croit que certains noms font **37 caractères** ou même **100 caractères**, alors qu’un nom normal fait seulement quelques caractères.

Quand Freezer passe à sa forme finale, le jeu essaie donc d’afficher un nom complètement faux et beaucoup trop long.  
Cela provoque un débordement dans le système d’affichage des noms, puis le moteur du jeu se bloque.

En clair :

> Le jeu ne plante pas à cause de la transformation de Freezer elle-même.  
> Il plante parce que le nom de Freezer forme finale pointe au mauvais endroit dans la ROM.

---

## 2. Architecture du texte concerné

La ROM française est basée sur la traduction anglaise Klepto v1.02.

La version française ajoute un moteur de dialogue personnalisé dans une zone étendue de la ROM, mais ce moteur ne concerne pas les noms utilisés en combat.

Les noms de personnages et d’ennemis en combat utilisent encore le système original du jeu.

La table concernée est :

```text
Table des noms de combat : table2
Offset PC : 0x12CE8
Adresse SNES : $82:ACE8
Nombre d’entrées : 105
Type : pointeurs 16-bit dans la banque $82
```

Chaque entrée pointe vers une chaîne de texte au format :

```text
[longueur sur 1 octet][caractères du nom]
```

Exemple simplifié :

```text
07 Freezer
```

Cela signifie :

- `07` = le nom fait 7 caractères
- `Freezer` = texte affiché

Dans les ROM japonaise et anglaise, cette table est correcte.  
Dans la ROM française, plusieurs pointeurs ont été modifiés ou cassés.

---

## 3. Causes du crash

Il y a **3 erreurs principales** dans la ROM française.

Ces 3 erreurs provoquent au total **13 entrées incorrectes** dans la table des noms de combat.

---

## 3.1 Erreur A — Le nom `Yajiro` déborde

Dans la version française, le nom `Yajiro` a été écrit directement à un endroit où il n’y avait pas assez de place.

Le nom original était plus court.  
Le traducteur a voulu écrire :

```text
Yajiro
```

Mais le dernier caractère, `o`, a été écrit sur l’octet suivant.

Problème : cet octet suivant n’était pas une lettre.  
C’était l’octet qui indiquait la longueur du nom suivant.

Normalement, cette longueur devait être :

```text
08
```

Mais elle est devenue :

```text
25
```

En hexadécimal :

```text
0x25 = 37
```

Donc le jeu croyait que certains noms faisaient **37 caractères**.

Cela cassait les entrées :

```text
85 à 95
```

Ces entrées correspondent à plusieurs unités spéciales ou de type Ōzaru / scénario.

---

## 3.2 Erreur B — Une forme de Freezer pointe au milieu de `Gokou`

Dans la ROM anglaise, l’entrée 100 pointait correctement vers le nom :

```text
Frieza
```

Dans la ROM française, elle aurait dû pointer vers :

```text
Freezer
```

Mais elle pointe au mauvais endroit :

```text
Au milieu du nom Gokou
```

Le jeu ne lit donc pas un vrai nom depuis son début.

Il tombe sur l’octet :

```text
0x25
```

Ce qui donne encore :

```text
37 caractères
```

Le jeu pense donc que le nom fait 37 caractères, alors que c’est faux.

---

## 3.3 Erreur C — Freezer forme finale pointe au milieu de `Dendé`

C’est l’erreur principale qui provoque le crash visible en jeu.

L’entrée 104 correspond à Freezer forme finale.

Elle devrait pointer vers :

```text
Freezer
```

Mais dans la ROM française, elle pointe au milieu du nom :

```text
Dendé
```

À cet endroit précis, le jeu lit l’octet :

```text
0x64
```

En décimal :

```text
0x64 = 100
```

Donc le jeu croit que le nom de Freezer fait **100 caractères**.

Le moteur essaie alors de copier environ 100 octets dans une petite zone mémoire prévue pour un nom court.  
Ces octets traversent d’autres chaînes de texte et vont même jusque dans une autre table de pointeurs.

Certains octets copiés sont ensuite interprétés comme des codes spéciaux par le moteur d’affichage.

Résultat :

```text
Blocage complet du moteur du jeu
```

C’est pour cela que le crash arrive exactement pendant la transformation de Freezer en 4e forme.

---

## 4. Correction appliquée

La correction consiste à :

1. remettre les pointeurs de Freezer vers le bon nom ;
2. restaurer la longueur cassée par le débordement de `Yajiro` ;
3. déplacer le nom `Yajiro` dans une zone libre de la ROM ;
4. faire pointer l’entrée de Yajiro vers ce nouvel emplacement ;
5. recalculer le checksum interne de la ROM.

---

## 5. Détail des modifications

| # | Offset PC | Offset `.smc` avec header | Adresse SNES | Ancien | Nouveau | Effet |
|---|---:|---:|---|---|---|---|
| 1 | `0x12DB0` | `0x12FB0` | `$82:ADB0` | `84 AE` | `7A AE` | L’entrée 100 pointe vers `Freezer` |
| 2 | `0x12DB8` | `0x12FB8` | `$82:ADB8` | `89 AE` | `7A AE` | Freezer forme finale pointe vers `Freezer` |
| 3 | `0x12E9B` | `0x1309B` | `$82:AE9B` | `25` | `08` | Restaure la longueur correcte du nom partagé |
| 4 | `0x175B6` | `0x177B6` | `$82:F5B6` | `FF ×7` | `06 45 05 1E 22 29 25` | Ajoute le nouveau nom `Yajiro` dans une zone libre |
| 5 | `0x12D8E` | `0x12F8E` | `$82:AD8E` | `95 AE` | `B6 F5` | L’entrée de Yajiro pointe vers le nouveau nom |
| 6 | `0x07FDC` | `0x081DC` | Header | `02 CC FD 33` | `5A 33 A5 CC` | Checksum interne recalculé |

---

## 6. Code des lettres utilisées pour `Yajiro`

Le nom `Yajiro` est stocké avec les mêmes codes de lettres que ceux utilisés par la traduction française.

```text
Y = 45
a = 05
j = 1E
i = 22
r = 29
o = 25
```

La nouvelle chaîne écrite en ROM est donc :

```text
06 45 05 1E 22 29 25
```

Ce qui signifie :

```text
06 = longueur du nom
45 05 1E 22 29 25 = Yajiro
```

---

## 7. Pourquoi le bug était dangereux

Le crash de Freezer était le problème le plus visible, mais ce n’était pas le seul effet possible.

Le débordement de `Yajiro` avait aussi cassé plusieurs autres entrées de la table.

Ces entrées pouvaient provoquer :

- noms trop longs ;
- affichage de texte corrompu ;
- boîtes de nom anormales ;
- comportements instables selon le combat ;
- potentiels blocages dans certains cas.

La correction nettoie donc toute la table des noms de combat concernée, pas seulement Freezer.

---

## 8. Ce qui n’a pas été modifié

Cette correction ne touche pas :

- les dialogues principaux ;
- le moteur de dialogue personnalisé de la traduction française ;
- les banques de dialogue `$86` et `$87` ;
- les graphismes ;
- les tilemaps ;
- les textes de scénario ;
- les techniques ;
- les mécaniques de combat.

La correction concerne uniquement :

```text
La table des noms des unités de combat
```

---

## 9. Vérifications effectuées

Après correction :

- les 105 entrées de la table des noms de combat ont été vérifiées ;
- aucune anomalie restante n’a été trouvée ;
- les 8 entrées liées aux formes de Freezer pointent bien vers `Freezer` ;
- l’entrée 104 de Freezer forme finale ne pointe plus au milieu de `Dendé` ;
- l’entrée 100 ne pointe plus au milieu de `Gokou` ;
- les entrées 85 à 95 ont de nouveau une longueur correcte ;
- le nom `Yajiro` est stocké proprement dans une zone libre ;
- le patch IPS appliqué sur la ROM française originale produit la ROM corrigée attendue ;
- le checksum interne a été recalculé.

---

## 10. MD5 de la ROM corrigée

```text
39481d86e43f4ba308fc1c25c1952a32
```

---

## 11. Checklist de test en jeu

Pour valider correctement la correction, il faut tester les points suivants.

### Combat final contre Freezer

- [ ] Freezer forme 1 fonctionne.
- [ ] Freezer forme 2 fonctionne.
- [ ] Freezer forme 3 fonctionne.
- [ ] Freezer passe en forme 4 sans crash.
- [ ] La phase Freezer 100% fonctionne.
- [ ] Le combat Super Saiyan fonctionne.
- [ ] Le jeu va jusqu’à la fin et aux crédits.

### Combat contre Ginyu

- [ ] Le combat contre Ginyu fonctionne.
- [ ] Les changements de nom liés au Body Change ne provoquent pas de bug.
- [ ] Aucun nom corrompu n’apparaît.

### Unités spéciales / Ōzaru

- [ ] Les unités 85 à 95 n’affichent plus de noms trop longs.
- [ ] Aucun texte corrompu n’apparaît dans ces combats.
- [ ] Aucun crash lié aux noms n’est visible.

### Yajiro

- [ ] Le nom `Yajiro` s’affiche correctement.
- [ ] Le nom ne déborde plus sur les données suivantes.
- [ ] Aucun autre nom autour de lui n’est cassé.

---

## 12. Conclusion

Le crash de Freezer forme finale dans la ROM française de **DBZ Super Saiya Densetsu** vient d’une table de noms de combat corrompue.

La forme finale de Freezer ne pointait pas vers le nom `Freezer`, mais vers le milieu du nom `Dendé`.

À cause de cela, le jeu lisait une fausse longueur de **100 caractères** et essayait d’afficher un nom beaucoup trop long.  
Ce débordement bloquait le moteur du jeu pendant la transformation.

La correction remet les bons pointeurs, répare le débordement causé par `Yajiro`, déplace proprement ce nom dans une zone libre, et rend la table des noms cohérente avec les versions japonaise et anglaise qui fonctionnent déjà.

En résumé :

```text
Cause du crash :
Freezer forme finale pointe au mauvais endroit dans la table des noms.

Effet :
Le jeu lit une longueur de 100 caractères et bloque.

Correction :
Freezer pointe de nouveau vers Freezer, Yajiro est relogé proprement, et la table est réparée.
```
