# ADR-0004 — Valkey, TypeScript 7 et Apache FOP

- **Statut** : accepté
- **Date** : 2026-10-16
- **Décideur** : Seydou Zakou
- **Complète** : ADR-0003

## 1. Valkey à la place de Redis

**Contexte** : depuis la version 8, Redis est distribué sous triple licence (RSALv2 / SSPLv1 /
AGPLv3). Valkey est un fork compatible au niveau du protocole, sous licence BSD.

**Décision** : Valkey est le seul magasin clé-valeur de la plateforme (OTP, jetons QR, limitation de
débit, caches, compteurs de bascule SMS).

**Conséquences** :
- Côté Java, on garde les clients standards compatibles : Spring Data Redis (Lettuce), Bucket4j.
- Images : `valkey/valkey` (tag stable épinglé) en local, en CI (Testcontainers) et en production.
- Toute introduction de Redis exige un nouvel ADR.

## 2. TypeScript 7

**Contexte** : TypeScript 7 (compilateur natif) est la version courante du registre npm ; il
accélère fortement la vérification de types sur un espace de travail à deux applications et
plusieurs paquets.

**Décision** : TypeScript 7 en mode strict pour `web/`.

**Conséquences** :
- Vérifier au prompt 14 la compatibilité des outils (typescript-eslint, orval, Vite, Vitest,
  Storybook/Ladle). Un outil incompatible est traité par sa solution de compatibilité documentée ;
  un retour à TypeScript 6 exige un nouvel ADR.
- Version épinglée dans le catalogue pnpm de l'espace de travail.

## 3. Apache FOP à la place de JasperReports

**Contexte** : la plateforme produit des PDF : reçus d'envoi et de paiement (A4 et ticket thermique
58/80 mm), états de taxes, états réglementaires par pays (BCEAO, BEAC/COBAC…), dossiers de
déclaration de soupçon, relevés.

**Décision** : Apache FOP (XSL-FO → PDF, licence Apache 2.0) pour tous les PDF ; Apache POI reste
utilisé pour les exports Excel ; CSV en natif.

**Conséquences** :
- Gabarits XSL-FO versionnés dans `backend/src/main/resources/templates/fo/`, alimentés par un XML
  ou un modèle de données intermédiaire produit par chaque module (pas de logique métier dans les
  gabarits).
- Polices embarquées couvrant les caractères accentués et les langues prévues (français, anglais,
  portugais, espagnol) ; libellés de taxes et mentions légales paramétrés par pays.
- Formats de page : A4 et rouleaux 58/80 mm ; QR de vérification intégré en image.
- Tests : rendu de chaque gabarit sur des données d'exemple (dont l'exemple 100 000 → 104 244 XOF),
  comparaison du texte extrait du PDF.
- Pas d'éditeur graphique de rapports : la mise en page se fait dans les gabarits XSL-FO.
