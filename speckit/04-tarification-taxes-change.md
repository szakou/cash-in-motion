# 04 — Tarification, taxes, taux de change et devis

> Entrée pour `/speckit-specify`. Référence : cahier des charges §4 (règles et exemple), §5.1
> (ADM-04 à ADM-06, ADM-11).

## Contexte et objectif

Le client doit connaître, avant de payer, le coût exact de son opération : frais, chaque taxe,
taux de change et montant reçu par le bénéficiaire. L'administration doit pouvoir faire évoluer
les tarifs et les taxes **pays par pays** (chaque pays de l'UEMOA, de la CEMAC et de la CEEAC a sa
propre fiscalité) sans jamais modifier les opérations déjà réalisées.

## Acteurs

Administrateur (paramétrage et approbation), Agent d'agence et Client (devis).

## Parcours utilisateurs

### P1 — Obtenir un devis
En tant qu'agent, je veux saisir un corridor et un montant envoyé (ou un montant à recevoir) et
obtenir un devis détaillé valable un temps limité.

- Étant donné le corridor Sénégal → Mali, un montant de 100 000 XOF, des frais de transfert de
  3 000 XOF et un message (200 XOF), quand je demande un devis, alors j'obtiens : TTA 500, TAF 544,
  THZUEMOA 0, total taxes 1 044, total payé 104 244 XOF, montant à payer 100 000 XOF.
- Étant donné un envoi de 1 000 000 XOF, quand je demande un devis, alors la TTA est plafonnée à
  2 000 XOF.
- Étant donné le corridor Sénégal → France et 655 957 XOF, quand je demande un devis, alors la
  THZUEMOA s'applique et le montant à payer est d'environ 1 000 EUR diminué de la marge de change
  (calcul seulement : ce corridor ne sera payable qu'avec un partenaire, en version 2).
- Étant donné le corridor Sénégal → Cameroun (XOF → XAF) et 100 000 XOF, quand je demande un
  devis, alors les taxes sénégalaises de l'envoi s'appliquent (dont la THZUEMOA, le Cameroun étant
  hors UEMOA), le taux XOF → XAF configuré est appliqué et le montant à payer est exprimé en XAF.
- Étant donné un pays d'envoi sans aucune règle de taxe validée ni confirmation « aucune taxe »,
  alors aucun devis n'est possible depuis ce pays.
- Étant donné un montant à recevoir de 100 000 XOF, quand je demande un devis inversé, alors le
  système calcule le montant à envoyer et le total à payer, cohérents au franc près.
- Étant donné un devis expiré, quand je l'utilise pour un envoi, alors c'est refusé et un nouveau
  devis est proposé.

### P1 — Paramétrer les grilles de frais
En tant qu'administrateur, je veux définir par corridor et type d'opération des paliers de montant
avec frais fixes et/ou pourcentage, minimum et maximum, et une date d'effet.

### P1 — Paramétrer les taxes de chaque pays
En tant qu'administrateur, je veux définir pour chaque pays d'opération ses taxes : code et
libellé, base (montant envoyé, frais), taux, plafond et sa devise, moment (envoi, paiement,
opérations wallet), condition selon le corridor (toujours, intra-zone, inter-zones, hors zone, ou
« hors zone monétaire du pays »), et qui la supporte (expéditeur ou bénéficiaire).

- Étant donné les taxes du Sénégal, alors elles ne s'appliquent qu'aux opérations réalisées au
  Sénégal (envoi depuis une agence sénégalaise, paiement dans une agence sénégalaise, wallet
  sénégalais).
- Étant donné un envoi Sénégal → Mali payé au Mali, alors l'envoi supporte les taxes du Sénégal et
  le paiement celles du Mali, chacune collectée pour son pays.

### P1 — Approuver à quatre yeux
En tant qu'administrateur, je veux qu'une modification de frais, de taxe ou de taux de change ne
s'applique qu'après approbation par un autre administrateur.

- Étant donné une grille créée par A, quand A tente de l'approuver, alors c'est refusé ; quand B
  l'approuve, alors elle s'applique à partir de sa date d'effet.

### P2 — Gérer les taux de change
En tant qu'administrateur, je veux saisir un taux de référence par paire de devises et une marge
par corridor ; les
parités XOF/EUR et XAF/EUR sont fixes (655,957), mais XOF et XAF restent deux devises distinctes
et le passage de l'une à l'autre utilise un taux configuré.

### P2 — Simuler l'impact
En tant qu'administrateur, je veux un simulateur qui calcule un devis avec des paramètres en
attente d'approbation, avant de les valider.

### P3 — Frais annexes
En tant qu'administrateur, je veux fixer les frais de message et de livraison à domicile.

## Règles métier

- Ordre de calcul : frais de transfert (palier) → frais annexes → rabais (jamais supérieur au total
  des frais) → taxes → arrondi ligne par ligne → total → montant à payer.
- Les taxes appartiennent à un pays : à l'envoi, celles du pays de l'agence (ou du wallet)
  émettrice ; au paiement, celles du pays de l'agence payeuse.
- Taxes initiales du **Sénégal** (exemple de référence) :
  - TTA : 0,5 % du montant envoyé, plafond 2 000 XOF par opération, à l'envoi (application au
    paiement du transfert : à clarifier) ;
  - TAF : 17 % des frais nets de rabais, à l'envoi (application au paiement : à clarifier) ;
  - THZUEMOA : taux et plafond paramétrables ; seulement si l'autre pays du corridor est hors UEMOA.
