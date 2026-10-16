# 01 — Socle backend (module `shared`)

## Objectif
Fournir les briques transverses que tous les modules réutiliseront : argent, identifiants, erreurs,
idempotence, outbox, pagination, horloge, chiffrement, configuration de test.

## Prérequis
00 terminé.

## Références
`CLAUDE.md` §4 (règles 1, 3, 7, 8), §5. Cahier §9.1, §9.2.

## Tâches
1. **Spring Modulith** : déclarer les modules de `CLAUDE.md` §3 (packages vides avec
   `package-info.java` `@ApplicationModule`). Test `ApplicationModules.verify()` + test ArchUnit
   interdisant l'accès aux packages non `api` d'un autre module.
2. **Money** (`shared.money`) : record `Money(BigDecimal amount, CurrencyCode currency)` immuable ;
   `plus`, `minus`, `multiply(BigDecimal rate)`, `percent(BigDecimal pct)`, `min`, `max`,
   `isNegative`, `round(RoundingPolicy)` ; refus d'opérer entre devises différentes
   (`CurrencyMismatchException`). `CurrencyCode` = enum ou value object ISO 4217 avec nombre de
   décimales (XOF=0, EUR=2, USD=2, GNF=0…). Convertisseur JPA `@Embeddable` (`amount`, `currency`)
   et sérialiseur Jackson (montant en chaîne).
3. **Identifiants** : `Ids.newId()` UUID v7 (lib `uuid-creator`), type générique `Id<T>` optionnel.
4. **Horloge** : bean `Clock` injectable (UTC) ; jamais `Instant.now()` direct (règle ArchUnit).
5. **Erreurs** : `BusinessException(code, httpStatus, args)` + `@RestControllerAdvice` produisant du
   RFC 7807 (`type`, `title`, `status`, `detail`, `code`, `correlationId`, `requestId`, `traceId`,
   `errors[]` pour la validation).
   Messages en français via `messages_fr.properties`.
6. **Idempotence** : annotation `@Idempotent` sur les endpoints ; filtre qui exige
   `Idempotency-Key`, stocke (clé, utilisateur, hash du corps, statut, réponse) dans la table
   `idempotency_keys` (TTL 48 h) ; rejeu → même réponse ; même clé + corps différent → 422
   `IDEMPOTENCY_KEY_REUSED` ; requête concurrente en cours → 409 `REQUEST_IN_PROGRESS`.
7. **Outbox** : table `outbox_events` (id, aggregate_type, aggregate_id, event_type, payload jsonb,
   occurred_at, published_at, attempts) ; `OutboxPublisher.append(event)` à appeler dans la
   transaction ; relais planifié (`@Scheduled`, `FOR UPDATE SKIP LOCKED`) qui publie sur l'exchange
   RabbitMQ `cim.events` (routing key = `event_type`), retry exponentiel, dead-letter.
8. **Pagination par curseur** : `CursorPage<T>(items, nextCursor)` + utilitaire d'encodage
   (base64 de `created_at,id`).
9. **Chiffrement applicatif** : `FieldEncryptor` AES-256-GCM, clé fournie par `CIM_DATA_KEY`
   (abstraction `KeyProvider` pour Vault/KMS plus tard), convertisseur JPA `@Convert` pour les
   colonnes sensibles ; `SecretHasher` Argon2id.
10. **Base JPA** : `BaseEntity` (id, created_at, updated_at, created_by, updated_by, version) avec
    auditing Spring Data ; `AppendOnlyEntity` sans update.
11. **OpenAPI** : springdoc, groupe `v1`, sécurité bearer, tâche Gradle `exportOpenApi` qui écrit
    `api/openapi.yaml`.
12. **Observabilité de base** : logs JSON (logstash-logback-encoder), masquage des champs
    sensibles (`idNumber`, `pin`, `answer`, `phone` partiellement) ; corrélation des requêtes selon
    la section « Corrélation des requêtes » ci-dessous.
13. **Support de test** : classe `IntegrationTest` (Testcontainers partagés, `@Sql` de nettoyage),
    `TestClock` réglable.

## Modèle de données
Migrations : `idempotency_keys`, `outbox_events`.

## Critères d'acceptation
- `Money.of("100000","XOF").percent("0.5")` = `500 XOF` ; opérations mixtes XOF/EUR → exception.
- Un POST `@Idempotent` rejoué 5 fois en parallèle produit un seul effet.
- Un événement ajouté puis transaction annulée n'est jamais publié.

## Tests exigés
Unitaires Money (arrondis, devises, bornes), intégration idempotence (concurrence), outbox
(rollback, retry, publication), chiffrement (aller-retour, IV unique), ArchUnit.

## Hors périmètre
Toute entité métier.


## Corrélation des requêtes (CorrelationId / RequestId) — exigence ajoutée le 2026-10-16

