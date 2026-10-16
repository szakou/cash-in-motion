# 14 — Socle Web (React, pnpm workspace)

## Objectif
Mettre en place les fondations communes aux deux applications Web : authentification, client API
généré, design system, layout, i18n, gestion des erreurs, tests.

## Prérequis
02 terminé ; `api/openapi.yaml` exporté.

## Références
`CLAUDE.md` §2, §5 (Web). Cahier §8 (ergonomie, compatibilité).

## Tâches
1. `packages/config` : tsconfig strict (`noUncheckedIndexedAccess`), ESLint (react, hooks,
   jsx-a11y, import), Prettier, Vitest preset.
2. `packages/api-client` : génération orval depuis `api/openapi.yaml` (hooks TanStack Query +
   schémas zod) ; instance axios/fetch qui ajoute le jeton, `Idempotency-Key` (UUID) sur les POST
   financiers, `Accept-Language: fr`, `X-Client-App` (`backoffice` ou `agency`) et
   `X-Correlation-ID` (UUID généré par action utilisateur, réutilisé pour les retries de cette
   action), et convertit les erreurs RFC 7807 en `ApiError` typée ; les messages d'erreur affichent
   « Référence : <correlationId> » avec un bouton copier ; Sentry reçoit le `correlationId` en tag.
3. `packages/ui` : shadcn/ui + Tailwind, thème clair/sombre, composants métier : `MoneyInput`
   (entier XOF, séparateur de milliers espace fine), `MoneyDisplay`, `formatMoney`, `PhoneInput`
   (E.164, défaut +221), `CountrySelect`, `CitySelect`, `DateRangePicker`, `DataTable` (TanStack
   Table, tri, filtres, pagination curseur, export CSV), `StatusBadge`, `ConfirmDialog`,
   `MaskedValue` (bouton « révéler » avec motif), `EmptyState`, `KpiCard`, `ChartCard` (ECharts).
4. Auth OIDC PKCE (`oidc-client-ts`) avec Keycloak, renouvellement silencieux, déconnexion,
   garde de routes par rôle (`<RequireRole roles=[...]>`), écran « accès refusé ».
5. Layout : barre latérale par rôle, en-tête (agence courante, utilisateur, déconnexion), fil
   d'Ariane, notifications toast, page 404.
6. i18n `react-i18next`, fichiers `fr.json` (seule langue V1), formats de date `dd/MM/yyyy HH:mm`
   dans le fuseau de l'agence ou du pays de l'utilisateur.
7. Gestion d'erreurs : boundary globale, mapping des `code` métier vers messages français, Sentry.
8. Tests : Vitest + Testing Library pour les composants ; MSW pour simuler l'API ; Playwright
   configuré (projet `e2e/`) avec un test de connexion.
9. Accessibilité WCAG AA : contrastes, focus visible, libellés de formulaires, navigation clavier.

## Critères d'acceptation
- Connexion/déconnexion fonctionnelles sur les deux apps avec les utilisateurs de démo.
- Un rôle sans accès voit l'écran « accès refusé ».
- `pnpm -r lint typecheck test build` passe.
- Storybook (ou Ladle) présente les composants de `packages/ui`.

## Hors périmètre
Écrans métier (15, 16).


## Mise à jour multi-pays (2026-10-16) — prévaut sur les sections ci-dessus

- `MoneyInput` / `formatMoney` : nombre de décimales et symbole selon la devise (XOF/XAF : 0, « FCFA » ;
  CDF : 2, « FC »…), lus depuis le référentiel ; jamais « entier XOF » en dur.
- `PhoneInput` : indicatif par défaut = pays de l'agence ou de l'utilisateur ; tous les indicatifs
  des pays du référentiel acceptés.
- i18n : `fr.json` de référence, structure prête pour `en`, `pt`, `es` ; locale selon l'utilisateur.
- TypeScript 7 (ADR-0004) ; vérifier la compatibilité de typescript-eslint, orval, Vitest et Ladle.
