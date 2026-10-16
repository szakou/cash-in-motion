# 03 — Référentiel multi-pays : pays, zones, devises, villes, corridors, pièces d'identité

> Entrée pour `/speckit-specify`. Référence : cahier des charges §3.1, §3.2, §5.1 (ADM-03) ;
> constitution, principe IX.

## Contexte et objectif

Cash In Motion s'adresse à toute l'Afrique de l'Ouest francophone (UEMOA) et à l'Afrique centrale
(CEMAC et autres pays de la CEEAC). Chaque pays a sa devise, son régulateur, son fuseau horaire,
ses langues, son format de numéro de téléphone et ses pièces d'identité. Le référentiel est
l'endroit unique où ces différences sont décrites, afin qu'ouvrir un pays soit une affaire de
paramétrage et non de développement. Il définit aussi où l'on peut envoyer et recevoir.

## Acteurs

Administrateur (gestion), tous les utilisateurs (lecture dans les formulaires).

## Parcours utilisateurs

### P1 — Décrire un pays
En tant qu'administrateur, je veux décrire chaque pays : zone (UEMOA, CEMAC, CEEAC hors CEMAC,
autre), devise, banque centrale et autorité de supervision, cellule de renseignement financier,
indicatif téléphonique et format des numéros nationaux, fuseau(x) horaire(s), langue par défaut et
langues disponibles, pièces d'identité acceptées.

- Étant donné le Cameroun, alors sa fiche indique : CEMAC, XAF, BEAC / COBAC, ANIF, +237,
  heure du Cameroun (UTC+1), français et anglais.
- Étant donné la République démocratique du Congo, alors elle a deux fuseaux (Kinshasa et
  Lubumbashi) : chaque agence précise le sien.

### P1 — Gérer le statut d'un pays
En tant qu'administrateur, je veux indiquer pour chaque pays s'il est **inactif**, **destination
seulement** (on peut y envoyer, le paiement passe par un partenaire), ou **en opération** (agences,
wallets et opérations locales).

- Étant donné un pays que je veux passer « en opération », alors le système vérifie une liste de
  contrôle d'ouverture : régulateur et cellule de renseignement renseignés, devise active, au moins
  une règle de taxe validée (ou « aucune taxe » explicitement confirmé), plafonds configurés, compte
  de cantonnement déclaré, route SMS active et testée, pièces d'identité définies, modèles de
  messages dans la langue par défaut. Tant qu'un point manque, le passage est refusé avec la liste
  des manques.
- Étant donné un pays repassé « inactif », alors les nouvelles opérations y sont refusées et
  l'historique reste consultable.

### P1 — Gérer les corridors
En tant qu'administrateur, je veux ouvrir un corridor (pays d'envoi → pays de réception) avec ses
devises d'envoi et de réception et ses montants minimum et maximum.

- Étant donné Sénégal → Mali, alors le corridor est « intra-zone » (UEMOA → UEMOA, XOF → XOF).
- Étant donné Sénégal → Cameroun, alors il est « inter-zones » (UEMOA → CEMAC, XOF → XAF).
- Étant donné Sénégal → France, alors il est « hors zone ».
- Étant donné un corridor dont le pays d'envoi n'est pas « en opération », alors son activation est refusée.

### P2 — Gérer régions et villes
En tant qu'administrateur, je veux ajouter, renommer ou désactiver des régions et villes par pays.

### P2 — Gérer les pièces d'identité
En tant qu'administrateur, je veux définir les types de pièces (CNI CEDEAO, CNI nationale, carte
CEMAC, passeport, carte consulaire, permis, carte de réfugié…), les pays où chacun est accepté,
son format de numéro et si une date d'expiration est exigée.

### P2 — Gérer les devises
En tant qu'administrateur, je veux définir les devises (code, nom, nombre de décimales, règle
d'arrondi, symbole d'affichage : « FCFA » pour XOF et XAF, etc.).

## Règles métier

- Zones et pays initiaux :

| Zone | Pays (ISO) | Devise | Banque centrale / supervision |
| --- | --- | --- | --- |
| UEMOA | SN, CI, ML, BF, NE, TG, BJ, GW | XOF | BCEAO |
| CEMAC | CM, GA, CG, TD, CF, GQ | XAF | BEAC / COBAC |
| CEEAC hors CEMAC | CD, AO, BI, RW, ST | CDF, AOA, BIF, RWF, STN | banque centrale nationale |
| Hors zone (destinations) | FR, ES, IT, BE, US, CA, GN, GH, NG, MA | EUR, USD, CAD, GNF, GHS, NGN, MAD | — |

- Au lancement, seul le Sénégal est « en opération » ; les autres pays de la région sont créés
  « inactifs » ou « destination seulement » selon les corridors ouverts.
- En version 1, un corridor n'est activable que si **les deux pays sont « en opération »** (paiement
  en agence Cash In Motion). Le statut « destination seulement » (paiement par un partenaire) ne
  sera utilisable qu'en version 2, avec les partenaires payeurs.
- Les exemples Sénégal → Mali et Sénégal → Cameroun des spécifications supposent ces pays ouverts ;
  les jeux de données de recette les passent « en opération ».
- Corridor intra-zone : origine et destination dans la même zone monétaire ; inter-zones : deux
  zones de la région (ex. UEMOA → CEMAC) ; hors zone : au moins un pays hors UEMOA/CEMAC/CEEAC.
  Ce classement sert aux taxes (fonctionnalité 04) et à la conformité (08).
- XOF et XAF ont la même parité fixe avec l'euro mais sont des devises distinctes : un corridor
  XOF → XAF exige un taux de change explicite.
- Un numéro de téléphone est toujours saisi et stocké au format international ; les formulaires
  proposent l'indicatif du pays de l'agence ou du client.
- Les dates et heures s'affichent dans le fuseau de l'agence ou du client.
- Suppression interdite pour toute donnée déjà utilisée : seulement désactivation.
- Toute modification est journalisée.

## Données manipulées

Zone, Devise, Pays (zone, devise, régulateur, cellule de renseignement, indicatif, format de
numéro, fuseaux, langues, statut, liste de contrôle d'ouverture), Région, Ville, Type de pièce
d'identité, Corridor (classement intra-zone / inter-zones / hors zone).

## Cas limites

- Pays à plusieurs fuseaux : le fuseau est porté par l'agence ; le fuseau par défaut du pays sert
  pour les clients wallet.
- Pays changeant de zone ou de devise (rare) : n'affecte jamais les opérations passées.
- Ville homonyme dans deux régions : distinguée par sa région.

## Critères de succès

- Décrire et ouvrir (hors démarches réglementaires) un nouveau pays en moins d'une journée, sans
  intervention technique.
- Un administrateur ouvre un nouveau corridor en moins de 3 minutes.

## Hors périmètre

Frais, taxes et taux de change (04), routes SMS (14), démarches d'agrément auprès des régulateurs.

## Points à clarifier

- [À CLARIFIER] Pays ouverts au pilote : Sénégal seul (transferts nationaux uniquement) ou
  Sénégal + un ou plusieurs pays voisins (ex. Mali) pour des transferts internationaux dès la V1 ?
- [À CLARIFIER] Ordre d'ouverture des pays suivants.
- [À CLARIFIER] Opère-t-on dans chaque pays via une entité locale agréée ou un partenaire agréé ?
  (impacte la liste de contrôle d'ouverture et les rapports réglementaires)
- Corridors ouverts au lancement.

## Dépendances

01.
