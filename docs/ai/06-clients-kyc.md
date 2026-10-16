# 06 — Clients, bénéficiaires et KYC (module `customer`)

## Objectif
Gérer les personnes (expéditeurs, bénéficiaires, clients wallet), leurs pièces d'identité et leur
niveau KYC, au guichet comme sur mobile.

## Prérequis
02, 03 terminés.

## Références
Cahier §3.1, §3.2, §6.2 MCL-01/02, §7.1, §7.2.

## Modèle de données
- `customers` (id, type `WALLET|WALK_IN`, first_names, last_name, birth_date, nationality,
  profession, address, city_id, country_iso2, phone E.164 unique si WALLET, email nullable,
  kyc_level `L0|L1|L2|L3`, kyc_status `NONE|PENDING|VERIFIED|REJECTED|EXPIRED`, risk_rating
  `LOW|MEDIUM|HIGH`, is_pep bool, status `ACTIVE|BLOCKED|CLOSED`, created_by_agency_id)
- `customer_id_documents` (id, customer_id, document_type_code, number **chiffré**,
  number_hash (pour recherche exacte), issuing_country, issued_at, expires_at, front_object_key,
  back_object_key, verified_at, verified_by)
- `customer_security_questions` (customer_id, question, answer_hash Argon2id)
- `beneficiaries` (id, owner_customer_id nullable, first_names, last_name, city_id nullable,
  country_iso2, phone nullable)
- `kyc_reviews` (id, customer_id, requested_level, submitted_at, decision `APPROVED|REJECTED|
  MORE_INFO`, reason, reviewer_id, decided_at, provider_reference nullable)
- `consents` (id, customer_id, purpose `TERMS|PRIVACY|MARKETING`, version, granted_at, revoked_at)

## Tâches
1. CRUD client au guichet (agent) : recherche par téléphone, nom, n° de pièce (via `number_hash`) ;
   déduplication (même pièce + même pays = même client).
2. Upload des pièces vers MinIO via URL pré-signée (PUT, 5 Mo max, `image/jpeg|png|application/pdf`),
   objets chiffrés côté serveur ; lecture via URL pré-signée courte (60 s) et journalisée.
3. KYC mobile progressif : L1 = téléphone vérifié OTP ; L2 = pièce + selfie (soumission →
   `kyc_reviews` PENDING, revue par `COMPLIANCE_OFFICER`, ou fournisseur via interface
   `IdentityVerificationProvider` avec implémentation `ManualReviewProvider` par défaut) ;
   L3 = justificatif d'adresse.
4. Plafonds par niveau KYC lus depuis la configuration de conformité (prompt 07) — exposer
   `CustomerApi.kycLevelOf(customerId)`.
5. Pièce expirée → `kyc_status=EXPIRED` ; client guichet : toute opération bloquée jusqu'à mise à
   jour de la pièce ; client wallet : niveau ramené à L1 (plafonds L1) jusqu'à mise à jour.
6. Question test : stockage haché, vérification `CustomerApi.verifySecretAnswer(...)` insensible à
   la casse et aux accents, 3 essais par transfert.
7. Droits RGPD : export JSON des données d'un client (`GET /api/v1/customers/{id}/data-export`,
   rôle conformité), anonymisation après expiration de la durée de conservation (job, 10 ans).
8. Chaque création/modification déclenche un contrôle sanctions/PEP via `ComplianceApi.screen(...)`
   (interface définie au prompt 07 ; ici, appeler un bouchon si 07 n'est pas encore livré).

## API
- `GET|POST /api/v1/customers`, `GET|PATCH /api/v1/customers/{id}`
- `POST /api/v1/customers/{id}/documents/upload-url`, `POST /api/v1/customers/{id}/documents`
- `POST /api/v1/customers/me/kyc/submit` (mobile), `GET /api/v1/customers/me`
- `GET /api/v1/compliance/kyc-reviews`, `POST /api/v1/compliance/kyc-reviews/{id}/decision`
- `GET|POST /api/v1/customers/{id}/beneficiaries`

## Critères d'acceptation
- Le n° de pièce n'apparaît jamais en clair en base, dans les logs ni dans les réponses API hors
  rôles autorisés (masqué `****1234`).
- Un client L1 ne peut pas dépasser les plafonds L1.

## Tests exigés
Chiffrement/hachage, déduplication, workflow KYC, URL pré-signées, masquage, export RGPD.

## Hors périmètre
Moteur AML (07), écrans (15, 16, 19).
