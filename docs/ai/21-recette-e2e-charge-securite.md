# 21 — Recette : tests E2E, charge et sécurité

## Objectif
Valider la plateforme complète avant le pilote : scénarios métier de bout en bout, performance,
résilience et sécurité.

## Prérequis
Tous les prompts précédents terminés.

## Références
Cahier §8 (exigences non fonctionnelles), §11 (recette et pilote).

## Tâches
1. **Jeu de données de recette** (`infra/seed/recette`) : 5 agences, 15 agents, 200 clients
   (niveaux KYC variés), grilles et taxes actives, listes de sanctions de test.
2. **Scénarios E2E** (Playwright pour le Web, `integration_test` / Patrol pour le mobile), au
   minimum :
   - jeu de données : SN, ML et CM « en opération » ;
   - envoi SN→ML de 100 000 XOF et paiement (montants de l'exemple §4.5) ;
   - envoi SN→CM (XOF → XAF) avec THZUEMOA, paiement au Cameroun et état de compensation ;
   - (V2, partenaires) envoi SN→FR avec change EUR ;
   - envoi bloqué par correspondance sanctions → libération par la conformité → paiement ;
   - annulation le jour même et remboursement ;
   - cash-in, P2P, cash-out complet entre app agent et app client ;
   - dépassement de plafond KYC L1 → passage L2 → opération acceptée ;
   - clôture de caisse avec écart et validation superviseur ;
   - export de l'état des taxes et rapprochement avec le grand livre.
3. **Charge** (k6) : 50 opérations financières/s soutenues 30 min (mix 40 % devis, 20 % envois,
   15 % paiements, 25 % wallet) ; objectifs : p95 devis < 300 ms, p95 transaction < 2 s, 0 erreur
   d'intégrité ; test de pic à 150/s ; rapport dans `docs/tests/charge.md`.
4. **Résilience** : coupure RabbitMQ, Valkey, fournisseur SMS pendant le test de charge → aucune
   perte d'opération, rattrapage de l'outbox, pas de double débit.
5. **Intégrité** : à la fin de chaque campagne, job d'intégrité ledger vert, Σ wallets + floats ≤ compte de
   cantonnement de chaque pays, rapprochement reporting ↔ ledger sans écart.
6. **Sécurité** : OWASP ZAP (scan authentifié) sur l'API et les apps Web, MobSF sur les APK/IPA,
   checklist OWASP ASVS niveau 2 et MASVS L2 remplie (`docs/security/asvs-checklist.md`),
   préparation du périmètre du test d'intrusion externe.
7. **Plan de recette** et PV : `docs/tests/plan-de-recette.md` (cas, résultat attendu, statut,
   preuve), liste des anomalies classées bloquante/majeure/mineure.

## Critères d'acceptation
- 100 % des scénarios E2E passent en preprod.
- Objectifs de charge atteints sans erreur d'intégrité.
- Aucune vulnérabilité High/Critical ouverte.

## Hors périmètre
Test d'intrusion par un prestataire externe (planifié séparément).
