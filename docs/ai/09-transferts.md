# 09 — Transferts : envoi, paiement, annulation (module `transfer`)

## Objectif
Implémenter le cœur métier : envoi d'argent en agence avec code de retrait, paiement au
bénéficiaire, annulation et remboursement, avec intégration pricing, conformité, ledger et caisse.

## Prérequis
04 à 08 terminés. **Revue humaine obligatoire.**

## Références
Cahier §3.3, §4, §5.3 AGW-01 à AGW-05, §6.1 MAG-06, §9.2.

## Modèle de données
- `transfers` (id, code unique, status, quote_id, quote_snapshot jsonb, corridor_id,
  sender_customer_id, beneficiary_id, beneficiary_snapshot jsonb, message nullable,
  send_amount, send_currency, transfer_fee, message_fee, delivery_fee, discount, tta, taf,
  thzuemoa, total_tax, total_paid, fx_rate, payout_amount, payout_currency,
  sending_agency_id, sending_agent_id, sent_at, paying_agency_id, paying_agent_id, paid_at,
  payout_id_document jsonb (chiffré), secret_question_attempts, cancel_reason, cancelled_by,
  journal_entry_send_id, journal_entry_payout_id, version)
- `transfer_status_history` (id, transfer_id, from_status, to_status, actor_id, reason, at)

## Machine à états
```
CREATED → PENDING_COMPLIANCE → AVAILABLE → PAID
CREATED → AVAILABLE
PENDING_COMPLIANCE → REJECTED → REFUNDED        (contre-passation complète)
REJECTED → FROZEN                                (sanction confirmée : SYS_FROZEN_FUNDS)
FROZEN → REFUNDED                                (dégel décidé par la conformité)
FROZEN → SURRENDERED                             (fonds remis à l'autorité sur décision, avec référence)
AVAILABLE → CANCELLED (même jour, avant paiement) → REFUNDED
AVAILABLE → BLOCKED (conformité) → AVAILABLE
AVAILABLE → EXPIRED (après N jours, paramétrable) → REFUNDED
```

## Tâches
1. **Code de retrait** : 10 chiffres aléatoires (SecureRandom) + chiffre de contrôle Luhn, unique,
   non séquentiel ; affiché groupé `123 456 789 0`.
2. **Envoi** `POST /api/v1/transfers` (`@Idempotent`, rôle `AGENCY_AGENT`) : vérifie caisse ouverte,
   agence active, plafond agence, float suffisant pour le total, consomme le devis (`PricingApi.consumeQuote`), pré-contrôle
   conformité, puis en **une transaction** : création du transfert, écriture ledger (recette
   « envoi »), historique, outbox `TransferSent`. Si `HOLD` → statut `PENDING_COMPLIANCE`, écriture
   passée quand même (fonds en transit), paiement impossible tant que non libéré.
3. **Consultation par code** `GET /api/v1/transfers/by-code/{code}` (agent payeur) : retourne
   bénéficiaire, montant à payer, présence d'une question test — jamais les données expéditeur
   complètes. Limitation de débit stricte (10/min/agent) contre l'énumération.
