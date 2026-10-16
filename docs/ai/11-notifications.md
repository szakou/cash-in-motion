# 11 — Notifications et routage SMS par pays (module `notification`)

> Notes techniques pour `/speckit-plan` de la fonctionnalité `speckit/14-notifications.md`.

## Objectif
Acheminer SMS, push et e-mails de façon fiable et asynchrone, avec un **fournisseur SMS configurable
par pays** (choisi d'après l'indicatif du destinataire), un fournisseur de secours, une route
internationale par défaut, des tests d'envoi et une supervision par pays et par fournisseur.

## Prérequis
01 (outbox), 02 (identité, périmètre pays), 03 (pays, indicatifs, locales). Consomme les
événements de 06, 09, 10.

## Modèle de données
- `sms_providers` (id, name, connector_type, credentials_ref (référence Vault/KMS — jamais le
  secret en base), supports_delivery_reports, supports_balance, status `ACTIVE|DISABLED`,
  created_at, updated_by)
- `sms_provider_costs` (provider_id, country_iso2, unit_cost, currency, valid_from)
- `sms_routes` (id, country_iso2 unique ou `*` pour la route internationale, primary_provider_id,
  secondary_provider_id nullable, sender_id, fallback_sender_id, status `TESTING|ACTIVE|SUSPENDED`,
  max_attempts_before_failover, critical_delivery_timeout_s (défaut 120), auto_failover_error_rate
  (ex. 0.2), auto_failover_window_s, failover_state `PRIMARY|SECONDARY_AUTO|SECONDARY_MANUAL`,
  last_successful_test_at, default_locale)
- `sms_route_tests` (id, route_id, provider_id, msisdn (chiffré), status, provider_message_id,
  error, requested_by, at)
- `notification_templates` (code, channel `SMS|PUSH|EMAIL`, locale, message_class
  `CRITICAL|TRANSACTIONAL|INFO`, subject, body (Mustache), active) — unicité (code, channel, locale)
- `notifications` (id, event_id, template_code, channel, message_class, recipient (chiffré),
  recipient_country_iso2, locale, rendered_length, segments, route_id, status `QUEUED|SENT|ACCEPTED|
  DELIVERED|FAILED|UNROUTABLE`, last_error, created_at)
- `notification_attempts` (id, notification_id, provider_id, attempt_no, is_failover, status,
  provider_message_id, error_code, cost, sent_at, delivered_at)
- `push_tokens` (id, owner_type, owner_id, device_id, fcm_token, platform, updated_at)

## Architecture du routage SMS
1. **Résolution du pays** : `libphonenumber` sur le numéro E.164 → `recipient_country_iso2`.
2. **Choix de la route** : `sms_routes[country]` si `ACTIVE`, sinon route `*`, sinon `UNROUTABLE`
   (+ alerte `SmsUnroutable`).
3. **Choix du fournisseur** : selon `failover_state` → principal ou secours.
4. **Connecteurs** : interface `SmsConnector { send(msisdn, senderId, text, encoding) ; parseDeliveryReport(...) ; balance() }`
   avec un registre par `connector_type`. Implémentations V1 : `GENERIC_HTTP` (URL, méthode,
   gabarit de corps, en-têtes, mapping des statuts — configurable sans code), `SMPP` (via
   jSMPP) et un ou deux agrégateurs nommés selon les choix métier ; `LOGGING` pour le dev.
5. **Bascule par message** : erreur ou timeout → retry (backoff) jusqu'à `max_attempts_before_failover`,
   puis secours. Pour `CRITICAL` : si `ACCEPTED` sans `DELIVERED` après `critical_delivery_timeout_s`,
   renvoi par le secours (job planifié).
6. **Bascule de route (disjoncteur)** : taux d'échec du principal sur `auto_failover_window_s`
   > seuil → `SECONDARY_AUTO` + alerte ; sondes périodiques pour revenir en `PRIMARY`.
   Implémentation : Resilience4j CircuitBreaker par (route, fournisseur) si son module est compatible
   Spring Boot 4, sinon compteurs Valkey ; retries via `@Retryable` de Spring Framework 7.
