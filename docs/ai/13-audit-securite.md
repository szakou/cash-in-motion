# 13 — Journal d'audit et durcissement sécurité (module `audit`)

## Objectif
Tracer de façon inaltérable toute action sensible et durcir l'API avant la recette.

## Prérequis
Tâches 1 à 4 (journal d'audit) : avec la fonctionnalité Spec Kit 02, juste après la 01.
Tâches 5 à 7 (durcissement sécurité) : après les fonctionnalités 09 à 13, avant la recette (21).

## Références
Cahier §8 (traçabilité, sécurité), §9.3, §7.1. `CLAUDE.md` §4 règles 8, 10.

## Tâches
1. Table `audit_log` (id, occurred_at, actor_id, actor_role, actor_ip, user_agent, action,
   target_type, target_id, before jsonb, after jsonb, reason, correlation_id, request_id, trace_id, prev_hash, hash) ;
   chaînage SHA-256 (`hash = sha256(prev_hash + contenu)`), triggers anti-UPDATE/DELETE, job de
   vérification de la chaîne, export quotidien vers stockage objet immuable (object lock).
2. Annotation `@Audited(action="PRICING_RULE_APPROVED")` + aspect ; couvrir : gestion des
   utilisateurs, paramétrage (référentiel, tarifs, taxes, change, plafonds), décisions KYC/AML,
   gel, annulations, clôtures de caisse avec écart, démasquage de données personnelles,
   renvoi de code, exports.
3. Démasquage motivé : `POST /api/v1/support/customers/{id}/reveal {field, reason}` → audit.
4. Consultation : `GET /api/v1/admin/audit?actor&action&target&from&to` (SUPER_ADMIN, conformité).
5. Durcissement : en-têtes de sécurité (HSTS, CSP stricte pour le Web, X-Content-Type-Options),
   CORS limité aux origines configurées, rate limiting Valkey (Bucket4j) par IP/utilisateur/endpoint
   (bucket4j), taille max des requêtes, validation stricte des entrées, désactivation des
   endpoints actuator sensibles en prod, rotation des clés (QR, données) documentée.
6. Analyse : OWASP Dependency-Check, SpotBugs + FindSecBugs dans le build ; corriger tout
   High/Critical.
7. Rédiger `docs/security/threat-model.md` (STRIDE sur : envoi, paiement par code, QR, P2P, admin).

## Critères d'acceptation
- Altérer une ligne d'audit en base est impossible (trigger) et détecté par la vérification.
- Toutes les actions listées produisent une entrée d'audit (test paramétré).
- Scan de dépendances sans vulnérabilité High/Critical.

## Tests exigés
Chaîne de hachage, couverture d'audit, rate limiting, en-têtes de sécurité.

## Hors périmètre
Test d'intrusion externe (21).
