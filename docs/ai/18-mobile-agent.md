# 18 — Mobile : App Agence (`mobile/apps/agent`)

## Objectif
Permettre à l'agent de réaliser cash-in et cash-out wallet par QR Code, de suivre son float et sa
caisse, et (souhaitable) d'envoyer/payer des transferts depuis le mobile.

## Prérequis
17, 10 terminés (09 pour MAG-06).

## Références
Cahier §6.1 (MAG-01 à MAG-07), §6.3 (séquence QR).

## Écrans
1. **Connexion** : OIDC agent + PIN local ; appareil enrôlé obligatoire.
2. **Accueil** : float disponible (alerte si bas), état de la caisse (ouvrir si fermée), boutons
   « Dépôt (cash-in) », « Retrait (cash-out) », « Payer un transfert », « Envoyer », opérations
   du jour.
3. **Cash-out** : scanner le QR du client → fiche (nom, montant, frais) → contrôle de pièce si
   exigé (saisie type + numéro, photo optionnelle) → « Accepter » → attente de la confirmation du
   client (temps réel SSE, délai max affiché) → reçu → instruction « Remettre X FCFA ».
4. **Cash-in (QR agent)** : saisie du montant (AmountKeypad) → QR affiché avec compte à rebours →
   confirmation du client en temps réel → reçu → « Encaisser X FCFA ».
5. **Cash-in (QR client)** : variante où l'agent scanne le QR profil du client puis saisit le montant.
6. **Historique** du jour et des 30 derniers jours, filtre par type, reçu partageable (SMS,
   WhatsApp via share sheet, PDF).
7. **Caisse** : ouverture, mouvements, clôture avec saisie du montant compté.
8. **Transferts (MAG-06, souhaitable)** : versions simplifiées des assistants d'envoi et de
   paiement du portail (prompt 16), mêmes règles.
9. **Impression (MAG-07, souhaitable)** : imprimante thermique Bluetooth ESC/POS 58 mm
   (`esc_pos_bluetooth` ou équivalent), appairage dans les paramètres.
10. **Paramètres** : profil, changement de PIN, imprimante, déconnexion.

## Règles
- Aucune opération si caisse fermée, float insuffisant, appareil rooté ou hors ligne.
- Bouton de validation désactivé après le premier appui jusqu'à réponse.
- Les montants à remettre/encaisser sont affichés en très grand.

## Critères d'acceptation
- Cash-in et cash-out de bout en bout avec l'app client (prompt 19) sur deux appareils.
- QR expiré ou déjà utilisé → message clair, aucune opération.

## Tests exigés
Tests de widgets par écran, tests d'intégration (`integration_test`) des parcours cash-in/cash-out
avec API simulée.

## Hors périmètre
Gestion d'équipe (portail Web 16).
