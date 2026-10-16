# 06 — Agences, caisses, float et commissions

> Entrée pour `/speckit-specify`. Référence : cahier des charges §5.1 (ADM-01, ADM-07), §5.3
> (AGW-06, AGW-08, AGW-09), §6.1 (MAG-04).

## Contexte et objectif

Le réseau d'agences est le canal principal. Chaque agence travaille avec un float prépayé
(valeur électronique que la plateforme lui doit) et chaque agent tient une caisse d'espèces.
Le siège doit ouvrir, piloter et approvisionner les agences ; les agences doivent maîtriser leur
liquidité et leur caisse au quotidien, et être rémunérées par des commissions.

## Acteurs

Administrateur, Superviseur d'agence, Agent d'agence.

## Parcours utilisateurs

### P1 — Ouvrir et gérer une agence
En tant qu'administrateur (global ou du pays), je veux créer une agence dans un pays « en
opération » (code, nom, type propre ou sous-agent, pays, adresse, ville, téléphone, fuseau
horaire, horaires, position géographique, plafonds journaliers d'envoi et de paiement, plan de
commission), la suspendre ou la clôturer.

- Étant donné une agence créée à Douala, alors son float et ses caisses sont en XAF et ses
  journées suivent le fuseau du Cameroun.
- Étant donné un pays qui n'est pas « en opération », alors on ne peut pas y créer d'agence.

- Étant donné une agence créée, quand elle est activée, alors son float existe avec un solde nul.
- Étant donné une agence suspendue, quand un de ses agents tente une opération, alors elle est refusée.

### P1 — Ouvrir et clôturer sa caisse
En tant qu'agent, je veux ouvrir ma caisse en début de journée avec un fonds de caisse, voir les
entrées et sorties d'espèces de la journée et la clôturer en déclarant le montant compté.

- Étant donné une caisse fermée, quand l'agent tente un envoi, alors c'est refusé (« caisse non ouverte »).
- Étant donné une clôture où le montant compté diffère du montant attendu, quand l'agent clôture,
  alors l'écart est calculé et la clôture attend la validation du superviseur.
- Étant donné une clôture sans écart, quand l'agent clôture, alors la caisse est fermée immédiatement.

### P1 — Suivre et approvisionner le float
En tant que superviseur, je veux voir le float disponible de l'agence et demander un
réapprovisionnement ; en tant qu'administrateur, je veux approuver la demande en saisissant la
référence du versement bancaire.

- Étant donné un float de 50 000 XOF, quand un envoi de 104 244 XOF est tenté, alors il est refusé
  (« float insuffisant »).
- Étant donné un float passant sous le seuil d'alerte de l'agence, alors superviseur et siège sont alertés.
- Étant donné une demande approuvée de 2 000 000 XOF, alors le float augmente de ce montant.

### P2 — Respecter les plafonds journaliers
- Étant donné un plafond d'envoi journalier atteint, quand un nouvel envoi est tenté, alors il est
  refusé jusqu'au lendemain.

### P2 — Calculer les commissions
En tant que superviseur, je veux consulter les commissions de l'agence et de chaque agent par
période, calculées automatiquement selon le plan de commission de l'agence.

### P3 — Trouver une agence
En tant que client, je veux voir les agences actives proches de moi avec leurs horaires.

## Règles métier

- Effets sur le float : envoi et dépôt wallet le diminuent ; paiement et retrait wallet
  l'augmentent ; réapprovisionnement validé l'augmente ; retrait de float le diminue.
- Effets sur la caisse : envoi et dépôt = entrée d'espèces ; paiement et retrait = sortie.
- Un agent ne peut avoir qu'une caisse ouverte à la fois ; une caisse est mono-devise, dans la
  devise du pays de l'agence ; le float de l'agence aussi.
- Les demandes de float d'un pays sont validées par un administrateur ayant ce pays dans son
  périmètre et alimentent le compte de cantonnement de ce pays.
- « Journée » (plafonds journaliers, caisse de la veille) = journée dans le fuseau de l'agence.
- Toute opération espèces exige une caisse ouverte au nom de l'agent.
- Commission = part des frais définie par le plan (ex. 40 % des frais d'envoi), attribuée à
  l'agence et à l'agent qui a réalisé l'opération.
- Les demandes de float et les clôtures avec écart sont journalisées.

## Données manipulées

Agence, Caisse (fonds d'ouverture, solde attendu, solde déclaré, écart, statut), Mouvement de
caisse, Demande de float, Plan de commission, Commission.

## Cas limites

- Caisse restée ouverte la veille : signalée, nouvelle opération interdite tant qu'elle n'est pas clôturée.
- Agence clôturée avec float non nul : clôture refusée tant que le float n'est pas soldé.
- Annulation d'un envoi : contre-passe aussi la commission correspondante.

## Critères de succès

- Écart entre caisse théorique et réelle visible en temps réel par le superviseur.
- Demande de float traitée de bout en bout sans échange hors système.

## Hors périmètre

Paiement effectif des commissions aux agences (virement) ; gestion RH des agents.

## Points à clarifier

- Plans de commission (taux par type d'opération, partage agence/agent).
- Seuil d'alerte de float par défaut.
- Les sous-agents ont-ils des règles différentes des agences propres ?

## Dépendances

01, 05.
