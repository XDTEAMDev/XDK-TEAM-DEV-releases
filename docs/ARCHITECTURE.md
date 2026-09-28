# Architecture — proposition v0 (en attente de validation)

> Nom de code provisoire : **Atelier** (dossier projet `.atelier/`, identifiant `com.xdkteamdev.atelier`).
> Tout ce qui dépend du nom définitif est centralisé dans une seule constante : il sera remplacé dès que le nom sera choisi.

## 1. Vue d'ensemble

```
┌──────────────────────────── Application desktop (Tauri 2) ────────────────────────────┐
│                                                                                        │
│  Front React + TS strict (WebView)          IPC typé                 Cœur Rust         │
│  ─ Accueil / projets                 ◄── commands (req/rép) ──►   ─ projects           │
│  ─ Onboarding                        ◄── channels (flux PTY) ──   ─ toolchain          │
│  ─ Atelier (onglets xterm.js)        ◄── events (état, logs)  ──  ─ pty                │
│  ─ Assets, Boutique, Publication                                  ─ devserver          │
│                                                                   ─ studio             │
│                                                                   ─ bridge (HTTP local)│
│                                                                   ─ assets / roblox    │
│                                                                   ─ secrets / account  │
└───────────────┬───────────────────────────────┬──────────────────────────┬─────────────┘
                │ spawn (PTY, PATH privé)       │ spawn                    │ HTTPS
                ▼                               ▼                          ▼
   Agents CLI (claude, codex, opencode)   rbxtsc --watch            Open Cloud, Gemini,
     │  stdio MCP                         rojo serve :port          Meshy, ElevenLabs,
     ├──► MCP « atelier » ──HTTP 127.0.0.1 + jeton──► bridge        Supabase (compte/licence)
     └──► MCP intégré de Roblox Studio ──► Studio
                                              ▲
                          rojo serve ─────────┘  (fichiers → Studio)
                          plugin compagnon ──► bridge (heartbeat, état, logs Studio)
```

Principe clé : **le cœur Rust est le seul à détenir les secrets et à parler aux API payantes**.
Les agents, le serveur MCP et le plugin Studio passent tous par le *bridge* local, qui exige un jeton aléatoire par session.

## 2. Décisions structurantes

