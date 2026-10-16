<!--
Usage : lancer `/speckit-constitution` et lui fournir ce fichier
(ex. « Établis la constitution du projet à partir de @speckit/constitution.md »).
Spec Kit l'enregistrera dans sa mémoire de projet (habituellement .specify/memory/constitution.md).
-->

# Constitution de Cash In Motion

Cash In Motion est une plateforme de transfert d'argent et de portefeuille électronique destinée à
l'ensemble de l'Afrique de l'Ouest et de l'Afrique centrale : les 8 pays de l'UEMOA (zone BCEAO,
franc CFA XOF), les 6 pays de la CEMAC (zone BEAC/COBAC, franc CFA XAF) et les autres pays de la
CEEAC (devises nationales). Le Sénégal est le premier pays d'opération. Parce qu'elle manipule
l'argent de clients et d'agences, chaque principe ci-dessous l'emporte sur la vitesse de livraison. Les termes DOIT,
NE DOIT PAS et DEVRAIT ont la force de MUST, MUST NOT et SHOULD.

## Principes fondamentaux

### I. Exactitude monétaire (NON NÉGOCIABLE)

- Tout montant DOIT être représenté en décimal exact accompagné de son code devise ISO 4217 ;
  les nombres à virgule flottante sont interdits pour l'argent, en code comme en base.
- Les règles d'arrondi DOIVENT être définies par devise (XOF : aucune décimale, arrondi au plus
  proche, demi vers le haut) et appliquées ligne par ligne, jamais sur un total recalculé.
- Une opération entre deux devises différentes sans taux de change explicite DOIT échouer.

*Justification : une erreur d'un franc répétée sur des millions d'opérations est une perte
financière et une faute réglementaire.*

### II. Grand livre en partie double, source unique des soldes (NON NÉGOCIABLE)

- Tout mouvement de valeur DOIT être enregistré par des écritures équilibrées (somme des débits =
  somme des crédits, par devise) dans la même transaction que l'opération métier.
- Aucun solde NE DOIT être modifié directement ; les soldes sont dérivés des écritures.
- Les écritures sont en ajout seul : une correction se fait par contre-passation, jamais par
  modification ou suppression.
