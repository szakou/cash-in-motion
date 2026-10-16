# 05 — Grand livre comptable et intégrité des fonds

> Entrée pour `/speckit-specify`. Référence : cahier des charges §3.4, §7.3 (cantonnement), §9.1 ;
> constitution, principes I à III.

## Contexte et objectif

La plateforme détient l'argent des clients (wallets) et doit à chaque agence la valeur de son
float, dans plusieurs pays et plusieurs devises (XOF, XAF, CDF…). Chaque franc doit être
traçable, chaque solde justifiable, et l'encours des clients de chaque pays couvert par le compte
bancaire de cantonnement de ce pays. Le grand livre est la seule source de vérité des
soldes ; les autres fonctionnalités l'utilisent pour enregistrer leurs mouvements.

## Acteurs

Système (enregistrement automatique), Administrateur et Responsable conformité (consultation,
contrôles et relevés pour les besoins finance).

## Parcours utilisateurs

### P1 — Enregistrer un mouvement équilibré
En tant que plateforme, chaque opération (envoi, paiement, dépôt, retrait, P2P, approvisionnement
de float, collecte de taxes, annulation) doit produire un ensemble d'écritures équilibrées.

- Étant donné un envoi national (Sénégal → Sénégal) de 100 000 XOF payé 104 244 XOF en agence,
  quand il est validé, alors le
  float de l'agence est diminué de 104 244, le compte « transferts en transit » augmente de
  100 000, les revenus de frais de 3 200, la TTA à reverser de 500 et la TAF à reverser de 544.
- Étant donné une écriture dont les débits ne sont pas égaux aux crédits, quand elle est soumise,
  alors elle est rejetée sans aucun effet.

### P1 — Empêcher les soldes négatifs et le double débit
- Étant donné un wallet de 100 000 XOF, quand 200 retraits de 1 000 XOF arrivent en même temps,
  alors exactement 100 réussissent et le solde final est 0.
- Étant donné un float d'agence insuffisant, quand un envoi le dépasserait, alors l'envoi est refusé.

### P1 — Corriger par contre-passation
En tant que plateforme, une annulation doit produire des écritures inverses, jamais modifier ni
supprimer les écritures d'origine.

### P2 — Bloquer temporairement des fonds
En tant que plateforme, je dois pouvoir réserver une partie d'un solde (ex. retrait en attente de
confirmation) puis la libérer ou la consommer.

- Étant donné 50 000 XOF disponibles et 30 000 réservés, quand un P2P de 30 000 est demandé, alors
  il est refusé (disponible = 20 000).

### P2 — Consulter soldes et relevés
En tant qu'administrateur, je veux consulter le solde (total, réservé, disponible) et le relevé
détaillé de tout compte sur une période.

### P2 — Contrôle d'intégrité quotidien
En tant que responsable conformité, je veux un contrôle automatique quotidien qui vérifie
l'équilibre global, la cohérence de chaque solde avec ses écritures, et que la somme des wallets
clients et des floats de chaque pays et chaque devise est couverte par le compte de cantonnement
correspondant, en tenant compte de ce que le pays doit aux autres pays ou de ce qu'ils lui doivent
(transferts inter-pays pas encore compensés), avec alerte sinon.

- Étant donné un transfert Sénégal → Cameroun payé mais pas encore compensé, alors le Cameroun n'est
  pas en alerte : sa créance sur le Sénégal compte dans sa couverture.

- Étant donné des wallets sénégalais totalisant 50 M XOF et un compte de cantonnement Sénégal de
  48 M XOF, alors une alerte « couverture insuffisante — Sénégal — XOF » est émise.

### P2 — Opération entre deux pays
En tant que plateforme, un envoi d'un pays vers un autre doit être enregistré dans les comptes du
pays d'envoi, puis le paiement dans ceux du pays de paiement, avec des comptes de liaison entre
pays qui permettent de savoir à tout moment ce que chaque pays doit à l'autre.

- Étant donné un envoi Sénégal → Cameroun de 100 000 XOF payé 99 000 XAF (taux d'exemple marge
  incluse), alors le pays de paiement doit être couvert des 99 000 XAF versés, la dette du pays
  d'envoi envers le pays de paiement est visible dans un état de compensation entre pays, et la
  différence de change est constatée en écart de change lors de la compensation.

## Règles métier

- Types de comptes : wallet client, float d'agence, comptes système (transferts en transit,
  revenus de frais, taxes à reverser — un compte par taxe et par pays, ex. TTA Sénégal —, fonds gelés sur décision de
  conformité, écarts de
  change, cantonnement bancaire, liaison entre pays, commissions d'agence à payer, rabais accordés).
- Tout compte appartient à un pays et à une devise.
- Un compte a une seule devise ; un mouvement multi-devises passe par le compte d'écarts de change.
- Comptes système créés automatiquement pour chaque pays passé « en opération » et chacune de ses devises.
- Les écritures ne sont jamais modifiées ni supprimées.
- Modèle de liquidité : envoi et dépôt diminuent le float de l'agence ; paiement et retrait
  l'augmentent ; approvisionnement validé l'augmente.
- Le détail débit/crédit de chaque type d'opération est documenté et validé par la finance.

## Données manipulées

Compte, Solde (total, réservé), Écriture de journal (type, référence de l'opération, date, auteur,
contre-passation éventuelle), Ligne d'écriture (compte, sens, montant, devise).

## Cas limites

- Opération partiellement échouée : aucune écriture partielle ne subsiste.
- Contre-passation d'une écriture déjà contre-passée : refusée.
- Compte clôturé : aucun nouveau mouvement.

## Critères de succès

- Zéro écart détecté par le contrôle d'intégrité sur toute la recette.
- Aucun double débit sous charge concurrente (test de 200 opérations simultanées).
- Tout solde est justifiable par la liste de ses écritures.

## Hors périmètre

Logique métier des opérations elles-mêmes (06, 09, 10, 12, 13) ; liaison avec la comptabilité générale
externe.

## Points à clarifier

- Plan de comptes définitif et correspondance avec la comptabilité générale de l'entreprise.
- Fréquence et format du rapprochement avec le relevé bancaire de cantonnement de chaque pays.
- [À CLARIFIER] Mode de règlement des soldes entre pays (compensation périodique, partenaire de
  règlement, banque correspondante).

## Dépendances

01, 03.
