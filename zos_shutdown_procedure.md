# Procédure de Shutdown Propre — z/OS 1.10 sous Hercules 3.13

**Système** : DUZA | **Sysplex** : ROCCAPL | **Emulateur** : Hercules 3.13

---

## Contexte

Cette procédure est adaptée à une installation z/OS 1.10.00 (HBB7750) tournant sous Hercules avec :
- **SA z/OS** (System Automation) actif — relance automatiquement certains composants (`RESTARTOPT=ALWAYS`)
- **NetView** (`NTVDZ`) comme gestionnaire réseau
- **Console operator** accessible directement dans le terminal Hercules (préfixe `.`)
- **Pas de console 3270** connectée sur le port telnet

> ⚠️ Ne jamais tuer Hercules sans avoir effectué le `QUIESCE` z/OS au préalable, sous peine de corrompre les fichiers CCKD.

---

## Procédure Rapide (si SA z/OS répond)

```
.VARY CN(*),ACTIVATE
.INGSHUT
```

Attendre le message `BLW002I SYSTEM WAIT STATE 'CCC'X`, puis dans Hercules :

```
quit
```

---

## Procédure Complète (manuelle)

### Étape 1 — Activer la console système

```
.VARY CN(*),ACTIVATE
```

Réponse attendue : `IEE712I VARY CN PROCESSING COMPLETE`

> Cette commande est nécessaire au premier accès car la console système n'est pas activée par défaut.

---

### Étape 2 — Arrêter JES2

```
.$PJES2
```

**Cas nominal** : JES2 liste les address spaces encore actifs (`$HASP607`).

Si des sessions TSO sont actives, les annuler :
```
.C U=<username>
```

> SA z/OS peut rejeter `$PJES2` avec `AOF566I PROCESSING REJECTED IN AOFRSD07`. Ce n'est pas bloquant, continuer à l'étape suivante.

---

### Étape 3 — Répondre aux WTORs en attente

Les messages `*NN XXXXX` sont des WTORs (Write To Operator with Reply) qui attendent une réponse. Les identifier et y répondre :

```
.R <numéro>,<réponse>
```

Exemples courants :
- `*NN IKT012D TCAS TERMINATION` → `.R NN,U`
- `*NN DSI803A NTVDZ` (NetView) → ignorer si toutes les tentatives échouent, ce WTOR ne bloque pas le shutdown

---

### Étape 4 — Halt EOD

```
.Z EOD
```

Réponse attendue : `IEE334I HALT EOD SUCCESSFUL`

> Cette commande arrête la journalisation SMF et prépare le système au quiesce. SA z/OS peut continuer à relancer des composants — c'est normal, ignorer.

---

### Étape 5 — Quiesce z/OS

```
.QUIESCE
```

Réponses attendues :
```
CPU000X: SIGP Stop (05) CPU000X ...
HHCCP011I CPU000X: Disabled wait state
          PSW=00020000 80000000 0000000000000CCC
*BLW002I SYSTEM WAIT STATE 'CCC'X - QUIESCE FUNCTION PERFORMED
```

> Le wait state `CCC` confirme que z/OS est proprement arrêté. Tous les buffers disque sont flushés.

---

### Étape 6 — Quitter Hercules

```
quit
```

---

## Récapitulatif des Commandes

| Commande | Description |
|---|---|
| `.VARY CN(*),ACTIVATE` | Activer la console système |
| `.$PJES2` | Arrêter JES2 |
| `.C U=<user>` | Annuler une session utilisateur |
| `.R <nn>,<réponse>` | Répondre à un WTOR |
| `.Z EOD` | Halt End Of Day |
| `.QUIESCE` | Quiesce z/OS (flush disques) |
| `quit` | Quitter Hercules |

---

## Messages Clés

| Message | Signification |
|---|---|
| `IEE712I VARY CN PROCESSING COMPLETE` | Console activée |
| `$HASP098 WILL TERMINATE` | JES2 en cours d'arrêt |
| `IEE334I HALT EOD SUCCESSFUL` | EOD accepté |
| `BLW002I SYSTEM WAIT STATE 'CCC'X` | z/OS quiescé, OK pour quitter |

---

## Comportements Connus de ce Système

- **SA z/OS relance automatiquement** RMF, RMFGAT, VTAM/NET avec `RESTARTOPT=ALWAYS` — ne pas perdre de temps à les arrêter manuellement, aller directement au `Z EOD` + `QUIESCE`.
- **NetView (NTVDZ)** répond aux WTORs `DSI803A` par des rejets en boucle — ignorer et continuer.
- **`$PJES2` rejeté par AOF566I** — normal si SA z/OS est actif, ne pas bloquer sur cette erreur.
- La commande `.` (point) est le préfixe pour envoyer des commandes MVS depuis la console Hercules.

---

## En Cas de Corruption CCKD

Si Hercules a été tué brutalement et qu'un fichier CCKD est corrompu au démarrage :

```bash
# Sauvegarder le fichier corrompu
mv fichier.CCKD fichier.CCKD.bak

# Recréer un disque 3390-3 vierge
dasdinit -z fichier.CCKD 3390-3 VOLSER
```

> Le type `3390-3` correspond à 3339 cylindres, 15 pistes/cylindre, 56832 bytes/piste.
