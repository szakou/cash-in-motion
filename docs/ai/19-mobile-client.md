# 19 — Mobile : App Client (`mobile/apps/client`)

## Objectif
Offrir au client final son wallet : inscription, KYC, solde, P2P, dépôt et retrait en agence par
QR Code, historique et notifications.

## Prérequis
17, 10 terminés (06 pour le KYC, 08 pour les agences proches).

## Références
Cahier §6.2 (MCL-01 à MCL-09), §6.3, §7.1 (consentements).

## Écrans
1. **Onboarding** : 3 écrans de présentation, acceptation CGU et politique de confidentialité
   (consentements versionnés), consentement marketing séparé et facultatif.
2. **Inscription** : téléphone (+221 par défaut) → OTP (renvoi après 60 s) → identité minimale
   (prénoms, nom, date de naissance) → création du PIN (2 saisies, refus des PIN triviaux
   `0000`, `1234`…) → activation biométrie optionnelle → wallet L1.
3. **Accueil** : solde (masquable d'un geste), plafonds restants, actions « Envoyer », « Déposer »,
   « Retirer », « Mon QR », dernières opérations.
4. **Envoyer (P2P)** : contact (`flutter_contacts`, permission demandée au besoin) ou numéro →
   nom masqué du destinataire → montant → récapitulatif frais/taxes/total → PIN → succès animé +
   reçu.
5. **Retirer (cash-out)** : montant → récapitulatif → QR affiché (luminosité max, compte à
   rebours) → l'agent scanne → écran de confirmation détaillé (agence, agent, montant) → PIN →
   reçu.
6. **Déposer (cash-in)** : scanner le QR de l'agent → récapitulatif → PIN → reçu ; ou « Mon QR »
   à présenter à l'agent.
7. **Historique** : liste paginée, filtres, détail, reçu partageable.
8. **Vérifier mon identité (KYC)** : niveau actuel et avantages du niveau supérieur ; L2 = photo
   recto/verso de la pièce (cadre guidé, contrôle de netteté) + selfie ; suivi du statut ;
   L3 = justificatif d'adresse.
9. **Agences proches** : carte (`flutter_map` + OpenStreetMap) et liste, horaires, itinéraire.
10. **Profil et sécurité** : changer le PIN, biométrie, appareils, langue, données personnelles
    (demande d'export), aide/contact support, déconnexion.
11. **Notifications** : centre de notifications, push à chaque mouvement.

## Règles
- Toute opération sortante exige le PIN (ou biométrie si activée et montant < seuil configurable).
- Affichage systématique des frais et taxes avant confirmation.
- Messages d'erreur en français simple (plafond atteint → proposer de passer au niveau KYC supérieur).
- Écrans sensibles protégés contre la capture d'écran.

## Critères d'acceptation
- Parcours inscription → KYC L2 → cash-in → P2P → cash-out réalisable de bout en bout.
- 3 PIN erronés → blocage temporaire avec message et délai.

## Tests exigés
Tests de widgets par écran, `integration_test` des parcours inscription, P2P, cash-out.

## Hors périmètre
Paiement marchand, Mobile Money externe, cartes (V2).


## Mise à jour multi-pays (2026-10-16) — prévaut sur les sections ci-dessus

- Inscription limitée aux pays `OPERATING` (sélecteur d'indicatif) ; devise, plafonds et langue par
  défaut du pays du client ; choix de langue parmi `available_locales` du pays.
- Agences proches filtrées sur le pays du wallet.
