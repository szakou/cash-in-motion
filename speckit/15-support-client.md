# 15 — Support client : recherche et traitement des réclamations

> Entrée pour `/speckit-specify`. Référence : cahier des charges §5.2 (SUP-01 à SUP-06).

## Contexte et objectif

Quand un client appelle (« mon bénéficiaire n'a pas reçu le code », « mon retrait n'a pas
abouti »), l'agent support doit retrouver l'opération en quelques secondes, comprendre ce qui
s'est passé et agir dans un cadre limité, sans exposer inutilement les données personnelles.

## Acteurs

Agent support, Superviseur d'agence (validation des annulations), Responsable conformité.

## Parcours utilisateurs

### P1 — Retrouver une opération
En tant qu'agent support, je veux rechercher un transfert ou une opération wallet par code,
téléphone, nom, agence, date ou montant.

- Étant donné un code de retrait, quand je le saisis, alors le transfert s'affiche en moins de 3 clics.
- Étant donné un numéro de téléphone, alors je vois les opérations où il apparaît (expéditeur,
  bénéficiaire ou client wallet).

### P1 — Comprendre ce qui s'est passé
En tant qu'agent support, je veux voir le détail complet et la chronologie d'une opération : chaque
statut avec date, acteur et motif, agences et agents, notifications envoyées et leur statut de remise.

### P1 — Données masquées par défaut
- Étant donné une fiche client, alors téléphone, numéro de pièce et adresse sont masqués ; pour les
  voir, je saisis un motif et l'action est journalisée.

### P2 — Agir dans un cadre limité
En tant qu'agent support, je veux renvoyer le code de retrait, demander une annulation (validée par
un superviseur), demander à la conformité le déblocage d'un transfert bloqué par des réponses erronées (après
vérification de l'identité), et
ouvrir un ticket de suivi.

### P2 — Partir de la référence d'erreur
En tant qu'agent support, quand un client ou un agent me communique la **référence** affichée sur
son écran d'erreur, je veux retrouver l'opération et l'historique technique complet de cette
action, afin de diagnostiquer sans lui redemander de détails.

- Étant donné une erreur affichée avec la référence R, quand je saisis R, alors je vois l'opération
  concernée (s'il y en a une), son statut et la chronologie de traitement, et je peux transmettre R
  à l'équipe technique pour l'analyse complète.

### P2 — Consulter un wallet
En tant qu'agent support, je veux consulter en lecture seule le wallet d'un client : solde,
opérations, niveau et statut d'identification.

## Règles métier

- Le support ne peut ni créer ni modifier un montant ; il ne peut ni payer ni rembourser lui-même.
- Un agent support limité à certains pays ne voit que les opérations dont l'origine ou la
  destination est dans ses pays ; les recherches acceptent les numéros de tous les pays.
- Toute révélation de donnée et toute action est journalisée avec son motif.
- Renvoi de code limité à 3 par 24 h et par transfert.
- Un ticket a un statut (ouvert, en cours, résolu), un historique et est lié à l'opération.

## Données manipulées

Vue opération (transfert ou wallet) et sa chronologie, Ticket de support, Journal de révélation.

## Cas limites

- Opération en attente de conformité : le support voit le statut mais pas le motif détaillé.
- Recherche trop large : résultats limités et invitation à préciser.

## Critères de succès

- Temps moyen pour retrouver une opération inférieur à 30 secondes.
- 100 % des révélations de données personnelles tracées.

## Hors périmètre

Centre d'appels, chat client (version 2).

## Points à clarifier

- Les tickets doivent-ils s'intégrer à un outil existant ?

## Dépendances

09, 10, 11, 14.
