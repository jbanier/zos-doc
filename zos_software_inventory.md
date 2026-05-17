# Trouver le software installé sur z/OS — Guide utilisateur TSO/ISPF

> **Contexte :** Ce guide est destiné à un utilisateur TSO standard (non opérateur, non RACF admin).  
> Il couvre les méthodes accessibles sans droits élevés pour localiser les produits installés,  
> notamment les bibliothèques compilateurs.

---

## Vue d'ensemble des approches

```
1. SMP/E (gestionnaire de produits)   →  source officielle, la plus fiable
2. Lister les datasets système         →  recherche par convention de nommage
3. Commandes opérateur via SDSF        →  selon droits disponibles
4. Catalogue système                   →  recherche dans le catalogue ICF
```

---

## Méthode 1 — SMP/E (System Modification Program/Extended)

SMP/E est **le gestionnaire de paquets de z/OS**, l'équivalent de `rpm -qa` ou `dpkg -l` sous Linux.  
C'est la source la plus fiable pour savoir ce qui est installé.

### Accéder à SMP/E via ISPF
```
=S.M    (si le panneau SDSF inclut SMP/E)
```
ou directement :
```
=10     (selon la configuration locale — peut varier)
```

### Via une commande ISPF directe
Dans le champ de commande ISPF :
```
SMPE
```

### Ce que vous cherchez dans SMP/E

Une fois dans SMP/E, naviguez vers **Global Zone** → **Installed FMIDs**.  
Vous y trouverez la liste de tous les produits installés avec :

| Champ | Signification |
|-------|--------------|
| FMID  | Identifiant unique du produit (ex. `HIG7780` = COBOL) |
| SYSMOD | Numéro de correctif appliqué |
| DATE  | Date d'installation |

> **Pour COBOL for z/OS**, le FMID commence typiquement par `HIG` (ex. `HIG7780` pour V6R1).

---

## Méthode 2 — Recherche dans le catalogue par convention de nommage

IBM respecte des **conventions de nommage strictes** pour ses bibliothèques système.  
On peut donc les retrouver par préfixe dans le catalogue ICF.

### Préfixes courants des produits IBM

| Produit | Préfixe dataset typique |
|---------|------------------------|
| COBOL for z/OS | `IGY.*` |
| PL/I for z/OS | `IBM.PLI.*` ou `IBMZ.PLI.*` |
| Assembler (HLASM) | `ASM.*` ou `SYS1.MACLIB` |
| C/C++ for z/OS | `CBC.*` |
| DB2 | `DSN.*` ou `DSNC.*` |
| CICS | `DFH.*` |
| IMS | `DFS.*` |
| MQ | `CSQ.*` |
| Java (IBM SDK) | `JAVA.*` ou `IBMJAVA.*` |
| z/OS base | `SYS1.*` |
| RACF | `SYS1.LINKLIB` (intégré au noyau) |

### Lister via DSLIST
```
=3.4
```
Dans le champ **Dsname Level**, tapez le préfixe :
```
IGY.*
```
Appuyez sur **Entrée**. Vous verrez tous les datasets COBOL installés.

> *Analogue Unix :* `find / -name "cobol*" -type d`

Faites de même pour les autres produits :
```
CBC.*      ← C/C++
ASM.*      ← Assembler HLASM
DSN.*      ← DB2
```

### Identifier la bibliothèque compilateur COBOL

Parmi les résultats `IGY.*`, cherchez un dataset dont le nom contient **SIGYCOMP** :

```
IGY.V6R1M0.SIGYCOMP    ← bibliothèque du compilateur COBOL
IGY.V6R1M0.SIGYSAMP    ← exemples fournis par IBM
IGY.V6R1M0.SIGYMAC     ← macros COBOL
```

Le numéro de version est encodé dans le nom : `V6R1M0` = Version 6, Release 1, Modification 0.

---

## Méthode 3 — Interroger le catalogue ICF directement (LISTCAT)

La commande TSO `LISTCAT` est l'équivalent de `ls` sur le catalogue système.

### Depuis la ligne de commande TSO
Appuyez sur **F6** (ou `=6` depuis ISPF) pour ouvrir une ligne de commande TSO, puis :

```tso
LISTCAT LEVEL(IGY) ALL
```

Résultat : liste tous les datasets dont le HLQ est `IGY`, avec volume, date de création, etc.

```tso
LISTCAT LEVEL(CBC) ALL       ← C/C++
LISTCAT LEVEL(SYS1) ALL      ← datasets système de base
LISTCAT LEVEL(DSN) ALL       ← DB2
```

