# 13 — Dépôt (cash-in) et retrait (cash-out) en agence par QR Code

> Entrée pour `/speckit-specify`. Référence : cahier des charges §6.1 (MAG-01 à MAG-05, MAG-07),
> §6.2 (MCL-05), §6.3 (séquence QR).

## Contexte et objectif

Le client doit pouvoir convertir des espèces en argent électronique et inversement dans n'importe
quelle agence, sans saisir son numéro ni montrer son code PIN à l'agent : un QR Code relie les deux
téléphones et chaque partie confirme de son côté. L'application agence est l'outil de l'agent pour
ces opérations.

## Acteurs

Client (application client), Agent d'agence (application agence).

## Parcours utilisateurs

### P1 — Retirer des espèces (cash-out)
En tant que client, je veux saisir le montant, afficher un QR Code, laisser l'agent le scanner,
confirmer avec mon PIN et recevoir mes espèces.

- Étant donné 50 000 XOF disponibles, quand je demande un retrait de 20 000 XOF, alors le total du devis
  (20 000 XOF, frais et taxes éventuelles) est réservé et un QR Code valable 5 minutes s'affiche.
- Étant donné que l'agent scanne le QR, alors il voit mon nom, le montant et les frais ; si le
  montant dépasse le seuil, il doit contrôler ma pièce d'identité.
- Étant donné que l'agent accepte, alors je vois le détail (agence, agent, montant) et confirme
  par PIN ; mon wallet est débité, le float de l'agence crédité, la caisse de l'agent enregistre
  une sortie de 20 000 XOF et l'agent voit en grand « Remettre 20 000 FCFA ».
- Étant donné que le QR expire ou que l'agent refuse, alors la réservation est libérée.
- Étant donné que deux agents scannent le même QR, alors un seul peut poursuivre.

### P1 — Déposer des espèces (cash-in)
En tant qu'agent, je veux saisir le montant reçu, afficher un QR Code, laisser le client le
scanner et confirmer, puis encaisser.

- Étant donné un float suffisant, quand le client confirme par PIN un dépôt de 30 000 XOF, alors son
  wallet est crédité, le float de l'agence débité, la caisse de l'agent enregistre une entrée de
  30 000 XOF et l'agent voit « Encaisser 30 000 FCFA ».
- Étant donné un float insuffisant, alors le dépôt est refusé avant la génération du QR.
- Étant donné que le dépôt ferait dépasser le plafond du client, alors il est refusé.

### P2 — Dépôt avec le QR « profil » du client
En tant que client, je veux présenter un QR permanent qui m'identifie ; l'agent le scanne, saisit
le montant et je confirme sur mon téléphone.

### P2 — Application agence au quotidien
En tant qu'agent, je veux voir sur l'accueil de l'application mon float, l'état de ma caisse et les
opérations du jour, partager un reçu, et imprimer sur une imprimante thermique Bluetooth.

## Règles métier

- Le QR Code ne contient aucune donnée personnelle ni montant modifiable ; il est à usage unique,
  signé, et expire après 5 minutes (paramétrable).
- Confirmation finale toujours faite par le client, avec son PIN, sur son propre téléphone.
- Contrôle de pièce obligatoire au-delà d'un seuil paramétrable.
- Dépôt et retrait ne sont possibles que dans une agence du pays du wallet (même devise).
- Toutes les opérations exigent : caisse de l'agent ouverte, agence active, téléphone de l'agent
  non modifié, contrôle de conformité non bloquant, plafonds respectés.
- Les deux écrans se mettent à jour en temps réel à chaque étape.
- Aucune opération n'est possible hors connexion.

## Données manipulées

Opération wallet (dépôt ou retrait : montant, frais, taxes, agence, agent, statut), Jeton QR
(objet, émetteur, montant, expiration, statut), Mouvement de caisse, Reçu.

## Cas limites

- Le client ferme l'application avant de confirmer : expiration, réservation libérée.
- Perte de réseau côté agent après la confirmation du client : l'agent retrouve le statut à la
  reconnexion avant de remettre les espèces.
- Capture d'écran du QR envoyée à un tiers : inutilisable après usage ou expiration.

## Critères de succès

- Retrait complet en moins de 60 secondes.
- Zéro double utilisation d'un QR en recette.
- Écarts de caisse imputables aux opérations QR : zéro.

## Hors périmètre

Retrait sans application (par code USSD/SMS), Mobile Money externe (version 2).

## Points à clarifier

- Frais de retrait et de dépôt (et taxes applicables).
- Seuil de montant imposant le contrôle de pièce.
- Les retraits sont-ils possibles dans toutes les agences, y compris sous-agents ?

## Dépendances

06, 11.
