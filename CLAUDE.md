# CLAUDE.md — Cash In Motion (§0 contexte racine)

> Ce fichier est lu par tout agent AI avant chaque tâche. Il fait autorité sur les conventions.
> Spécifications fonctionnelles : `speckit/NN-*.md` puis `specs/` (générés par Spec Kit) ; le
> cahier des charges `docs/cahier-des-charges.md` est le document d'origine (voir constitution).
> Le développement suit **GitHub Spec Kit** : constitution et entrées de fonctionnalités dans
> `speckit/` (voir `speckit/README.md`). Les fichiers `docs/ai/NN-*.md` sont des **notes techniques**
> à fournir à `/speckit-plan`, pas des prompts à exécuter directement.
> En cas de divergence : constitution Spec Kit > spécification validée > ce fichier > cahier des
> charges > notes `docs/ai/`.

## 1. Le produit

Cash In Motion est une plateforme de **transfert d'argent** (envoi/paiement en agence, national et
international) et de **portefeuille électronique** (wallet, P2P, cash-in/cash-out en agence par
QR Code), **multi-pays** : UEMOA (XOF, BCEAO), CEMAC (XAF, BEAC/COBAC) et autres pays de la CEEAC
(CDF, AOA, BIF, RWF, STN). Le **Sénégal** est le premier pays ouvert. Tout ce qui varie par pays
(devise, fuseau, langues, pièces, taxes, plafonds, régulateur, cellule de renseignement financier,
fournisseurs SMS) est **de la configuration par pays, jamais du code**.

Applications :

| Application | Dossier | Utilisateurs |
| --- | --- | --- |
| API backend | `backend/` | toutes les applications |
| Back-office Web | `web/apps/backoffice` | Super admin, Admin, Conformité, Support |
| Portail Agence Web | `web/apps/agency` | Superviseur, Agent d'agence |
| App mobile Agence | `mobile/apps/agent` | Agent d'agence |
| App mobile Client | `mobile/apps/client` | Client final (wallet) |

## 2. Stack (imposée — ne pas substituer sans ADR)

| Couche | Technologie |
| --- | --- |
| Backend | Java 25 (LTS), Spring Boot 4.1 (Spring Framework 7, Jakarta EE 11), Spring Modulith 2.1, Spring Security 7 (resource server JWT), Spring Data JPA (Hibernate 7), Jackson 3, Flyway, springdoc-openapi 3, MapStruct, Gradle 9 Kotlin DSL |
| Base | PostgreSQL 18 |
| Cache / jetons | Valkey (licence BSD, protocole compatible Redis), client Spring Data Redis / Lettuce |
| Documents PDF | Apache FOP (XSL-FO → PDF : reçus A4 et ticket 58/80 mm, rapports, dossiers de déclaration) ; Apache POI pour Excel |
| Messagerie | RabbitMQ 4.3 (via outbox transactionnelle) |
| IAM | Keycloak 26.7+ (OIDC) |
| Stockage objet | S3-compatible (MinIO en local) |
| Web | React 19 + TypeScript 7 (strict), Vite 8, pnpm workspaces, TanStack Router 1 + Query 5, react-hook-form 7 + zod 4, shadcn/ui + Tailwind 4, Apache ECharts 6, client API généré depuis OpenAPI (orval) |
| Mobile | Flutter stable courante, 3.47+ (Dart 3), melos, Riverpod, go_router, dio, freezed/json_serializable, mobile_scanner, qr_flutter, local_auth, flutter_secure_storage |
| Tests | JUnit (version gérée par Spring Boot), AssertJ, Testcontainers, ArchUnit, Spring Modulith tests ; Vitest + Testing Library + Playwright ; flutter_test + mocktail + integration_test ; k6 |
| Infra | Docker, docker-compose (dev), Kubernetes/Helm (prod), GitHub Actions ou GitLab CI, Prometheus/Grafana/Loki/OpenTelemetry, Sentry |

### Pièges de version à éviter (les modèles IA ont surtout appris Spring Boot 3)

- Jackson 3 : paquets `tools.jackson.*` (et non `com.fasterxml.jackson.databind`) ; les annotations
  restent dans `com.fasterxml.jackson.annotation`.
