# 08 — Conformité : sanctions, plafonds et surveillance anti-blanchiment

> Entrée pour `/speckit-specify`. Référence : cahier des charges §7.2 ; constitution, principe IV.

## Contexte et objectif

La plateforme doit empêcher les opérations avec des personnes sanctionnées, faire respecter les
plafonds réglementaires et détecter les comportements suspects, conformément à la réglementation
LBC/FT de chaque zone (UEMOA, CEMAC) et de chaque pays. Le responsable conformité doit disposer d'outils de revue et de déclaration.

## Acteurs

Responsable conformité, Administrateur (paramètres de plafonds), Système (contrôles automatiques).

## Parcours utilisateurs

### P1 — Contrôler chaque opération avant exécution
En tant que plateforme, je dois vérifier avant tout mouvement de fonds : expéditeur et bénéficiaire
contre les listes de sanctions et PEP, statut du client (gelé, pièce expirée), plafonds.
Résultat : autoriser, mettre en attente de revue, ou refuser.

- Étant donné un bénéficiaire « Mamadou Diallo » et une entrée de liste « DIALLO Mamadou », quand un
  envoi est demandé, alors l'opération est mise en attente et une correspondance potentielle est
  créée pour revue.
- Étant donné un client gelé, quand il tente une opération, alors elle est refusée.
- Étant donné un plafond mensuel atteint, quand une nouvelle opération est demandée, alors elle est
  refusée avec le motif « plafond atteint ».
- Étant donné une opération refusée, alors aucun mouvement de fonds n'a lieu.

### P1 — Revoir les correspondances et décider
En tant que responsable conformité, je veux une file des correspondances potentielles, comparer les
informations, et décider « faux positif » (l'opération continue) ou « correspondance confirmée »
(l'opération est refusée et le client gelé).

### P1 — Paramétrer les plafonds
En tant qu'administrateur ou responsable conformité, je veux définir des plafonds (montant et
nombre) par opération, par jour et par mois, par **pays**, niveau d'identification, agence ou
corridor, dans la devise du pays.

- Étant donné des plafonds différents au Sénégal et au Gabon pour le niveau 1, alors chaque client
  est soumis aux plafonds de son pays.

### P2 — Surveiller les flux après coup
En tant que responsable conformité, je veux des alertes automatiques sur des schémas suspects :
fractionnement sous un seuil, fréquence anormale, montants ronds répétés, un expéditeur vers de
nombreux bénéficiaires, un bénéficiaire recevant de nombreux expéditeurs, corridor à risque, dépôt
suivi d'un retrait rapide.

- Étant donné un seuil de 500 000 XOF, quand un client fait 5 envois de 490 000 XOF en 24 h, alors
  une alerte « fractionnement » est créée.

### P2 — Traiter les alertes et déclarer
En tant que responsable conformité, je veux assigner, annoter, clore ou escalader une alerte, et
générer le dossier de déclaration de soupçon pour la cellule de renseignement financier du pays
concerné (CENTIF dans l'UEMOA, ANIF dans la CEMAC, cellule nationale ailleurs), au format de ce pays.
Un responsable conformité limité à un pays ne traite que les alertes de ce pays.

### P2 — Gérer les listes
En tant que responsable conformité, je veux importer les listes de sanctions (ONU, UE, OFAC,
nationale) et tenir une liste interne.

### P3 — Geler et dégeler
En tant que responsable conformité, je veux geler ou dégeler un client ou un wallet avec un motif.

## Règles métier

- Le contrôle préalable s'applique à toutes les opérations : envoi, paiement, P2P, dépôt, retrait.
- La comparaison des noms tolère l'ordre des mots, les accents et les petites différences
  d'orthographe ; le seuil de similarité est paramétrable.
- Une opération en attente de conformité ne peut pas être payée ni retirée tant qu'elle n'est pas libérée.
- Les règles de surveillance et leurs seuils sont paramétrables et activables individuellement.
- Toute décision de conformité est motivée et journalisée.
- Conservation des dossiers : 10 ans minimum, ou plus si la loi du pays l'exige.
- Une alerte est rattachée au pays de l'opération ; une alerte impliquant deux pays (envoi
  inter-pays) est visible des responsables conformité des deux pays.
- Les listes de sanctions nationales sont rattachées à leur pays.

## Données manipulées

Entrée de liste de surveillance, Résultat de contrôle, Plafond, Règle de surveillance, Alerte,
Déclaration de soupçon, Gel.

## Cas limites

- Liste de sanctions indisponible ou non mise à jour depuis plus de 24 h : les opérations sont mises
  en attente plutôt qu'autorisées (ou selon décision à clarifier).
- Homonyme récurrent déjà classé faux positif : ne doit pas bloquer à nouveau la même personne
  identifiée par sa pièce.

## Critères de succès

- 100 % des opérations passent par le contrôle préalable.
- Taux de faux positifs mesuré et ajustable par le seuil.
- Délai moyen de traitement d'une correspondance suivi dans le tableau de bord conformité.

## Hors périmètre

Connexion à un fournisseur commercial de listes (choix ultérieur).

## Points à clarifier

- Valeurs initiales des plafonds par niveau d'identification.
- Comportement si les listes ne sont pas à jour : bloquer ou autoriser avec alerte ?
- Corridors classés à risque.

## Dépendances

01, 07.
