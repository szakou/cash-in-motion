# 10 — Paiement, annulation, expiration et remboursement d'un transfert

> Entrée pour `/speckit-specify`. Référence : cahier des charges §5.3 (AGW-04, AGW-05), §6.1 (MAG-06).

## Contexte et objectif

Le bénéficiaire retire son argent dans n'importe quelle agence du réseau en présentant le code et
sa pièce d'identité. La plateforme doit garantir qu'un transfert n'est payé qu'une seule fois, à la
bonne personne, et permettre d'annuler ou de rembourser proprement.

## Acteurs

Agent d'agence (payeur), Superviseur d'agence (annulation), Bénéficiaire, Expéditeur.

## Parcours utilisateurs

### P1 — Payer un transfert
En tant qu'agent, je veux saisir le code, vérifier l'identité du bénéficiaire, poser la question
test si elle existe, et remettre les espèces avec un reçu de paiement.

- Étant donné un transfert disponible de 100 000 XOF, quand l'agent saisit le code, alors il voit
  le nom du bénéficiaire, le montant à payer et l'existence d'une question test — jamais les
  coordonnées complètes de l'expéditeur.
- Étant donné une pièce dont le nom ne correspond pas au bénéficiaire, alors le paiement est refusé.
- Étant donné une réponse fausse à la question test trois fois de suite, alors le transfert est
  bloqué ; il ne peut être débloqué que par le responsable conformité (sur demande du support,
  après vérification de l'identité du bénéficiaire).
- Étant donné un paiement validé, alors le float de l'agence payeuse augmente de 100 000 XOF, la
  caisse de l'agent enregistre une sortie de 100 000 XOF, le statut passe à « payé » et
  l'expéditeur est notifié.
- Étant donné deux agents qui saisissent le même code au même moment, alors un seul paiement aboutit.

### P1 — Protéger contre la recherche de codes
- Étant donné un agent qui saisit de nombreux codes invalides en peu de temps, alors ses tentatives
  sont temporairement bloquées et la conformité est alertée.

### P1 — Annuler un envoi le jour même
En tant que superviseur, je veux annuler un envoi non payé de mon agence le jour même, avec un
motif, et rembourser l'expéditeur.

- Étant donné un envoi disponible du jour, quand le superviseur l'annule, alors le montant envoyé et
  les taxes sont contre-passés, les frais selon le paramètre de remboursement, la caisse enregistre
  la sortie du remboursement et l'expéditeur est notifié.
- Étant donné un envoi déjà payé, alors l'annulation est impossible.

### P2 — Expiration
En tant que plateforme, un transfert non retiré après un délai paramétrable passe « expiré » ;
l'expéditeur peut alors être remboursé en agence.

### P2 — Recevoir directement sur son wallet
En tant que bénéficiaire disposant d'un wallet, si l'expéditeur l'a choisi, je reçois le montant
directement sur mon wallet dès que le transfert est disponible.

### P2 — Payer depuis l'application mobile agence
En tant qu'agent, je veux réaliser le paiement depuis mon téléphone avec les mêmes contrôles.

## Règles métier

- Machine à états complète (référence unique ; toute autre transition est interdite) :
  - créé → disponible ; créé → en attente de conformité ;
  - en attente de conformité → disponible (libéré) ; en attente de conformité → rejeté ;
  - rejeté → remboursé (faux positif écarté mais opération refusée, ou décision de rejet simple) ;
  - rejeté → gelé (correspondance de sanctions confirmée : fonds conservés sur un compte de fonds
    gelés) ; gelé → remboursé (dégel décidé par la conformité) ; gelé → remis à l'autorité (sur
    décision d'une autorité, avec sa référence) ;
  - disponible → payé ; disponible → annulé → remboursé ; disponible → bloqué → disponible ;
    disponible → expiré → remboursé.
- Le montant payé est exactement le montant à payer enregistré à l'envoi.
- Le paiement exige une caisse ouverte et un contrôle de conformité non bloquant.
- L'agent payeur ne peut pas être l'agent émetteur du même transfert (paramétrable).
- Le remboursement des frais lors d'une annulation est paramétrable ; les taxes sont toujours
  contre-passées.
- Chaque transition est historisée (acteur, date, motif).

## Données manipulées

Transfert (statuts, agence et agent payeurs, date de paiement, pièce du bénéficiaire, tentatives
de question test, motif d'annulation), Historique des statuts, Reçu de paiement.

## Cas limites

- Agence payeuse dans une autre devise : non prévu en version 1 (paiement uniquement dans la devise
  de réception du corridor).
- Envoi annulé après 23 h 59 heure locale de l'agence émettrice : refusé (plus « le jour même »).
- Agence payeuse dans un autre pays que l'agence émettrice : le paiement se fait dans la devise du
  pays de paiement prévue par le corridor, et les taxes du pays de paiement s'appliquent.
- Transfert bloqué par la conformité pendant le paiement : paiement refusé.

## Critères de succès

- Zéro double paiement en recette, y compris sous charge.
- Paiement réalisé en moins de 90 secondes par un agent formé.
- 100 % des transitions interdites rejetées lors des tests.

## Hors périmètre

Paiement par des partenaires étrangers (version 2).

## Points à clarifier

- Délai d'expiration d'un transfert non retiré.
- Les frais sont-ils remboursés en cas d'annulation ?
- Tolérance acceptée sur le nom du bénéficiaire (fautes d'orthographe).

## Dépendances

09 ; 11 pour le parcours « recevoir directement sur son wallet ».
