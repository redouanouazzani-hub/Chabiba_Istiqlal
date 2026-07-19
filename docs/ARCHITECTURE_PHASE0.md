# Portail de la Jeunesse Istiqlalienne — Architecture & Modèle de données (Phase 0)

**Stack validée :** Laravel 12 · PHP 8.3 · Filament 3 (back-office) · Livewire 3 + Blade + Tailwind (front public, RTL) · PostgreSQL · Redis · Meilisearch · Leaflet/OSM
**Rôles :** Claude = développement · Lovable = exploration UI (maquettes) · Ressource interne = relecture & maintenance

---

## 1. Vue d'ensemble du modèle de données

Le modèle s'articule autour de **4 axes** issus des notes de cadrage :

1. **Territoire** — l'épine dorsale : tout (membres, rôles, activités, statistiques, validation) est rattaché à un niveau national / régional / provincial / local.
2. **Personnes** — utilisateurs, adhérents, demandes d'adhésion, cartes de membre.
3. **Vie de l'organisation** — activités, inscriptions, congrès, contenus (actualités, documents, médias), participation (propositions, sondages, formations).
4. **Gouvernance & sécurité** — rôles/permissions territorialisés, audit, notifications.

---

## 2. Axe Territoire

### `regions`
| Champ | Type | Notes |
|---|---|---|
| id | bigint PK | |
| name_fr / name_ar | string | Bilingue |
| slug | string unique | |
| latitude / longitude | decimal nullable | Carte des sections |
| sort_order | smallint | |

Les 12 régions administratives du Maroc, seedées dès le départ.

### `provinces`
| Champ | Type | Notes |
|---|---|---|
| id | bigint PK | |
| region_id | FK regions | cascade restrict |
| name_fr / name_ar | string | |
| slug | string unique | |

### `sections`
| Champ | Type | Notes |
|---|---|---|
| id | bigint PK | |
| province_id | FK provinces | |
| name_fr / name_ar | string | |
| slug | string unique | |
| address_fr / address_ar | text nullable | |
| latitude / longitude | decimal nullable | Affichage Leaflet |
| contact_email / contact_phone | string nullable | Annuaire de contact |
| is_active | boolean | |

> **Règle métier :** un membre appartient à une **section** ; la région et la province se déduisent par jointure. Les droits d'un administrateur régional couvrent toutes les sections de sa région (scoping automatique via Global Scopes Eloquent).

---

## 3. Axe Personnes

### `users`
Compte d'accès (authentification). Distinct de la fiche adhérent : un visiteur peut créer un compte pour déposer une demande avant d'être membre.

| Champ | Type | Notes |
|---|---|---|
| id | bigint PK | |
| name | string | |
| email | string unique | |
| phone | string nullable, unique | Notifications SMS |
| password | string | bcrypt/argon2id |
| locale | enum(fr, ar) | Langue préférée |
| two_factor_secret / recovery_codes | text nullable chiffrés | 2FA (Fortify) — **obligatoire pour tout rôle admin** |
| email_verified_at | timestamp | |
| is_active | boolean | Désactivation sans suppression |

### `membership_applications` — demandes d'adhésion (module Adhésion)
| Champ | Type | Notes |
|---|---|---|
| id | bigint PK | |
| user_id | FK users | Le demandeur |
| section_id | FK sections | Choisie dans le formulaire |
| first_name_fr / last_name_fr | string | |
| first_name_ar / last_name_ar | string | |
| cin_number | string **chiffré** (cast encrypted) | |
| birth_date | date | Contrôle tranche d'âge de l'organisation |
| gender | enum | Statistiques exigées (âge/genre/région) |
| address, city | string | |
| cin_document_path | string | Stockage **disque privé chiffré**, servi par URL signée temporaire |
| photo_path | string | Disque privé |
| signature_data | text nullable | Signature électronique (canvas) |
| status | enum(draft, submitted, under_review, approved, rejected) | Machine à états |
| reviewed_by | FK users nullable | Responsable local ayant statué |
| reviewed_at | timestamp nullable | |
| rejection_reason_fr / _ar | text nullable | |

