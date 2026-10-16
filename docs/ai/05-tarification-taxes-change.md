# 05 — Tarification, taxes et change (module `pricing`)

## Objectif
Calculer de façon déterministe et versionnée le devis complet d'une opération : frais, taxes,
rabais, taux de change, total et montant à payer.

## Prérequis
03 terminé. **Revue humaine obligatoire.**

## Références
Cahier §4 (règles de calcul, formules, exemple), §5.1 ADM-04 à ADM-06. `CLAUDE.md` §4 règle 5, §7.

## Modèle de données (toutes versionnées : `valid_from`, `valid_to` nullable, `created_by`)
- `fee_grids` (id, corridor_id nullable, operation_type `TRANSFER_SEND|TRANSFER_PAYOUT|WALLET_P2P|
  WALLET_CASH_IN|WALLET_CASH_OUT`, currency, status `DRAFT|PENDING_APPROVAL|ACTIVE|RETIRED`)
- `fee_tiers` (id, fee_grid_id, min_amount, max_amount, fixed_fee, percent_fee, min_fee, max_fee)
- `ancillary_fees` (code `MESSAGE|HOME_DELIVERY`, currency, amount, valid_from, valid_to)
- `tax_rules` (id, code `TTA|TAF|THZUEMOA`, base `SENT_AMOUNT|NET_FEES`, rate, cap_amount nullable,
  cap_currency, applies_to jsonb (types d'opération), condition `ALWAYS|CROSS_UEMOA_ONLY`,
  payer `SENDER|RECEIVER`, valid_from, valid_to, status)
- `fx_rates` (id, base_currency, quote_currency, reference_rate, margin_percent, source
  `MANUAL|FIXED_PEG|PROVIDER`, valid_from, valid_to) — seed EUR/XOF = 655.957 `FIXED_PEG`
- `promo_codes` (code, discount_type `PERCENT_OF_FEES|FIXED`, value, max_uses, used, valid_from,
  valid_to) — V2, créer la table seulement
- `quotes` (id, payload jsonb, created_at, expires_at, used bool) — snapshot du devis

## Tâches
1. `QuoteEngine.quote(QuoteRequest)` → `Quote` immuable. Entrées : type d'opération, corridor,
   montant envoyé **ou** montant à recevoir (calcul inverse), options (message, livraison), rabais.
   Sortie : toutes les lignes du cahier §3.3 + identifiants des règles appliquées + `expiresAt`.
2. Algorithme (respecter l'ordre) :
   1. Frais de transfert = palier contenant le montant : `fixed + percent × montant`, borné min/max.
   2. Frais annexes (message, livraison) si demandés.
   3. Rabais ≤ total des frais.
   4. Pour chaque `tax_rule` active applicable (type, condition UEMOA) : base × taux, plafonné.
   5. Arrondi par ligne selon la devise (XOF : unité, HALF_UP).
   6. Total = montant + frais − rabais + taxes payées par l'expéditeur.
   7. Montant à payer = montant × (taux de référence × (1 − marge)) arrondi devise de réception,
      moins taxes à la charge du bénéficiaire le cas échéant.
3. Calcul inverse (montant à recevoir donné) par recherche itérative bornée, résultat cohérent au
   franc près.
4. Le devis est persisté dans `quotes` (TTL configurable, 15 min par défaut) ; les transferts
   référencent `quoteId` et réutilisent **exactement** ses valeurs.
5. Workflow quatre yeux : création → `PENDING_APPROVAL` → approbation par un autre `ADMIN`
   → `ACTIVE`. Chevauchement de périodes interdit pour une même clé.
6. Façade `PricingApi.quote(...)`, `PricingApi.getQuote(id)`, `PricingApi.consumeQuote(id)`.

## API
- `POST /api/v1/quotes`
- `GET|POST|PATCH /api/v1/admin/pricing/{fee-grids|tax-rules|fx-rates|ancillary-fees}`
- `POST /api/v1/admin/pricing/{type}/{id}/approve`

## Critères d'acceptation (cas de test obligatoires)
| Cas | Attendu |
| --- | --- |
| SN→ML, 100 000 XOF, frais 3 000, message 200 | TTA 500, TAF 544, THZUEMOA 0, total 104 244, à payer 100 000 XOF |
| SN→ML, 1 000 000 XOF | TTA plafonnée à 2 000 |
| SN→FR, 655 957 XOF | THZUEMOA appliquée, à payer ≈ 1 000 EUR − marge |
| Rabais > frais | rabais ramené aux frais |
| Règle modifiée après devis | devis existant inchangé |
| Devis expiré | `QUOTE_EXPIRED` |

## Tests exigés
Unitaires exhaustifs (paliers, bornes, arrondis, plafonds, UEMOA/hors UEMOA, calcul inverse),
tests de propriétés (total ≥ montant, taxes ≥ 0), intégration du workflow d'approbation.

## Hors périmètre
Écrans d'administration (15).


## Mise à jour multi-pays (2026-10-16) — prévaut sur les sections ci-dessus

- `tax_rules` : ajouter `country_iso2` (pays dont la taxe relève), `applies_at` (`SEND|PAYOUT|WALLET`),
  remplacer `condition` par `zone_condition` (`ALWAYS|INTRA_ZONE|INTER_ZONE|OUT_OF_REGION|OUTSIDE_OWN_ZONE`).
- Moteur de devis : taxes d'envoi = règles du pays de l'agence/wallet émetteur ; taxes de paiement =
  règles du pays payeur, supportées selon `payer` (déduites du montant payé si `RECEIVER`).
- Un pays `OPERATING` doit avoir au moins une règle `ACTIVE` ou un drapeau `no_tax_confirmed`.
- `fx_rates` : seed XOF/EUR et XAF/EUR `FIXED_PEG` 655.957 ; XOF↔XAF via taux `MANUAL` avec marge.
- Ajouter le cas de test SN → CM (XOF → XAF) avec THZUEMOA.
- Taux de change : `fx_rates` = taux de **référence par paire de devises** (retirer
  `margin_percent` de cette table) ; nouvelle table `corridor_fx_margins` (corridor_id,
  margin_percent, valid_from, valid_to, status) pour la **marge par corridor**, soumise au même
  workflow d'approbation.