- Les taxes des autres pays sont saisies lors de leur ouverture (voir liste de contrôle d'ouverture,
  fonctionnalité 03).
- Les taxes sont à la charge de l'expéditeur par défaut ; la règle est paramétrable par taxe.
- Montant à payer = montant envoyé × taux de change (taux de référence de la paire de devises
  diminué de la marge du corridor), arrondi selon la devise de réception, moins les taxes
  supportées par le bénéficiaire le cas échéant.
- Deux règles actives ne peuvent pas se chevaucher dans le temps pour la même clé.
- Un devis fige toutes les valeurs ; une opération créée depuis un devis utilise exactement ces
  valeurs, même si un tarif change ensuite.
- Durée de validité d'un devis paramétrable (15 minutes par défaut).

## Données manipulées

Grille de frais et ses paliers, Frais annexe, Règle de taxe, Taux de change, Devis (avec le détail
complet et les règles appliquées), Code promotionnel (préparé pour une version ultérieure).

## Cas limites

- Montant hors de tous les paliers : devis refusé avec message explicite.
- Aucune règle de taxe active pour le pays et « aucune taxe » non confirmé : devis refusé,
  alerte administrateur (voir parcours « Obtenir un devis »).
- Montant inférieur au minimum ou supérieur au maximum du corridor : refus.

## Critères de succès

- 100 % des exemples chiffrés du cahier des charges reproduits à l'identique.
- Un devis est obtenu en moins de 300 ms dans 95 % des cas.
- Aucune opération existante n'est modifiée par un changement de tarif (vérifié par test).

## Hors périmètre

Codes promotionnels et campagnes de rabais (version 2).

## Points à clarifier

- [À CLARIFIER] Liste des taxes sur les transferts et la monnaie électronique de chaque pays visé
  (UEMOA, CEMAC, CEEAC), avec base, taux et plafond.
- [À CLARIFIER] Taxes du pays de paiement : supportées par le bénéficiaire (déduites du montant
  payé) ou prépayées par l'expéditeur dans le total ?
- [À CLARIFIER] La TAF s'applique-t-elle aux frais seulement (hypothèse retenue) ou au montant envoyé ?
- [À CLARIFIER] Taux, base, plafond et corridors concernés par la THZUEMOA.
- [À CLARIFIER] TTA et TAF s'appliquent-elles aussi au paiement du transfert (« envoi/retrait »
  dans le cahier initial) ? Un transfert Sénégal → Sénégal les supporterait-il deux fois ?
- [À CLARIFIER] TTA et TAF s'appliquent-elles aux opérations wallet (P2P, dépôt, retrait) ?
- Qui peut accorder un rabais au guichet, et avec quel plafond ?

## Dépendances

03.
