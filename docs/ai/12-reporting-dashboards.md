# 12 — Reporting, tableaux de bord et états réglementaires (module `reporting`)

## Objectif
Fournir les indicateurs des tableaux de bord (admin, agence) et les exports comptables, fiscaux et
réglementaires.

## Prérequis
09, 10 terminés.

## Références
Cahier §5.1 ADM-09, ADM-10 ; §5.3 AGW-07 ; §7.3.

## Tâches
1. Projection de lecture `operation_facts` (une ligne par opération finalisée : date locale dans le fuseau de l'agence,
   type, agence, agent, corridor, devise, montant, frais, chaque taxe, rabais, commission, statut)
   alimentée par les événements (idempotente) — jamais de requêtes analytiques sur les tables
   transactionnelles.
2. Agrégats matérialisés `daily_agency_stats` (rafraîchis toutes les 5 min) ; périodes jour,
   semaine (lundi), mois, année.
3. Endpoint `GET /api/v1/reports/dashboard?scope=global|agency&agencyId&period=day|week|month|year&date=`
   renvoyant : nombre et montant d'envois/paiements/cash-in/cash-out/P2P, frais, taxes par code,
   commissions, top 10 agences et corridors, série temporelle pour graphiques, comparaison à la
   période précédente (%). Périmètre appliqué (agence ↔ son agence uniquement).
4. Exports asynchrones (`POST /api/v1/reports/exports` → job → fichier MinIO → URL pré-signée) :
   journal des opérations (CSV/Excel), **état des taxes à reverser** (TTA, TAF, THZUEMOA par
   période), commissions par agence, balance du grand livre, encours de monnaie électronique et
   rapprochement cantonnement, état périodique BCEAO (gabarit paramétrable).
5. Rapprochement : comparaison `operation_facts` ↔ ledger, écarts listés.

## Critères d'acceptation
- Les chiffres du tableau de bord égalent la somme des opérations du ledger sur la même période.
- Un superviseur ne voit que son agence.
- Export de 100 000 lignes en < 60 s.

## Tests exigés
Projection idempotente, agrégats sur jeu de données connu, découpage de périodes dans le fuseau de chaque pays,
contrôle de périmètre, exports.

## Hors périmètre
Graphiques Web (15, 16).


## Mise à jour multi-pays (2026-10-16) — prévaut sur les sections ci-dessus

- `operation_facts` : ajouter `country_iso2`, `counterpart_country_iso2`, `local_date` calculée dans
  le fuseau de l'agence/du client ; agrégats par pays et devise.
- Vue consolidée : conversion explicite dans une devise de consolidation paramétrable, taux affichés.
- États de taxes par pays ; états réglementaires par régulateur (gabarits BCEAO, BEAC/COBAC,
  autres) ; état de compensation inter-pays (prompt 04).
