> **Document d'origine (15/10/2026), centré sur le Sénégal.** Il est remplacé par
> `speckit/constitution.md` (périmètre multi-pays UEMOA/CEMAC/CEEAC, principes, stack), les ADR de
> `docs/adr/` et les spécifications `speckit/NN-*.md`. La section 10 (stack : Java 21, Spring Boot 3,
> Redis, JasperReports) est **obsolète** ; les exemples chiffrés (§4.5) restent valables.

# Cash In Motion — Cahier des charges & spécifications techniques

Oct 15, 2026 · @Seydou Zakou

## 1. Contexte, objectifs et périmètre

Cash In Motion est une plateforme de transfert d'argent et de portefeuille électronique (wallet), opérée depuis le Sénégal, qui permet d'envoyer de l'argent en agence ou de pair à pair, et de déposer/retirer des espèces via QR Code.

**Objectifs**

- Permettre aux agences d'effectuer des envois et des paiements (retraits) nationaux et internationaux, avec calcul automatique des frais, taxes et taux de change.
- Offrir aux clients un wallet mobile : consultation du solde, transfert P2P, cash-in/cash-out en agence par QR Code.
- Donner à l'administration une configuration centralisée (pays, villes, grilles tarifaires, taux de taxes, taux de change) et un pilotage par tableaux de bord.
- Garantir la conformité réglementaire (protection des données, LBC/FT – AML/KYC, réglementation BCEAO sur la monnaie électronique).

**Périmètre**

| Livrable | Utilisateurs | Canal |
| --- | --- | --- |
| Back-office Web | Administrateur, Support client, Compliance | Navigateur |
| Portail Agence Web | Agent / Superviseur d'agence | Navigateur |
| App mobile Agence | Agent d'agence | Android (prioritaire), iOS |
| App mobile Client | Client final | Android, iOS |
| API & services backend | Toutes les applications, partenaires | REST / Webhooks |

**Hors périmètre (V1)** : cartes bancaires virtuelles, crédit/microcrédit, paiement marchand en ligne, intégration directe aux opérateurs Mobile Money (prévue en V2).

## 2. Acteurs et rôles

L'accès est contrôlé par rôles (RBAC) ; chaque utilisateur interne est rattaché à une agence ou au siège.

| Rôle | Application | Responsabilités principales |
| --- | --- | --- |
| Super administrateur | Back-office Web | Gestion des administrateurs, paramètres système sensibles, audit |
| Administrateur | Back-office Web | Agences, pays/régions/villes, grilles de frais, taux de taxes et de change, plafonds, tableau de bord global |
| Responsable conformité | Back-office Web | Validation KYC, revue des alertes AML, gel/déblocage de transactions et comptes, déclarations CENTIF |
| Support client | Back-office Web | Recherche et consultation des transactions, historique, tickets, actions limitées (renvoi de code, signalement) |
| Superviseur d'agence | Portail Agence + mobile | Gestion des agents de son agence, caisse et float, tableaux de bord de l'agence, annulations |
| Agent d'agence | Portail Agence + mobile | Envoi, paiement, cash-in/cash-out client par QR Code, clôture de caisse |
| Client (wallet) | App mobile Client | Inscription/KYC, solde, P2P, cash-in/cash-out en agence, historique |
| Expéditeur / Bénéficiaire non inscrits | Agence | Personnes servies au guichet sans wallet |
| Partenaires (V2) | API | Correspondants étrangers, opérateurs Mobile Money, banques |

## 3. Modèle de données fonctionnel

Les entités du cahier initial sont reprises et complétées par celles nécessaires au wallet, à la comptabilité et à la conformité.

### 3.1 Expéditeur (Client)

| Champ | Type | Obligatoire | Règle |
| --- | --- | --- | --- |
| Nom et prénoms | Texte | Oui | Contrôle listes de sanctions/PEP à la saisie |
| Adresse | Texte | Oui |  |
| Ville | Référentiel | Oui | Issue du paramétrage admin |
| Pays | Référentiel ISO 3166 | Oui | Pays d'envoi autorisé |
| Téléphone | E.164 | Oui | Unique par client wallet, vérifié par OTP |
| Type de pièce d'identité | Liste (CNI CEDEAO, passeport, carte consulaire, permis…) | Oui |  |
| Numéro de pièce | Texte | Oui | Chiffré au repos ; + date d'expiration et scan recto/verso |
| Question test | Texte | Non | Utilisée au paiement si renseignée |
| Réponse | Texte | Non | Stockée hachée, jamais affichée |
| Ajouts proposés | Date de naissance, nationalité, profession, source des fonds, niveau KYC | Selon montant | Exigés par la LBC/FT |

