# Ecclesia — Contexte projet

## Stack
- **Frontend** : `index.html` unique (SPA vanilla JS) — pas de framework, pas de build tool
- **Backend** : Supabase (Postgres + Realtime + RLS + Auth)
- **Hébergement** : GitHub Pages
- **Repo** : https://github.com/Ecclesia-CS/Vote-assertions
- **Code source** : https://raw.githubusercontent.com/Ecclesia-CS/Vote-assertions/main/index.html
- **Supabase URL** : https://fcdhbgsqzvxepzvjweod.supabase.co

---

## Schéma BDD

### `sessions`
| Colonne | Type | Notes |
|---|---|---|
| id | BIGSERIAL PK | |
| title | TEXT | |
| description | TEXT | |
| status | TEXT | `open` / `closed` / `archived` |
| position | INTEGER | ordre d'affichage |
| created_at | TIMESTAMPTZ | |

### `assertions`
| Colonne | Type | Notes |
|---|---|---|
| id | BIGSERIAL PK | |
| session_id | BIGINT FK → sessions | |
| text | TEXT | |
| author | TEXT | |
| status | TEXT | `pending` / `approved` / `rejected` |
| created_at | TIMESTAMPTZ | |

### `votes`
| Colonne | Type | Notes |
|---|---|---|
| id | BIGSERIAL PK | |
| assertion_id | BIGINT FK → assertions | CASCADE |
| session_id | BIGINT FK → sessions | |
| pseudo | TEXT | |
| vote | TEXT | `accord` / `desaccord` / `passe` |
| created_at | TIMESTAMPTZ | |
| — | UNIQUE (assertion_id, pseudo) | un vote par assertion par pseudo |

### `config`
| Colonne | Type | Notes |
|---|---|---|
| key | TEXT PK | ex : `moderation_policy` |
| value | TEXT | `open` / `closed` / `ai` |

### `admin_roles`
| Colonne | Type | Notes |
|---|---|---|
| id | UUID PK | gen_random_uuid() |
| email | TEXT UNIQUE | email autorisé comme animateur |
| created_at | TIMESTAMPTZ | |

Animateurs autorisés : `ecclesia.debat@ik.me`, `jules.becquemont@gmail.com`, `max.reinaudo@gmail.com`, `antoine.doumas92@gmail.com`, `lyssandre.school@icloud.com`

### Vue `assertion_stats`
Agrège assertions + votes : `total_votes`, `nb_accord`, `nb_desaccord`, `nb_passe`, par assertion.

### RLS résumé
- **sessions** : lecture publique, insert/update réservés aux admins authentifiés (`admin_roles`)
- **assertions** : lecture des `approved`, insert libre (`pending`), update réservé aux admins
- **votes** : lecture et insert libres, UNIQUE empêche le double vote
- **config** : lecture publique, update réservé aux admins
- **admin_roles** : lecture publique (`anon` + `authenticated`), pas d'insert/update côté client

---

## Authentification admin

L'accès animateur est géré via **Supabase Auth** (magic link / OTP) — plus de mot de passe en clair dans le code.

### Flow de connexion
1. L'animateur clique "🔒 Animer" → modal avec champ email
2. L'app vérifie que l'email est dans `admin_roles` (lecture anon)
3. Si autorisé : `db.auth.signInWithOtp({ email, options: { shouldCreateUser: true, emailRedirectTo: "https://ecclesia-cs.github.io/Vote-assertions/" } })`
4. L'animateur reçoit un magic link par email, clique dessus
5. Supabase redirige vers l'app avec un `?code=...` (PKCE flow)
6. `onAuthStateChange` détecte la session, vérifie `admin_roles`, active `adminMode`

### Configuration Supabase Auth
- **Flow** : PKCE (`flowType: "pkce"` dans le client)
- **Confirm email** : OFF
- **Site URL** : `https://ecclesia-cs.github.io/Vote-assertions/`
- **Redirect URLs** : `https://ecclesia-cs.github.io/Vote-assertions/`, `https://ecclesia-cs.github.io/Vote-assertions/**`
- **SMTP** : ⚠️ service intégré Supabase (rate limit : 3 emails/heure) — à migrer vers Resend pour la prod

