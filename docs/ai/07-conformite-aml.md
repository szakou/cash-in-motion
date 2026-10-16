# 07 — Conformité AML/LBC-FT et sanctions (module `compliance`)

## Objectif
Contrôler chaque opération avant exécution (sanctions/PEP, plafonds) et surveiller les flux après
exécution (règles AML), avec une file de revue pour la conformité.

## Prérequis
API de 06 disponible.

## Références
Cahier §7.2, §5.2, §2 (Responsable conformité).

## Modèle de données
- `watchlist_entries` (id, source `UN|EU|OFAC|NATIONAL|INTERNAL`, full_name, aliases jsonb,
  birth_date, nationality, list_reference, imported_at)
- `screening_results` (id, subject_type, subject_id, context_type, context_id, score, matched_entry_id,
  status `CLEAR|POTENTIAL_MATCH|CONFIRMED_MATCH|FALSE_POSITIVE`, reviewed_by, reviewed_at)
- `limits` (id, scope `KYC_LEVEL|AGENCY|CORRIDOR|GLOBAL`, scope_value, operation_type, period
  `PER_OPERATION|DAILY|MONTHLY`, max_amount, max_count, currency, valid_from, valid_to)
- `aml_rules` (code, description, params jsonb, severity `LOW|MEDIUM|HIGH`, enabled)
- `aml_alerts` (id, rule_code, subject_type, subject_id, related_operations jsonb, severity, status
  `OPEN|IN_REVIEW|CLOSED_NO_ACTION|ESCALATED|REPORTED`, assignee, notes, created_at, closed_at)
- `str_reports` (id, alert_id, payload jsonb, generated_at, sent_at) — déclaration CENTIF

## Tâches
1. **Pré-contrôle synchrone** `ComplianceApi.preCheck(OperationContext)` → `ALLOW | HOLD | DENY`
   avec motifs : screening expéditeur + bénéficiaire (similarité Jaro-Winkler + normalisation
   accents/ordre des noms, seuil configurable 0,88), vérification des plafonds (cumul jour/mois
   depuis les opérations confirmées), client bloqué, pièce expirée, corridor à risque.
   `HOLD` → l'opération passe en `PENDING_COMPLIANCE`.
2. **Import des listes** : job d'import CSV/XML (format ONU consolidé) + saisie manuelle de liste
   interne ; interface `SanctionsProvider` pour un fournisseur externe ultérieur.
3. **Surveillance asynchrone** : consommateur des événements `TransferSent`, `TransferPaid`,
   `WalletOperationCompleted` évaluant les règles :
   `STRUCTURING` (≥ N opérations sous le seuil en 24 h), `HIGH_FREQUENCY`, `ROUND_AMOUNTS`,
   `MANY_BENEFICIARIES` (même expéditeur → ≥ N bénéficiaires distincts / 7 j),
   `MANY_SENDERS` (même bénéficiaire ← ≥ N expéditeurs / 7 j), `HIGH_RISK_CORRIDOR`,
   `RAPID_IN_OUT` (cash-in suivi de cash-out < 1 h). Paramètres dans `aml_rules.params`.
4. File de revue : assignation, notes, décision ; décision sur une opération `PENDING_COMPLIANCE`
   → `ComplianceDecisionMade` (libère ou refuse l'opération via événement).
5. Gel/dégel d'un client ou d'un wallet (`ComplianceApi.freeze/unfreeze`).
6. Génération du dossier de déclaration de soupçon (PDF Apache FOP + JSON) pour la CENTIF.
7. Conservation : purge interdite avant 10 ans (contrainte applicative + documentation).

## API
- `GET /api/v1/compliance/alerts`, `PATCH /api/v1/compliance/alerts/{id}`
- `GET /api/v1/compliance/screenings?status=POTENTIAL_MATCH`, `POST .../{id}/decision`
- `GET|POST|PATCH /api/v1/admin/compliance/{limits|rules}`
- `POST /api/v1/compliance/watchlists/import`
- `POST /api/v1/compliance/customers/{id}/freeze|unfreeze`

## Critères d'acceptation
- « Mamadou Diallo » vs entrée « DIALLO Mamadou » → `POTENTIAL_MATCH`.
- 5 envois de 490 000 XOF en 24 h avec seuil 500 000 → alerte `STRUCTURING`.
- Un `DENY` n'écrit rien dans le grand livre.

## Tests exigés
Unitaires par règle (paramètres limites), similarité de noms (jeu de cas), intégration
pré-contrôle + plafonds, flux d'alerte complet.

## Hors périmètre
Écrans (15).


## Mise à jour multi-pays (2026-10-16) — prévaut sur les sections ci-dessus

- `limits` : ajouter `country_iso2` ; `aml_alerts` et `str_reports` : `country_iso2` (et
  `counterpart_country_iso2` pour les opérations inter-pays) ; `watchlist_entries.source NATIONAL`
  rattachée à un pays.
- Format de déclaration de soupçon par pays (CENTIF UEMOA, ANIF CEMAC, autres) : gabarits paramétrables.
