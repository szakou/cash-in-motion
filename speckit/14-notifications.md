# 14 — Notifications et fournisseurs SMS configurables par pays

> Entrée pour `/speckit-specify`. Décrit le QUOI et le POURQUOI, sans choix technique.
> Référence : cahier des charges §5.2 (SUP-04), §6.2 (MCL-07) ; constitution, principe IX.

## Contexte et objectif

Dans les pays de l'UEMOA, de la CEMAC et de la CEEAC, le SMS est le canal qui fait foi : c'est par
SMS que le bénéficiaire reçoit son code de retrait et que le client reçoit ses codes de
vérification. Aucun fournisseur ne couvre bien tous les pays : la qualité de remise, le coût et
les règles d'identifiant d'expéditeur varient d'un pays et d'un opérateur à l'autre. L'administration
doit donc pouvoir **configurer, pour chaque pays, le fournisseur SMS à utiliser**, avec un
fournisseur de secours, sans intervention technique. Les notifications sur téléphone et les e-mails
complètent le SMS. Un message qui échoue ne doit jamais bloquer ni annuler une opération.

## Acteurs

Expéditeur, Bénéficiaire, Client wallet, Agent support, Superviseur, Responsable conformité,
Administrateur (fournisseurs, routage, modèles), Système.

## Parcours utilisateurs

### P1 — Configurer les fournisseurs SMS
En tant qu'administrateur, je veux enregistrer les comptes de fournisseurs SMS (nom, type de
connecteur parmi ceux pris en charge, identifiants d'accès, pays couverts, coût unitaire indicatif
par pays) afin de pouvoir les affecter aux pays.

- Étant donné un fournisseur enregistré, quand je consulte sa fiche, alors ses identifiants
  d'accès ne sont jamais réaffichés (seulement « configuré le … par … ») ; je peux les remplacer.
- Étant donné un fournisseur affecté à au moins un pays actif, quand je tente de le supprimer,
  alors c'est refusé ; je peux seulement le désactiver après l'avoir remplacé partout.

### P1 — Affecter un fournisseur à chaque pays
En tant qu'administrateur, je veux définir pour chaque pays : le fournisseur principal, un
fournisseur de secours, l'identifiant d'expéditeur à utiliser dans ce pays (nom alphanumérique
ou numéro court), et l'état de la route (en test, active, suspendue).

- Étant donné le Sénégal avec le fournisseur A en principal et B en secours, quand un SMS part vers
  un numéro +221, alors il est envoyé par A avec l'identifiant d'expéditeur du Sénégal.
- Étant donné un bénéficiaire au Cameroun (+237) d'un envoi fait au Sénégal, alors son SMS part par
  le fournisseur configuré pour le **Cameroun** (pays du numéro du destinataire), pas celui du Sénégal.
- Étant donné un numéro d'un pays sans route configurée, alors le message part par la route
  internationale par défaut ; s'il n'y en a pas, il est marqué « non routable » et le support et
  l'administration sont alertés.
- Étant donné un pays où l'identifiant alphanumérique n'est pas autorisé ou pas encore enregistré,
  alors l'identifiant de remplacement configuré pour ce pays est utilisé.

### P1 — Tester avant d'activer
En tant qu'administrateur, je veux envoyer un SMS de test vers un numéro de mon choix par une route
donnée et voir le résultat (accepté, remis, échec et motif) avant d'activer la route.

- Étant donné une route jamais testée avec succès, quand je tente de l'activer, alors c'est refusé.
- Étant donné un pays d'opération sans route active, alors l'ouverture de ce pays aux opérations
  est bloquée (constitution, principe IX).

### P1 — Basculer automatiquement sur le secours
En tant que plateforme, si le fournisseur principal d'un pays échoue, je dois réessayer puis
basculer sur le fournisseur de secours, pour que les messages critiques arrivent.

- Étant donné un refus ou une absence de réponse du fournisseur principal, alors le message est
  renvoyé par le fournisseur de secours après le nombre d'essais configuré.
- Étant donné un code de retrait accepté par le principal mais non remis après le délai configuré
  (par défaut 2 minutes), alors il est renvoyé par le secours (un éventuel doublon est préféré à
  une non-réception pour les messages critiques).
- Étant donné un taux d'échec du fournisseur principal d'un pays dépassant le seuil configuré sur
  une période, alors la route bascule entièrement sur le secours, l'administration est alertée,
  et le retour au principal se fait automatiquement après rétablissement ou manuellement.

### P1 — Informer lors d'un transfert
- Étant donné un envoi disponible, alors l'expéditeur reçoit un SMS de confirmation (montant, code,
  pays de destination) et le bénéficiaire un SMS avec l'expéditeur, le montant, la devise et le code.
- Étant donné un transfert payé, annulé ou remboursé, alors l'expéditeur en est informé.

### P1 — Informer lors d'une opération wallet et envoyer les codes de vérification
- Étant donné un P2P, un dépôt ou un retrait terminé, alors chaque client concerné reçoit une
  notification sur son téléphone (ou un SMS s'il n'a pas activé les notifications) avec le montant
  et le nouveau solde.
- Étant donné une demande de code de vérification, alors le client reçoit un SMS avec le code, sa
  durée de validité et l'avertissement de ne jamais le partager.

