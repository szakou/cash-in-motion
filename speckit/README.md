# Cash In Motion — Entrées Spec Kit

Ce dossier contient les textes à fournir aux commandes Spec Kit :

- `constitution.md` → à donner **une seule fois** à `/speckit-constitution`.
- `NN-*.md` → une fonctionnalité chacun, à donner à `/speckit-specify` **dans l'ordre**.
  Ils décrivent le QUOI et le POURQUOI, sans choix technique.

Les détails techniques (modèle de données, API, recettes du grand livre, écrans) restent dans
`docs/ai/` : ce sont les **notes à fournir à `/speckit-plan`** (voir la correspondance ci-dessous).

## Démarrage

Prérequis sur le poste : Git, `uv` (pour installer la CLI Spec Kit), Docker, JDK 25, Node + pnpm,
Flutter stable, et l'agent IA choisi (ex. Claude Code).

```bash
cd Cash-In-Motion
git init && git add -A && git commit -m "docs: cadrage initial (constitution, specs, notes, ADR)"
uv tool install specify-cli
specify init . --integration claude      # ou copilot, etc. ; conserver les fichiers existants
specify extension add git                 # optionnel : branches numérotées par fonctionnalité
```

Avant `/speckit-plan` de chaque fonctionnalité, trancher ses points **[À CLARIFIER]** avec le métier
(ou laisser `/speckit-clarify` les poser).

1. `/speckit-constitution` avec `@speckit/constitution.md`.
2. Pour chaque fonctionnalité, dans l'ordre :

```text
/speckit-specify   @speckit/NN-<nom>.md
/speckit-clarify   (répondre aux points « À clarifier »)
/speckit-plan      Respecter la constitution. Notes techniques : @docs/ai/<prompts listés ci-dessous>
/speckit-checklist
/speckit-tasks
/speckit-analyze
/speckit-implement
/speckit-converge  (répéter implement/converge jusqu'à « Converged »)
```

Spec Kit numérote lui-même ses dossiers de fonctionnalités (`001-…`) dans l'ordre où vous lancez
`/speckit-specify` ; en suivant l'ordre ci-dessous, ses numéros correspondront à ceux des fichiers.

## Ordre et correspondances

| # | Fonctionnalité | Dépend de | Notes pour `/speckit-plan` (`docs/ai/`) | Revue humaine |
| --- | --- | --- | --- | --- |
| 01 | Accès sécurisé et utilisateurs internes (+ socle des applications) | — | 00, 01, 02, 14, 17, 20 | |
| 02 | Journal d'audit | 01 | 13 (tâches 1 à 4) | |
| 03 | Référentiel | 01 | 03, 15 | |
| 04 | Tarification, taxes, change, devis | 03 | 05, 15 | Oui |
| 05 | Grand livre comptable | 01, 03 | 04 | Oui |
| 06 | Agences, caisses, float, commissions | 01, 05 | 08, 15, 16 | Oui |
| 07 | Clients et KYC au guichet | 01, 03 | 06 | |
| 08 | Conformité AML et sanctions | 01, 07 | 07, 15 | Oui |
| 09 | Envoi d'un transfert en agence | 04–08, 14 | 09, 16, 18 | Oui |
| 10 | Paiement, annulation, remboursement | 09 (11) | 09, 16, 18 | Oui |
| 11 | Wallet : inscription, sécurité, KYC mobile | 01, 05, 07, 08, 14 | 02, 06, 10, 17, 19 | Oui |
| 12 | Transfert P2P | 11 | 10, 19 | Oui |
| 13 | Dépôt et retrait par QR Code | 06, 11 | 10, 18, 19 | Oui |
| 14 | Notifications et fournisseurs SMS par pays | 01, 02, 03 | 11, 15 | Oui |
| 15 | Support client | 09, 10, 11, 14 | 09, 11, 15 | |
| 16 | Tableaux de bord et rapports | 09, 10, 12, 13 | 12, 15, 16 | |

Recette globale après la 16 : durcissement sécurité (`docs/ai/13`, tâches 5 à 7), puis E2E,
charge et sécurité (`docs/ai/21`).

## Remarques

- **Périmètre multi-pays** : UEMOA (XOF, BCEAO), CEMAC (XAF, BEAC/COBAC) et autres pays de la
  CEEAC. Tout ce qui varie par pays (devise, fuseau, langues, pièces, taxes, plafonds, régulateur,
  fournisseurs SMS) est du paramétrage (constitution, principe IX). Le Sénégal est le premier pays
  ouvert ; les exemples chiffrés utilisent ses taxes.
- **SMS** : le code de retrait part par SMS ; il est recommandé de lancer la 14 **avant la 09**
  (ordre 01 → 08, puis 14, puis 09 → 13, 15, 16). Spec Kit numérotera alors ses dossiers dans cet
  ordre, différent des numéros de fichiers : c'est sans conséquence.
- Les points marqués **[À CLARIFIER]** dans 04 (base de la TAF, THZUEMOA, qui paie les taxes,
  taxes sur le wallet) doivent être tranchés avec le métier avant `/speckit-plan` de la 04.
- En cas de divergence : constitution > spécification validée > cahier des charges > notes `docs/ai/`.