**Workflow :** submitted → notification au responsable local de la section → approved crée automatiquement le `member` + la `membership_card` → notification au candidat. Chaque transition est journalisée (activitylog).

### `members` — fiche adhérent
| Champ | Type | Notes |
|---|---|---|
| id | bigint PK | |
| user_id | FK users unique | |
| section_id | FK sections | |
| application_id | FK membership_applications | Traçabilité |
| member_number | string unique | Format ex. `JI-2026-000001` |
| joined_at | date | |
| status | enum(active, expired, suspended) | Piloté par la cotisation |
| directory_visibility | enum(hidden, members_only, full) | Confidentialité annuaire |
| bio, profile_photo_path | nullable | Profil espace membre |

### `membership_cards` — carte de membre numérique
| Champ | Type | Notes |
|---|---|---|
| id | bigint PK | |
| member_id | FK members | |
| card_uid | uuid unique | Contenu du QR code |
| qr_signature | string | **HMAC-SHA256** (clé serveur) — vérification hors ligne impossible à falsifier |
| valid_from / valid_until | date | Suit la période de cotisation |
| status | enum(active, expired, revoked) | |

Endpoint public `GET /verify/{card_uid}` : affiche validité + nom + section (aucune donnée sensible), après vérification de signature.

### `contributions` — cotisations (sans paiement en ligne)
| Champ | Type | Notes |
|---|---|---|
| id | bigint PK | |
| member_id | FK members | |
| period_year | smallint | |
| amount | decimal | |
| method | enum(cash, transfer, exemption) | Encaissement hors ligne, **enregistré** par le responsable local |
| recorded_by | FK users | |
| receipt_number | string nullable | |
| recorded_at | timestamp | |

> Le paiement en ligne étant hors périmètre, ce module trace les cotisations encaissées physiquement et pilote le statut du membre + les rappels automatiques.

### `badges` & `member_badge` (pivot)
Badges de participation (espace membre) : `code`, `name_fr/ar`, `icon`, `rule_type` (ex. N activités suivies, formation complétée), attribution automatique via listeners d'événements.

---

## 4. Axe Vie de l'organisation

### `activities` — agenda & événements (module Vie associative)
| Champ | Type | Notes |
|---|---|---|
| id | bigint PK | |
| title_fr / title_ar | string | |
| description_fr / description_ar | text | |
| type | enum(meeting, training, congress, campaign, social, other) | Filtre agenda |
| level | enum(national, regional, local) | |
| region_id / section_id | FK nullable | Selon le niveau |
| venue_fr / venue_ar | string | |
| latitude / longitude | decimal nullable | |
| starts_at / ends_at | datetime | |
| capacity | int nullable | Quota |
| registration_open | boolean | |
| poster_path | string nullable | Affiche téléchargeable |
| status | enum(draft, published, cancelled, completed) | |
| created_by | FK users | |