- Spring Boot 4 : starters et auto-configurations découpés par technologie ; vérifier les noms de
  starters et de propriétés dans la documentation 4.1, pas de mémoire.
- Hibernate 7 / Jakarta Persistence 3.2 ; Spring Security 7 (DSL lambda uniquement) ;
  annotations de nullité JSpecify (`org.jspecify.annotations`).
- Spring Framework 7 fournit `@Retryable` et `@ConcurrencyLimit` en natif ; Resilience4j seulement
  pour le disjoncteur (vérifier son module compatible Boot 4, sinon implémenter avec Spring).
- Java 25 : threads virtuels activés (`spring.threads.virtual.enabled=true`), pas de
  `synchronized` long autour d'appels réseau.
- Valkey : pas de bibliothèque « Valkey » dédiée côté Spring ; utiliser Spring Data Redis (Lettuce),
  Bucket4j et Testcontainers avec l'image `valkey/valkey` ; ne jamais introduire Redis dans la stack.
- PDF : Apache FOP uniquement (pas de JasperReports, iText ni OpenPDF) ; gabarits XSL-FO versionnés
  dans `backend/src/main/resources/templates/fo/`, polices embarquées (accents, symbole FCFA),
  formats de page A4 et ticket 58/80 mm.
- TypeScript 7 (compilateur natif) : si un outil de l'espace Web ne le supporte pas encore
  (ex. une version de typescript-eslint), le signaler et appliquer la solution de compatibilité
  documentée par l'outil plutôt que de revenir en TypeScript 6 sans ADR.
- En cas de doute sur une API, consulter la documentation de la version exacte avant de coder.

## 3. Arborescence du monorepo

```
cash-in-motion/
├── CLAUDE.md                  ← ce fichier
├── AGENTS.md                  ← pointe vers CLAUDE.md
├── backend/                   ← Gradle, package racine app.cashinmotion
│   └── src/main/java/app/cashinmotion/
│       ├── shared/            ← Money, ids, erreurs, idempotence, outbox, sécurité commune
│       ├── identity/          ← utilisateurs internes, rôles, périmètre agence
│       ├── referential/       ← pays, régions, villes, devises, corridors, types de pièces
│       ├── pricing/           ← grilles de frais, taxes, taux de change, moteur de devis
│       ├── ledger/            ← grand livre en partie double (SEULE source des soldes)
│       ├── customer/          ← clients, bénéficiaires, KYC
│       ├── agency/            ← agences, caisses, float, commissions
│       ├── transfer/          ← envois, paiements, annulations
│       ├── wallet/            ← wallets, P2P, cash-in/out QR
│       ├── compliance/        ← sanctions/PEP, règles AML, alertes, plafonds
│       ├── notification/      ← SMS, push, e-mail, webhooks
│       ├── reporting/         ← tableaux de bord, exports, états fiscaux
│       └── audit/             ← journal d'audit inaltérable
├── web/
│   ├── apps/backoffice/  apps/agency/
│   └── packages/ui/  packages/api-client/  packages/config/
├── mobile/
│   ├── apps/agent/  apps/client/
│   └── packages/core/         ← API, auth, QR, design system, i18n
├── infra/                     ← docker-compose, keycloak realm, helm, observabilité
├── api/openapi.yaml           ← exporté du backend en CI, source des clients générés
├── docs/
│   ├── cahier-des-charges.md
│   ├── adr/                   ← décisions d'architecture (ADR-NNNN-titre.md)
│   └── ai/                    ← notes techniques NN-*.md pour /speckit-plan
├── speckit/                   ← constitution + entrées /speckit-specify
├── .specify/                  ← créé par Spec Kit (mémoire, modèles, scripts)
└── specs/                     ← créé par Spec Kit (spec, plan, tasks par fonctionnalité)
```

## 4. Règles d'or (non négociables)