4. **Paiement** `POST /api/v1/transfers/{code}/payout` (`@Idempotent`) : statut `AVAILABLE`, devise
   de paiement = devise de l'agence, pièce du bénéficiaire saisie et contrôlée (nom
   correspondant au bénéficiaire, tolérance de similarité), réponse secrète si définie (3 essais
   puis `BLOCKED`), pré-contrôle conformité, caisse ouverte ; écriture « paiement » (débit
   `SYS_TRANSFERS_IN_TRANSIT`, crédit float de l'agence payeuse), mouvement de caisse `OUT` ;
   outbox `TransferPaid`.
5. **Annulation** (superviseur, même jour, avant paiement) et **remboursement** : contre-passation
   de l'envoi ; frais remboursés ou non selon paramètre `refund_fees_on_cancel` ; taxes
   contre-passées.
6. **Expiration** : job quotidien → `EXPIRED`, remboursement à la demande de l'expéditeur.
7. **Reçus** : PDF générés avec Apache FOP (gabarits XSL-FO, polices embarquées) pour l'envoi et le paiement,
   format A4 et ticket 58/80 mm,
   avec toutes les lignes du cahier §3.3, QR de vérification ; `GET /api/v1/transfers/{id}/receipt`.
8. **Recherche support** `GET /api/v1/support/transfers?code&phone&name&agencyId&from&to&amount&reference`
   (`reference` = `correlationId` affiché à l'utilisateur, recherché sur `transfer_status_history`,
   `wallet_operations`, `idempotency_keys` et `audit_log`)
   (rôle `SUPPORT_AGENT`, données masquées) + `GET /api/v1/support/transfers/{id}/timeline`.
9. Réception directe en wallet (MCL-06) : si le bénéficiaire a un wallet et que l'option est
   choisie, paiement automatique sur le wallet à la libération (crédit wallet au lieu de caisse).

## Règles métier
- Montant payé = `payout_amount` du snapshot, jamais recalculé.
- L'agent payeur ne peut pas être l'agent émetteur pour un même transfert (paramétrable).
- Toute transition est historisée.

## Critères d'acceptation
- Scénario complet SN→ML de 100 000 XOF : reçu à 104 244 XOF, paiement de 100 000 XOF au Mali,
  ledger équilibré via le compte de liaison SN↔ML (XOF), comptes de taxes du Sénégal crédités
  (500 / 544). Même scénario en national SN→SN via `SYS_TRANSFERS_IN_TRANSIT`.
- Deux paiements simultanés du même code : un seul réussit.
- Réponse secrète fausse 3 fois → `BLOCKED`.

## Tests exigés
Intégration bout en bout (Testcontainers), concurrence, machine à états (toutes transitions
interdites), annulation/remboursement, reçus générés, limitation de débit.

## Hors périmètre
Partenaires payeurs étrangers par API (V2) — prévoir l'interface `PayoutPartner` sans implémentation.


## Mise à jour multi-pays (2026-10-16) — prévaut sur les sections ci-dessus

- Ajouter `sending_country_iso2`, `paying_country_iso2` ; devise de paiement = devise du corridor.
- Taxes : celles du pays émetteur à l'envoi, celles du pays payeur au paiement (prompt 05).
- Déblocage d'un transfert `BLOCKED` : rôle `COMPLIANCE_OFFICER` uniquement (le support en fait la
  demande). Compte système `SYS_FROZEN_FUNDS` par pays/devise pour l'état `FROZEN`.
- **Recette inter-pays équilibrée par devise** — exemple SN → CM, 100 000 XOF envoyés, taux figé
  0,99 → 99 000 XAF payés :

  | Moment | Compte | Débit | Crédit |
  | --- | --- | --- | --- |
  | Envoi (SN, XOF) | AGENCY_FLOAT agence SN | total payé | |
  | | SYS_FEE_REVENUE / SYS_TAX_*_PAYABLE (SN) | | frais / taxes |
  | | SYS_INTERCOUNTRY_SETTLEMENT SN↔CM (XOF) | | 100 000 |
  | Paiement (CM, XAF) | SYS_INTERCOUNTRY_SETTLEMENT SN↔CM (XAF) | 99 000 | |
  | | AGENCY_FLOAT agence CM | | 99 000 |

  Chaque écriture est équilibrée dans sa devise. Le compte de liaison XOF (dette du Sénégal) et le
  compte de liaison XAF (créance du Cameroun) sont rapprochés au taux figé dans l'état de
  compensation ; la marge de change est constatée en `SYS_FX_GAIN_LOSS` lors de la compensation.
  Pour un transfert national (SN → SN), le compte `SYS_TRANSFERS_IN_TRANSIT` reste utilisé comme
  décrit plus haut.
