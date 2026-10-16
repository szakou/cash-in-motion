# 02 — Journal d'audit et traçabilité des actions

> Entrée pour `/speckit-specify`. Référence : cahier des charges §8 (traçabilité), §9.3 ;
> constitution, principe V.

## Contexte et objectif

Régulateur, auditeurs et direction doivent pouvoir reconstituer qui a fait quoi, quand et
pourquoi, sur toute action sensible, avec la garantie que ces traces n'ont pas été modifiées.

## Acteurs

Super administrateur et Responsable conformité (consultation, y compris pour le compte des auditeurs externes), Système (enregistrement).

## Parcours utilisateurs

### P1 — Tracer automatiquement les actions sensibles
En tant que plateforme, je dois enregistrer chaque action sensible : gestion des utilisateurs,
modifications du référentiel, des tarifs, taxes, taux de change et plafonds, approbations,
décisions d'identification et de conformité, gels, annulations, clôtures de caisse avec écart,
révélations de données personnelles, renvois de code, exports.

- Étant donné qu'un administrateur modifie le taux de la TAF, alors une entrée indique l'auteur, la
  date, la valeur avant, la valeur après et le motif.

### P1 — Garantir l'inaltérabilité
- Étant donné une entrée d'audit, quand quelqu'un tente de la modifier ou de la supprimer, alors
  c'est impossible ; et toute altération éventuelle est détectée par un contrôle d'intégrité.

### P2 — Consulter et filtrer
En tant que super administrateur ou responsable conformité, je veux rechercher dans le journal par
auteur, type d'action, objet concerné et période, et exporter le résultat.

### P3 — Conservation externe
En tant que responsable conformité, je veux qu'une copie quotidienne du journal soit archivée hors
de la plateforme, non modifiable.

## Règles métier

- Chaque entrée contient : date et heure, auteur, rôle, adresse réseau, pays concerné, action,
  objet, valeurs avant et après, motif s'il est exigé, identifiant de corrélation.
- Un responsable conformité limité à certains pays ne consulte que les entrées de ces pays.
- Les entrées sont chaînées de façon à rendre toute altération détectable.
- Conservation : 10 ans minimum.
- La consultation du journal est elle-même journalisée.

## Données manipulées

Entrée d'audit, Résultat du contrôle d'intégrité, Archive quotidienne.

## Cas limites

- Action système (tâche automatique) : l'auteur est « système » avec le nom de la tâche.
- Valeurs avant/après contenant des données sensibles : enregistrées masquées.

## Critères de succès

- 100 % des actions de la liste produisent une entrée (vérifié par tests).
- Toute altération simulée est détectée par le contrôle d'intégrité.

## Hors périmètre

Journaux techniques d'exploitation.

## Points à clarifier

- Liste complémentaire d'actions à tracer souhaitée par la conformité ou l'auditeur externe.

## Dépendances

01. Les fonctionnalités 03 à 16 enregistrent leurs propres actions sensibles dans ce journal dès leur livraison.
