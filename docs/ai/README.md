# Prompts de développement — Cash In Motion

> **Depuis l'adoption de Spec Kit**, ces fichiers servent de **notes techniques pour
> `/speckit-plan`**. Le point d'entrée est `speckit/README.md` (ordre des fonctionnalités et
> correspondance avec ces notes). Les sections « Tâches » et « Tests exigés » ci-dessous sont
> indicatives : `/speckit-tasks` génère la liste de tâches officielle.

Ces prompts pilotent des agents AI (Claude Code ou équivalent) pour construire la plateforme de bout
en bout. Chaque fichier est autonome mais suppose que l'agent a lu `CLAUDE.md` (contexte racine,
stack, règles d'or) et peut consulter `docs/cahier-des-charges.md`.

## Ordre d'exécution et dépendances

| # | Prompt | Dépend de | Parallélisable avec |
| --- | --- | --- | --- |
| 00 | [Monorepo et infra locale](00-monorepo-infra-locale.md) | — | — |
| 01 | [Socle backend (shared)](01-backend-socle.md) | 00 | — |
| 02 | [Identité, authentification, RBAC](02-identite-auth-rbac.md) | 01 | 03 |
| 03 | [Référentiel](03-referentiel.md) | 01 | 02 |
| 04 | [Grand livre](04-grand-livre.md) | 01, 03 | 05 |
| 05 | [Tarification, taxes, change](05-tarification-taxes-change.md) | 03 | 04 |
| 06 | [Clients et KYC](06-clients-kyc.md) | 02, 03 | 07 |
| 07 | [Conformité AML et sanctions](07-conformite-aml.md) | 06 (API) | 08 |
| 08 | [Agences, caisses, float](08-agences-caisses-float.md) | 02, 04 | 07 |
| 09 | [Transferts : envoi, paiement, annulation](09-transferts.md) | 04–08 | — |
| 10 | [Wallet, P2P, cash-in/out QR](10-wallet-qr.md) | 04–08 | 11 |
| 11 | [Notifications](11-notifications.md) | 01, 02, 03 | 10, 12 |
| 12 | [Reporting et tableaux de bord](12-reporting-dashboards.md) | 09, 10 | 11 |
| 13 | [Audit et durcissement sécurité](13-audit-securite.md) | 01 (audit) ; 09, 10 (durcissement) | 12 |
| 14 | [Socle Web](14-web-socle.md) | 02 + `api/openapi.yaml` | 17 |
| 15 | [Web — Back-office](15-web-backoffice.md) | 14, 03, 05, 07, 12 | 16 |
| 16 | [Web — Portail Agence](16-web-portail-agence.md) | 14, 08, 09 | 15 |
| 17 | [Socle mobile Flutter](17-mobile-socle.md) | 02 + `api/openapi.yaml` | 14 |
| 18 | [Mobile — App Agence](18-mobile-agent.md) | 17, 10 | 19 |
| 19 | [Mobile — App Client](19-mobile-client.md) | 17, 10 | 18 |
| 20 | [DevOps, CI/CD, observabilité](20-devops-observabilite.md) | 00 (peut démarrer tôt) | tous |
| 21 | [Recette : E2E, charge, sécurité](21-recette-e2e-charge-securite.md) | tout | — |

Chemin critique : 00 → 01 → 03 → 04/05 → 08 → 09 → 10 → 18/19 → 21.

## Comment utiliser ces notes

Ces notes ne sont plus exécutées directement. Pour chaque fonctionnalité Spec Kit, donnez à
`/speckit-plan` les notes listées dans la colonne « Notes pour /speckit-plan » de
`speckit/README.md`, par exemple :

```text
/speckit-plan Respecte la constitution. Notes techniques : @docs/ai/04-grand-livre.md
```

Conseils :

- Une branche et une PR par fonctionnalité ; revue humaine obligatoire pour le grand livre, la
  tarification, les transferts, le wallet et la conformité (notes 04, 05, 07, 09, 10).
- L'ordre et les dépendances du tableau ci-dessus sont indicatifs ; l'ordre de référence est
  celui de `speckit/README.md`.
- L'avancement est suivi par Spec Kit (`tasks.md` de chaque fonctionnalité) ; `PROGRESS.md` est archivé.
- Après chaque prompt backend qui modifie l'API, relancer `./gradlew exportOpenApi` puis
  `pnpm --filter api-client generate` et `melos run generate`.

## Structure commune des prompts

Chaque prompt contient : Objectif · Prérequis · Références · Tâches · Modèle de données · API ·
Règles métier · Critères d'acceptation · Tests exigés · Hors périmètre.
