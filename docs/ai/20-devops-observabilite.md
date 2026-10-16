# 20 — DevOps, CI/CD et observabilité

## Objectif
Automatiser build, tests, analyse de sécurité et déploiement, et rendre la plateforme observable.

## Prérequis
00 terminé ; peut avancer en parallèle des autres prompts.

## Références
Cahier §8 (disponibilité, sauvegarde, exploitabilité), §9.4 ; constitution (politique de versions) ; ADR-0003, ADR-0004.

## Tâches
1. **CI** (GitHub Actions ou GitLab CI, au choix documenté dans un ADR) avec détection des chemins
   modifiés :
   - backend : `./gradlew build` (tests + Testcontainers), JaCoCo (seuils de `CLAUDE.md` §8),
     SpotBugs/FindSecBugs, OWASP Dependency-Check, export OpenAPI et vérification de diff ;
   - web : `pnpm -r lint typecheck test build`, Playwright sur PR vers `main` ;
   - mobile : `melos run analyze test`, build APK `dev` ;
   - images Docker multi-stage (backend JRE 25 distroless, cache AOT Java 25 activé, web nginx non-root), scan Trivy,
     SBOM (CycloneDX), signature (cosign).
2. **CD** : Helm chart `infra/helm/cash-in-motion` (backend, web-backoffice, web-agency),
   environnements `recette`, `preprod`, `prod` ; déploiement auto en recette sur `main`,
   promotion manuelle approuvée vers preprod/prod ; migrations Flyway en job pré-déploiement ;
   rolling update, readiness/liveness probes, HPA sur CPU et latence.
3. **Secrets** : Vault (ou KMS du cloud choisi) + External Secrets Operator ; aucun secret en clair
   dans les manifests.
4. **Observabilité** : OpenTelemetry (traces backend → Tempo/Jaeger), Prometheus (métriques
   Micrometer : latence par endpoint, taux d'erreur, opérations par type, échecs ledger, profondeur
   de l'outbox, files RabbitMQ), Loki (logs JSON), tableaux Grafana (technique + métier : envois/min,
   float bas, alertes AML ouvertes), alertes (SLO 99,9 %, p95 > 2 s, outbox > 1 000, job
   d'intégrité ledger en échec). Recherche par référence : `correlationId` et `requestId` sont des
   champs JSON (**pas** des labels Loki, pour éviter l'explosion de cardinalité) ; tableau Grafana
   « Recherche par référence » (requête `{app="cim-backend"} | json | correlationId="<ref>"`) lié aux
   traces (Tempo) via `traceId` ; propagation `traceparent` côté ingress.
5. **Sauvegardes** : PostgreSQL PITR (WAL archiving, RPO ≤ 5 min), sauvegardes chiffrées,
   test de restauration mensuel automatisé (RTO ≤ 1 h), procédure dans `docs/runbooks/`.
6. **Runbooks** : incident ledger, file outbox bloquée, fournisseur SMS en panne, rotation de clés,
   restauration.
7. **Mobile** : Fastlane pour publication Play Store (track interne) et TestFlight.

## Critères d'acceptation
- Une PR qui casse un test ou introduit une vulnérabilité Critical est bloquée.
- Déploiement recette automatique < 15 min après merge.
- Tableau Grafana métier opérationnel en recette.

## Hors périmètre
Choix définitif de l'hébergeur (décision métier/juridique).