> *Analogue Unix :* `find /usr -maxdepth 2 -type d | grep -i cobol`

### Variante — chercher dans tout le catalogue
```tso
LISTCAT LEVEL(*) ENTRIES(IGY.*.SIGYCOMP)
```
> ⚠️ Cette commande peut être longue si le catalogue est volumineux.

---

## Méthode 4 — Commandes opérateur via SDSF (si autorisé)

Ces commandes sont émises depuis SDSF et donnent des informations système en temps réel.

### Accéder à la console SDSF
```
=S.MAS    ou    =S.DA
```
Puis dans le champ de commande SDSF, tapez `/` suivi de la commande opérateur.

### Commandes utiles

| Commande | Information retournée | Analogue Unix |
|----------|----------------------|---------------|
| `/D PROG,APF` | Liste des bibliothèques APF autorisées (contient souvent les compilateurs) | `cat /etc/ld.so.conf` |
| `/D PROG,LNK` | LNKLIST — bibliothèques dans le link list système | `echo $PATH` |
| `/D PROG,LPA` | LPA — modules chargés en mémoire basse | `lsmod` |
| `/D IPLINFO` | Informations sur le dernier IPL (boot) et version z/OS | `uname -a` |
| `/D PROD,INSTALLED` | **Liste des produits IBM installés** (z/OS 2.2+) | `rpm -qa` |

> ⚠️ Si la commande retourne `IEE345I COMMAND REJECTED`, vous n'avez pas les droits suffisants.  
> Contactez votre administrateur système.

### La commande la plus utile : `D PROD,INSTALLED`
```
/D PROD,INSTALLED
```
Retourne directement la liste des produits IBM avec leur version, par exemple :
```
PRODUCT            VERSION  RELEASE  FMID
COBOL FOR Z/OS     6        1        HIG7780
C/C++ FOR Z/OS     2        1        HCCA310
HLASM TOOLKIT      1        6        HAS1B60
```

---

## Méthode 5 — Chercher dans la LNKLIST via ISPF

La **Link List** contient les bibliothèques accessibles à tous les programmes sans `STEPLIB` explicite.  
Les compilateurs y sont souvent référencés.

### Via SYS1.PARMLIB
```
BROWSE 'SYS1.PARMLIB(LNKLST00)'
```
ou selon la configuration :
```
BROWSE 'SYS1.PARMLIB(LNKLST01)'
```

Ce membre liste toutes les bibliothèques dans le link list. Cherchez-y `IGY`, `CBC`, `ASM`, etc.

> *Analogue Unix :* `cat /etc/environment` ou `echo $PATH`

---

## Récapitulatif — Trouver la bibliothèque compilateur COBOL

Voici la démarche recommandée dans l'ordre :

```
1.  =3.4  →  Dsname Level : IGY.*
            Cherchez un dataset nommé *.SIGYCOMP

2.  TSO LISTCAT LEVEL(IGY) ALL
            Vérifiez le nom exact et le volume

3.  BROWSE 'SYS1.PARMLIB(LNKLST00)'
            Confirmez que la lib est dans le link list

4.  SDSF : /D PROD,INSTALLED    (si droits disponibles)
            Liste officielle de tous les produits
```

Une fois le nom trouvé (ex. `IGY.V6R1M0.SIGYCOMP`), utilisez-le dans votre JCL :
```jcl
//STEPLIB  DD  DSN=IGY.V6R1M0.SIGYCOMP,DISP=SHR
```

---

## Tableau de référence rapide

| Objectif | Commande / Chemin | Analogue Unix |
|----------|------------------|---------------|
| Lister produits installés | `/D PROD,INSTALLED` (SDSF) | `rpm -qa` |
| Trouver datasets d'un produit | `=3.4` → préfixe `IGY.*` | `find / -name "libcobol*"` |
| Lister le catalogue | `TSO LISTCAT LEVEL(IGY) ALL` | `locate cobol` |
| Voir le link list | `BROWSE 'SYS1.PARMLIB(LNKLST00)'` | `echo $PATH` |
| Voir les APF libs | `/D PROG,APF` (SDSF) | `cat /etc/ld.so.conf` |
| Gestionnaire de paquets | `SMPE` (ISPF) | `rpm` / `dpkg` |

---

> **Conseil :** Si vous n'avez pas les droits pour les commandes opérateur, la méthode `=3.4`  
> avec le préfixe `IGY.*` reste la plus rapide et ne nécessite aucun droit particulier.  
> Votre administrateur système (`SYSADMIN`) ou DBA peut également vous fournir la liste  
> des bibliothèques à utiliser dans vos JCL.
