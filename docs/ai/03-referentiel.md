# 03 — Référentiel (pays, régions, villes, devises, corridors)

## Objectif
Construire les données de référence administrables utilisées par la tarification, les transferts et
les formulaires.

## Prérequis
01 terminé (02 recommandé pour la sécurité des endpoints admin).

## Références
Cahier §3.1, §3.2, §5.1 ADM-03.

## Modèle de données
- `currencies` (code ISO 4217 PK, name, minor_units, rounding_mode, active)
- `countries` (iso2 PK, iso3, name_fr, dial_code, currency_code, is_uemoa bool, can_send bool,
  can_receive bool, active)
- `regions` (id, country_iso2, name, active)
- `cities` (id, region_id, country_iso2, name, active)
- `id_document_types` (code PK ex. `CNI_CEDEAO`, `PASSPORT`, `CONSULAR_CARD`, `DRIVING_LICENSE`,
  label_fr, allowed_countries jsonb, requires_expiry bool, active)
- `corridors` (id, origin_country, destination_country, send_currency, receive_currency,
  is_cross_uemoa computed = origine ou destination hors UEMOA, max_amount, active)

## Tâches
1. Entités, repositories, services, contrôleurs CRUD admin avec validation (zod côté Web plus tard).
2. Seed Flyway (`R__referential_seed.sql`) : devises XOF, EUR, USD, GNF, GHS, NGN, MAD, CAD ; les
   8 pays UEMOA (SN, CI, ML, BF, NE, TG, BJ, GW) avec `is_uemoa=true` ; FR, US, GN, GH, NG, MA, CA,
   ES, IT ; régions du Sénégal (14) et leurs chefs-lieux ; types de pièces ci-dessus ;
   corridors de départ SN→(pays UEMOA, FR, GN).
3. Endpoint public de lecture mis en cache (cache Spring local puis Valkey, 10 min, invalidation sur écriture).
4. Désactivation logique uniquement (jamais de suppression si référencé).
5. Exposer une façade `ReferentialApi` (module `api`) : `findCorridor(origin, dest)`,
   `currencyOf(country)`, `isUemoa(country)`, `roundingFor(currency)`.

## API
- `GET /api/v1/referential/{countries|regions|cities|currencies|id-document-types|corridors}`
  (filtres `?country=`, `?region=`, `?canSend=`)
- `POST|PATCH /api/v1/admin/referential/...` (rôle `ADMIN`)

## Règles métier
- Un corridor n'est actif que si ses deux pays sont actifs et respectivement `can_send`/`can_receive`.
- `is_cross_uemoa` détermine l'application de la taxe THZUEMOA (prompt 05).

## Critères d'acceptation
- Le Back-office peut ajouter une ville, désactiver un pays, créer un corridor.
- `ReferentialApi.findCorridor("SN","ML")` renvoie XOF→XOF, `is_cross_uemoa=false` ;
  `SN→FR` renvoie XOF→EUR, `true`.

## Tests exigés
Repository + contrôleur (validation, droits), cache invalidé, seed chargé.

## Hors périmètre
Frais, taxes, change (05).


## Mise à jour multi-pays (2026-10-16) — prévaut sur les sections ci-dessus

- `zones` (code `UEMOA|CEMAC|CEEAC_OTHER|OTHER`, name, central_bank, supervisor).
- `countries` : ajouter `zone_code`, `status` (`INACTIVE|DESTINATION_ONLY|OPERATING`),
  `default_timezone`, `default_locale`, `available_locales` jsonb, `national_number_pattern`,
  `regulator`, `fiu_name` (CENTIF, ANIF…), `opening_checklist` (calculée). Remplacer `is_uemoa`
  par `zone_code`.
- `country_timezones` (country_iso2, timezone) pour les pays multi-fuseaux (CD).
- `corridors.zone_relation` calculé : `INTRA_ZONE|INTER_ZONE|OUT_OF_REGION` (remplace `is_cross_uemoa`).
- Seed : UEMOA (SN, CI, ML, BF, NE, TG, BJ, GW — XOF), CEMAC (CM, GA, CG, TD, CF, GQ — XAF),
  CEEAC hors CEMAC (CD — CDF, AO — AOA, BI — BIF, RW — RWF, ST — STN), destinations hors zone ;
  fuseaux IANA (ex. Africa/Dakar, Africa/Abidjan, Africa/Niamey, Africa/Douala, Africa/Kinshasa,
  Africa/Lubumbashi, Africa/Kigali) ; indicatifs ; SN seul `OPERATING`.
- Endpoint `GET /api/v1/admin/countries/{iso2}/opening-checklist` + transition `OPERATING`
  refusée tant que la checklist n'est pas verte (taxes, plafonds, cantonnement, route SMS testée,
  pièces, modèles de messages dans la locale par défaut).
- Règle V1 (prévaut sur « Règles métier » ci-dessus) : un corridor n'est activable que si les deux
  pays sont `OPERATING` ; `DESTINATION_ONLY` réservé aux partenaires payeurs (V2).
- Corridors du seed : SN→SN actif ; les autres corridors (SN→pays UEMOA, SN→CM, SN→FR, SN→GN) créés
  inactifs. Le jeu de données de recette passe ML et CM en `OPERATING` et active SN→ML et SN→CM.
