# Ecclesia — Contexte projet

## Stack
- **Frontend** : `index.html` unique (SPA vanilla JS, ~1144 lignes) — pas de framework, pas de build tool
- **Backend** : Supabase (Postgres + Realtime + RLS)
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

### Vue `assertion_stats`
Agrège assertions + votes : `total_votes`, `nb_accord`, `nb_desaccord`, `nb_passe`, par assertion.

### RLS résumé
- **sessions** : lecture publique (sauf archived), insert/update libres
- **assertions** : lecture des `approved`, insert si `pending`, update libre (modération)
- **votes** : lecture et insert libres, UNIQUE empêche le double vote
- **config** : lecture et update libres

---

## Architecture du code (index.html)

### État global (`state`)
```js
state = {
  loading, view,           // vue courante : 'onboarding' | 'session-pick' | 'main' | 'admin'
  pseudo, adminMode,
  sessions, currentSessionId,
  assertions, myVotes,     // assertions approuvées + votes du pseudo courant
  allAssertions,           // toutes assertions (admin)
  moderationPolicy,        // 'open' | 'closed' | 'ai'
  activeTab,               // onglet vote courant : 'vote' | 'results' | 'submit'
  adminTab,                // onglet admin : 'sessions' | 'moderation' | 'policy' | 'merge'
  mergeTargets, aiLoading, aiResult,  // fusion IA
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
          ├── tab: vote      (voter sur les assertions)
          ├── tab: results   (voir les stats)
          └── tab: submit    (soumettre une assertion)
      → admin
          ├── tab: sessions    (créer / gérer les sessions)
          ├── tab: moderation  (approuver / rejeter les assertions)
          ├── tab: policy      (politique de modération)
          └── tab: merge       (fusion IA)
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

## Fonctionnalités prévues / en cours
- [ ] Modération IA automatique (bouton désactivé, `disabled: true`)
- [ ] ...
