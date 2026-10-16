# 11 — Wallet client : inscription, sécurité, identification en ligne et consultation

> Entrée pour `/speckit-specify`. Référence : cahier des charges §6.2 (MCL-01 à MCL-03, MCL-09), §7.1.

## Contexte et objectif

Le client doit pouvoir ouvrir un portefeuille électronique depuis son téléphone, le sécuriser,
élever son niveau d'identification pour augmenter ses plafonds, et consulter son solde et son
historique. C'est la porte d'entrée de toutes les fonctions wallet (P2P, dépôt, retrait).

## Acteurs

Client, Responsable conformité (validation des identifications).

## Parcours utilisateurs

### P1 — S'inscrire
En tant que client, je veux m'inscrire avec mon numéro de téléphone, le valider par un code reçu
par SMS, renseigner mon identité minimale et créer un code PIN.

- Étant donné un numéro d'un pays « en opération » (ex. +221 Sénégal), quand je saisis le code reçu dans les 5 minutes, alors mon
  numéro est validé ; après 3 codes faux, je dois en redemander un.
- Étant donné un numéro déjà inscrit, quand je tente de m'inscrire, alors on me propose de me connecter.
- Étant donné un PIN trivial (0000, 1234…), alors il est refusé.
- Étant donné une inscription terminée au Sénégal, alors mon wallet en XOF est ouvert au niveau 1 ;
  au Cameroun, il serait ouvert en XAF.
- Étant donné un numéro d'un pays qui n'est pas « en opération », alors l'inscription est refusée
  avec la liste des pays disponibles.
- Étant donné l'inscription, alors j'accepte les conditions et la politique de confidentialité ;
  le consentement marketing est séparé et facultatif.

### P1 — Se connecter et protéger l'accès
En tant que client, je veux ouvrir l'application avec mon PIN ou ma biométrie, sur un appareil
reconnu.

- Étant donné 3 PIN erronés, alors l'accès est bloqué temporairement avec indication du délai.
- Étant donné un nouvel appareil, alors une validation par code SMS est exigée avant toute opération.
- Étant donné 2 minutes d'inactivité, alors l'application se verrouille.

### P1 — Consulter son solde et son historique
En tant que client, je veux voir mon solde disponible (masquable), mes plafonds restants et
l'historique détaillé de mes opérations, avec un reçu pour chacune.

### P2 — Élever son niveau d'identification
En tant que client, je veux photographier ma pièce (recto/verso) et prendre un selfie pour passer
au niveau 2, puis fournir un justificatif d'adresse pour le niveau 3, et suivre l'état de ma demande.

- Étant donné une demande de niveau 2 soumise, quand le responsable conformité l'approuve, alors
  mon niveau et mes plafonds augmentent et je suis notifié.
- Étant donné une demande rejetée, alors je reçois le motif et peux soumettre à nouveau.

### P2 — Gérer sa sécurité et ses données
En tant que client, je veux changer mon PIN, activer ou désactiver la biométrie, voir mes appareils,
et demander l'export de mes données.

## Règles métier

- Niveaux : 1 (téléphone vérifié), 2 (pièce + selfie validés), 3 (justificatif d'adresse validé) ;
  chaque niveau a ses plafonds (fonctionnalité 08).
- Un numéro de téléphone = un seul wallet.
- Un wallet gelé reste consultable mais aucune opération n'est possible.
- Les écrans sensibles ne peuvent pas être capturés.
- Pièce expirée : le niveau redescend à 1 jusqu'à mise à jour.
- Le wallet est rattaché au pays de résidence du client (celui de son numéro à l'inscription) :
  devise, plafonds, taxes, régulateur, fuseau et langue par défaut de ce pays.
- Langue : français par défaut ; le client peut choisir une autre langue disponible dans son pays
  (anglais, portugais, espagnol…).

## Données manipulées

Client wallet, Wallet (solde, réservé, statut), Appareil enrôlé, Demande d'identification
(documents, statut, motif), Consentement.

## Cas limites

- SMS non reçu : renvoi possible après 60 secondes, limité par heure.
- Téléphone rooté ou modifié : opérations financières interdites, consultation autorisée.
- Changement de numéro de téléphone : procédure assistée par le support (hors application).

## Critères de succès

- Inscription complète en moins de 3 minutes.
- Application utilisable en 3G sur un téléphone d'entrée de gamme.
- Décision sur une demande de niveau 2 en moins de 24 h ouvrées.

## Hors périmètre

P2P (12), dépôt et retrait (13), notifications (14).

## Points à clarifier

- Plafonds de chaque niveau.
- Vérification d'identité automatique par un prestataire dès le lancement, ou revue manuelle ?
- Âge minimum pour ouvrir un wallet.

## Dépendances

01, 05, 07, 08, 14 (codes de vérification par SMS).
