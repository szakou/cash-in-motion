# 08 — Agences, caisses, float et commissions (module `agency`)

## Objectif
Gérer le réseau d'agences, leur liquidité (float électronique et caisse espèces) et les
commissions.

## Prérequis
02, 04 terminés.

## Références
Cahier §5.1 ADM-01, ADM-07 ; §5.3 AGW-06, AGW-08, AGW-09 ; §6.1 MAG-04.

## Modèle de données
- `agencies` (id, code unique `AG-0001`, name, type `OWNED|SUB_AGENT`, address, city_id,
  country_iso2, phone, opening_hours jsonb, status `ACTIVE|SUSPENDED|CLOSED`, daily_send_limit,
  daily_payout_limit, float_account_id, commission_plan_id, geo_lat, geo_lng)
- `tills` (id, agency_id, agent_user_id, currency, status `OPEN|CLOSED|PENDING_VALIDATION`,
  opened_at, closed_at, opening_balance, declared_closing_balance, expected_closing_balance,
  variance, validated_by)
- `till_movements` (id, till_id, direction `IN|OUT`, amount, operation_type, operation_id, at) —
  livre de caisse des espèces physiques, hors grand livre (voir modèle de liquidité, prompt 04)
- `float_requests` (id, agency_id, amount, direction `TOP_UP|WITHDRAW`, status `REQUESTED|APPROVED|
  REJECTED|SETTLED`, requested_by, approved_by, bank_reference, created_at)
- `commission_plans` (id, name, rules jsonb : part des frais par type d'opération, ex. 40 %)
- `commissions` (id, agency_id, agent_user_id, operation_type, operation_id, amount, status
  `ACCRUED|PAID`, period)

## Tâches
1. CRUD agences (admin) ; création → ouverture automatique du compte `AGENCY_FLOAT` dans le ledger.
2. Caisse par agent : ouverture (solde d'ouverture), chaque opération espèces ajoute un
   `till_movement` (envoi et cash-in = `IN`, paiement et cash-out = `OUT`), clôture avec solde
   déclaré → écart calculé, écart ≠ 0 → `PENDING_VALIDATION` par le superviseur.
3. Float : envoi de transfert et cash-in **débitent** le float de l'agence ; paiement et cash-out le
   **créditent** (tableau du prompt 04). Refus `INSUFFICIENT_FLOAT` si le débit rendrait le float
   négatif. Événement `FloatLow` sous un seuil configurable par agence.
4. Demandes de réapprovisionnement : agence demande → admin approuve avec référence bancaire →
   écriture `SYS_SAFEGUARD_BANK`/`AGENCY_FLOAT`.
5. Commissions : écouter `TransferSent`, `TransferPaid`, `WalletOperationCompleted` et calculer la
   commission selon le plan ; écriture `SYS_FEE_REVENUE` → `SYS_AGENCY_COMMISSION_PAYABLE`.
6. Façade `AgencyApi` : `assertAgencyActive`, `floatAccountOf`, `openTillOf(agentId)`,
   `checkDailyLimit`.
7. Recherche d'agences proches (pour l'app client) : `GET /api/v1/public/agencies/nearby?lat&lng&radiusKm`.

## API
- `GET|POST|PATCH /api/v1/admin/agencies`
- `POST /api/v1/agencies/{id}/tills/open`, `POST /api/v1/tills/{id}/close`, `GET /api/v1/tills/{id}`
- `GET /api/v1/agencies/{id}/float`, `POST /api/v1/agencies/{id}/float-requests`,
  `POST /api/v1/admin/float-requests/{id}/approve|reject`
- `GET /api/v1/agencies/{id}/commissions?period=`

## Critères d'acceptation
- Un agent sans caisse ouverte ne peut faire aucune opération espèces (`TILL_NOT_OPEN`).
- Agence suspendue → toutes ses opérations refusées.
- Clôture avec écart → statut en attente de validation.

## Tests exigés
Cycle caisse complet, float insuffisant, plafond journalier, calcul de commission, droits par rôle.

## Hors périmètre
Écrans (15, 16, 18).


## Mise à jour multi-pays (2026-10-16) — prévaut sur les sections ci-dessus

- `agencies` : `country_iso2` (pays `OPERATING` obligatoire), `timezone` (défaut du pays),
  devise du float et des caisses = devise du pays.
- Plafonds journaliers et « caisse de la veille » calculés dans `agencies.timezone`.
- Approbation des demandes de float limitée aux administrateurs ayant le pays dans leur périmètre.
