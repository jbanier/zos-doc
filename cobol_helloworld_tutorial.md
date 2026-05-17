# Mini tutoriel COBOL z/OS — Hello World de A à Z

> **Environnement :** IBM COBOL for z/OS · TSO/ISPF · JCL soumis via ISPF · Résultats en SYSOUT (spool JES)  
> **Prérequis :** Être connecté en TSO sous `IBMUSER` et avoir accès à ISPF.

---

## Vue d'ensemble des étapes

```
1. Créer la source COBOL   →   IBMUSER.COBOL.SOURCE(HELLO)
2. Créer le JCL de compile →   IBMUSER.JCL(HELLOC)
3. Soumettre la compilation →   SUB  →  job HELLOC
4. Créer le JCL d'exécution →  IBMUSER.JCL(HELLOR)
5. Soumettre l'exécution    →   SUB  →  job HELLOR
6. Consulter les résultats  →   =S.ST  (SDSF/Spool)
```

---

## Étape 1 — Allouer les bibliothèques nécessaires

Si elles n'existent pas encore, il faut créer trois PDS :

| Dataset                  | Type | Utilité                     |
|--------------------------|------|-----------------------------|
| `IBMUSER.COBOL.SOURCE`   | PDS  | Sources COBOL               |
| `IBMUSER.COBOL.OBJ`      | PDS  | Modules objet (output compile) |
| `IBMUSER.COBOL.LOAD`     | PDSE | Load modules (exécutables)  |
| `IBMUSER.JCL`            | PDS  | Jobs JCL                    |

### Allouer via ISPF
```
=3.2
```
Pour chaque dataset, remplissez les champs :

```
Option      ===> A                    (Allocate)
Dataset name===> IBMUSER.COBOL.SOURCE
Volume      ===>                      (laisser vide = SMS)
Generic unit===>
Space units ===> TRACKS
Primary qty ===> 5
Secondary   ===> 5
Directory   ===> 10                   (0 pour séquentiel)
Record fmt  ===> FB
Record len  ===> 80
Block size  ===> 32720
Dataset type===> PDS
```

> Répétez l'opération pour `IBMUSER.COBOL.OBJ` (RECFM=FB, LRECL=80),  
> `IBMUSER.COBOL.LOAD` (RECFM=U, LRECL=0, type PDSE),  
> et `IBMUSER.JCL` (RECFM=FB, LRECL=80).

---

## Étape 2 — Écrire le programme COBOL

### Ouvrir l'éditeur
```
EDIT 'IBMUSER.COBOL.SOURCE(HELLO)'
```
ou depuis `=3.4` → entrez dans `IBMUSER.COBOL.SOURCE` → tapez `HELLO` dans le champ **Name** + Entrée.

### Saisir le code source

Copiez le programme suivant dans l'éditeur ISPF :

```cobol
000100 IDENTIFICATION DIVISION.
000200 PROGRAM-ID. HELLO.
000300 AUTHOR. IBMUSER.
000400*
000500 ENVIRONMENT DIVISION.
000600*
000700 DATA DIVISION.
000800*
000900 PROCEDURE DIVISION.
001000     DISPLAY 'HELLO WORLD FROM Z/OS COBOL'
001100     STOP RUN.
```

> **Attention :** En COBOL fixe (RECFM=FB LRECL=80) :
> - Colonnes 1–6   : numéros de séquence (optionnels mais conventionnels)
> - Colonne 7       : indicateur (`*` = commentaire, `-` = continuation)
> - Colonnes 8–11  : **Area A** (divisions, sections, noms de paragraphe)
> - Colonnes 12–72 : **Area B** (instructions)
> - Colonnes 73–80 : identification (ignorées par le compilateur)

### Sauvegarder
```
F3
```
---

## Étape 3 — Écrire le JCL de compilation

### Ouvrir l'éditeur
```
EDIT 'IBMUSER.JCL(HELLOC)'
```

### Saisir le JCL

```jcl
//HELLOC   JOB (ACCT),'HELLO COMPILE',CLASS=A,MSGCLASS=X,
//             MSGLEVEL=(1,1),NOTIFY=&SYSUID
//*
//* COMPILATION IBM COBOL FOR Z/OS
//*
//COBCL    EXEC PGM=IGYCRCTL,
//             PARM='OBJECT,APOST,LIST,MAP,XREF'
//STEPLIB  DD  DSN=IGY.V6R1M0.SIGYCOMP,DISP=SHR     <-- adapter la version
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  DSN=IBMUSER.COBOL.SOURCE(HELLO),DISP=SHR
//SYSLIN   DD  DSN=IBMUSER.COBOL.OBJ(HELLO),
//             DISP=(NEW,CATLG,DELETE),
//             SPACE=(TRK,(5,5)),
//             DCB=(RECFM=FB,LRECL=80,BLKSIZE=3200)
//SYSUT1   DD  UNIT=SYSALLDA,SPACE=(CYL,(1,1))
//SYSUT2   DD  UNIT=SYSALLDA,SPACE=(CYL,(1,1))
//SYSUT3   DD  UNIT=SYSALLDA,SPACE=(CYL,(1,1))
//SYSUT4   DD  UNIT=SYSALLDA,SPACE=(CYL,(1,1))
//SYSUT5   DD  UNIT=SYSALLDA,SPACE=(CYL,(1,1))
//SYSUT6   DD  UNIT=SYSALLDA,SPACE=(CYL,(1,1))
//SYSUT7   DD  UNIT=SYSALLDA,SPACE=(CYL,(1,1))
//*
//* LINK-EDIT
//*
//LKED     EXEC PGM=IEWL,COND=(8,LT,COBCL),
//             PARM='LIST,MAP,RENT'
//SYSPRINT DD  SYSOUT=*
//SYSLIN   DD  DSN=IBMUSER.COBOL.OBJ(HELLO),DISP=SHR
//SYSLMOD  DD  DSN=IBMUSER.COBOL.LOAD(HELLO),
//             DISP=(NEW,CATLG,DELETE),
//             SPACE=(TRK,(5,5,3))
//SYSUT1   DD  UNIT=SYSALLDA,SPACE=(CYL,(1,1))
```

