# 10 — Wallet, P2P et cash-in/cash-out par QR Code (module `wallet`)

## Objectif
Offrir le portefeuille électronique client : solde, historique, transfert P2P, dépôt et retrait en
agence par QR Code sécurisé.

## Prérequis
04 à 08 terminés. **Revue humaine obligatoire.**

## Références
Cahier §6.1 (MAG-02 à MAG-05), §6.2 (MCL-03 à MCL-07), §6.3 (séquence QR), §9.3 (QR).

## Modèle de données
- `wallets` (id, customer_id unique, currency XOF, ledger_account_id, status `ACTIVE|FROZEN|CLOSED`,
  created_at)
- `wallet_operations` (id, type `CASH_IN|CASH_OUT|P2P_OUT|P2P_IN|TRANSFER_IN`, status
  `INITIATED|AWAITING_CONFIRMATION|PENDING_COMPLIANCE|COMPLETED|FAILED|EXPIRED|CANCELLED`,
  wallet_id, counterparty_wallet_id, agency_id, agent_user_id, amount, fee, taxes jsonb, quote_id,
  qr_token_id, journal_entry_id, failure_reason, created_at, completed_at)
- `qr_tokens` (id, token_hash, purpose `CASH_IN|CASH_OUT`, issuer_type `CUSTOMER|AGENT`, issuer_id,
  amount, currency, expires_at, consumed_at, scanned_by, status)

## Tâches
1. Création automatique du wallet à l'atteinte du KYC L1 (compte ledger `CUSTOMER_WALLET`).
2. `GET /api/v1/wallets/me` (solde disponible, bloqué, plafonds restants),
   `GET /api/v1/wallets/me/operations?cursor`.
3. **P2P** `POST /api/v1/wallets/p2p` (`@Idempotent`, signature appareil, PIN) : destinataire par
   téléphone ; si inexistant → `RECIPIENT_NOT_FOUND` (V1, pas d'invitation) ; devis via
   `PricingApi` (type `WALLET_P2P`), pré-contrôle conformité, écriture ledger, notifications.
   `GET /api/v1/wallets/p2p/preview?phone=` renvoie le nom masqué du destinataire (« Awa D*** »).
4. **QR Code** : format de charge utile `cim://qr/v1/{tokenId}.{signature}` ; signature HMAC-SHA256
   (clé dans Vault) ; jeton aléatoire 128 bits, stocké haché ; usage unique ; TTL 5 min ;
   ne contient ni montant modifiable ni donnée personnelle.
5. **Cash-out** (séquence cahier §6.3) :
   1. Client `POST /api/v1/qr/cash-out {amount}` → `hold` du montant + frais sur son wallet, jeton.
   2. Agent `POST /api/v1/qr/{token}/scan` → vérif signature/expiration/statut, retourne nom,
      montant, frais, exigence de pièce (au-dessus d'un seuil) ; statut `AWAITING_CONFIRMATION`.
   3. Agent `POST /api/v1/qr/{token}/accept` (pièce saisie si exigée).
   4. Client `POST /api/v1/qr/{token}/confirm` (PIN) → écriture : débit wallet (montant + frais +
      taxes), crédit float agence (montant), crédit frais/taxes ; release du hold ; mouvement de
      caisse `OUT` pour les espèces remises.
   5. Notifications temps réel aux deux apps (SSE ou WebSocket `GET /api/v1/stream`).
   Expiration ou refus → release du hold.
6. **Cash-in** : l'agent génère le QR (`POST /api/v1/qr/cash-in {amount}`), le client scanne et
   confirme par PIN → débit float agence (`INSUFFICIENT_FLOAT` sinon), crédit wallet, mouvement de
   caisse `IN` pour les espèces reçues.
   Variante : le client présente son QR « profil » statique signé et l'agent saisit le montant.
7. Plafonds KYC et float vérifiés à chaque étape ; opérations idempotentes ; recettes ledger du
   prompt 04.
8. Réception de transfert international sur wallet (événement de 09).

## Critères d'acceptation
- Un jeton QR ne peut être consommé qu'une fois, même avec 2 agents qui scannent simultanément.
- QR expiré → `QR_EXPIRED`, hold libéré.
- Wallet gelé → toute opération refusée sauf consultation.
- Σ wallets clients = Σ comptes `CUSTOMER_WALLET` du ledger.

## Tests exigés
Séquences cash-in/cash-out complètes, P2P (solde insuffisant, plafond, gelé), concurrence sur
jeton, expiration, signature falsifiée, idempotence.

## Hors périmètre
Mobile Money externe (V2), cartes.


## Mise à jour multi-pays (2026-10-16) — prévaut sur les sections ci-dessus

- `wallets.currency` = devise du pays du wallet (`wallets.country_iso2`, pays de l'indicatif à
  l'inscription) ; plafonds, taxes et fuseau de ce pays.
- Dépôt et retrait uniquement dans une agence du pays du wallet ; P2P limité au même pays en V1.
- Cash-out : le `hold` porte sur le **total du devis** (montant + frais + taxes), pas seulement
  montant + frais.
