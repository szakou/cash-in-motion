# 09 — Envoi d'un transfert en agence

> Entrée pour `/speckit-specify`. Référence : cahier des charges §3.3, §4.5, §5.3 (AGW-01 à AGW-03).

## Contexte et objectif

C'est le service principal : un expéditeur se présente au guichet, paie en espèces, et son
bénéficiaire pourra retirer l'argent dans une autre agence grâce à un code. Le parcours doit être
rapide (moins de 2 minutes), exact au franc près et impossible à dupliquer par erreur.

## Acteurs

Agent d'agence, Expéditeur, Bénéficiaire (destinataire des notifications).

## Parcours utilisateurs

### P1 — Envoyer de l'argent
En tant qu'agent, je veux, en quatre étapes (expéditeur → bénéficiaire → montant et devis →
confirmation), enregistrer l'envoi, encaisser le client et lui remettre un code et un reçu.

- Étant donné un expéditeur identifié, un bénéficiaire au Mali, 100 000 XOF, des frais de 3 000 XOF
  et un message, quand l'agent confirme, alors : le client paie 104 244 XOF, un code de retrait
  unique est généré, le reçu détaille toutes les lignes (montant, frais de transfert, frais de
  message, chaque taxe du pays d'envoi — au Sénégal TTA, TAF, THZUEMOA —, total taxe, rabais, total, taux, montant à payer), le float de
  l'agence diminue de 104 244 XOF et la caisse de l'agent enregistre une entrée de 104 244 XOF.
- Étant donné que l'agent clique deux fois sur « Confirmer », alors un seul envoi est créé.
- Étant donné un devis expiré au moment de confirmer, alors l'agent doit recalculer avant de valider.
- Étant donné un envoi mis en attente par la conformité, alors l'agent est informé, le code n'est
  pas communiqué au client et l'argent est conservé en transit jusqu'à décision.

### P1 — Notifier les parties
- Étant donné un envoi disponible, alors l'expéditeur reçoit une confirmation et le bénéficiaire,
  s'il a un téléphone, reçoit le code et le montant.

### P2 — Rendre la monnaie
En tant qu'agent, je veux saisir le montant reçu en espèces et voir la monnaie à rendre.

### P2 — Réimprimer un reçu
En tant qu'agent, je veux réimprimer le reçu d'un envoi de mon agence (mention « duplicata »).

### P2 — Envoyer depuis l'application mobile agence
En tant qu'agent, je veux réaliser le même envoi depuis mon téléphone, avec les mêmes règles.

## Règles métier

- L'envoi se fait depuis une agence d'un pays « en opération », dans la devise de ce pays, vers
  tout pays de destination d'un corridor actif (ex. Sénégal → Cameroun : payé en XOF, reçu en XAF).
- Le reçu est édité dans la langue de l'agence et mentionne les taxes sous leur libellé national.
- Préconditions : agent avec caisse ouverte, agence active, plafond journalier non dépassé, float
  suffisant, expéditeur sans pièce expirée, devis valide, contrôle de conformité non bloquant.
- Le code de retrait est unique, non prévisible, de 10 chiffres plus un chiffre de contrôle,
  affiché par groupes (123 456 789 0).
- Les montants de l'envoi sont exactement ceux du devis.
- Statuts : créé → (en attente de conformité) → disponible.
- Le reçu existe au format A4 et ticket (58/80 mm) et porte un QR de vérification.
- Le message au bénéficiaire est limité à 160 caractères et génère des frais.

## Données manipulées

Envoi (code, date et heure, agent et agence émetteurs, expéditeur, bénéficiaire, montant envoyé,
frais de transfert, de message, de livraison, chaque taxe, total taxe, rabais, total, taux de
change, montant à payer, devises, statut, historique des statuts), Reçu.

## Cas limites

- Coupure réseau après confirmation : l'agent peut vérifier si l'envoi a été créé et réimprimer le
  reçu, sans risque de doublon.
- Bénéficiaire sans téléphone : le code est remis uniquement à l'expéditeur.
- Corridor désactivé entre le devis et la confirmation : refus.

## Critères de succès

- Parcours complet réalisé en moins de 2 minutes par un agent formé.
- Zéro doublon en recette, y compris en cas de double clic ou de coupure réseau.
- Reçu conforme ligne à ligne à l'exemple chiffré du cahier des charges.

## Hors périmètre

Paiement, annulation et expiration (10) ; envoi depuis le wallet client vers une agence (version 2).

## Points à clarifier

- Livraison à domicile : disponible au lancement ? sur quels corridors ?
- Le reçu doit-il être signé par le client (version papier) ?

## Dépendances

04, 05, 06, 07, 08, 14 (envoi du code par SMS).
