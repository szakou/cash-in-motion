# 07 — Clients, bénéficiaires et identification au guichet (KYC)

> Entrée pour `/speckit-specify`. Référence : cahier des charges §3.1, §3.2, §5.3 (AGW-01), §7.1, §7.2.

## Contexte et objectif

La loi impose d'identifier chaque expéditeur et chaque bénéficiaire. Au guichet, l'agent doit
retrouver rapidement un client existant ou en créer un, enregistrer sa pièce d'identité et ses
bénéficiaires, sans doublon et en protégeant ces données.

## Acteurs

Agent d'agence, Responsable conformité, Agent support (lecture masquée), Client (personne servie).

## Parcours utilisateurs

### P1 — Retrouver ou créer un expéditeur
En tant qu'agent, je veux rechercher un client par téléphone, nom ou numéro de pièce, et le créer
s'il n'existe pas.

- Étant donné un client déjà enregistré avec la pièce CNI n° X émise au Sénégal, quand je saisis ce
  numéro, alors sa fiche s'affiche au lieu d'en créer une seconde.
- Étant donné un nouveau client, quand je saisis nom et prénoms, adresse, ville, pays, téléphone,
  type et numéro de pièce, date d'expiration et photos recto/verso, alors il est créé.
- Étant donné une pièce expirée, quand je sélectionne le client, alors un avertissement bloque toute
  opération jusqu'à mise à jour de la pièce (pour un client wallet, voir fonctionnalité 11 : retour
  au niveau 1).

### P1 — Enregistrer une question test
En tant qu'expéditeur, je veux pouvoir définir une question et une réponse secrètes que le
bénéficiaire devra donner au retrait.

- Étant donné une réponse enregistrée, quand on consulte la fiche, alors la réponse n'est jamais
  affichée, à personne.

### P1 — Gérer les bénéficiaires
En tant qu'agent, je veux enregistrer les bénéficiaires d'un expéditeur (nom et prénoms, pays,
ville facultative, téléphone recommandé) et les réutiliser.

### P2 — Compléter l'identification selon le montant
En tant que plateforme, je dois demander des informations supplémentaires (date de naissance,
nationalité, profession, origine des fonds) au-delà de seuils de montant.

### P2 — Exercer les droits sur les données
En tant que responsable conformité, je veux produire l'export des données d'un client sur demande
et corriger ses données ; l'effacement n'est possible qu'après la durée légale de conservation.

## Règles métier

- Champs obligatoires d'un expéditeur : nom et prénoms, adresse, ville, pays, téléphone, type et
  numéro de pièce. Question test et réponse facultatives.
- Champs d'un bénéficiaire : nom et prénoms et pays obligatoires ; ville, téléphone et message facultatifs.
- Unicité : même type + numéro de pièce + pays d'émission = même personne.
- Les pièces acceptées, leur format de numéro et les informations complémentaires exigées
  dépendent du pays de l'agence (paramétrés dans le référentiel).
- Le téléphone est saisi au format international, avec l'indicatif du pays de l'agence proposé
  par défaut ; un expéditeur ou un bénéficiaire peut avoir un numéro d'un autre pays.
- Les règles de protection des données (consentements, durée de conservation, droits) suivent la
  loi du pays où le client est servi.
- Le numéro de pièce est protégé au repos et affiché masqué (ex. ****1234) sauf rôle et motif autorisés.
- Chaque création ou modification déclenche un contrôle sanctions/PEP (fonctionnalité 08) ; tant que 08 n'est pas livrée, le contrôle est simulé et aucune opération financière n'est encore possible.
- Les consentements (conditions, confidentialité, marketing) sont enregistrés avec leur version et leur date.
- Conservation minimale de 10 ans après la fin de la relation.

## Données manipulées

Client (type : guichet ou wallet), Pièce d'identité (type, numéro, pays, dates, images), Question
secrète, Bénéficiaire, Consentement, Niveau et statut d'identification.

## Cas limites

- Même téléphone pour deux personnes au guichet : autorisé pour les clients guichet, interdit pour les wallets.
- Nom saisi avec ou sans accents : la recherche le retrouve.
- Pièce sans date d'expiration (selon le type) : acceptée.

## Critères de succès

- Retrouver un client existant en moins de 10 secondes.
- Zéro doublon créé pour une même pièce lors de la recette.
- Aucun numéro de pièce en clair hors des écrans autorisés.

## Hors périmètre

Inscription et identification en ligne depuis l'app client (11), moteur de règles AML (08).

## Points à clarifier

- Seuils de montant déclenchant les informations complémentaires.
- Pièces acceptées pour les bénéficiaires étrangers.

## Dépendances

01, 03.
