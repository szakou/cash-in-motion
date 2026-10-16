# 01 — Accès sécurisé et gestion des utilisateurs internes

> Entrée pour `/speckit-specify`. Décrit le QUOI et le POURQUOI, sans choix technique.
> Référence : cahier des charges §2, §5.1 (ADM-02), §9.3.

## Contexte et objectif

Cash In Motion est utilisé par le siège (administration, conformité, support) et par un réseau
d'agences. Chaque personne ne doit voir et faire que ce que son rôle et son agence autorisent.
Cette fonctionnalité pose la base de confiance de toute la plateforme : connexion forte, rôles,
périmètre d'agence et gestion du cycle de vie des comptes internes.

## Acteurs

Super administrateur, Administrateur, Responsable conformité, Agent support, Superviseur
d'agence, Agent d'agence.

## Parcours utilisateurs

### P1 — Se connecter de façon sécurisée
En tant qu'utilisateur interne, je veux me connecter avec mon identifiant, mon mot de passe et un
second facteur, afin que personne ne puisse agir en mon nom.

- Étant donné un utilisateur actif avec second facteur configuré, quand il saisit des identifiants
  corrects et un code valide, alors il accède à l'espace correspondant à son rôle.
- Étant donné un utilisateur sans second facteur configuré, quand il se connecte pour la première
  fois, alors il est obligé de le configurer avant d'accéder à quoi que ce soit.
- Étant donné un utilisateur suspendu, quand il tente de se connecter, alors l'accès est refusé
  avec un message neutre.
- Étant donné une session inactive depuis la durée configurée, quand l'utilisateur revient, alors
  il doit se reconnecter.

### P1 — Ne voir que son périmètre
En tant que superviseur ou agent, je ne dois voir que les données de mon agence, afin de protéger
les clients et les autres agences.

- Étant donné un agent de l'agence A, quand il tente d'ouvrir une opération de l'agence B, alors
  l'accès est refusé.
- Étant donné un superviseur, quand il consulte la liste des agents, alors seuls ceux de son agence
  apparaissent.
- Étant donné un responsable conformité limité au Cameroun, quand il ouvre la file d'alertes,
  alors il ne voit que les alertes du Cameroun.
- Étant donné un administrateur global, alors il voit tous les pays et peut filtrer par pays.

### P1 — Gérer les comptes internes
En tant qu'administrateur, je veux créer, modifier, suspendre et réactiver des utilisateurs
internes et leur attribuer un rôle et, le cas échéant, une agence.

- Étant donné un administrateur, quand il crée un agent sans agence, alors la création est refusée.
- Étant donné un administrateur, quand il tente de créer un super administrateur, alors c'est refusé
  (réservé au super administrateur).
- Étant donné un utilisateur suspendu, quand on consulte son historique, alors ses actions passées
  restent visibles et attribuées.

### P2 — Gérer son équipe d'agence
En tant que superviseur, je veux créer et suspendre les agents de mon agence sans passer par le siège.

- Étant donné un superviseur, quand il crée un agent, alors l'agent est rattaché d'office à son agence.

### P3 — Voir son profil
En tant qu'utilisateur interne, je veux voir mon rôle, mon agence et ma dernière connexion.

## Règles métier

- Rôles : Super administrateur, Administrateur, Responsable conformité, Agent support,
  Superviseur d'agence, Agent d'agence. Un utilisateur a un seul rôle.
- Tout rôle d'agence exige une agence de rattachement (donc un pays).
- Les rôles du siège ont un périmètre géographique : **global** ou limité à un ou plusieurs pays
  (ex. administrateur pays, responsable conformité pays, support pays).
- Seul un administrateur global peut créer ou modifier un administrateur pays ; un administrateur
  pays ne gère que les utilisateurs et paramètres de ses pays.
- Second facteur obligatoire pour tout utilisateur interne.
- Un compte n'est jamais supprimé : il est suspendu ou clôturé.
- Toute création, modification de rôle, suspension et réactivation est journalisée (qui, quand,
  avant/après).
- Une matrice rôle × action documentée définit les droits ; elle est vérifiable.

## Données manipulées

Utilisateur interne : nom complet, e-mail, téléphone, rôle, agence (si rôle d'agence), périmètre
pays (global ou liste de pays, pour les rôles du siège), langue d'interface, statut
(actif, suspendu, clôturé), second facteur configuré, dernière connexion.

## Cas limites

- Agence suspendue : ses utilisateurs peuvent se connecter mais ne peuvent effectuer aucune opération.
- Changement d'agence d'un agent : interdit si une caisse est ouverte à son nom.
- Tentatives de connexion répétées en échec : blocage temporaire.

## Critères de succès

- 100 % des actions refusées hors rôle ou hors périmètre lors des tests de la matrice des droits.
- Un nouvel agent est opérationnel en moins de 5 minutes après sa création.
- Aucune connexion interne possible sans second facteur.

## Hors périmètre

Connexion des clients du wallet (fonctionnalité 11), journal d'audit consultable (02).

## Points à clarifier

- Durée d'inactivité avant déconnexion (proposition : 15 min Web, 2 min mobile).
- Nombre d'échecs avant blocage et durée du blocage.
- Un superviseur peut-il changer le rôle d'un agent en superviseur ?
- [À CLARIFIER] Faut-il des rôles dédiés « Finance » (états de taxes, rapprochements) et « Auditeur »
  (lecture seule) ? En version 1, ces besoins sont couverts par Administrateur et Responsable conformité.

## Dépendances

Aucune (première fonctionnalité). Elle inclut la mise en place du socle permettant aux
applications de démarrer.
