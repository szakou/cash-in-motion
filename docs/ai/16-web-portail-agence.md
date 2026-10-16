# 16 — Web : Portail Agence (guichet)

## Objectif
Construire `web/apps/agency`, l'outil de guichet pour envoyer et payer des transferts, gérer la
caisse et suivre l'activité de l'agence. Parcours d'envoi en moins de 2 minutes.

## Prérequis
14, 08, 09 terminés.

## Références
Cahier §5.3 (AGW-01 à AGW-09), §3.1–3.3, §4.5 (exemple de calcul).

## Écrans
| Route | Rôles | Contenu |
| --- | --- | --- |
| `/` | AGENCY_AGENT, AGENCY_SUPERVISOR | Tableau de bord agence : jour/semaine/mois/année, nombre et montants envoyés/payés, frais, commissions, float, état de la caisse |
| `/till` | AGENCY_AGENT | Ouvrir la caisse, mouvements du jour, clôture avec comptage (billetage optionnel), écart |
| `/send` | AGENCY_AGENT | Assistant d'envoi en 4 étapes (ci-dessous) |
| `/payout` | AGENCY_AGENT | Paiement par code (ci-dessous) |
| `/transfers` · `/:id` | AGENCY_AGENT, AGENCY_SUPERVISOR | Historique de l'agence, filtres, réimpression du reçu, annulation (superviseur) |
| `/customers` | AGENCY_AGENT | Recherche/création de clients |
| `/float` | AGENCY_SUPERVISOR | Solde du float, demandes de réapprovisionnement |
| `/team` | AGENCY_SUPERVISOR | Agents de l'agence, caisses à valider |
| `/commissions` | AGENCY_SUPERVISOR | Commissions par période |

### Assistant d'envoi
1. **Expéditeur** : recherche par téléphone/pièce ou création (champs cahier §3.1, scan de pièce,
   question test optionnelle). Alerte si pièce expirée.
2. **Bénéficiaire** : bénéficiaires récents de l'expéditeur ou nouveau (§3.2), pays de destination,
   ville, message optionnel (frais affichés), livraison optionnelle.
3. **Montant et devis** : saisie montant envoyé **ou** à recevoir ; appel `POST /quotes` avec
   anti-rebond ; tableau détaillé (frais, chaque taxe, rabais, total, taux, montant à payer) ;
   compte à rebours de validité du devis.
4. **Confirmation** : récapitulatif, encaissement (montant reçu, monnaie à rendre), validation →
   affichage du code en grand, impression du reçu (A4 ou ticket), SMS envoyés.
   Statut `PENDING_COMPLIANCE` : message clair, pas de code communiqué tant que non libéré.

### Paiement
Saisie du code (masque `000 000 000 0`, validation Luhn côté client) → fiche bénéficiaire et
montant → saisie de la pièce du bénéficiaire → question test si présente → confirmation →
reçu de paiement.

## Règles d'interface
- Raccourcis clavier (F2 envoi, F3 paiement), utilisable sans souris.
- Double soumission impossible (bouton verrouillé + `Idempotency-Key` stable pendant l'étape).
- Messages d'erreur métier explicites (`INSUFFICIENT_FLOAT`, `TILL_NOT_OPEN`, `QUOTE_EXPIRED`,
  `TRANSFER_NOT_PAYABLE`…).
- Fonctionne sur écran 1366×768.

## Critères d'acceptation
- Scénario de l'exemple §4.5 réalisable de bout en bout et reçu conforme.
- Un agent ne voit que son agence.

## Tests exigés
Playwright : envoi complet, paiement complet, annulation par superviseur, clôture de caisse avec
écart ; tests de composants de l'assistant (validation par étape).

## Hors périmètre
App mobile agent (18).


## Mise à jour multi-pays (2026-10-16) — prévaut sur les sections ci-dessus

- Indicatif, devise, fuseau et langue de l'interface proviennent du pays de l'agence ; libellés de
  taxes nationaux sur le devis et le reçu.