> **Points clés du JCL :**
> - `NOTIFY=&SYSUID` : vous envoie un message TSO à la fin du job
> - `COND=(8,LT,COBCL)` : le link-edit ne s'exécute que si la compilation a réussi (RC < 8)
> - Adaptez `IGY.V6R1M0.SIGYCOMP` au nom exact de la bibliothèque compilateur sur votre système

### Sauvegarder
```
F3
```

---

## Étape 4 — Soumettre le job de compilation

Depuis l'éditeur du JCL `HELLOC`, tapez dans la ligne de commande :
```
SUB
```
puis **Entrée**.

ISPF affiche un message du type :
```
JOB HELLOC(JOB01234) SUBMITTED
```

> *Analogue Unix :* `sbatch helloc.sh` (soumission à un scheduler)

---

## Étape 5 — Vérifier la compilation dans le spool

### Accéder au spool JES (SDSF)
```
=S.ST
```
Vous voyez la liste de vos jobs. Repérez `HELLOC`.

### Codes retour à vérifier

| RC | Signification |
|----|---------------|
| 0  | Compilation parfaite |
| 4  | Warnings (souvent acceptable) |
| 8  | Erreurs — le link-edit n'a pas tourné |
| 12+ | Erreurs graves |

### Consulter le listing de compilation
Placez le curseur sur le job `HELLOC` et tapez `S` + Entrée pour entrer dans le job.  
Puis tapez `S` sur le step `COBCL` → `SYSPRINT` pour voir le listing COBOL complet.

> Recherchez `RETURN CODE` en bas du listing pour confirmer le RC.

---

## Étape 6 — Écrire le JCL d'exécution

### Ouvrir l'éditeur
```
EDIT 'IBMUSER.JCL(HELLOR)'
```

### Saisir le JCL

```jcl
//HELLOR   JOB (ACCT),'HELLO RUN',CLASS=A,MSGCLASS=X,
//             MSGLEVEL=(1,1),NOTIFY=&SYSUID
//*
//* EXECUTION DU PROGRAMME HELLO
//*
//RUN      EXEC PGM=HELLO
//STEPLIB  DD  DSN=IBMUSER.COBOL.LOAD,DISP=SHR
//SYSOUT   DD  SYSOUT=*
//SYSPRINT DD  SYSOUT=*
```

### Sauvegarder et soumettre
```
F3
```
Rouvrez le membre et tapez :
```
SUB
```

---

## Étape 7 — Consulter les résultats d'exécution

```
=S.ST
```

Repérez le job `HELLOR`. Le RC devrait être `0000`.

Entrez dans le job (`S`) → step `RUN` → DD `SYSOUT`.

Vous devriez voir :
```
HELLO WORLD FROM Z/OS COBOL
```

---

## Récapitulatif des commandes directes utilisées

| Action | Commande ISPF | Analogue Unix |
|--------|--------------|---------------|
| Éditer la source COBOL | `EDIT 'IBMUSER.COBOL.SOURCE(HELLO)'` | `vi ~/cobol/HELLO.cbl` |
| Éditer le JCL compile | `EDIT 'IBMUSER.JCL(HELLOC)'` | `vi ~/jcl/helloc.jcl` |
| Soumettre un job | `SUB` (depuis l'éditeur) | `sbatch helloc.sh` |
| Voir les jobs en cours | `=S.ST` | `squeue` / `bjobs` |
| Voir le spool d'un job | `S` sur le job dans SDSF | `cat slurm-XXXX.out` |
| Allouer un dataset | `=3.2` | `mkdir` |

---

## En cas de problème

| Symptôme | Cause probable | Action |
|----------|---------------|--------|
| RC=8 à la compilation | Erreur syntaxe COBOL | Consulter `SYSPRINT` du step COBCL |
| RC=8 au link-edit | Module objet introuvable | Vérifier `IBMUSER.COBOL.OBJ(HELLO)` |
| `806 ABEND` à l'exécution | Load module introuvable | Vérifier `STEPLIB` dans HELLOR |
| `S0C7 ABEND` | Erreur de données en mémoire | Revoir la logique COBOL |
| JOB non soumis | Erreur JCL syntaxique | Vérifier le JCL ligne par ligne |

---

> **Conseil :** Une fois à l'aise, vous pouvez combiner compilation et exécution dans un seul JCL en ajoutant un step `EXEC PGM=HELLO` après le link-edit, conditionné par `COND=(8,LT,LKED)`.
