# Guide de navigation TSO/ISPF — Commandes de base

> Ce guide couvre les commandes essentielles pour naviguer dans les bibliothèques et éditer des fichiers sous TSO/ISPF.  
> Les analogies Unix sont indiquées en italique pour faciliter la transition.

---

## Concepts de base

| Concept z/OS | Analogie Unix | Description |
|---|---|---|
| Dataset séquentiel (PS) | Fichier plat | Un seul fichier, lecture linéaire |
| PDS / PDSE | Répertoire | Bibliothèque contenant des **membres** |
| Membre | Fichier dans un dossier | Unité élémentaire dans un PDS |
| HLQ (High Level Qualifier) | `~` (home) | Préfixe utilisateur, ex. `IBMUSER` |
| Volume | Partition / disque | Support physique du dataset |

---

## 1. Lister des datasets

### Via le menu ISPF
```
Menu primaire → 3 (Utilities) → 4 (DSLIST)
Raccourci : =3.4
```

### Commande directe en ligne de commande ISPF
Dans le champ **Option** ou **Command**, tapez :
```
DSLIST IBMUSER.*
```
> *Analogue Unix :* `ls ~` ou `ls /home/ibmuser/`

Pour filtrer par préfixe plus précis :
```
DSLIST IBMUSER.ASM.*
```
> *Analogue Unix :* `ls ~/ASM/`

---

## 2. Naviguer dans un PDS (lister ses membres)

### Depuis la DSLIST
Placez le curseur sur le dataset et tapez `M` + Entrée.

### Commande directe
Dans le champ de commande ISPF :
```
MEMLIST 'IBMUSER.ASM.SOURCE'
```
> *Analogue Unix :* `ls ~/ASM/SOURCE/`

---

## 3. Consulter un fichier (lecture seule — Browse)

### Depuis la liste des membres
Placez le curseur sur le membre et tapez `B` + Entrée.

### Commande directe
```
BROWSE 'IBMUSER.ASM.SOURCE(MYPGM)'
```
> *Analogue Unix :* `cat ~/ASM/SOURCE/MYPGM` ou `less ~/ASM/SOURCE/MYPGM`

### Navigation dans le Browse

| Commande / Touche | Action | Analogue Unix |
|---|---|---|
| **F8** | Page suivante | `Space` dans `less` |
| **F7** | Page précédente | `b` dans `less` |
| **F10** | Scroll gauche | — |
| **F11** | Scroll droite | — |
| `FIND texte` | Rechercher | `grep texte` ou `/texte` dans `less` |
| `FIND texte PREV` | Recherche vers le haut | `?texte` dans `less` |
| `TOP` | Aller au début | `gg` dans `vi` |
| `BOTTOM` | Aller à la fin | `G` dans `vi` |
| **F3** | Quitter | `q` dans `less` |

---

## 4. Éditer un fichier

### Depuis la liste des membres
Placez le curseur sur le membre et tapez `E` + Entrée.

### Commande directe
```
EDIT 'IBMUSER.ASM.SOURCE(MYPGM)'
```
> *Analogue Unix :* `vi ~/ASM/SOURCE/MYPGM`

### Commandes essentielles dans l'éditeur ISPF

#### Navigation

| Commande / Touche | Action | Analogue Unix (`vi`) |
|---|---|---|
| **F8** | Page suivante | `Ctrl+F` |
| **F7** | Page précédente | `Ctrl+B` |
| `TOP` | Aller au début | `gg` |
| `BOTTOM` | Aller à la fin | `G` |
| `LOCATE 150` | Aller à la ligne 150 | `:150` |

#### Recherche et remplacement

| Commande | Action | Analogue Unix |
|---|---|---|
| `FIND texte` | Chercher un texte | `/texte` |
| `FIND texte PREV` | Chercher vers le haut | `?texte` |
| `CHANGE /ancien/nouveau/` | Remplacer la 1ère occurrence | `:s/ancien/nouveau/` |
| `CHANGE /ancien/nouveau/ ALL` | Remplacer toutes les occurrences | `:%s/ancien/nouveau/g` |

#### Édition de lignes (préfixes de ligne)

Ces commandes se tapent **dans la numérotation de ligne** à gauche :

| Préfixe | Action | Analogue Unix |
|---|---|---|
| `I` | Insérer une ligne après | `o` dans `vi` |
| `D` | Supprimer la ligne | `dd` dans `vi` |
| `Dnn` | Supprimer `nn` lignes (ex. `D5`) | `5dd` dans `vi` |
| `R` | Répéter (dupliquer) la ligne | `yy` + `p` dans `vi` |
| `C` | Copier la ligne (+ `A` ou `B` pour coller) | `yy` dans `vi` |
| `M` | Déplacer la ligne (+ `A` ou `B` pour coller) | `dd` + `p` dans `vi` |
| `A` | Coller **après** la ligne marquée | `p` dans `vi` |
| `B` | Coller **avant** la ligne marquée | `P` dans `vi` |

#### Sauvegarde et sortie

| Commande / Touche | Action | Analogue Unix (`vi`) |
|---|---|---|
| **F3** | Sauvegarder et quitter | `:wq` |
| `SAVE` | Sauvegarder sans quitter | `:w` |
| `CANCEL` | Quitter **sans** sauvegarder | `:q!` |
| `UNDO` | Annuler la dernière action | `u` dans `vi` |

---

## 5. Créer un nouveau membre

Dans la liste des membres du PDS, tapez le nom souhaité dans le champ **Name** en haut et appuyez sur Entrée. L'éditeur s'ouvre sur un fichier vide.

Ou en commande directe :
```
EDIT 'IBMUSER.ASM.SOURCE(NEWPGM)'
```
> *Analogue Unix :* `vi ~/ASM/SOURCE/NEWPGM`

---

## 6. Copier / Renommer un dataset ou membre

### Via le menu ISPF
```
=3.3  (Move/Copy Utility)
```

### Depuis la DSLIST — commandes de ligne
Placez le curseur sur le dataset :

| Commande | Action |
|---|---|
| `C` | Copier le dataset |
| `R` | Renommer le dataset |
| `D` | Supprimer le dataset |

> *Analogue Unix :* `cp`, `mv`, `rm`

---

## 7. Allouer un nouveau dataset

```
=3.2  (Data Set Utility → Allocate)
```
> *Analogue Unix :* `mkdir` pour un PDS, `touch` pour un séquentiel

---

## 8. Référence rapide des raccourcis de navigation ISPF

| Raccourci | Destination |
|---|---|
| `=3.4` | DSLIST — lister des datasets |
| `=3.3` | Move/Copy Utility |
| `=3.2` | Allocate / Delete / Rename dataset |
| `=2` | Éditeur ISPF direct |
| `=1` | Browse direct |
| **F3** | Retour / Quitter l'écran courant |
| **F1** | Aide contextuelle |
| **F12** | Annuler / Retour sans valider |

---

> **Conseil :** En mode TSO natif (hors ISPF), la commande `LISTDS 'IBMUSER.*' MEMBERS` liste les datasets et leurs membres directement dans le terminal, similaire à un `find ~ -type f`.