- Un contrôle d'intégrité automatique DOIT vérifier quotidiennement l'équilibre global, la
  cohérence des soldes et, pour chaque pays et chaque devise, la couverture de toute la valeur
  électronique due (wallets clients + floats d'agences) par le compte de cantonnement correspondant,
  augmenté ou diminué du solde net des comptes de liaison avec les autres pays (transferts
  inter-pays non encore compensés).
- Toutes les agences fonctionnent sur float prépayé : envoi et dépôt débitent le float de l'agence,
  paiement et retrait le créditent ; les espèces physiques sont suivies dans un livre de caisse.

### III. Opérations financières sûres par conception

- Toute commande qui déplace de l'argent DOIT être idempotente : la même demande rejouée produit
  le même résultat et un seul effet.
- Les accès concurrents à un même compte DOIVENT être sérialisés de façon à rendre impossibles le
  double débit et le solde négatif non autorisé.
- Chaque objet financier (transfert, opération wallet, dossier KYC, alerte) DOIT suivre une machine
  à états explicite ; toute transition non déclarée est refusée et toute transition est historisée.
- Tarifs, taxes, taux de change et plafonds DOIVENT être versionnés avec une date d'effet ; une
  opération fige les valeurs appliquées et n'est jamais recalculée a posteriori.
- En cas de doute (donnée manquante, service tiers indisponible, règle ambiguë), le système DOIT
  refuser ou mettre en attente l'opération plutôt que l'exécuter.
- Aucun appel à un service externe (SMS, vérification d'identité, partenaire) NE DOIT avoir lieu à
  l'intérieur d'une transaction financière ; les effets externes passent par une file d'événements
  fiable (outbox).

### IV. Conformité réglementaire intégrée

- Chaque opération DOIT passer un pré-contrôle de conformité (listes de sanctions et PEP, statut du
  client, plafonds par niveau KYC, agence et corridor) avant tout mouvement de fonds.
- Les flux DOIVENT être surveillés par des règles LBC/FT paramétrables produisant des alertes
  revues par un responsable conformité, avec possibilité de déclaration à la CENTIF.
- Les données KYC et transactionnelles DOIVENT être conservées au moins 10 ans et ne peuvent être
  effacées avant ce terme.
- Les taxes sont propres à chaque pays (ex. Sénégal : TTA, TAF, THZUEMOA) : chaque taxe est
  rattachée à un pays d'opération, paramétrable, jamais écrite en dur, et collectée sur un compte
  dédié par pays et par devise permettant son reversement à l'administration concernée.
- Les obligations réglementaires dépendent de la zone : BCEAO et CENTIF nationales (GIABA) pour
  l'UEMOA ; BEAC, COBAC et ANIF nationales (GABAC) pour la CEMAC ; banque centrale et cellule de
  renseignement financier nationales pour les autres pays. Plafonds, déclarations et rapports sont
  donc paramétrés et produits par pays.

### V. Protection des données personnelles et sécurité

- Le traitement des données respecte la loi de protection des données de chaque pays d'opération
  (au Sénégal : loi n° 2008-12, CDP), y compris ses règles de localisation et de transfert des
  données, et le RGPD pour les résidents de l'UE : minimisation, finalité, consentement explicite pour le marketing, droits
  d'accès et de rectification.
- Les données sensibles (numéros de pièce, documents d'identité) DOIVENT être chiffrées au repos ;
  les secrets (réponse à la question test, PIN) DOIVENT être hachés avec un algorithme adapté.
- Aucune donnée personnelle ni aucun secret NE DOIT apparaître dans les journaux techniques.
- Les données personnelles sont masquées par défaut dans les interfaces internes ; leur
  révélation exige un motif et est journalisée.
- L'autorisation DOIT être vérifiée côté serveur à chaque requête : rôle ET périmètre (un agent ou
  superviseur n'accède qu'à son agence).
- L'authentification multifacteur est obligatoire pour tout utilisateur interne ; les appareils
  mobiles sont enrôlés et les opérations financières mobiles exigent PIN ou biométrie.
- Toute action d'écriture d'un utilisateur interne DOIT produire une entrée d'audit inaltérable
  (acteur, action, cible, avant/après, horodatage).
- Chaque requête reçue DOIT être identifiée par un identifiant de corrélation et un identifiant de
  requête présents dans **toutes** les lignes de journal produites par son exécution, propagés aux
  traitements asynchrones, événements et appels externes qu'elle déclenche, et renvoyés au client
  (référence d'erreur communicable au support).

### VI. Tests comme preuve (NON NÉGOCIABLE pour le cœur financier)

- Chaque règle métier d'une spécification DOIT être couverte par au moins un test automatisé
  écrit avant ou avec son implémentation.
- Les fonctionnalités touchant au grand livre, à la tarification, aux transferts et au wallet
  DOIVENT inclure : tests d'intégration sur une vraie base, tests de concurrence (double soumission,
  accès simultanés), tests de toutes les transitions d'état interdites.
- Couverture minimale : 80 % sur le grand livre et la tarification, 70 % ailleurs.
- Les exemples chiffrés des spécifications (ex. envoi de 100 000 XOF → 104 244 XOF payés) DOIVENT
  devenir des tests d'acceptation exécutables.

### VII. Simplicité et modularité

- Architecture en monolithe modulaire : modules métier aux frontières explicites, communiquant
  uniquement par leurs interfaces publiques et des événements ; ces frontières sont vérifiées
  automatiquement.
- Pas d'abstraction, de service ou de dépendance sans besoin démontré par une spécification ;
  toute exception est justifiée dans le plan (section de suivi de complexité).
- Les contrats d'API sont décrits en OpenAPI et les clients Web et mobile en sont générés.

### VIII. Expérience utilisateur adaptée au terrain

- Les interfaces sont en français par défaut ; l'anglais, le portugais et l'espagnol (Cameroun
  anglophone, Guinée-Bissau, São Tomé-et-Principe, Angola, Guinée équatoriale) et les langues
  locales doivent pouvoir être ajoutés sans modification de code.
- Dates et heures sont affichées dans le fuseau du pays concerné (agence ou client) ; les
  notions « jour », « le jour même » et les périodes de rapport suivent ce fuseau.
- Les applications mobiles DOIVENT rester utilisables en 3G et en connexion instable, sur des
  appareils d'entrée de gamme ; aucune opération financière n'est mise en file hors ligne.
- Le parcours d'envoi au guichet DOIT pouvoir être réalisé en moins de 2 minutes.
- Frais, taxes et montant final sont toujours affichés avant toute confirmation.

### IX. Multi-pays par configuration

- Tout ce qui varie d'un pays à l'autre DOIT être une donnée de configuration par pays, jamais du
  code : devise, fuseau horaire, langues, indicatif et format téléphonique, pièces d'identité
  acceptées, taxes, plafonds, régulateur et cellule de renseignement financier, fournisseurs SMS,
  formats de rapports réglementaires.
- Ouvrir un nouveau pays DOIT se faire par paramétrage et validation, sans nouvelle version.
- Les montants de devises différentes (XOF, XAF, CDF…) ne sont jamais additionnés sans
  conversion explicite ; les rapports consolidés indiquent la devise et le taux utilisés.
- Les SMS, canal critique dans ces marchés, sont acheminés par le fournisseur configuré pour le
  pays du destinataire, avec un fournisseur de secours ; l'absence de fournisseur pour un pays
  d'opération est une anomalie bloquante pour l'ouverture de ce pays.

## Contraintes techniques

| Domaine | Choix imposé |
| --- | --- |
| Backend | Java 25 (LTS), Spring Boot 4.1 (Spring Framework 7, Jakarta EE 11, Hibernate 7, Jackson 3), Spring Modulith 2.1, Spring Security 7, Spring Data JPA, Flyway, springdoc-openapi 3, Gradle 9 (Kotlin DSL) |
| Données | PostgreSQL 18 ; Valkey (jetons, OTP, limitation de débit ; Redis exclu pour raison de licence) ; RabbitMQ 4.3 (événements) ; stockage objet S3-compatible (documents KYC) |
| Identité | Keycloak 26.7+ (OIDC), rôles : SUPER_ADMIN, ADMIN, COMPLIANCE_OFFICER, SUPPORT_AGENT, AGENCY_SUPERVISOR, AGENCY_AGENT, CUSTOMER |
| Web | React 19, TypeScript 7 (strict), Vite 8, TanStack Router 1 / Query 5, shadcn/ui + Tailwind 4, ECharts 6, zod 4 ; deux applications : Back-office et Portail Agence |
| Mobile | Flutter stable courante (3.47 ou plus), Riverpod (dernière version majeure), go_router ; deux applications : Agence et Client |
| Documents | Apache FOP pour tous les PDF (reçus, rapports, déclarations) ; Apache POI pour Excel |
| Infrastructure | Docker, Kubernetes/Helm, CI avec analyse de sécurité (dépendances, images), observabilité OpenTelemetry/Prometheus/Grafana/Loki, Sentry |
| Monorepo | `backend/`, `web/`, `mobile/`, `infra/`, `api/openapi.yaml`, `docs/` |
| Conventions | Code et schéma en anglais ; textes affichés et documentation en français ; identifiants UUID v7 ; horodatages UTC, affichage dans le fuseau IANA du pays concerné ; erreurs RFC 7807 avec code métier stable ; montants sérialisés en chaîne |

Toute dérogation à ces choix exige une décision d'architecture écrite (ADR) dans `docs/adr/`.

### Politique de versions

- On n'utilise que des versions sous **support open source actif** ; aucune version dont le support
  se termine dans les 6 mois n'est adoptée.
- Java reste sur la **LTS courante** (25, support jusqu'en 2030 au moins) ; pas de version non-LTS
  en production.
- Spring Boot suit sa branche mineure la plus récente : montée de version mineure planifiée tous
  les 6 mois (4.1 → 4.2 → …), avant la fin du support de la branche en cours.
- Les versions des bibliothèques sont gérées par la nomenclature Spring Boot (BOM) et un catalogue
  unique (`gradle/libs.versions.toml`, `package.json` de l'espace pnpm, `pubspec` racine) ; aucune
  version libre dans un module.
- Le code généré DOIT suivre les API de ces versions : pas d'API Spring Boot 3, Jackson 2
  (`com.fasterxml.jackson.databind`) ou Hibernate 6 ; la compilation, les analyses et les recettes
  OpenRewrite de la CI bloquent ces régressions.
- Une mise à jour de sécurité critique est appliquée sous 7 jours.

## Flux de développement et portes qualité

1. Chaque fonctionnalité suit : specify → clarify → plan → checklist → tasks → analyze →
   implement → converge.
2. Une spécification ne contient aucun choix technique ; tout point incertain est marqué à
   clarifier et tranché avant le plan.
3. Le plan DOIT vérifier sa conformité à chaque principe de cette constitution avant et après la
   conception ; toute violation non justifiée bloque la suite.
4. Une fonctionnalité n'est terminée que si : la compilation et tous les tests passent, la
   couverture minimale est atteinte, le contrat OpenAPI et les migrations sont à jour, aucune
   vulnérabilité haute ou critique n'est ouverte.
5. Revue humaine obligatoire avant fusion pour toute fonctionnalité touchant au grand livre, à la
   tarification, aux transferts, au wallet ou à la conformité.
6. Le cahier des charges (`docs/cahier-des-charges.md`, version initiale du 15/10/2026, centrée sur
   le Sénégal) n'est plus qu'un document d'origine : il est remplacé par cette constitution, les
   ADR et les spécifications validées ; ses exemples chiffrés (§4.5) restent valables. Sa section
   technique (§10) est obsolète.

## Gouvernance

- Cette constitution prévaut sur toute autre pratique ou consigne d'agent.
- Toute modification passe par une proposition écrite, une revue par le responsable technique et
  le responsable conformité pour les principes II à V, puis une mise à jour de version :
  majeure (principe retiré ou redéfini), mineure (principe ou section ajoutés), correctif
  (clarification).
- Les revues de code et l'étape d'analyse vérifient la conformité à cette constitution.

**Version** : 1.3.0 | **Ratifiée** : 2026-10-16 | **Dernière modification** : 2026-10-16

*1.2.0 : passage à Java 25 / Spring Boot 4.1 et versions courantes de la stack ; politique de versions.*
*1.2.1 : Valkey, TypeScript 7, Apache FOP (ADR-0004).*
*1.2.2 : couverture du cantonnement = wallets + floats ± liaisons inter-pays ; statut du cahier des charges clarifié.*
*1.3.0 : corrélation obligatoire des journaux par requête (principe V).*

*1.1.0 : extension du périmètre à l'UEMOA, la CEMAC et la CEEAC ; principe IX (multi-pays par
configuration) ; taxes, régulateurs, protection des données, langues et fuseaux par pays.*