| Sujet | Choix proposé | Raison |
|---|---|---|
| Shell desktop | **Tauri 2** | Binaire ~10 Mo, Rust pour PTY / fichiers / trousseau, pas de Node embarqué dans l'app elle-même. |
| Terminaux | `portable-pty` (ConPTY sous Windows) + `@xterm/xterm` (addons fit, webgl, web-links). Flux via `tauri::ipc::Channel` | Le Channel est ordonné et binaire, bien plus rapide que les events pour un terminal. |
| Toolchain | **Privée**, dans le dossier de données de l'app : Node LTS officiel (vérifié par `SHASUMS256.txt`), Rojo (release GitHub, SHA-256 épinglé), agents installés via `npm --prefix`. PATH injecté **uniquement** dans les processus enfants. | Aucun droit admin, aucun conflit avec une installation existante, versions épinglées, désinstallation propre. |
| roblox-ts | `devDependency` épinglée **dans chaque projet** (pas d'install globale) | Chaque projet compile avec sa version ; mises à jour maîtrisées. |
| Instructions agents | `AGENTS.md` = source unique ; `CLAUDE.md` = `@AGENTS.md` + spécificités Claude | Codex et OpenCode lisent `AGENTS.md` ; Claude Code lit `CLAUDE.md` et gère les imports `@`. |
| Outils pour agents | **Deux** serveurs MCP : le nôtre (assets, génération, upload, diagnostics, publication) **et le serveur MCP intégré à Roblox Studio** (exploration du DataModel, exécution Luau, playtest, capture) | Roblox maintient désormais un MCP officiel dans Studio : on ne le réimplémente pas. |
| Notre MCP | Adaptateur **stdio** en TypeScript (bundle esbuild, lancé avec le Node privé) qui relaie vers le bridge HTTP du cœur | Le stdio est supporté par tous les CLI ; les clés ne quittent jamais le cœur. |
| Source de vérité du jeu | **Les fichiers du projet**. Code : Rojo « fully managed ». Monde : modèles `.rbxm` + description JSON de la map | Publication reproductible (`rojo build` → Open Cloud), tout est versionné et annulable. Les éditions faites à la main dans Studio ne sont pas la référence (voir risques). |
| Auth Roblox | **OAuth 2.0 PKCE** (app enregistrée par nous) ; clé API Open Cloud en repli pour les endpoints qui n'acceptent pas l'OAuth | Zéro setup pour l'utilisateur. Contrainte : 10 utilisateurs max avant la revue de l'app par Roblox. |
| Stockage local | SQLite (`rusqlite`, FTS5) + fichiers | Index des projets, banque d'assets avec recherche plein texte, historique des publications, file de tâches. |
| Secrets | crate `keyring` (Gestionnaire d'identification Windows / Trousseau macOS) | Exigence : jamais de clé en clair. |
| Historique projet | Dépôt git dans chaque projet ; **instantanés écrits par le cœur via `gix`** (gitoxide), sans binaire git requis | Bouton « Annuler ce que l'IA vient de faire » ; les agents exploitent déjà git pour lire les diffs. |
| Front | React 19, Vite, Tailwind 4, TanStack Router + Query, Zustand, primitives Radix (style shadcn) | Standard, typé, testable avec Vitest. |
| Erreurs | `AppError` (thiserror) → `{ code, message_fr, details? }` ; détails techniques repliés + journal fichier | Jamais de stack trace brute côté utilisateur. |
| Monorepo | npm workspaces + workspace Cargo | Aucun outil supplémentaire à installer. |
| Emplacement des projets | `~/Atelier/Projets` par défaut (**hors** Documents / OneDrive / iCloud), modifiable | OneDrive qui synchronise `node_modules` = lenteurs et fichiers verrouillés sous Windows. |

## 3. Arborescence du dépôt

```
.
├─ apps/
│  └─ desktop/
│     ├─ src/                         # Front
│     │  ├─ app/                      # routes, layout, providers
│     │  ├─ features/
│     │  │  ├─ projects/              # P1 accueil, création
│     │  │  ├─ onboarding/            # P2 assistant premier lancement
│     │  │  ├─ workshop/              # P3 onglets terminaux, état Studio, diagnostics
│     │  │  ├─ assets/                # P4 banque, générateurs
│     │  │  ├─ shop/                  # P5 game passes, developer products
│     │  │  ├─ publish/               # P6 publication, historique
│     │  │  └─ account/               # P6 compte, licence
│     │  ├─ components/ui/            # design system
│     │  ├─ lib/ipc/                  # wrappers typés des commandes Tauri
│     │  └─ i18n/fr.ts                # toutes les chaînes UI
│     └─ src-tauri/
│        ├─ src/
│        │  ├─ lib.rs                 # assemblage des plugins, état global
│        │  ├─ commands/              # couche IPC fine (validation + appel des modules)
│        │  ├─ error.rs               # AppError → message FR
│        │  ├─ db/                    # SQLite + migrations
│        │  ├─ projects/              # P1 index, instanciation de template, instantanés gix
│        │  ├─ toolchain/             # P2 manifeste, téléchargement vérifié, détection
│        │  ├─ agents/                # P2 adaptateurs par agent (install, login, config MCP)
│        │  ├─ pty/                   # P3 sessions portable-pty
│        │  ├─ devserver/             # P3 rbxtsc -w, rojo serve, parsing des diagnostics
│        │  ├─ studio/                # P2/P3 détection, plugins, ouverture
│        │  ├─ bridge/                # P3/P4 serveur axum 127.0.0.1 + jeton
│        │  ├─ assets/                # P4 banque, génération, post-traitement
│        │  ├─ roblox/                # P4+ client Open Cloud, OAuth
│        │  ├─ secrets/               # keyring
│        │  └─ account/               # P6 Supabase, licence
│        ├─ capabilities/             # permissions Tauri minimales
│        └─ tauri.conf.json
├─ packages/
│  ├─ project-template/               # P1 template roblox-ts (embarqué comme ressource)
│  ├─ shared/                         # types partagés (projet, asset, erreurs)
│  ├─ mcp-server/                     # P4 adaptateur MCP stdio
│  └─ studio-plugin/                  # P3 plugin compagnon (roblox-ts → .rbxm)
├─ backend/supabase/                  # P6 migrations SQL, edge functions (Stripe, licence)
├─ docs/
└─ .github/workflows/                 # CI Windows + macOS (+ Linux), release signée
```

## 4. Structure d'un projet utilisateur

```
mon-jeu/
├─ AGENTS.md  CLAUDE.md               # conventions pour les agents (FR)
├─ package.json  tsconfig.json        # roblox-ts épinglé
├─ default.project.json               # arbre Rojo
├─ src/
│  ├─ server/  main.server.ts, services/
│  ├─ client/  main.client.ts, controllers/
│  └─ shared/  network.ts, generated/assets.ts (écrit par l'app, jamais à la main)
├─ world/                             # P5 modèles .rbxm + map.json
├─ .atelier/
│  ├─ project.json                    # id, nom, template@version, universeId/placeId, port Rojo
│  ├─ diagnostics.json                # P3 erreurs de compilation (lisibles par l'agent)
│  ├─ studio-output.log               # P3 sortie Studio (via plugin)
│  └─ build/                          # place générée (ignoré par git)
└─ .mcp.json  opencode.json  …        # P2/P4 config MCP écrite par l'app
```

`src/shared/generated/assets.ts` : registre `nom → rbxassetid://…` régénéré par l'app à chaque upload.
L'« insertion automatique de l'assetId » se fait donc dans un fichier dédié, déterministe, et le code l'importe — pas de rechercher/remplacer dans le code de l'utilisateur.

## 5. Flux de données principaux

1. **Création de projet (P1)** — UI `create_project(nom, template)` → le cœur calcule le slug (accents, noms réservés Windows, collisions) → copie le template embarqué dans un dossier temporaire → substitue les variables → renomme atomiquement → écrit `.atelier/project.json` → enregistre en SQLite → commit initial via `gix` → (P2) `npm install` avec le Node privé, progression affichée.
2. **Session d'agent (P3)** — onglet → `pty_spawn(projectId, agent)` → PTY avec `cwd` = projet, `PATH` privé, `ATELIER_BRIDGE_URL` + jeton → sortie via Channel vers xterm.js, entrée via `pty_write`, `pty_resize`. Instantané git automatique à l'ouverture et à la fermeture.
3. **Boucle de dev (P3)** — à l'ouverture du projet : `rbxtsc -w` + `rojo serve --port <libre>` → diagnostics parsés → badge UI + message expliqué en français + `.atelier/diagnostics.json` + outil MCP `get_build_errors`.
4. **Asset (P4)** — l'agent appelle `generate_icon("épée en feu")` → adaptateur MCP → bridge → le cœur lit la clé Gemini dans le trousseau → génère → post-traite (redimensionnement, PNG) → banque (fichier + SQLite FTS) → `upload_to_roblox` (Open Cloud Assets, polling de l'opération, résultat de modération) → mise à jour de `generated/assets.ts` → renvoie `{ nom, assetId }` à l'agent.
5. **Publication (P6)** — build complet → `rojo build -o .atelier/build/place.rbxl` → Open Cloud *Place Publishing* → version enregistrée (SQLite + tag git) → lien vers le jeu.
6. **Licence (P6)** — Stripe Checkout (web) → webhook (edge function Supabase) → table `licenses` → connexion dans l'app (lien magique / OAuth via deep link) → jeton de licence signé Ed25519, vérifiable hors ligne avec période de grâce, stocké dans le trousseau.

## 6. Intégration Roblox Studio

- **Détection** : Windows `%LOCALAPPDATA%\Roblox\Versions\*\RobloxStudioBeta.exe` ; macOS `/Applications/RobloxStudio.app`. Processus en cours détecté via `sysinfo`.
- **Synchro code** : `rojo serve` (un port par projet). Le plugin Rojo officiel exige un clic « Connect ». Option à évaluer en P3 : plugin Rojo pré-configuré (« baked ») avec connexion automatique au port du projet (licence MPL-2.0, compatible).
- **Plugin compagnon** (copié dans le dossier Plugins de Studio, aucune action utilisateur) : heartbeat vers le bridge, identification du projet ouvert (marqueur `StringValue` injecté par `default.project.json`), relais de `LogService` pour que l'agent lise les erreurs d'exécution.
- **MCP intégré de Studio** : ajouté à la config MCP de chaque agent (Windows `cmd.exe /c %LOCALAPPDATA%\Roblox\mcp.bat`, macOS `…/RobloxStudio.app/Contents/MacOS/StudioMCP`). Nécessite d'activer « Enable Studio as MCP server » dans Studio : étape guidée de l'onboarding. Règle dans `AGENTS.md` : **le code se modifie dans `src/`, jamais via Studio** (Rojo écraserait les modifications) ; Studio MCP sert à inspecter, exécuter du Luau de test, lancer des playtests.
- **Ouvrir dans Studio** : projet non publié → `rojo build` d'une place locale puis ouverture ; projet publié → deep link `roblox-studio:` sur le placeId.

## 7. Sécurité et confidentialité

- Bridge lié à `127.0.0.1` uniquement, port aléatoire, jeton par session transmis par variable d'environnement, vérification de l'en-tête `Origin`.
- Capabilities Tauri minimales ; CSP stricte ; aucune exécution de commande arbitraire depuis le front (liste blanche : agents connus, rbxtsc, rojo, npm).
- Aucun secret dans l'environnement des agents : ils n'ont que l'URL et le jeton du bridge.
- Métadonnées de la banque partagée (noms, tags écrits par d'autres utilisateurs) = **entrée non fiable** pour l'agent : longueur bornée, caractères filtrés, présentées comme des données (risque d'injection de prompt).
- Aucune télémétrie par défaut. Seuls échanges avec nos serveurs : compte/licence, consultation de la banque partagée, synchro cloud *activée explicitement*, contribution d'un asset *sur action explicite*.

## 8. Tests et CI

- **Rust** : `cargo test` (instanciation de template, slugs, anti path-traversal, parsing des diagnostics…), `clippy -D warnings`, `rustfmt`.
- **Front** : `tsc --noEmit` strict, ESLint, Vitest + Testing Library, IPC mocké via `@tauri-apps/api/mocks`.
- **Template** : job CI qui instancie un projet, lance `npm ci && npx rbxtsc` puis `rojo build` → garantit qu'un projet neuf compile toujours.
- **E2E** : WebdriverIO + `tauri-driver` (Windows/Linux ; pas de driver WKWebView sur macOS → tests manuels documentés).
- **CI** : matrice `windows-latest`, `macos-latest`, `ubuntu-latest` ; release signée en P6.

## 9. Découpage par phase

| Phase | Livrable | Critère de validation |
|---|---|---|
| 1 | Monorepo, app Tauri qui démarre, accueil + création de projet depuis le template, `AGENTS.md`/`CLAUDE.md` | Un projet créé compile avec `rbxtsc` et se construit avec `rojo build` ; tests Rust + front verts |
| 2 | Toolchain privée, adaptateurs d'agents, assistant de connexion, détection de Studio + plugins | Sur une machine vierge : 1er lancement → agent connecté et Studio prêt, sans terminal ni droits admin |
| 3 | Onglets PTY multi-agents, dev server, diagnostics, état Studio, « Ouvrir dans Studio » | Un agent modifie le code → Studio se met à jour ; une erreur de compilation est visible dans l'UI et lue par l'agent |
| 4 | Banque d'assets, générateurs, upload Open Cloud, serveur MCP | L'agent génère une icône, l'uploade, et l'utilise dans le code via `generated/assets.ts` |
| 5 | Génération de maps, game passes / developer products + code boutique | Une map « île pirate » et un game pass fonctionnels en playtest ; `ProcessReceipt` idempotent |
| 6 | Publication + historique, compte, licence, mises à jour signées, signature Windows, notarisation macOS | Installation propre sur Windows (SmartScreen) et macOS (Gatekeeper) ; mise à jour auto |
| 7 | Isolation VM (optionnelle) | Diagnostic de virtualisation au démarrage ; agent fonctionnel dans la VM |

## 10. Risques principaux

| Risque | Impact | Parade |
|---|---|---|
| Les CLI d'agents changent vite (installation, auth, flags, config MCP) | Onboarding cassé | Un adaptateur par agent + manifeste de versions testé en CI, mis à jour avec l'app |
| OAuth Roblox limité à 10 utilisateurs avant revue ; endpoints boutique documentés en **Beta** | Blocage à la sortie publique | Lancer l'enregistrement et la revue tôt ; repli clé API guidée |
| Pas d'API publique (à ma connaissance) pour **créer** une expérience | 1re publication pas 100 % automatique | Création guidée sur le Creator Hub, puis sélection de l'expérience lors du consentement OAuth |
| Terrain et éditions faites à la main dans Studio non capturés par Rojo | Pertes à la publication | Monde décrit en fichiers ; terrain généré par un script serveur à partir d'un seed + style (P5) ; avertissement clair dans l'UI |
| Limites Open Cloud : quotas d'upload audio, délais de modération, décalque ≠ image ID, limites de triangles des meshes | Assets refusés ou inutilisables | Post-traitement (décimation via `meshopt`, formats), file de tâches avec reprise, messages explicites |
| Agents exécutés avec les droits de l'utilisateur | Sécurité | Garde-fous des agents eux-mêmes, instantanés git, puis isolation VM en P7 |
| Public jeune (beaucoup de créateurs Roblox sont mineurs) ; les abonnements IA ont des âges minimaux | Juridique (RGPD, paiements, CGU des fournisseurs IA) | Vérification d'âge à la création de compte, CGU adaptées, à valider avec un juriste |
| Vente numérique B2C mondiale (TVA UE, etc.) | Charge fiscale | Stripe Tax, ou un « merchant of record » |
| Antivirus / SmartScreen / Gatekeeper | Installation bloquée | Signature dès les premières bêtas, binaires téléchargés vérifiés par hash |
| Phase 7 : WHPX exige l'activation d'une fonctionnalité Windows (admin + redémarrage), HVF exige un entitlement signé | Contredit le « zéro setup » | Évaluer WSL2 (Windows) et Virtualization.framework (macOS) comme alternatives à QEMU |

## 11. Questions ouvertes

Bloquantes pour la phase 1 :

1. **Nom de l'app et identifiant** (`com.xdkteamdev.<nom>`). L'identifiant fixe les dossiers de données, les entrées du trousseau et le canal de mises à jour : il est coûteux à changer ensuite. Éviter « Roblox » dans le nom (marque déposée).
2. **Dépôt** : ce dépôt contient aujourd'hui les versions de l'outil de nettoyage de disque. Code de l'app ici (à la racine de cette branche) ou dans un dépôt dédié, ce dépôt-ci servant alors aux releases et au `latest.json` des mises à jour ?
3. **Template de départ** : un seul template « Jeu vide » bien structuré (recommandé), ou aussi « Obby » / « Simulateur » dès la phase 1 ?

À trancher avant la phase concernée (ma recommandation entre parenthèses) :

4. Agents au lancement (Claude Code, Codex, OpenCode ; Gemini CLI plus tard).
5. Auth Roblox (OAuth 2.0 + repli clé API). Qui enregistre l'app OAuth côté Roblox (compte perso ou groupe) ?
6. Connexion Studio (plugin Rojo pré-configuré en auto-connexion si la licence et la maintenance le permettent).
7. Modèle de licence (achat unique, N machines, essai gratuit ?) et prestataire de paiement (Stripe + Stripe Tax, ou un merchant of record).
8. Signature : compte Apple Developer et certificat Windows (Azure Trusted Signing ou certificat OV/EV) — à prévoir dès la phase 2 pour les bêtas.
9. Tranche d'âge visée (impact juridique et choix des agents).
