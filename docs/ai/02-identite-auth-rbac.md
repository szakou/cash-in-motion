# 02 — Identité, authentification, RBAC et périmètre

## Objectif
Sécuriser l'API par Keycloak (OIDC/JWT), gérer les utilisateurs internes et appliquer RBAC +
périmètre d'agence côté serveur.

## Prérequis
01 terminé.

## Références
Cahier §2 (acteurs), §5.1 ADM-02, §9.3. `CLAUDE.md` §4 règle 9, §6.

## Tâches
1. Spring Security resource server : validation JWT Keycloak (issuer `cim`), extraction des rôles
   realm → `GrantedAuthority` `ROLE_*`. Endpoints publics : `/actuator/health`, `/swagger-ui/**`,
   `/api/v1/public/**` (inscription client, OTP).
2. `CurrentUser` (id, keycloakId, rôles, agencyId nullable, type `INTERNAL|CUSTOMER`) injectable.
3. Module `identity` : entité `internal_users` (id, keycloak_id, full_name, email, phone, role,
   agency_id, status `ACTIVE|SUSPENDED|CLOSED`, mfa_enabled, last_login_at).
   Service `UserAdminService` qui crée/suspend l'utilisateur **dans Keycloak** (Admin API via client
   `cim-admin-cli`) et localement, de façon cohérente (compensation si l'un échoue).
4. Règles : `ADMIN` ne peut pas créer de `SUPER_ADMIN` ; `AGENCY_SUPERVISOR` ne gère que les agents
   de son agence ; tout utilisateur `AGENCY_*` doit avoir une `agency_id`.
5. Périmètre : composant `ScopeGuard.assertCanAccessAgency(agencyId)` + filtre de requête
   réutilisable pour restreindre les listes à l'agence de l'utilisateur.
6. MFA obligatoire pour les rôles internes : vérifier la claim `acr`/`amr` ; refuser (403
   `MFA_REQUIRED`) sinon. Configurer la politique OTP dans le realm.
7. *(À réaliser avec la fonctionnalité Spec Kit 11 « Wallet », pas avec la 01.)* Authentification mobile client : flux OTP SMS + PIN (le PIN est un facteur local vérifié par le
   backend) — **définir ici** l'API, implémentation SMS au prompt 11 via interface `OtpSender`.
   Stockage OTP dans Valkey (TTL 5 min, 3 essais, anti-rejeu).
8. *(Fonctionnalité 11, et 13 pour l'app agence.)* Enrôlement d'appareil : table `devices` (id, owner_type, owner_id, device_fingerprint,
   public_key, platform, status, enrolled_at, last_seen_at). Les apps mobiles signent les requêtes
   financières (en-tête `X-Device-Signature`, Ed25519) — vérifier côté serveur.
9. Verrouillage : 3 PIN erronés → compte bloqué 30 min (configurable), alerte.

## API
- `GET /api/v1/me`
- `GET|POST /api/v1/admin/users`, `PATCH /api/v1/admin/users/{id}` (suspendre, réactiver, rôle)
- `POST /api/v1/public/customers/otp/request`, `POST /api/v1/public/customers/otp/verify`
- `POST /api/v1/customers/me/pin` (création/changement), `POST /api/v1/devices/enroll`

## Critères d'acceptation
- Un agent de l'agence A reçoit 403 sur toute ressource de l'agence B.
- Un utilisateur interne sans MFA reçoit 403 `MFA_REQUIRED`.
- Matrice rôle × endpoint documentée dans `docs/security/rbac-matrix.md` et testée.

## Tests exigés
Tests paramétrés de la matrice RBAC (JWT de test signés localement), tests OTP (expiration,
essais, rejeu), test de verrouillage PIN, test de signature d'appareil.

## Hors périmètre
Écrans de connexion (14, 17).


## Mise à jour multi-pays (2026-10-16) — prévaut sur les sections ci-dessus

- `internal_users` : ajouter `country_scope` (`GLOBAL` ou liste ISO2 via table `user_country_scopes`)
  pour les rôles du siège ; un `ADMIN` pays ne gère que ses pays ; seul un `ADMIN` global gère les
  administrateurs pays.
- `ScopeGuard` : ajouter `assertCanAccessCountry(iso2)` et filtres de liste par pays, combinés au
  périmètre agence.
- OTP et inscription client : refuser les numéros des pays non `OPERATING`.
