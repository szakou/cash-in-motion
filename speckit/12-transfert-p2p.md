# 12 — Transfert de wallet à wallet (P2P)

> Entrée pour `/speckit-specify`. Référence : cahier des charges §6.2 (MCL-04).

## Contexte et objectif

Le client doit pouvoir envoyer instantanément de l'argent à un autre client Cash In Motion à partir
de son numéro de téléphone, en connaissant le coût exact avant de confirmer.

## Acteurs

Client émetteur, Client destinataire.

## Parcours utilisateurs

### P1 — Envoyer à un autre client
En tant que client, je veux choisir un contact ou saisir un numéro, vérifier le nom du
destinataire, saisir le montant, voir les frais et taxes, puis confirmer avec mon PIN.

- Étant donné un destinataire inscrit, quand je saisis son numéro, alors je vois son nom
  partiellement masqué (« Awa D*** ») pour confirmer que c'est la bonne personne.
- Étant donné un solde suffisant, quand je confirme 10 000 XOF, alors mon solde diminue du montant
  plus les frais et taxes, le solde du destinataire augmente de 10 000 XOF instantanément, et nous
  recevons chacun une notification et un reçu.
- Étant donné un solde insuffisant, alors l'opération est refusée avant la saisie du PIN.
- Étant donné un plafond dépassé, alors l'opération est refusée et on me propose d'élever mon
  niveau d'identification.
- Étant donné un numéro non inscrit, alors l'opération est refusée avec un message clair.
- Étant donné que je tape deux fois sur « Confirmer », alors un seul transfert est effectué.

### P2 — Retrouver mes destinataires récents
En tant que client, je veux voir mes derniers destinataires pour renvoyer rapidement.

### P3 — Ajouter un motif
En tant que client, je veux ajouter un court motif visible par le destinataire.

## Règles métier

- Frais et taxes selon la tarification du type « P2P » (fonctionnalité 04).
- Contrôle de conformité préalable sur les deux clients.
- L'émetteur et le destinataire ne peuvent pas être le même wallet.
- En version 1, le P2P est limité aux wallets d'un même pays ; un P2P vers un autre pays est
  refusé avec un message clair (voir points à clarifier).
- Wallet gelé (émetteur ou destinataire) : refus.
- Confirmation par PIN, ou biométrie si activée et montant inférieur à un seuil paramétrable.
- Une opération P2P ne peut pas être annulée par le client ; une erreur se règle via le support.

## Données manipulées

Opération wallet (type P2P, montant, frais, taxes, émetteur, destinataire, statut, motif), Reçu.

## Cas limites

- Connexion perdue après confirmation : l'application vérifie le statut au retour, sans doublon.
- Opération mise en attente par la conformité : fonds réservés, destinataire non crédité tant que
  non libérée.

## Critères de succès

- Destinataire crédité en moins de 5 secondes dans 95 % des cas.
- Zéro doublon en recette.

## Hors périmètre

Envoi vers un numéro non inscrit avec invitation (version 2), demande de paiement.

## Points à clarifier

- Frais du P2P (gratuit, fixe, pourcentage) et taxes applicables dans chaque pays.
- [À CLARIFIER] P2P entre pays d'une même zone (ex. Sénégal → Côte d'Ivoire, XOF → XOF) ou entre
  zones (XOF → XAF) : à ouvrir dès la version 1 ?
- Seuil de montant pour lequel la biométrie remplace le PIN.

## Dépendances

11.
