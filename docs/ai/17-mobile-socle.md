# 17 — Socle mobile Flutter (`mobile/packages/core`)

## Objectif
Fournir le socle partagé des deux apps mobiles : réseau, authentification, sécurité de
l'appareil, QR, design system, i18n, stockage, gestion hors ligne légère.

## Prérequis
02 terminé ; `api/openapi.yaml` exporté.

## Références
`CLAUDE.md` §2, §5 (Mobile). Cahier §6, §8 (réseau 3G, compatibilité), §9.3 (mobile).

## Tâches
1. Client API : génération depuis OpenAPI (`openapi_generator` dio) ou modèles `freezed` écrits
   à la main si la génération est insatisfaisante ; intercepteurs : jeton, `Idempotency-Key`,
   `X-Device-Signature` (Ed25519 via `cryptography`), `X-Client-App` (`mobile-agent` ou
   `mobile-client`), `X-Correlation-ID` (un par action utilisateur, conservé pour ses retries ;
   affiché comme « Référence » dans `ErrorView`, envoyé en tag Sentry), retry exponentiel **uniquement** sur
   requêtes idempotentes, timeouts adaptés 3G, mapping RFC 7807 → `AppException`.
2. Auth : OIDC PKCE (`flutter_appauth`) pour l'agent ; OTP + PIN pour le client ; jetons dans
   `flutter_secure_storage` ; verrouillage auto après 2 min d'inactivité ; déverrouillage PIN ou
   biométrie (`local_auth`).
3. Enrôlement d'appareil : génération de paire de clés au premier lancement, stockage sécurisé,
   appel `POST /devices/enroll`.
4. Sécurité : certificate pinning (dio `badCertificateCallback` + empreintes configurables),
   détection root/jailbreak (`flutter_jailbreak_detection` ou équivalent) avec blocage des
   opérations financières, `FLAG_SECURE` sur les écrans sensibles, obfuscation au build
   (`--obfuscate --split-debug-info`), attestation (Play Integrity / App Attest) — interface prête.
5. QR : widget scanner (`mobile_scanner`) avec cadre de visée, torche, retour haptique, parsing
   `cim://qr/v1/...` ; widget d'affichage (`qr_flutter`) avec compte à rebours d'expiration et
   luminosité maximale.
6. Design system : thème Material 3 aux couleurs de la marque, composants `MoneyText`,
   `AmountKeypad`, `PinPad`, `PrimaryButton` (état chargement, anti double-tap), `StatusChip`,
   `ReceiptCard`, `EmptyView`, `ErrorView` avec réessai.
7. i18n `flutter_localizations` + ARB `fr` (wolof et anglais prévus), formatage montants XOF
   (`100 000 FCFA`) et dates dans le fuseau du pays.
8. Navigation `go_router` avec gardes d'authentification ; état Riverpod ; architecture
   feature-first.
9. Connectivité : bannière hors ligne, file de réessai pour les lectures, **aucune opération
   financière mise en file hors ligne**.
10. Notifications push : `firebase_messaging`, enregistrement du jeton (`POST /push-tokens`),
    ouverture contextuelle de l'écran concerné.
11. Flavors `dev`, `staging`, `prod` (URL API, issuer Keycloak, empreintes TLS).
12. Sentry Flutter, journalisation sans données personnelles.

## Critères d'acceptation
- `melos run analyze test` passe ; les deux apps démarrent en flavor `dev` sur Android 8+ et iOS 15+.
- Une requête financière sans signature d'appareil valide est rejetée par le backend.

## Tests exigés
Unitaires des intercepteurs, du parsing QR, du formatage ; tests de widgets PinPad, AmountKeypad.

## Hors périmètre
Écrans métier (18, 19).


## Mise à jour multi-pays (2026-10-16) — prévaut sur les sections ci-dessus

- i18n : ARB `fr` + `en`, `pt`, `es` prêts ; formatage montants selon la devise du pays
  (XOF/XAF « FCFA », CDF « FC »…) ; dates dans le fuseau du pays.
