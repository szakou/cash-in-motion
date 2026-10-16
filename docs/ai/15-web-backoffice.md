# 15 — Web : Back-office (Admin, Conformité, Support)

## Objectif
Construire `web/apps/backoffice` couvrant administration, paramétrage, conformité et support.

## Prérequis
14, 03, 05, 07, 12 terminés (06, 08, 13 pour les écrans correspondants).

## Références
Cahier §5.1 (ADM-01 à ADM-12), §5.2 (SUP-01 à SUP-06), §7.2.

## Écrans (route · rôles · contenu)
| Route | Rôles | Contenu |
| --- | --- | --- |
| `/` | ADMIN, SUPER_ADMIN | Tableau de bord global : KPI (volume, montants, frais, taxes par code), courbes par période, top agences/corridors, sélecteur jour/semaine/mois/année, comparaison période précédente |
| `/agencies` · `/agencies/:id` | ADMIN | Liste filtrable, création/édition, suspension, plafonds, float, utilisateurs de l'agence, carte |
| `/float-requests` | ADMIN | Demandes de réapprovisionnement à approuver/rejeter |
| `/users` | ADMIN, SUPER_ADMIN | Utilisateurs internes, rôles, suspension |
| `/settings/referential/*` | ADMIN | Pays, régions, villes, devises, types de pièces, corridors |
| `/settings/pricing/*` | ADMIN | Grilles de frais (éditeur de paliers), frais annexes, règles de taxes, taux de change ; historique des versions ; workflow d'approbation quatre yeux (badge « en attente ») ; **simulateur de devis** |
| `/settings/limits` | ADMIN, COMPLIANCE_OFFICER | Plafonds par niveau KYC, agence, corridor |
| `/compliance/alerts` · `/:id` | COMPLIANCE_OFFICER | File d'alertes AML, détail avec opérations liées, notes, décision, génération STR |
| `/compliance/screenings` | COMPLIANCE_OFFICER | Correspondances potentielles sanctions/PEP, décision |
| `/compliance/kyc` | COMPLIANCE_OFFICER | Revue KYC : visionneuse pièce/selfie, approuver/rejeter/demander plus |
| `/compliance/watchlists` | COMPLIANCE_OFFICER | Import de listes |
| `/support/transfers` · `/:id` | SUPPORT_AGENT | Recherche multicritère, détail, chronologie des statuts, données masquées + révélation motivée, renvoi du code, demande d'annulation |
| `/support/customers/:id` | SUPPORT_AGENT | Fiche client, wallet (lecture), opérations, statut KYC |
| `/reports` | ADMIN, COMPLIANCE_OFFICER | Exports (taxes à reverser, opérations, commissions, BCEAO), suivi des jobs, téléchargement |
| `/audit` | SUPER_ADMIN, COMPLIANCE_OFFICER | Journal d'audit filtrable |

## Règles d'interface
- Toute action irréversible passe par `ConfirmDialog` avec résumé.
- Formulaires de paramétrage : validation zod alignée sur le backend, aperçu de l'impact (simulateur).
- Montants toujours avec devise ; taxes détaillées par code.
- Données personnelles masquées par défaut.

## Critères d'acceptation
- Un ADMIN crée une grille de frais, un second ADMIN l'approuve, le simulateur reflète la nouvelle
  grille à partir de sa date d'effet.
- Le support retrouve un transfert par code en < 3 clics et voit sa chronologie.
- Tableau de bord cohérent avec l'API `reports/dashboard`.

## Tests exigés
Tests de composants pour les formulaires de paramétrage ; Playwright : création d'agence,
approbation de grille, revue d'alerte AML, recherche support.

## Hors périmètre
Portail agence (16).


## Mise à jour multi-pays (2026-10-16) — prévaut sur les sections ci-dessus

- Sélecteur de pays global dans l'en-tête (limité au périmètre de l'utilisateur) ; toutes les listes
  et tableaux de bord filtrables par pays.
- `/settings/countries` : fiche pays (zone, devise, fuseaux, langues, régulateur, CRF) et
  **liste de contrôle d'ouverture** avec bouton « Passer en opération ».
- `/settings/sms/providers`, `/settings/sms/routes` (principal/secours, sender ID, état, test
  d'envoi, bascule manuelle), `/monitoring/sms` (qualité par pays et fournisseur),
  `/settings/notification-templates` (par langue, compteur de segments).
- Taxes : éditeur par pays (`/settings/pricing/tax-rules?country=`).