7. **Accusés de remise** : endpoint webhook par fournisseur
   `POST /api/v1/webhooks/sms/{providerId}/dlr` (signature/secret partagé vérifiés), mise à jour
   idempotente des tentatives.
8. **Encodage** : détection GSM-7 vs UCS-2, calcul des segments ; alerte si un modèle dépasse
   1 segment pour la classe `CRITICAL`.
9. **Débit** : limitation par fournisseur (bucket Valkey via Bucket4j) selon le contrat.
10. **Secrets** : identifiants via Vault/KMS (`credentials_ref`), rotation sans redéploiement ;
    jamais logués ; le formulaire admin est en écriture seule.

## Consommateurs d'événements
`OtpRequested` (CRITICAL), `TransferSent` (CRITICAL pour le bénéficiaire, TRANSACTIONAL pour
l'expéditeur), `TransferPaid`, `TransferCancelled`, `TransferRefunded`, `WalletOperationCompleted`,
`KycDecisionMade`, `FloatLow`, `AmlAlertRaised` (e-mail au responsable conformité du pays),
`SmsUnroutable`, `SmsRouteFailover`, `SmsProviderBalanceLow`. Files RabbitMQ par canal, DLQ,
idempotence par (event_id, recipient, channel) sauf renvoi de secours.

## API
- `GET|POST|PATCH /api/v1/admin/sms/providers` (secrets en écriture seule)
- `GET|POST|PATCH /api/v1/admin/sms/routes` ; `POST /api/v1/admin/sms/routes/{id}/test` ;
  `POST /api/v1/admin/sms/routes/{id}/activate|suspend|failover|restore`
- `GET /api/v1/admin/sms/stats?country&provider&from&to` (volumes, taux d'acceptation/remise,
  délais, bascules, coûts)
- `GET|POST|PATCH /api/v1/admin/notification-templates`
- `POST /api/v1/support/transfers/{id}/resend-code {target, reason, forceSecondary}` (max 3 / 24 h)
- `POST /api/v1/push-tokens`
- Périmètre : un `ADMIN` pays ne modifie que les routes de ses pays (à confirmer en clarification).

## Écrans (Back-office, prompt 15)
Fournisseurs, routes par pays (carte ou tableau avec état, principal/secours, dernier test),
assistant de test d'envoi, tableau de bord qualité par pays/fournisseur, modèles par langue avec
compteur de caractères et segments.

## Règles et critères d'acceptation
- Un SMS vers +237 depuis une opération au Sénégal passe par la route du Cameroun.
- Route jamais testée avec succès → activation refusée ; pays sans route active → checklist
  d'ouverture du pays (prompt 03) rouge.
- Panne simulée du principal (WireMock) → messages critiques remis via secours, bascule de route
  automatique, retour automatique après rétablissement.
- Un échec fournisseur n'affecte jamais la transaction métier ; un événement rejoué n'envoie
  qu'un message (hors renvoi de secours CRITICAL).
- Aucun secret fournisseur ni numéro en clair dans les logs.

## Tests exigés
Résolution pays/route (table de numéros de tous les pays de la région), bascule par message et
par route (disjoncteur), DLR idempotents, encodage/segments, rendu multi-locale, connecteur
GENERIC_HTTP configurable, droits par périmètre pays.

## Hors périmètre
Webhooks partenaires, campagnes marketing, WhatsApp/voix (prévoir l'ajout de canaux).

## Corrélation (2026-10-16)
Chaque consommateur restaure le `correlationId` reçu dans l'en-tête `x-correlation-id` ; il est
stocké sur `notifications` et `notification_attempts`, transmis aux fournisseurs qui l'acceptent
(référence client du message) et présent sur toutes les lignes de journal d'envoi, de bascule et
d'accusé de remise. Les accusés de remise (webhooks) retrouvent le `correlationId` d'origine par
`provider_message_id`.
