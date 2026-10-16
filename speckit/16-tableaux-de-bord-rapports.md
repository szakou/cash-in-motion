# 16 — Tableaux de bord, rapports et états réglementaires

> Entrée pour `/speckit-specify`. Référence : cahier des charges §5.1 (ADM-09, ADM-10), §5.3 (AGW-07), §7.3.

## Contexte et objectif

Le siège doit piloter l'activité globale, les agences doivent suivre leur propre activité, et la
finance doit produire les états de taxes à reverser et les états réglementaires. Les chiffres
affichés doivent toujours correspondre exactement au grand livre.

## Acteurs

Administrateur, Responsable conformité, Superviseur d'agence, Agent d'agence. Les besoins de
l'équipe finance sont couverts par le rôle Administrateur (périmètre pays) en version 1.

## Parcours utilisateurs

### P1 — Tableau de bord global
En tant qu'administrateur, je veux voir pour une période (jour, semaine, mois, année) : nombre et
montant des envois, paiements, dépôts, retraits et P2P ; frais encaissés ; taxes collectées par
type et par pays (ex. TTA, TAF, THZUEMOA pour le Sénégal) ; commissions ; top 10 agences et corridors ; évolution dans le temps ;
comparaison avec la période précédente.

- Étant donné une journée avec des opérations connues, alors chaque indicateur est égal à la somme
  des opérations de cette journée dans le fuseau du pays concerné.

### P1 — Tableau de bord agence
En tant que superviseur ou agent, je veux le même type de tableau de bord limité à mon agence
(quotidien, hebdomadaire, mensuel, annuel), avec float et état des caisses.

- Étant donné un superviseur de l'agence A, alors aucune donnée d'une autre agence n'apparaît.

### P1 — État des taxes à reverser
En tant qu'administrateur (besoins finance), je veux pour un pays et une période un état détaillé et totalisé
de chaque taxe de ce pays collectée (ex. TTA, TAF, THZUEMOA au Sénégal), par opération et par
agence, exportable, prêt pour la déclaration à l'administration fiscale du pays.

### P2 — Exports
En tant qu'administrateur, je veux exporter en tableur ou PDF : journal des opérations,
commissions par agence, balance des comptes, encours de monnaie électronique et rapprochement
avec le compte de cantonnement de chaque pays, états réglementaires périodiques de chaque pays
selon son régulateur (BCEAO pour l'UEMOA, BEAC / COBAC pour la CEMAC, banque centrale nationale
ailleurs), état de compensation entre pays.

- Étant donné un export volumineux, alors il est préparé en arrière-plan et je suis averti quand il
  est prêt au téléchargement.

### P2 — Rapprochement
En tant qu'administrateur (besoins finance), je veux une comparaison automatique entre les chiffres des
rapports et le grand livre, avec la liste des écarts éventuels.

## Règles métier

- Les périodes sont calculées dans le fuseau du pays (ou de l'agence) ; la semaine commence le lundi.
- Les indicateurs sont présentés par pays et dans la devise du pays ; une vue consolidée
  multi-pays n'additionne jamais deux devises sans conversion explicite et affiche la devise de
  consolidation et les taux utilisés.
- Un utilisateur limité à certains pays ne voit que ces pays.
- Les tableaux de bord sont actualisés au moins toutes les 5 minutes.
- Les opérations annulées et remboursées apparaissent séparément et ne gonflent pas les volumes.
- Accès aux exports journalisé.

## Données manipulées

Indicateurs agrégés par période, agence, corridor et type d'opération ; Export (type, période,
statut, fichier) ; Écart de rapprochement.

## Cas limites

- Opération à cheval sur minuit : rattachée à la date de sa validation.
- Agence fermée en cours de période : ses chiffres restent dans l'historique.

## Critères de succès

- Écart nul entre tableaux de bord et grand livre sur toute la recette.
- Affichage d'un tableau de bord en moins de 2 secondes.
- Export de 100 000 lignes en moins de 60 secondes.

## Hors périmètre

Outil décisionnel externe, prévisions.

## Points à clarifier

- [À CLARIFIER] Formats exacts des états réglementaires (BCEAO, BEAC/COBAC, autres) et des
  déclarations fiscales de chaque pays d'opération.
- Devise de consolidation des vues multi-pays (XOF, EUR ?).
- Périodicité de reversement des taxes.

## Dépendances

09, 10, 12, 13.
