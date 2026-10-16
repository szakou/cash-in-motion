# ADR-0003 — Stack backend : Java 25 et Spring Boot 4.1

- **Statut** : accepté
- **Date** : 2026-10-16
- **Remplace** : choix initial « Java 21 / Spring Boot 3.3+ » du cahier des charges

## Contexte

Cash In Motion manipule de l'argent réglementé dans plusieurs pays (UEMOA, CEMAC, CEEAC). Le
backend doit garantir l'exactitude des montants, des transactions sûres sous forte concurrence,
une sécurité et une traçabilité auditables, et des intégrations bancaires et télécoms (SMPP,
ISO 20022, rapports réglementaires). Le code est produit en grande partie par des agents IA
encadrés par Spec Kit.

## Décision

1. **Langage et framework** : Java + Spring Boot plutôt que Go/Echo, TypeScript/NestJS,
   Rust/Actix, Groovy/Grails, Ruby/Rails, PHP/Laravel ou Python/FastAPI, pour :
   - décimal exact natif (`BigDecimal`) intégré à la base et à la sérialisation ;
   - transactions, verrouillage et reprise éprouvés ;
   - briques prêtes : Spring Security (OIDC, MFA, RBAC), Spring Modulith (frontières de modules
     vérifiées), outbox, résilience, observabilité ;
   - bibliothèques de référence pour SMPP, ISO 20022, documents PDF (jSMPP, Prowide, Apache FOP) ;
   - support long terme, outillage de sécurité, familiarité des auditeurs et partenaires bancaires ;
   - vivier de développeurs dans la région ; typage strict qui cadre le code généré par IA.
2. **Versions** :
   - **Java 25** : LTS courante (GA 16/09/2025, support Oracle Premier jusqu'au 30/09/2030) ;
     Java 21 reste supportée mais plus ancienne, sans les améliorations des threads virtuels (Java 24).
   - **Spring Boot 4.1** (GA 30/06/2026, support OSS jusqu'au 31/07/2027) ; la branche 3.5 n'a plus
     de support OSS depuis le 30/06/2026 et 4.0 s'arrête le 31/12/2026.
   - Composants alignés : Spring Framework 7, Jakarta EE 11, Hibernate 7, Jackson 3, Spring
     Security 7, Spring Modulith 2.1, springdoc-openapi 3, Gradle 9.
   - Données et infrastructure : PostgreSQL 18 (fin de vie 11/2030), Valkey (voir ADR-0004), RabbitMQ
     4.3, Keycloak 26.7+.

## Conséquences

- Positives : support le plus long possible, correctifs de sécurité assurés, threads virtuels
  adaptés aux nombreux appels externes (SMS, KYC, partenaires).
- Négatives : les agents IA connaissent mieux Spring Boot 3 ; risque de code dans l'ancien style.
  Mesures : versions inscrites dans la constitution et `CLAUDE.md` (liste de pièges), catalogue de
  versions unique, compilation stricte et recettes OpenRewrite en CI.
- Certaines bibliothèques secondaires (SDK de fournisseurs SMS/KYC, Resilience4j)
  doivent être vérifiées pour Spring Boot 4 à l'étape `/speckit-plan` des fonctionnalités concernées.
- Montée de version mineure de Spring Boot tous les 6 mois (4.2 attendue fin 2026).

## Alternatives écartées

Kotlin + Spring Boot (même socle, acceptable si l'équipe le préfère, nouvel ADR requis) ; NestJS
(pas de décimal natif, écosystème plus fragmenté) ; Go (peu de briques prêtes, pas de décimal
natif) ; Rust (vitesse de développement, rareté des profils) ; Grails (typage dynamique, petite
communauté) ; Rails, Laravel, FastAPI (typage faible ou optionnel pour un cœur financier).

## Sources des versions (consultées le 2026-10-16)

- Spring Boot : https://endoflife.date/spring-boot et https://github.com/spring-projects/spring-boot/releases
- Spring Modulith : https://github.com/spring-projects/spring-modulith/releases
- Java : https://endoflife.date/oracle-jdk
- PostgreSQL : https://endoflife.date/postgresql
- RabbitMQ : https://endoflife.date/rabbitmq — Redis : https://endoflife.date/redis
- Keycloak : https://github.com/keycloak/keycloak/releases
- Paquets Web : registre npm (`npm view`)