### État admin dans `state`
```js
adminMode: false,        // true si session Auth valide + email dans admin_roles
adminUser: null,         // objet user Supabase Auth
adminEmailInput: "",     // saisie dans la modal
adminEmailSent: false,   // true après envoi du magic link
adminAuthError: "",      // message d'erreur éventuel
```

---

## Architecture du code (index.html)

### État global (`state`)
```js
state = {
  loading, view,              // vue courante
  pseudo,                     // pseudo participant (localStorage)
  adminMode, adminUser,       // auth animateur (Supabase Auth)
  adminEmailInput, adminEmailSent, adminAuthError,
  sessions, currentSessionId,
  assertions, myVotes,
  moderationPolicy,           // 'open' | 'closed' | 'ai'
  adminTab,                   // 'sessions' | 'stats' | 'moderate' | 'policy' | 'merge'
  mergeTargets, aiLoading, aiResult,
}
```

### Fonctions principales
| Fonction | Rôle |
|---|---|
| `render()` | Re-rendu complet de l'app |
| `loadSessions()` | Charge les sessions depuis Supabase |
| `loadAssertions()` | Charge les assertions approuvées de la session courante |
| `loadMyVotes()` | Charge les votes du pseudo courant |
| `loadAllAssertions()` | Charge toutes les assertions (admin) |
| `loadConfig()` | Charge la politique de modération |
| `listenAuth()` | Écoute `onAuthStateChange`, vérifie `admin_roles`, active `adminMode` |
| `checkAdminRole(email)` | Vérifie si un email est dans `admin_roles` |
| `sendMagicLink()` | Vérifie l'email puis envoie le magic link via Supabase Auth |
| `doAdminLogout()` | `db.auth.signOut()` + reset état admin |
| `handleVote(id, vote)` | Enregistre un vote |
| `handleSubmit()` | Soumet une assertion |
| `handleModerate(id, status)` | Modère une assertion (admin) |
| `handleAiMerge()` | Fusionne des assertions via LLM |
| `handleApplyMerge()` | Applique la fusion proposée par l'IA |
| `setPolicy(key)` | Change la politique de modération |
| `subscribeRealtime()` | Abonnement Supabase Realtime |

### Vues
```
onboarding → (saisie pseudo)
  → session-pick → (choix session)
      → main
          ├── tab: voter     (voter sur les assertions)
          ├── tab: résultats (voir les stats)
          └── tab: proposer  (soumettre une assertion)
      → admin (accès via magic link Supabase Auth)
          ├── tab: sessions    (créer / gérer les sessions)
          ├── tab: stats       (statistiques détaillées)
          ├── tab: modérer     (approuver / rejeter les assertions)
          ├── tab: politique   (politique de modération)
          └── tab: fusion IA   (fusion d'assertions par LLM)
```

---

## Palette & style
- Fond : `#f8f7f4` — Texte : `#1a1a1a`
- Vert (accord) : `#eaf3de` / `#3b6d11`
- Rouge (désaccord) : `#fcebeb` / `#791f1f`
- Jaune (neutre) : `#faeeda` / `#633806`
- Violet (sélection) : `#7f77dd` / `#eeedfe`
- Max-width : 480px (mobile-first)

---

## Fonctionnalités existantes
- [x] Multi-sessions avec statuts (open / closed / archived)
- [x] Vote (accord / désaccord / passe) avec stats en temps réel
- [x] Soumission d'assertions par les participants
- [x] Modération manuelle (admin)
- [x] Politique de modération (ouverte / fermée)
- [x] Fusion d'assertions par IA (LLM via API Anthropic)
- [x] Realtime Supabase
- [x] Détection de session par URL (`?session=ID`)
- [x] Authentification admin via Supabase Auth (magic link, PKCE flow)
- [x] Liste blanche animateurs via table `admin_roles`
- [x] RLS renforcée (écriture réservée aux admins authentifiés)

## Fonctionnalités prévues / en cours
- [ ] Configurer SMTP externe (Resend) pour lever le rate limit email (3/h → 100/h)
- [ ] Vérifier le flow PKCE de bout en bout une fois le rate limit levé
- [ ] Modération IA automatique (bouton désactivé, `disabled: true`)