### 3.2 Bénéficiaire

| Champ | Type | Obligatoire |
| --- | --- | --- |
| Nom et prénoms | Texte | Oui |
| Ville | Référentiel | Non |
| Pays | Référentiel | Oui |
| Message | Texte (≤ 160 caractères) | Non — génère des frais de message |
| Téléphone (ajout proposé) | E.164 | Recommandé — notification SMS du code |

Au paiement, l'agent saisit la pièce d'identité du bénéficiaire (type, numéro), qui est rattachée à la transaction.

### 3.3 Envoi (Transaction de transfert)

| Champ | Description |
| --- | --- |
| Code | Référence unique de retrait (ex. 10 chiffres + clé de contrôle), non devinable |
| Date et heure | Horodatage UTC + fuseau de l'agence |
| Agent (nom, adresse) | Agent et agence émettrice ; agent et agence payeuse au retrait |
| Montant envoyé | En monnaie d'envoi |
| Frais de transfert | Selon grille tarifaire (corridor, tranche) |
| Frais de message / livraison | Optionnels |
| Taxes TTA, TAF, THZUEMOA | Détail par taxe |
| Total taxe | Somme des taxes |
| Rabais | Optionnel, plafonné aux frais |
| Total (monnaie d'envoi) | Montant payé par l'expéditeur |
| Taux de change | Figé à la création de l'envoi |
| Montant à payer (monnaie de réception) | Montant remis au bénéficiaire |
| Statut (ajout) | Créé, En attente conformité, Disponible, Payé, Annulé, Remboursé, Bloqué |

### 3.4 Entités complémentaires

- **Agence** : code, nom, adresse, ville, pays, statut, type (propre / sous-agent), plafonds, compte float.
- **Utilisateur** : identifiants, rôle, agence, MFA, statut.
- **Wallet** : client, devise (XOF), solde, solde bloqué, plafonds selon niveau KYC, statut.
- **Opération wallet** : type (cash-in, cash-out, P2P, réception de transfert), montant, frais, QR Code / jeton, statut.
- **Écriture comptable** : grand livre en partie double (débit/crédit) — source de vérité de tous les soldes.
- **Paramétrage** : pays, régions, villes, devises, corridors, grilles de frais, taux de taxes, taux de change, plafonds.
- **Alerte AML / Dossier KYC / Journal d'audit**.

## 4. Règles de calcul : frais, taxes et change

Tous les taux, barèmes et plafonds sont paramétrables par l'administrateur, versionnés avec une date d'effet, et figés sur chaque transaction au moment de sa création.

### 4.1 Frais

- **Frais de transfert** : grille par corridor (pays d'envoi → pays de réception) et par tranche de montant ; valeur fixe ou pourcentage, avec minimum et maximum.
- **Frais de message** et **frais de livraison** : montants fixes paramétrables, appliqués seulement si l'option est choisie.
- **Rabais** : remise sur les frais (code promo ou décision du superviseur), jamais supérieure au total des frais.

### 4.2 Taxes (Sénégal)

| Taxe | Base (hypothèse) | Taux | Plafond | Application |
| --- | --- | --- | --- | --- |
| TTA — Taxe sur les Transferts d'Argent | Montant envoyé | 0,5 % | 2 000 XOF par opération | Envoi et retrait |
| TAF — Taxe sur les Activités Financières | Frais facturés (commissions) | 17 % | — | Envoi et retrait |
| THZUEMOA — Taxe Hors Zone UEMOA | Montant envoyé | À paramétrer | À paramétrer | Flux dont l'origine ou la destination est hors UEMOA |

### 4.3 Formules

```latex
\text{Frais} = F_{transfert} + F_{message} + F_{livraison}
```

```latex
TTA = \min(0{,}005 \times M_{envoy\acute{e}},\ 2000)
```

```latex
TAF = 0{,}17 \times (\text{Frais} - \text{Rabais})
```

```latex
\text{Total} = M_{envoy\acute{e}} + \text{Frais} - \text{Rabais} + TTA + TAF + THZUEMOA
```

```latex
M_{\grave{a}\ payer} = M_{envoy\acute{e}} \times \text{Taux de change}
```

Arrondis : montants en XOF arrondis à l'unité (pas de centimes) ; règle d'arrondi par devise paramétrable. Les calculs utilisent des décimaux exacts, jamais des flottants.

### 4.4 Taux de change

- Taux de référence saisi ou importé par l'admin, avec une marge commerciale par corridor.
- XOF ↔ EUR : parité fixe (655,957 XOF pour 1 EUR) ; XOF ↔ XOF (UEMOA) : taux 1.
- Le taux affiché au devis est garanti jusqu'à validation de l'envoi (durée de validité paramétrable).

### 4.5 Exemple — Sénégal → Mali (zone UEMOA)

| Ligne | Montant (XOF) |
| --- | --- |
| Montant envoyé | 100 000 |
| Frais de transfert | 3 000 |
| Frais de message | 200 |
| TTA (0,5 % × 100 000) | 500 |
| TAF (17 % × 3 200) | 544 |
| THZUEMOA | 0 (corridor intra-UEMOA) |
| Total taxe | 1 044 |
| **Total payé par l'expéditeur** | **104 244** |
| Montant à payer au bénéficiaire (taux 1) | 100 000 |

Les montants de frais de l'exemple sont illustratifs.

## 5. Exigences fonctionnelles — Web

Identifiants : ADM = Administration, SUP = Support, AGW = Agence Web. Priorité M = indispensable V1, S = souhaitable V1, C = V2.

### 5.1 Back-office Administrateur

| ID | Exigence | Priorité |
| --- | --- | --- |
| ADM-01 | Créer, modifier, suspendre, clôturer une agence (coordonnées, type, plafonds journaliers/mensuels, horaires) | M |
| ADM-02 | Gérer les utilisateurs internes et leurs rôles ; MFA obligatoire | M |
| ADM-03 | Paramétrer pays d'envoi/réception, régions, villes, devises | M |
| ADM-04 | Paramétrer les corridors et grilles de frais (tranches, fixe/%, min/max, date d'effet) | M |
| ADM-05 | Paramétrer les taux de taxes (TTA, TAF, THZUEMOA) et leurs plafonds, avec historique | M |
| ADM-06 | Gérer les taux de change et marges par corridor | M |
| ADM-07 | Approvisionner et suivre le float des agences (demandes, validations, soldes) | M |
| ADM-08 | Plafonds wallet par niveau KYC | M |
| ADM-09 | Tableau de bord global : volumes, montants, frais, taxes collectées, top agences/corridors, évolution | M |
| ADM-10 | Rapports exportables (CSV/Excel/PDF), dont état des taxes à reverser | M |
| ADM-11 | Principe des quatre yeux pour les changements sensibles (taux, frais, plafonds) | S |
| ADM-12 | Gestion des campagnes de rabais / codes promo | C |

### 5.2 Support client

| ID | Exigence | Priorité |
| --- | --- | --- |
| SUP-01 | Rechercher une transaction par code, téléphone, nom, agence, date, montant | M |
| SUP-02 | Consulter le détail complet et l'historique des statuts (chronologie, agents, écritures) | M |
| SUP-03 | Données personnelles masquées par défaut ; démasquage journalisé et motivé | M |
| SUP-04 | Renvoyer le SMS du code au bénéficiaire / à l'expéditeur | M |
| SUP-05 | Ouvrir un ticket, demander une annulation ou un remboursement (validé par un superviseur) | S |
| SUP-06 | Consulter un wallet client : solde, opérations, statut KYC (lecture seule) | M |

### 5.3 Portail Agence

| ID | Exigence | Priorité |
| --- | --- | --- |
| AGW-01 | Rechercher ou créer un expéditeur (KYC au guichet, scan de pièce) | M |
| AGW-02 | Simuler un envoi (devis : frais, taxes, taux, montant à payer) | M |
| AGW-03 | Créer l'envoi, encaisser, générer le code et imprimer le reçu ; SMS aux deux parties | M |
| AGW-04 | Payer un transfert : saisie du code, contrôle de la question test, KYC du bénéficiaire, reçu | M |
| AGW-05 | Annulation avant paiement (même jour, par superviseur) et remboursement | M |
| AGW-06 | Gestion de caisse : ouverture, mouvements, clôture, écart de caisse | M |
| AGW-07 | Tableaux de bord journalier, hebdomadaire, mensuel, annuel : nombre et montant des envois/paiements, frais, commissions | M |
| AGW-08 | Consultation du float et demande de réapprovisionnement | M |
| AGW-09 | Commissions agent/agence calculées automatiquement | S |

## 6. Exigences fonctionnelles — Mobile

### 6.1 App mobile Agence

| ID | Exigence | Priorité |
| --- | --- | --- |
| MAG-01 | Connexion agent (identifiant + PIN + appareil enrôlé), session courte | M |
| MAG-02 | Cash-in : l'agent génère ou scanne un QR Code, le client confirme sur son app, le wallet est crédité contre espèces | M |
| MAG-03 | Cash-out : le client génère un QR Code de retrait, l'agent le scanne, contrôle l'identité et remet les espèces | M |
| MAG-04 | Solde du float agent visible en temps réel ; blocage si float insuffisant | M |
| MAG-05 | Historique des opérations du jour et reçu numérique (SMS/partage) | M |
| MAG-06 | Envoi/paiement de transfert depuis le mobile (mêmes règles que AGW) | S |
| MAG-07 | Impression Bluetooth sur imprimante thermique | S |

### 6.2 App mobile Client

| ID | Exigence | Priorité |
| --- | --- | --- |
| MCL-01 | Inscription par téléphone + OTP, création d'un code PIN, biométrie optionnelle | M |
| MCL-02 | KYC progressif : niveau 1 (téléphone), niveau 2 (pièce + selfie), niveau 3 (justificatif d'adresse) ; plafonds associés | M |
| MCL-03 | Affichage du solde et de l'historique | M |
| MCL-04 | Transfert P2P vers un numéro ou un contact, avec frais affichés avant confirmation par PIN | M |
| MCL-05 | Cash-in / cash-out en agence via QR Code | M |
| MCL-06 | Recevoir un transfert international directement dans le wallet | S |
| MCL-07 | Notifications push et SMS de chaque mouvement | M |
| MCL-08 | Localisation des agences proches | S |
| MCL-09 | Interface en français (wolof et anglais en V2) | M |

### 6.3 Séquence QR Code — cash-out

1. Le client saisit le montant dans son app ; le serveur crée une demande et renvoie un QR Code signé (jeton à usage unique, validité 5 minutes).
2. L'agent scanne le QR Code ; le serveur vérifie signature, expiration, plafonds, solde du client et statut AML.
3. L'agent voit nom, montant et frais, contrôle la pièce si le montant dépasse le seuil, puis valide.
4. Le client confirme avec son PIN ; le serveur passe les écritures (débit wallet client, crédit float agent) en une seule transaction.
5. Les deux apps affichent le reçu ; l'agent remet les espèces.

Le cash-in suit la même séquence en sens inverse (crédit wallet, débit float). Le QR Code ne contient jamais de données personnelles ni de montant modifiable, seulement une référence signée.

## 7. Conformité réglementaire

La plateforme doit être exploitée sous un agrément BCEAO (émetteur de monnaie électronique ou adossement à un établissement agréé) ; les références ci-dessous sont à valider par un juriste dans leur version en vigueur.

### 7.1 Protection des données (RGPD et loi sénégalaise)

- Loi sénégalaise n° 2008-12 sur les données à caractère personnel : déclaration/autorisation auprès de la CDP avant mise en production.
- RGPD applicable aux clients résidant dans l'UE (corridors Europe → Afrique).
- Minimisation, finalités déclarées, consentement explicite pour le marketing, politique de confidentialité dans les apps.
- Droits des personnes (accès, rectification, opposition) via un formulaire et un circuit back-office ; l'effacement est limité par l'obligation de conservation LBC/FT.
- Chiffrement des données sensibles au repos (pièces d'identité, documents KYC) et en transit (TLS 1.2+).
- Registre des traitements, analyse d'impact (AIPD), notification des violations.
- Encadrement des transferts de données hors Sénégal (choix de l'hébergement, voir section 10).

### 7.2 AML/KYC (LBC/FT)

| Exigence | Mise en œuvre |
| --- | --- |
| Identification des clients | KYC au guichet et KYC mobile progressif ; vérification de la validité de la pièce |
| Filtrage sanctions et PEP | Contrôle de l'expéditeur et du bénéficiaire à chaque opération (listes ONU, UE, OFAC, nationales) ; blocage en cas de correspondance |
| Seuils et plafonds | Plafonds par opération, jour et mois, par niveau KYC et par agence |
| Surveillance des transactions | Règles : fractionnement, fréquence anormale, montants ronds répétés, corridors à risque, bénéficiaires multiples |
| Gestion des alertes | File de revue pour le responsable conformité, statut et justification tracés |
| Déclaration de soupçon | Export du dossier vers la CENTIF |
| Conservation | Données KYC et transactions conservées au moins 10 ans après la fin de la relation |
| Travel rule | Données expéditeur/bénéficiaire transmises avec chaque transfert transfrontalier |

### 7.3 Autres obligations

- Reporting périodique à la BCEAO (volumes, encours de monnaie électronique, incidents).
- Cantonnement : encours des wallets couverts à 100 % par un compte de cantonnement en banque, rapprochement quotidien.
- Collecte et reversement des taxes TTA/TAF à l'administration fiscale avec un état détaillé.

## 8. Exigences non fonctionnelles

| Domaine | Exigence cible |
| --- | --- |
| Disponibilité | 99,9 % mensuel pour les API de transaction ; maintenance planifiée hors heures d'ouverture |
| Performance | Devis < 300 ms, validation d'une transaction < 2 s au 95e centile |
| Capacité | 50 transactions/s en pointe au lancement, architecture extensible horizontalement |
| Intégrité | Grand livre en partie double, transactions ACID, idempotence de toutes les opérations financières |
| Réseau | Apps mobiles utilisables en 3G et en connexion instable (reprise, retry idempotent, payloads légers) |
| Sécurité | OWASP ASVS niveau 2, MFA pour tout utilisateur interne, tests d'intrusion avant production |
| Traçabilité | Journal d'audit inaltérable de toute action (qui, quoi, quand, avant/après) |
| Sauvegarde | RPO ≤ 5 min, RTO ≤ 1 h ; sauvegardes chiffrées et testées chaque mois |
| Compatibilité | Android 8+, iOS 15+, navigateurs Chrome/Edge/Firefox récents |
| Ergonomie | Interface en français, parcours guichet en moins de 2 minutes, accessibilité WCAG AA pour le Web |
| Exploitabilité | Supervision, alertes, logs centralisés, tableaux de bord techniques |

## 9. Spécifications techniques

L'architecture cible est un backend unique exposé par une passerelle API, organisé en modules métier, dont le grand livre comptable est la seule source de vérité des soldes.

```
[Back-office Web] [Portail Agence Web] [App Agence Flutter] [App Client Flutter]
                         │  HTTPS
      Passerelle API (rate limiting) + Keycloak (OIDC, MFA, RBAC)
                         │
 Backend Spring Boot (monolithe modulaire)
   Paramétrage · Transfert · Wallet · Conformité · Grand livre · Notification
                         │
 PostgreSQL · Redis · RabbitMQ · Stockage objet (KYC)
                         │
 Externes : SMS/Push · KYC & sanctions · Banque (cantonnement) · Partenaires payeurs
```

Les quatre applications n'accèdent jamais à la base directement ; tout mouvement d'argent passe par le module Grand livre dans une transaction de base de données.

### 9.1 Principes d'architecture

- **Grand livre en partie double** : chaque opération produit des écritures équilibrées (wallet client, float agence, compte de frais, comptes de taxes TTA/TAF/THZUEMOA, compte de transit des transferts). Les soldes sont dérivés des écritures, jamais modifiés à la main.
- **Idempotence** : toute requête financière porte un en-tête `Idempotency-Key` ; un rejeu renvoie le résultat initial sans double débit.
- **Concurrence** : verrouillage pessimiste (`SELECT … FOR UPDATE`) sur les comptes débités, ordre de verrouillage déterministe pour éviter les interblocages.
- **Machine à états** des transactions : transitions autorisées explicites, chaque transition journalisée.
- **Outbox transactionnelle** : les événements (SMS, AML, webhooks) sont écrits dans la même transaction que l'opération puis publiés vers RabbitMQ.
- **Montants** en décimal exact (`NUMERIC(19,4)` en base, `BigDecimal` en code) avec code devise ISO 4217.

### 9.2 API (REST, OpenAPI 3)

| Domaine | Exemples d'endpoints |
| --- | --- |
| Devis | `POST /v1/quotes` — frais, taxes, taux, montant à payer |
| Transferts | `POST /v1/transfers`, `GET /v1/transfers/{code}`, `POST /v1/transfers/{code}/payout`, `POST /v1/transfers/{id}/cancel` |
| Clients / KYC | `POST /v1/customers`, `POST /v1/customers/{id}/kyc-documents` |
| Wallet | `GET /v1/wallets/me`, `POST /v1/wallets/p2p`, `GET /v1/wallets/me/transactions` |
| QR Code | `POST /v1/qr/cash-out`, `POST /v1/qr/{token}/scan`, `POST /v1/qr/{token}/confirm` |
| Agences / caisse | `GET /v1/agencies/{id}/float`, `POST /v1/tills/{id}/close` |
| Paramétrage | `/v1/admin/countries`, `/v1/admin/fee-grids`, `/v1/admin/tax-rates`, `/v1/admin/fx-rates` |
| Rapports | `GET /v1/reports/dashboard?period=day\|week\|month\|year` |

Conventions : versionnement dans l'URL, pagination par curseur, erreurs au format RFC 7807, horodatages ISO 8601 UTC, webhooks signés HMAC pour les partenaires.

### 9.3 Sécurité

- **Authentification** : OIDC via Keycloak ; MFA (TOTP) pour tous les utilisateurs internes ; PIN + appareil enrôlé + biométrie optionnelle pour le mobile ; jetons d'accès courts (5–15 min) et rotation des refresh tokens.
- **Autorisation** : RBAC + périmètre (un agent ne voit que son agence) contrôlé côté serveur.
- **Mobile** : certificate pinning, détection root/jailbreak, obfuscation, pas de données sensibles en clair sur l'appareil, attestation d'appareil (Play Integrity / App Attest).
- **QR Code** : jeton aléatoire à usage unique, signé, expirant en 5 minutes, lié à l'émetteur ; confirmation PIN obligatoire.
- **Données** : chiffrement au repos (disque + colonnes sensibles), clés dans Vault/KMS, masquage dans les logs, séparation des environnements.
- **Anti-fraude** : limitation de débit, blocage après 3 PIN erronés, alertes sur connexions inhabituelles, vélocité par appareil.
- **Audit** : journal en ajout seul, horodaté, exporté vers un stockage immuable.

### 9.4 Environnements et qualité

- Environnements : développement, recette, pré-production (données anonymisées), production.
- Tests : unitaires (couverture ≥ 80 % sur les modules Grand livre et Taxes), intégration (Testcontainers), contrats d'API, end-to-end, charge (k6/Gatling), test d'intrusion externe.

## 10. Proposition de stack technologique

Recommandation : un backend JVM (Spring Boot) sur PostgreSQL, un front Web React, et une seule base de code mobile Flutter pour les deux apps — robuste pour la finance, cohérent avec une équipe déjà à l'aise sur la JVM et PostgreSQL.

| Couche | Choix recommandé | Pourquoi | Alternative |
| --- | --- | --- | --- |
| Backend / API | Java 21 + Spring Boot 3 (monolithe modulaire, Spring Modulith) | Transactions ACID, `BigDecimal`, écosystème sécurité mature (Spring Security), référence en fintech | Kotlin + Spring Boot ; NestJS (TypeScript) |
| Base de données | PostgreSQL 16 | ACID, verrous fins, JSONB, partitionnement des tables d'opérations | — |
| Cache, verrous, limitation de débit | Redis | Jetons QR à courte durée, OTP, rate limiting, idempotence | — |
| Messagerie / événements | RabbitMQ (V1) | Notifications, AML asynchrone, webhooks partenaires ; outbox transactionnelle | Kafka si volume élevé (V2) |
| Identité et accès | Keycloak (OIDC/OAuth2) | SSO back-office, MFA, RBAC, gestion des sessions | Spring Authorization Server |
| Front Web (Admin, Support, Agence) | React + TypeScript + Vite, TanStack Query, shadcn/ui + Tailwind | Riche en tableaux et formulaires, grand vivier de développeurs | Vue 3 + shadcn-vue |
| Graphiques Web | Apache ECharts | Tableaux de bord performants | Recharts |
| Mobile (Agence + Client) | Flutter (Dart) | Une base pour Android et iOS, performant sur appareils d'entrée de gamme, scan QR (mobile\_scanner), biométrie (local\_auth) | React Native |
| Stockage sécurisé mobile | flutter\_secure\_storage (Keystore/Keychain) | Jetons et clés d'appareil | — |
| Documents KYC | Stockage objet S3-compatible (MinIO ou cloud) chiffré | Pièces d'identité, selfies | — |
| Vérification d'identité | Fournisseur KYC (OCR + liveness) | Accélère le KYC mobile niveau 2 | Revue manuelle en V1 |
| Filtrage sanctions/PEP | Service spécialisé via API, ou listes importées + recherche floue | Couverture et mises à jour des listes | — |
| SMS / OTP / push | Agrégateur SMS local + Firebase Cloud Messaging | Couverture des opérateurs sénégalais | — |
| Rapports | JasperReports (PDF), Apache POI (Excel) | Reçus, états fiscaux et réglementaires | — |
| Conteneurs et déploiement | Docker + Kubernetes (ou Docker Compose au démarrage) | Montée en charge progressive | — |
| CI/CD | GitLab CI ou GitHub Actions ; SonarQube, OWASP Dependency-Check, Trivy | Qualité et sécurité à chaque livraison | — |
| Secrets et clés | HashiCorp Vault (ou KMS du cloud) | Clés de signature QR, chiffrement des données | — |
| Observabilité | Prometheus + Grafana, Loki, OpenTelemetry, Sentry | Métriques, logs, traces, erreurs mobiles | — |
| Hébergement | Datacenter au Sénégal (souveraineté des données) ou cloud avec région Afrique | Conformité CDP, latence | À trancher avec le juriste |

**Pourquoi pas Grails pour ce projet ?** Grails reste possible (même JVM), mais Spring Boot offre plus de bibliothèques financières et de sécurité, un support à long terme plus large et un recrutement plus facile, ce qui compte pour un produit régulé.

**Pourquoi un monolithe modulaire ?** Une équipe de taille réduite livre plus vite avec un seul déploiement ; les modules (Ledger, Transfert, Wallet, Conformité, Paramétrage, Notification) ont des frontières nettes et pourront être extraits en services si la charge l'exige.

## 11. Planning, livrables et points ouverts

Estimation indicative : environ 9 mois jusqu'au lancement commercial avec une équipe de 6 à 8 personnes (2 backend, 1 front Web, 2 mobile, 1 QA, 1 DevOps à temps partiel, 1 chef de projet/PO).

1. **Cadrage et conception** (6 semaines) : ateliers métier, validation juridique, maquettes UX, modèle de données, contrats d'API.
2. **Socle technique** (4 semaines) : infrastructure, CI/CD, Keycloak, grand livre, paramétrage.
3. **Lot 1 — Transfert en agence** (10 semaines) : back-office admin, portail agence, envoi/paiement, taxes, caisse, reçus.
4. **Lot 2 — Wallet et mobile** (10 semaines) : app client, app agence, P2P, cash-in/cash-out QR Code, KYC mobile.
5. **Lot 3 — Conformité et pilotage** (6 semaines, en partie en parallèle) : AML, support, tableaux de bord, rapports réglementaires.
6. **Recette, sécurité et pilote** (6 semaines) : tests de charge, test d'intrusion, pilote sur 3 à 5 agences.
7. **Généralisation** : déploiement progressif, formation, support.

**Livrables** : spécifications détaillées, maquettes, code source, spécification OpenAPI, apps publiées (Play Store, App Store), documentation d'exploitation, plan de tests et PV de recette, rapport de test d'intrusion.

**Points ouverts à trancher**

- [ ] Statut réglementaire : agrément EME propre ou partenariat avec une banque/EME agréé ?
- [ ] Base exacte de la TAF : frais uniquement (hypothèse retenue) ou montant envoyé ?
- [ ] Taux, base et plafond de la THZUEMOA, et corridors concernés.
- [ ] Taxes payées par l'expéditeur (hypothèse) ou déduites du montant reçu ?
- [ ] TTA et TAF appliquées aussi aux opérations wallet (P2P, cash-in, cash-out) ?
- [ ] Liste des pays et corridors du lancement, et partenaires payeurs à l'étranger.
- [ ] Seuils KYC et plafonds par niveau.
- [ ] Hébergement : Sénégal ou cloud régional.
- [ ] Intégration Mobile Money (Orange Money, Wave…) dès la V1 ou en V2 ?