**But** : retrouver dans les journaux **toutes les lignes** produites par l'exécution d'une requête
HTTP (contrôleur, sécurité, services, accès base, appels sortants, événements et traitements
asynchrones qu'elle déclenche), avec une seule valeur de recherche.

### Identifiants (package `app.cashinmotion.shared.observability`)
| Clé MDC | En-tête HTTP | Origine | Rôle |
| --- | --- | --- | --- |
| `correlationId` | `X-Correlation-ID` (requête et réponse) | repris du client s'il est valide (`^[A-Za-z0-9._-]{8,64}$`), sinon généré (UUID v7) | fil de bout en bout d'une action utilisateur : traverse requêtes, événements, jobs et appels sortants |
| `requestId` | `X-Request-ID` (réponse) | **toujours** généré par le serveur (UUID v7), jamais repris du client | une exécution HTTP précise (un rejeu ou un retry a un nouveau `requestId`, même `correlationId`) |
| `traceId`, `spanId` | `traceparent` (W3C) | Micrometer Tracing + OpenTelemetry | lien avec les traces distribuées |
| `httpMethod`, `httpRoute` | — | gabarit de route (`/api/v1/transfers/{code}`), jamais l'URL réelle (données) | |
| `clientApp` | `X-Client-App` | valeurs contrôlées : `backoffice`, `agency`, `mobile-agent`, `mobile-client` | |
| `userId`, `agencyId`, `countryIso2` | — | ajoutés après authentification ; identifiants internes uniquement (jamais e-mail, téléphone, nom) | |

### Tâches
1. `CorrelationFilter` (`OncePerRequestFilter`, ordre `Ordered.HIGHEST_PRECEDENCE`, donc **avant
   Spring Security**) : valide ou génère `correlationId`, génère `requestId`, remplit le MDC, ajoute
   `X-Correlation-ID` et `X-Request-ID` à **toutes** les réponses (y compris 401, 403, 404, 500 et
   réponses produites par Spring Security), puis `MDC.clear()` dans un `finally`. Le
   `correlationId` est aussi exposé en attribut de requête et via `CorrelationContext.current()`.
2. Après authentification (filtre placé après le filtre Bearer de Spring Security) : ajout de
   `userId`, `agencyId`, `countryIso2` au MDC.
3. Journal d'accès : une ligne INFO en fin de requête (méthode, route, statut, durée en ms, taille de
   réponse, `clientApp`) ; les endpoints actuator de santé sont exclus.
4. Erreurs RFC 7807 : champs `correlationId` et `requestId` systématiques ; c'est la **référence**
   affichée par les applications Web et mobiles et communiquée au support.
5. **Propagation asynchrone** : bibliothèque Micrometer `context-propagation` +
   `ContextPropagatingTaskDecorator` sur tous les `TaskExecutor` (`@Async`, exécuteurs applicatifs,
   threads virtuels) ; règle ArchUnit interdisant `new Thread(...)` et `Executors.*` non décorés.
6. **Outbox et RabbitMQ** : colonnes `correlation_id` et `causation_request_id` sur `outbox_events` ;
   le relais publie avec les en-têtes `x-correlation-id`, `x-causation-id`, `traceparent` ; chaque
   consommateur restaure le MDC (`correlationId` d'origine, `messageId` comme `requestId`,
   `consumer`), traite, puis nettoie le MDC.
7. **Appels sortants** (fournisseurs SMS, KYC, sanctions, Admin API Keycloak, partenaires) :
   intercepteur `RestClient` qui ajoute `X-Correlation-ID` et `traceparent`, et journalise
   fournisseur, opération, statut et durée, sans corps ni secret.
8. **Jobs planifiés** (relais outbox, intégrité du grand livre, expirations, exports) : un
   `correlationId` par exécution (`job-<nom>-<uuid v7>`), MDC `jobName`.
9. **Persistance de la corrélation** : colonne `correlation_id` sur `idempotency_keys`,
   `journal_entries`, `transfer_status_history`, `wallet_operations`, `notifications`,
   `notification_attempts`, `audit_log`, `aml_alerts` — pour relier journaux et données.
10. **Format** : en recette et production, JSON avec les clés MDC au premier niveau ; en local,
    motif console incluant `[%X{correlationId:-}] [%X{requestId:-}]`.
11. **Sécurité** : valeur entrante validée (jeu de caractères, longueur) pour empêcher l'injection
    dans les journaux ; jamais utilisée pour une décision d'autorisation ; une valeur invalide est
    remplacée et l'événement journalisé en DEBUG.

### Critères d'acceptation
- Pour une requête `POST /api/v1/transfers`, **toutes** les lignes de journal émises pendant son
  exécution (sécurité, contrôleur, services, repositories, appel au moteur de devis, outbox)
  portent le même `correlationId` et le même `requestId`.
- Le message RabbitMQ `TransferSent` et les lignes du consommateur de notifications (y compris
  l'appel au fournisseur SMS) portent le même `correlationId`.
- Un `X-Correlation-ID` entrant valide est repris ; absent ou invalide → généré ; les deux en-têtes
  sont présents dans la réponse, y compris en 401, 403 et 500.
- Aucune valeur de MDC ne fuit d'une requête à l'autre (test de 200 requêtes concurrentes sur
  threads virtuels).
- Le corps d'erreur RFC 7807 contient `correlationId` et `requestId`.

### Tests exigés
Tests d'intégration avec capture et analyse des journaux JSON (`OutputCaptureExtension`) ; tests de
propagation `@Async`, outbox → RabbitMQ → consommateur (Testcontainers), appel sortant (WireMock
vérifie l'en-tête) ; test de concurrence anti-fuite du MDC ; test ArchUnit des exécuteurs.