### `activity_registrations`
`activity_id`, `member_id`, `status` enum(**confirmed, waitlisted**, cancelled, attended), `position` (ordre liste d'attente), timestamps. Promotion automatique du premier de la liste d'attente en cas de désistement (job + notification).

### `activity_reports` — comptes-rendus post-événement
`activity_id`, `summary_fr/ar`, `attendees_count`, `pdf_path` (généré automatiquement), photos via medialibrary.

### `congresses` — archives des congrès
`edition_number` (ex. 14), `name_fr/ar`, `city`, `held_from/held_to`, `summary`, documents & résolutions rattachés via `documents` + medialibrary. Le 14ᵉ Congrès (Salé, 10-11 juillet 2026) est le premier enregistrement seedé.

### `posts` — actualités & communiqués
`type` enum(news, communique, press_release), `title_fr/ar`, `slug`, `excerpt`, `body_fr/ar` (éditeur riche), `cover_path`, `is_featured` (à la une accueil), `published_at`, `author_id`. Indexé Meilisearch.

### `documents` — bibliothèque documentaire (module Médiathèque)
| Champ | Type | Notes |
|---|---|---|
| id | bigint PK | |
| category | enum(statutes, minutes, speech, charter, report, resolution, press) | Catégories de la note de cadrage |
| title_fr / title_ar | string | |
| file (medialibrary) | | PDF/DOCX |
| visibility | enum(**public, members, admins**) | Contrôle d'accès par document |
| region_id | FK nullable | Filtre par région |
| documentable | morphs nullable | Rattachement congrès/activité |
| published_at | date | |

### `albums` + medialibrary — galerie photos ; `videos` — vidéothèque (URL YouTube ou fichier), `theme`, bilingue.

### Participation (module Espace participatif)
- **`proposals`** : `member_id`, `title_fr/ar`, `body`, `status` enum(pending, published, under_study, adopted, rejected), modération avant publication.
- **`proposal_votes`** : pivot unique (proposal, member).
- **`polls` / `poll_options` / `poll_votes`** : sondages, résultats agrégés en temps réel (Livewire polling), vote unique par membre, résultats anonymisés.
- **`courses` / `course_enrollments`** : catalogue de formations (leadership, plaidoyer, numérique), inscription, `completed_at`, génération d'**attestation PDF** (dompdf) avec numéro vérifiable.
- **`discussion_threads` / `discussion_messages`** : fils par thématique, `is_approved` (modération a priori), modérateurs = rôle dédié.

### Contact & newsletter
- **`contact_messages`** : `topic` enum, `region_id` nullable, routage automatique → assignation au responsable concerné (`assigned_to`), `status` (new, in_progress, answered).
- **`newsletter_subscribers`** : email, `locale`, `confirmed_at` (double opt-in), token de désinscription.
- **`faqs`** : question/réponse bilingues, `sort_order`, recherche par mot-clé.

### Organigramme (module Accueil)
**`leadership_positions`** : `person_name_fr/ar`, `title_fr/ar` (Secrétaire Général, membre du BE…), `photo`, `message_fr/ar` (mot du SG), `sort_order`, `congress_id` (organigramme historisé par mandat). Les 29 membres du Bureau Exécutif issus de la démo seront seedés.

---

## 5. Axe Gouvernance & sécurité

### Rôles (spatie/laravel-permission) + scoping territorial

| Rôle | Portée | Capacités clés |
|---|---|---|
| `super_admin` | — | Tout, gestion technique |
| `national_admin` | national | Contenus nationaux, stats globales, gestion des rôles, exports |
| `regional_admin` | 1 région | Membres & activités de sa région, stats régionales |
| `local_admin` | 1 section | **Validation des adhésions** de sa section, activités locales, enregistrement cotisations |
| `moderator` | national ou régional | Modération propositions/discussions |
| `editor` | national | Actualités, médiathèque, FAQ |
| `member` | — | Espace membre, participation |

Le rattachement territorial se fait par la table **`role_assignments`** (`user_id`, `role`, `scope_type` region/section nullable, `scope_id`) et s'applique via un middleware + Global Scopes : un `local_admin` ne voit **jamais** une demande hors de sa section, par construction et pas seulement par interface.

### Sécurité intégrée dès la phase 0
- 2FA (TOTP) **imposée** à tout compte disposant d'un rôle admin (middleware bloquant).
- Champs sensibles (CIN) : cast `encrypted` Laravel (AES-256, clé hors base).
- Pièces d'identité : disque `private` (hors webroot), accès uniquement par **URL signée** expirant en 5 min, réservé aux valideurs du ressort.
- `spatie/activitylog` sur : applications, members, contributions, roles, documents internes — journal exigé par la note "Admin", consultable dans Filament, **non modifiable** (aucune route d'édition/suppression).
- Rate limiting : formulaires publics (adhésion, contact, newsletter) + throttle login + verrouillage progressif.
- En-têtes : CSP stricte, HSTS, X-Frame-Options, Referrer-Policy (middleware dédié).
- Uploads : validation MIME réelle (finfo), taille max, re-encodage des images (supprime les payloads), noms de fichiers aléatoires.
- Sauvegardes : `spatie/laravel-backup` chiffrées, quotidiennes, copies hors site, test de restauration mensuel.
- Registre de traitement + dossier **CNDP (loi 09-08)** : données d'appartenance politique = données sensibles → consentement explicite dans le formulaire, finalités documentées, durées de conservation définies (à valider avec l'organisation).

---

## 6. Arborescence Laravel du projet

```
portail-ji/
├── app/
│   ├── Enums/                    # ApplicationStatus, ActivityLevel, Visibility…
│   ├── Models/                   # ~30 modèles ci-dessus
│   ├── Filament/
│   │   ├── Resources/            # Un Resource par entité admin
│   │   ├── Pages/Dashboard.php   # KPIs par ressort territorial
│   │   └── Widgets/              # Stats adhésions (région/âge/genre), activité
│   ├── Livewire/
│   │   ├── Public/               # Accueil, Agenda, Actualités, Médiathèque, FAQ…
│   │   ├── Membership/           # Formulaire multi-étapes, suivi de demande
│   │   ├── Member/               # Dashboard, carte, annuaire, profil
│   │   └── Participation/        # Propositions, sondages, formations, discussions
│   ├── Policies/                 # Autorisation fine par modèle (scoping territorial)
│   ├── Services/
│   │   ├── MembershipService.php # Machine à états de la demande
│   │   ├── CardService.php       # Génération QR + signature HMAC
│   │   ├── StatsService.php      # Agrégats démographiques
│   │   └── WaitlistService.php   # Quotas & liste d'attente
│   ├── Notifications/            # Email/SMS/database (statut demande, rappels…)
│   └── Http/Middleware/          # SecurityHeaders, EnforceTwoFactorForAdmins, SetLocale
├── database/
│   ├── migrations/               # ~35 migrations, ordre : territoire → users → membres → contenus
│   └── seeders/                  # Régions/provinces, rôles, organigramme (29 BE), 14ᵉ Congrès
├── resources/views/
│   ├── layouts/ (app + rtl)      # dir="rtl" automatique si locale=ar
│   └── livewire/…
├── lang/fr/ · lang/ar/
└── tests/Feature/                # Workflow adhésion, scoping des rôles, vérif QR, quotas
```

**Routage bilingue :** préfixe `/fr` et `/ar`, middleware `SetLocale`, `<html dir="rtl">` en arabe, Tailwind logical properties (`ms-*`, `me-*`) pour un RTL sans dédoublement de CSS.

---

## 7. Ordre de développement (rappel feuille de route)

| Étape | Contenu | Sortie |
|---|---|---|
| 0.1 | Init projet, auth + 2FA, migrations territoire & users, seeders régions/rôles | Base démarrable |
| 0.2 | Filament + RBAC territorial + audit + middleware sécurité | Back-office squelette sécurisé |
| 1.1 | Accueil public (bannière, actualités, organigramme, accès rapides) + Contact/FAQ/newsletter | Portail vitrine en ligne |
| 1.2 | Adhésion : formulaire multi-étapes → validation locale → membre + carte QR + page de vérification | Campagne d'adhésion digitale |
| 2.x | Espace membre, Vie associative, Médiathèque, stats & exports admin | Vie interne complète |
| 3.x | Participation, recherche Meilisearch, notifications SMS, durcissement final, formation admins | Portail complet |

---

*Document de phase 0 — à faire valider par le Bureau Exécutif avant lancement du développement (étape 0.1).*
