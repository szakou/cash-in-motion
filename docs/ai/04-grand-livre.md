# 04 — Grand livre en partie double (module `ledger`)

## Objectif
Implémenter la seule source de vérité des soldes : comptes, journaux, écritures équilibrées,
verrouillage, soldes dérivés, rapprochement.

## Prérequis
01, 03 terminés. **Revue humaine obligatoire.**

## Références
`CLAUDE.md` §4 règles 1, 2, 4. Cahier §3.4, §7.3 (cantonnement), §9.1.

## Modèle de données
- `ledger_accounts` (id, code unique, type `ASSET|LIABILITY|REVENUE|EXPENSE|EQUITY`,
  owner_type `CUSTOMER_WALLET|AGENCY_FLOAT|SYSTEM`, owner_id nullable, currency,
  normal_side `DEBIT|CREDIT`, allow_negative bool default false, status, created_at)
- `ledger_balances` (account_id PK, balance NUMERIC(19,4), blocked NUMERIC(19,4), version,
  updated_at) — dénormalisation maintenue **dans la même transaction** que les écritures
- `journal_entries` (id, entry_type, reference_type, reference_id, description, posted_at,
  posted_by, reversal_of nullable) — ajout seul
- `ledger_lines` (id, journal_entry_id, account_id, side `DEBIT|CREDIT`, amount > 0, currency) —
  ajout seul

Comptes système créés par seed (un par devise active) :
`SYS_TRANSFERS_IN_TRANSIT`, `SYS_FEE_REVENUE`, `SYS_TAX_TTA_PAYABLE`, `SYS_TAX_TAF_PAYABLE`,
`SYS_TAX_THZUEMOA_PAYABLE`, `SYS_FX_GAIN_LOSS`, `SYS_SAFEGUARD_BANK` (cantonnement),
`SYS_AGENCY_COMMISSION_PAYABLE`, `SYS_DISCOUNTS`.

## Tâches
1. `LedgerApi.post(PostingRequest)` : liste de lignes ; valide l'équilibre par devise, montants > 0,
   comptes actifs, même devise compte/ligne ; verrouille les comptes concernés
   (`SELECT … FOR UPDATE` trié par id) ; refuse si un compte non `allow_negative` deviendrait
   négatif (`INSUFFICIENT_FUNDS`) ; écrit journal + lignes + met à jour `ledger_balances`.
2. `LedgerApi.reverse(journalEntryId, reason)` : contre-passation intégrale (jamais de suppression).
3. `LedgerApi.hold/release(accountId, amount, ref)` : montant bloqué (utilisé par wallet/QR).
4. `LedgerApi.balanceOf(accountId)` (disponible = balance − blocked), `statement(accountId, period,
   cursor)`.
5. `LedgerApi.openAccount(...)` idempotent par (owner_type, owner_id, currency, purpose).
6. Recettes d'écritures (classe `PostingRecipes`) pour : envoi de transfert, paiement de transfert,
   annulation/remboursement, cash-in, cash-out, P2P, approvisionnement float, collecte de taxes.
   Documenter chaque recette dans `docs/ledger/recettes.md` (tableau débit/crédit).
7. Job de contrôle d'intégrité quotidien : Σ lignes = `ledger_balances` pour chaque compte ;
   Σ débits = Σ crédits global ; Σ (wallets clients + floats d'agences) ≤ solde `SYS_SAFEGUARD_BANK` → alerte sinon.
8. Triggers PostgreSQL interdisant `UPDATE`/`DELETE` sur `journal_entries` et `ledger_lines`.

## Modèle de liquidité des agences (décision)
Toutes les agences fonctionnent sur **float prépayé** : `AGENCY_FLOAT` est un compte de passif
(valeur électronique que la plateforme doit à l'agence). Les espèces physiques appartiennent à
l'agence et sont suivies hors grand livre dans le livre de caisse (prompt 08).

| Opération | Effet sur le float de l'agence |
| --- | --- |
| Envoi de transfert (l'agence encaisse les espèces) | débit du total payé |
| Paiement de transfert (l'agence décaisse) | crédit du montant payé |
| Cash-in wallet (l'agence encaisse) | débit du montant déposé |
| Cash-out wallet (l'agence décaisse) | crédit du montant retiré |
| Réapprovisionnement validé | crédit |

## Exemple de recette — envoi national SN → SN en agence (100 000 XOF, frais 3 200, TTA 500, TAF 544)
(Pour un envoi entre deux pays, le compte de liaison remplace `SYS_TRANSFERS_IN_TRANSIT` : voir prompt 09.)
| Compte | Débit | Crédit |
| --- | --- | --- |
| AGENCY_FLOAT (agence émettrice) | 104 244 | |
| SYS_TRANSFERS_IN_TRANSIT | | 100 000 |
| SYS_FEE_REVENUE | | 3 200 |
| SYS_TAX_TTA_PAYABLE | | 500 |
| SYS_TAX_TAF_PAYABLE | | 544 |

## Critères d'acceptation
- Toute écriture déséquilibrée est refusée sans effet de bord.
- 200 débits concurrents sur un même wallet de 100 000 XOF à 1 000 XOF chacun : exactement 100
  réussissent, solde final 0, aucun interblocage.
- Le job d'intégrité détecte une incohérence injectée en test.

## Tests exigés
Unitaires (équilibre, signes, devises), intégration Testcontainers (concurrence, triggers,
contre-passation), tests de propriétés (jqwik) : toute séquence de postings valides garde Σ = 0.

## Hors périmètre
Logique métier des transferts/wallet (09, 10) — seules les recettes sont fournies ici.


## Mise à jour multi-pays (2026-10-16) — prévaut sur les sections ci-dessus

- `ledger_accounts` : ajouter `country_iso2` ; comptes système créés par (pays `OPERATING`, devise).
- Comptes de taxes par code et par pays : `SYS_TAX_<CODE>_PAYABLE` + `country_iso2`.
- Nouveau type `SYS_INTERCOUNTRY_SETTLEMENT` (un par paire de pays et devise) pour les transferts
  inter-pays : l'envoi crédite la liaison côté pays émetteur, le paiement la débite côté pays payeur ;
  état de compensation `GET /api/v1/reports/intercountry-settlement`.
- Contrôle d'intégrité de couverture par (pays, devise) : Σ wallets + Σ floats du pays ≤
  `SYS_SAFEGUARD_BANK` du pays + solde net des `SYS_INTERCOUNTRY_SETTLEMENT` du pays (créances −
  dettes envers les autres pays).
- Écart de change des transferts inter-pays : constaté en `SYS_FX_GAIN_LOSS` lors de la
  compensation entre pays (pas au paiement).
- Compte `SYS_FROZEN_FUNDS` par (pays, devise) : fonds des transferts gelés pour sanction confirmée.
- Recette détaillée d'un envoi inter-pays : section multi-pays du prompt 09.
