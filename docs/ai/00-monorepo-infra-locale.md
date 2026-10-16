# 00 — Monorepo et infrastructure locale

## Objectif
Créer le squelette du monorepo et un environnement local complet démarrable en une commande, sans
aucune logique métier.

## Prérequis
Aucun. Lire `CLAUDE.md` §2, §3, §10.

## Références
Constitution (contraintes techniques, politique de versions), ADR-0003, ADR-0004, cahier des charges §9 (architecture).

## Tâches
1. Initialiser l'arborescence de `CLAUDE.md` §3 (dossiers vides avec `README.md` d'une ligne).
2. `.gitignore` racine (Java/Gradle, Node/pnpm, Flutter/Dart, IDE, `.env`), `.editorconfig`,
   `.gitattributes` (LF), `README.md` racine (présentation + démarrage rapide).
3. `backend/` : projet Gradle Kotlin DSL, Gradle 9, Java 25 toolchain, Spring Boot 4.1, dépendances de
   `CLAUDE.md` §2, application `app.cashinmotion.CashInMotionApplication`, endpoint
   `/actuator/health`, profil `local`. Version catalog `gradle/libs.versions.toml`.
4. `web/` : workspace pnpm (`pnpm-workspace.yaml`), deux apps Vite React TS (`apps/backoffice`
   port 5173, `apps/agency` port 5174) affichant une page « Hello », packages `ui`, `api-client`,
   `config` (tsconfig strict partagé, ESLint flat config, Prettier).
5. `mobile/` : workspace melos, deux apps Flutter (`apps/agent` id `app.cashinmotion.agent`,
   `apps/client` id `app.cashinmotion.client`), package `packages/core` vide, `analysis_options.yaml`
   strict (`very_good_analysis` ou `flutter_lints` + règles strictes).
6. `infra/docker-compose.yml` avec : `postgres:18` (base `cim`, utilisateur `cim`), `valkey/valkey` (tag stable épinglé),
   `rabbitmq:4.3-management`, `quay.io/keycloak/keycloak:26.7` (mode dev, import de realm),
   `minio` (bucket `kyc-documents` créé au démarrage), `mailpit` (SMTP de test). Volumes nommés,
   healthchecks, `.env.example`.
7. `infra/keycloak/realm-cim.json` : realm `cim`, clients `cim-backoffice` (public, PKCE),
   `cim-agency` (public, PKCE), `cim-mobile-agent`, `cim-mobile-client` (public, PKCE),
   `cim-api` (bearer-only), rôles realm de `CLAUDE.md` §6, un utilisateur de démo par rôle
   (mots de passe de dev documentés dans `infra/README.md`).
8. `Makefile` racine : `make up`, `make down`, `make backend`, `make web`, `make test`.
9. `docs/adr/ADR-0001-monolithe-modulaire.md` et `ADR-0002-grand-livre-partie-double.md`
   (contexte, décision, conséquences, 1 page chacun).

## Critères d'acceptation
- `make up` démarre tous les services sains en < 2 min.
- `./gradlew build` passe ; `GET :8080/actuator/health` → `UP` avec Postgres, Valkey, RabbitMQ.
- `pnpm -r build` et `melos run analyze` passent.
- Connexion Keycloak possible avec chaque utilisateur de démo.

## Tests exigés
- Test `@SpringBootTest` de démarrage du contexte avec Testcontainers (Postgres, Valkey, RabbitMQ).

## Hors périmètre
CI/CD (prompt 20), toute entité métier.