### P2 — Superviser la qualité par pays et par fournisseur
En tant qu'administrateur, je veux un tableau de bord par pays et par fournisseur : volumes,
taux d'acceptation, taux de remise, délai moyen de remise, échecs par motif, bascules, coût estimé
sur la période ; et une alerte si le crédit SMS d'un compte fournisseur passe sous un seuil
(lorsque le fournisseur l'expose).

### P2 — Messages dans la langue du destinataire
En tant qu'administrateur, je veux gérer les modèles de messages par langue (français par défaut ;
anglais, portugais, espagnol au besoin) ; la langue utilisée est celle du client si elle est
connue, sinon la langue par défaut du pays du destinataire.

### P2 — Renvoyer le code de retrait
En tant qu'agent support, je veux renvoyer le SMS du code au bénéficiaire ou à l'expéditeur, au
plus 3 fois par 24 h, avec un motif, et pouvoir forcer l'envoi par le fournisseur de secours.

### P2 — Alerter les équipes
- Float sous le seuil → superviseur et siège ; alerte de conformité de gravité élevée → e-mail au
  responsable conformité du pays ; décision d'identification → client.

### P3 — Centre de notifications
En tant que client, je veux retrouver mes notifications passées dans l'application.

## Règles métier

- **Routage** : la route est choisie d'après le pays de l'indicatif du numéro du destinataire.
- **Configuration par pays** : fournisseur principal, fournisseur de secours (facultatif mais
  recommandé), identifiant d'expéditeur et identifiant de remplacement, état de la route, nombre
  d'essais avant bascule, délai de remise attendu pour les messages critiques, seuil de bascule
  automatique, langue par défaut.
- **Route internationale par défaut** : utilisée pour les pays sans route dédiée.
- **Classes de messages** : critique (codes de vérification, code de retrait), transactionnel
  (confirmations), information (alertes, rappels). Seuls les messages critiques déclenchent le
  renvoi par le secours en cas de non-remise.
- Un SMS fait au plus 160 caractères en alphabet standard (70 si caractères spéciaux) ; les
  modèles indiquent leur longueur et le nombre de segments ; il ne contient jamais de numéro de
  pièce, de PIN ni de réponse secrète.
- Exemple : « Cash In Motion : Moussa Ndiaye vous a envoyé 100 000 FCFA. Code : 123 456 789 0.
  Retrait en agence avec pièce d'identité. »
- Les identifiants d'accès aux fournisseurs sont conservés de façon sécurisée et ne sont jamais
  affichés ni journalisés.
- Toute modification de fournisseur, de route ou de modèle est tracée dans le journal d'audit.
- Un même événement ne produit qu'un message par destinataire et par canal, sauf renvoi par le
  secours d'un message critique non remis.
- Chaque message conserve : route et fournisseur utilisés, tentatives, bascules, statuts
  (en file, envoyé, accepté, remis, échec), motif d'échec, coût estimé.
- Les messages transactionnels sont toujours envoyés ; les consentements marketing s'appliquent
  aux seuls messages promotionnels.

## Données manipulées

Fournisseur SMS (compte, type de connecteur, identifiants protégés, pays couverts, coûts
indicatifs, statut), Route SMS par pays (principal, secours, identifiants d'expéditeur, paramètres
de bascule, état, dernier test), Route internationale par défaut, Modèle de message (classe,
canal, langue, texte), Notification (destinataire, pays, route, fournisseur, tentatives, statuts,
coût), Test d'envoi, Préférences de notification du client.

## Cas limites

- Principal et secours indisponibles : le message reste en file, de nouvelles tentatives sont
  faites, le support voit l'échec et peut renvoyer ; l'opération n'est jamais bloquée.
- Numéro mal formé ou indicatif inconnu : message « non routable », alerte au support.
- Changement de fournisseur d'un pays pendant que des messages sont en file : les nouveaux envois
  utilisent la nouvelle route, ceux en cours se terminent sur l'ancienne.
- Accusé de remise jamais reçu (fournisseur sans accusé) : statut « accepté » considéré final
  pour les messages non critiques.

## Critères de succès

- 95 % des SMS critiques remis en moins de 60 secondes dans chaque pays d'opération.
- Ouvrir la route SMS d'un nouveau pays en moins de 30 minutes, sans intervention technique.
- En cas de panne du fournisseur principal d'un pays, 95 % des messages critiques remis par le
  secours en moins de 5 minutes.
- Aucune opération bloquée par un échec d'envoi ; zéro doublon pour les messages non critiques.

## Hors périmètre

Campagnes marketing, notifications aux partenaires (version 2), messages vocaux, messageries
instantanées (prévoir la possibilité d'ajouter d'autres canaux plus tard).

## Points à clarifier

- [À CLARIFIER] Fournisseurs SMS pressentis pour les premiers pays (Sénégal puis suivants), et un
  agrégateur régional comme route internationale par défaut ?
- [À CLARIFIER] Identifiant d'expéditeur « CASHINMOTION » : à enregistrer dans quels pays, auprès de qui ?
- Délai de remise attendu avant renvoi par le secours, par classe de message.
- Les SMS au bénéficiaire sont-ils facturés via les frais de message ?
- Qui peut modifier les routes : administrateur global seulement, ou aussi l'administrateur du pays ?

## Dépendances

01, 02, 03 (pays, indicatifs, langues) ; utilisée par 09, 10, 11, 12, 13.