1. **Argent = `BigDecimal` + devise.** Jamais `double`/`float`. Type valeur `Money(amount, currency)`
   dans `shared`. En base : `NUMERIC(19,4)` + `CHAR(3)` (ISO 4217). XOF : 0 décimale, arrondi
   `HALF_UP` (règle d'arrondi par devise dans le référentiel).
2. **Les soldes ne se modifient jamais directement.** Tout mouvement passe par
   `LedgerService.post(JournalEntry)` : écritures équilibrées (Σ débits = Σ crédits par devise),
   dans la même transaction que l'opération métier. Les tables d'écritures sont en ajout seul.
3. **Idempotence** : toute commande financière (POST qui déplace de l'argent) exige l'en-tête
   `Idempotency-Key` (UUID). Rejeu = même réponse, aucun double effet.
4. **Concurrence** : verrouiller les comptes débités (`SELECT … FOR UPDATE`) dans un ordre
   déterministe (tri par id de compte). Pas de verrouillage optimiste sur les soldes.
5. **Paramètres versionnés** : frais, taxes, taux de change et plafonds ont `valid_from`/`valid_to`.
   Une transaction fige une copie (snapshot) des valeurs appliquées ; un changement ultérieur ne
   modifie jamais une transaction existante.
6. **Machines à états explicites** pour transferts, opérations wallet, KYC, alertes : transitions
   autorisées déclarées dans une enum, toute autre transition lève `IllegalStateTransitionException`.
7. **Événements via outbox** : écrire dans `outbox_event` dans la transaction, un relais publie vers
   RabbitMQ. Jamais d'appel réseau externe (SMS, KYC, partenaire) dans une transaction DB.
8. **Données personnelles** : n° de pièce, réponses secrètes, documents KYC chiffrés au repos
   (chiffrement applicatif AES-GCM, clé via Vault/KMS) ; réponse à la question test **hachée**
   (Argon2id). Jamais de donnée personnelle ni de secret dans les logs.
9. **Autorisation côté serveur** : RBAC + périmètre (un agent/superviseur ne voit que son agence).
   Ne jamais se fier au client.
10. **Audit** : toute action d'écriture d'un utilisateur interne produit une entrée d'audit
    (acteur, action, cible, avant/après, IP, horodatage).
11. **Frontières de modules** : un module n'accède qu'à l'API publique (`api` package) des autres
    modules ; pas de jointure JPA entre modules ; vérifié par Spring Modulith + ArchUnit.
12. **Pas de secret dans le dépôt.** Configuration par variables d'environnement.
13. **Corrélation** : toute ligne de journal porte `correlationId` et `requestId` (MDC) ; ces
    valeurs suivent l'exécution partout (threads, `@Async`, outbox, RabbitMQ, appels sortants,
    jobs). Ne jamais créer de thread ou d'exécuteur sans propagation de contexte ; ne jamais
    écrire dans les journaux autrement que par SLF4J. Détail : `docs/ai/01-backend-socle.md`,
    section « Corrélation des requêtes ».

## 5. Conventions de code

- **Langue** : code, noms de tables/colonnes, commits en **anglais** ; textes affichés, messages
  d'erreur utilisateur et documentation en **français** (clés i18n, `fr` par défaut).
- **Identifiants** : UUID v7 (`shared.Ids.newId()`). Codes lisibles à part (ex. code de retrait).
- **Dates** : `timestamptz` en UTC ; affichage et notion de « journée » dans le fuseau IANA de
  l'agence ou du pays du client (`countries.default_timezone`, `agencies.timezone`).
- **Pays** : toute entité opérationnelle porte `country_iso2` (agence, wallet, compte du grand
  livre, règle de taxe, plafond, alerte, route SMS) ; les utilisateurs du siège ont un périmètre
  pays (`GLOBAL` ou liste) appliqué comme le périmètre agence.
- **Tables** : snake_case, pluriel (`transfers`, `ledger_entries`), colonnes `created_at`,
  `updated_at`, `created_by`, `version` sur les tables mutables.
- **Migrations** : Flyway `V{yyyyMMddHHmm}__{module}_{description}.sql`, jamais modifiées après merge.
- **API REST** : `/api/v1/...`, JSON camelCase, pagination par curseur (`?cursor=&limit=`),
  erreurs RFC 7807 (`application/problem+json`) avec `code` métier stable (ex. `TRANSFER_NOT_PAYABLE`),
  montants sérialisés en chaîne (`"amount": "100000"`, `"currency": "XOF"`).
- **En-têtes de corrélation** : `X-Correlation-ID` (envoyé par les clients, une valeur par action
  utilisateur, réutilisée pour ses retries ; renvoyé par le serveur), `X-Request-ID` (généré par le
  serveur, renvoyé), `X-Client-App` ; les erreurs RFC 7807 contiennent `correlationId` et
  `requestId`, affichés comme « Référence » à l'utilisateur.
- **Structure d'un module backend** : `api/` (façade publique + DTO + événements), `domain/`
  (entités, règles, états), `application/` (services, cas d'usage), `infrastructure/` (JPA,
  clients externes), `web/` (contrôleurs). Logique métier dans `domain`/`application`, pas dans
  les contrôleurs.
- **Web** : composants fonctionnels, aucun appel `fetch` manuel (client généré + TanStack Query),
  formulaires `react-hook-form` + `zod`, montants formatés via `formatMoney()` de `packages/ui`.
- **Mobile** : architecture feature-first (`features/<x>/{data,domain,presentation}`), Riverpod,
  aucun secret en clair hors `flutter_secure_storage`.

## 6. Rôles (valeurs exactes)

`SUPER_ADMIN`, `ADMIN`, `COMPLIANCE_OFFICER`, `SUPPORT_AGENT`, `AGENCY_SUPERVISOR`,
`AGENCY_AGENT`, `CUSTOMER`.

## 7. Taxes (rappel — par pays, paramétrables, jamais en dur)

Chaque règle de taxe appartient à un pays (`country_iso2`) et s'applique aux opérations réalisées
dans ce pays (envoi : pays émetteur ; paiement : pays payeur). Taxes initiales du **Sénégal** :

| Code | Base | Taux initial | Plafond |
| --- | --- | --- | --- |
| `TTA` | montant envoyé | 0,5 % | 2 000 XOF / opération |
| `TAF` | frais nets de rabais | 17 % | — |
| `THZUEMOA` | montant envoyé | à paramétrer | à paramétrer ; seulement si l'autre pays du corridor est hors UEMOA |

Taxes à la charge de l'expéditeur (hypothèse en attente de validation — garder la règle paramétrable :
`tax_payer = SENDER | RECEIVER`).

## 8. Définition de « terminé » (pour chaque prompt)

- `./gradlew build` (ou `pnpm -r build && pnpm -r test`, ou `melos run test`) passe sans erreur.
- Tests unitaires + intégration écrits pour chaque règle métier listée dans le prompt.
- Couverture ≥ 80 % sur `ledger` et `pricing`, ≥ 70 % ailleurs.
- `api/openapi.yaml` régénéré si l'API change ; clients Web/Mobile régénérés.
- Migration Flyway ajoutée si le schéma change.
- `tasks.md` de la fonctionnalité à jour (tâches cochées) et `/speckit-converge` à « Converged ».
- Aucun TODO silencieux : toute hypothèse est consignée dans la spec ou le plan de la fonctionnalité.

## 9. Comment travailler

1. Suivre le flux Spec Kit de la fonctionnalité active (spec → plan → tasks → implement → converge).
2. Implémenter par petits commits (Conventional Commits), une tâche de `tasks.md` à la fois.
3. Ne pas élargir le périmètre de la spécification ; proposer les idées hors périmètre comme futures fonctionnalités.
4. En cas d'ambiguïté bloquante : choisir l'option la plus sûre (refuser plutôt qu'accepter une
   opération financière douteuse), la documenter, continuer.

## 10. Commandes utiles

```bash
docker compose -f infra/docker-compose.yml up -d     # Postgres, Valkey, RabbitMQ, Keycloak, MinIO, Mailpit
cd backend && ./gradlew bootRun                      # API sur :8080, Swagger sur /swagger-ui
cd web && pnpm install && pnpm dev                   # backoffice :5173, agency :5174
cd mobile && melos bootstrap && melos run test
```
