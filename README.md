# MistralChat — Clone de Claude sur Laragon

> Application web chatbot PHP/JS sans framework.
> **Streaming SSE** token par token · **SQLite WAL** · **HTTP Basic Auth** · **DOMPurify** · **CSRF**

---

## Table des matières

1. [Structure du projet](#structure-du-projet)
2. [Description détaillée de chaque fichier](#description-détaillée)
3. [Flux de données — 3 cas](#flux-de-données)
4. [Tableau de sécurité](#sécurité)
5. [Installation sur Laragon](#installation)
6. [Dépendances](#dépendances)
7. [Schéma SQLite](#schéma-sqlite)

---

## Structure du projet

```
mistralchat/
│
├── index.php                      # Shell PHP/HTML : injecte le token CSRF dans <meta>
├── setup.php                      # Script d'installation : génère .htpasswd + .htaccess, vérifie prérequis
│
├── .env                           # Variables sensibles — JAMAIS commité
├── .env.example                   # Template .env — seul fichier de config commité
├── .htaccess                      # Généré par setup.php (chemin absolu AuthUserFile)
├── .htpasswd                      # Généré par setup.php (PHP pur, sans CLI htpasswd)
├── .gitignore                     # Exclut : .env · .htaccess · .htpasswd · *.db · storage/
│
├── api/
│   ├── chat.php                   # Endpoint SSE : stream Mistral, gestion conv. existante et nouvelle
│   └── history.php                # Endpoint REST : CRUD conversations avec validation CSRF
│
├── config/
│   ├── config.php                 # Bootstrap PHP : charge .env, constantes, init DB, session CSRF
│   └── system_prompt.txt          # Persona et instructions permanentes du bot (texte brut éditable)
│
├── src/
│   ├── Database.php               # Singleton PDO SQLite : connexion, WAL, schéma, triggers
│   ├── MistralClient.php          # Wrapper API Mistral : streamMessage() + sendMessage()
│   └── ConversationManager.php    # CRUD conversations/messages + token budget + génération de titre
│
├── assets/
│   ├── css/
│   │   └── style.css              # Layout Grid, composants, thèmes clair/sombre, animation .streaming
│   ├── js/
│   │   ├── app.js                 # Orchestration, state (isStreaming, currentConvId), event listeners
│   │   ├── api.js                 # Réseau : streamChat (fetch+ReadableStream) + REST + header CSRF
│   │   └── ui.js                  # DOM : bulles, DOMPurify, markdown, erreurs mi-stream, scroll
│   └── img/
│       └── avatar-bot.svg         # Icône SVG du bot (inline dans index.php recommandé)
│
└── storage/
    └── mistralchat.db             # Base SQLite auto-créée au premier lancement (exclue du git)
```

---

## Description détaillée

---

### `index.php`

**Rôle** : Shell PHP/HTML unique — génère le token CSRF dans un `<meta>` et sert l'interface SPA.

**Responsabilités** :
- Démarre la session PHP via `require_once 'config/config.php'` (qui initialise le CSRF token)
- Émet `<meta name="csrf-token" content="<?= htmlspecialchars($_SESSION['csrf_token']) ?>">` dans `<head>` — seule raison pour laquelle ce fichier est `.php` et non `.html`
- Fournit la structure HTML : `<aside id="sidebar">`, `<main id="chat">`, `<form id="input-form">`, `<textarea id="message-input">`
- Charge `assets/css/style.css`, puis les modules JS avec `type="module"` : `app.js` en `defer`
- Ne contient aucune logique métier — tout est délégué aux modules JS
- L'authentification HTTP Basic est appliquée par Apache **avant** que PHP exécute ce fichier — le JS ne voit jamais de page de login

**Méthodes / fonctions clés** : aucune (fichier de rendu, pas de classes)

**Dépendances** : `config/config.php` (pour la session et le CSRF token)

**Points de sécurité** :
- `htmlspecialchars()` obligatoire sur la valeur du CSRF token insérée dans le HTML (protection XSS dans l'attribut)
- Ne jamais émettre de variables PHP sensibles (clé API, chemin de fichier) dans le HTML généré

---

### `setup.php`

**Rôle** : Script d'installation unique à exécuter en premier — génère les fichiers de sécurité et vérifie les prérequis.

**Responsabilités** :
- Vérifie les prérequis : PHP ≥ 8.1, extensions `pdo_sqlite` et `curl` activées, dossier `storage/` accessible en écriture
- Lit `AUTH_USER` et `AUTH_PASS` depuis `.env` via `parse_ini_file()`
- Génère `.htpasswd` au format Apache **sans CLI** : utilise `crypt($password, '$apr1$' . substr(md5(random_bytes(6)), 0, 8) . '$')` pour le format APR1-MD5 compatible Apache, ou `'{SHA}' . base64_encode(sha1($password, true))` comme alternative SHA1 plus simple
- Génère `.htaccess` complet avec le chemin **absolu** correct pour `AuthUserFile` : utilise `realpath(__DIR__) . DIRECTORY_SEPARATOR . '.htpasswd'` pour obtenir le chemin Windows-compatible (`C:/laragon/www/mistralchat/.htpasswd`) — cette étape est critique sur Windows car Apache exige un chemin absolu pour `AuthUserFile`
- Crée le dossier `storage/` s'il est absent
- Affiche un rapport HTML : ✓/✗ pour chaque étape, avec les corrections à effectuer si une étape échoue
- Se protège contre les ré-exécutions accidentelles : si `.htpasswd` existe déjà, demande une confirmation explicite (`?force=1` en query string)

**Méthodes / fonctions clés** :
```
checkPrerequisites(): array        // Retourne ['php_version' => bool, 'pdo_sqlite' => bool, 'curl' => bool, ...]
generateHtpasswd(string $user, string $pass): string  // Retourne la ligne formatée .htpasswd
generateHtaccess(string $htpasswdAbsPath): string     // Retourne le contenu complet du .htaccess
writeFile(string $path, string $content): bool        // Écrit avec gestion d'erreur explicite
renderReport(array $steps): void   // Affiche le résultat HTML
```

**Dépendances** : `.env`

**Points de sécurité** :
- Doit être inaccessible après installation : `.htaccess` généré inclut une règle `FilesMatch` qui bloque `setup.php`
- Ne jamais afficher `AUTH_PASS` en clair dans le rapport de résultat
- Valider que `.env` a été chargé correctement avant de lire `AUTH_USER` / `AUTH_PASS` — si absent, afficher une erreur explicite et arrêter

---

### `.env`

**Rôle** : Stockage de toutes les variables sensibles, absent du dépôt git.

**Contenu attendu** :
```ini
MISTRAL_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxx
MISTRAL_MODEL=mistral-large-latest
MISTRAL_TEMPERATURE=0.7
MISTRAL_MAX_TOKENS=2048
AUTH_USER=admin
AUTH_PASS=motdepasse_fort
APP_ENV=dev
```

**Points de sécurité** :
- Présent dans `.gitignore` — ne jamais commiter
- Bloqué par `.htaccess` via `<FilesMatch "\.env">` — inaccessible en HTTP
- `parse_ini_file()` dans `config/config.php` échoue silencieusement si le fichier est absent : vérifier le retour et lever une `RuntimeException` explicite plutôt que de continuer avec des constantes vides

---

### `.env.example`

**Rôle** : Template commité avec des valeurs fictives — documentation des variables attendues.

**Contenu** : identique à `.env` mais avec des valeurs neutres (`your_mistral_api_key_here`, `your_username`, etc.) et des commentaires expliquant le rôle de chaque variable. C'est le seul fichier de configuration commité.

---

### `.htaccess`

**Rôle** : Configuration Apache complète — généré par `setup.php` avec le chemin absolu correct.

**Responsabilités** :

**1. HTTP Basic Auth** — protège l'intégralité du projet :
```apache
AuthType Basic
AuthName "MistralChat"
AuthUserFile "C:/laragon/www/mistralchat/.htpasswd"
Require valid-user
```

**2. Blocage des fichiers sensibles** — inaccessibles en HTTP même si l'auth est contournée :
```apache
<FilesMatch "(\.env|\.htpasswd|\.gitignore|setup\.php)$">
    Require all denied
</FilesMatch>
```

**3. Blocage du dossier storage/** :
```apache
<DirectoryMatch "storage">
    Require all denied
</DirectoryMatch>
```

**4. Headers de sécurité** :
```apache
<IfModule mod_headers.c>
    Header always set X-Frame-Options "DENY"
    Header always set X-Content-Type-Options "nosniff"
    Header always set Referrer-Policy "no-referrer"
    Header always set X-Accel-Buffering "no"
</IfModule>
```

**5. Désactivation gzip pour SSE** — critique : gzip bufferise les réponses et casse le streaming token par token :
```apache
<IfModule mod_deflate.c>
    SetEnvIf Request_URI "api/chat\.php$" no-gzip dont-vary
</IfModule>
```

**Points de sécurité** :
- Le chemin absolu dans `AuthUserFile` est généré dynamiquement par `setup.php` — ne pas éditer manuellement
- Sans la règle `FilesMatch` sur `.htpasswd`, le fichier serait lisible en HTTP et exposerait les credentials hashés

---

### `.htpasswd`

**Rôle** : Fichier de credentials Apache généré par `setup.php` — une ligne par utilisateur au format `user:hash`.

**Format** : `admin:$apr1$xxxxxxxx$xxxxxxxxxxxxxxxxxxxx` (APR1-MD5) ou `admin:{SHA}base64hash` (SHA1).

**Points de sécurité** : Exclu du git. Bloqué en HTTP par `.htaccess`. Jamais édité manuellement.

---

### `.gitignore`

**Rôle** : Exclusions git — protège les fichiers sensibles et les artefacts locaux.

**Contenu** :
```
.env
.htaccess
.htpasswd
storage/
*.db
*.db-wal
*.db-shm
```

Note : `.htaccess` est exclu car généré par `setup.php` avec un chemin absolu spécifique à chaque machine.

---

### `api/chat.php`

**Rôle** : Endpoint SSE — point d'entrée des messages utilisateur, orchestre le streaming Mistral.

**Responsabilités** :
1. **Nettoyage du buffer** : `if (ob_get_level() > 0) { ob_end_clean(); }` — obligatoire sur Laragon/Windows où Apache mod_php peut avoir un buffer actif ; sans ça, les `flush()` n'ont aucun effet
2. **Configuration du timeout** : `set_time_limit(0)` empêche Apache de couper la connexion (défaut 30s) ; `ignore_user_abort(true)` permet de terminer la sauvegarde SQLite même si le navigateur ferme l'onglet avant la fin du stream
3. **Validation de l'entrée** : lit `php://input`, décode JSON, valide que `message` est présent et non vide — en cas d'erreur, émet `event: error` et termine
4. **Gestion conversation nouvelle** : si `conversation_id === null`, appelle `ConversationManager::createConversation()` puis émet `event: init\ndata: {"conversation_id":N}\n\n` + `flush()` **avant** les premiers tokens
5. **Sauvegarde du message utilisateur** : `ConversationManager::saveMessage($convId, 'user', $message)` avant le stream
6. **Construction du contexte** : `ConversationManager::getMessages($convId)` → inclut le system prompt en position 0, puis `truncateToTokenBudget()` si le contexte dépasse `MISTRAL_MAX_TOKENS`
7. **Headers SSE** : `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`
8. **Streaming** : instancie `MistralClient::streamMessage($messages, $onChunk)` où `$onChunk` émet `data: {"token":"..."}\n\n` + `ob_flush()` + `flush()` à chaque token ; accumule la réponse complète dans `$fullResponse`
9. **Fin du stream nominal** : émet `data: [DONE]\n\n` + `flush()`, puis sauvegarde `$fullResponse` en SQLite via `saveMessage($convId, 'assistant', $fullResponse)`
10. **Génération de titre asynchrone** : si c'est une nouvelle conversation (`$isNew === true`), appelle `ConversationManager::generateTitle($convId, $message)` (appel Mistral non-streamé), puis émet `event: title\ndata: {"title":"..."}\n\n` — fait après la fin du stream pour ne pas bloquer les premiers tokens
11. **Gestion des erreurs** : tout `catch(\Exception $e)` émet `event: error\ndata: {"message":"..."}\n\n` + `flush()` et termine proprement

**Méthodes / fonctions clés** : fichier procédural, pas de classes. Opérations clés listées ci-dessus.

**Dépendances** : `config/config.php` · `src/MistralClient.php` · `src/ConversationManager.php`

**Points de sécurité** :
- Ne pas inclure la clé API dans les réponses d'erreur (utiliser `$e->getMessage()` filtré)
- Valider que `conversation_id` est un entier si fourni (`filter_var($id, FILTER_VALIDATE_INT)`)
- `ignore_user_abort(true)` est nécessaire mais attention : si le client se déconnecte, PHP continue — ne pas faire d'opérations coûteuses après la déconnexion (la sauvegarde SQLite est légère, c'est acceptable)

---

### `api/history.php`

**Rôle** : Endpoint REST — CRUD des conversations avec validation CSRF sur toutes les mutations.

**Responsabilités** :
- Route les requêtes selon la méthode HTTP + paramètre `action` :

| Méthode HTTP | `?action=` | Opération | CSRF requis |
|---|---|---|---|
| `GET` | `list` | `$manager->listConversations()` | Non |
| `GET` | `get&id=X` | `$manager->getConversationMessages(X)` | Non |
| `POST` | `create` | `$manager->createConversation()` → retourne `{id}` | Oui |
| `DELETE` | `delete&id=X` | `$manager->deleteConversation(X)` | Oui |
| `PATCH` | `rename&id=X` | `$manager->renameConversation(X, $body['title'])` | Oui |

- **Validation CSRF** sur `POST`/`DELETE`/`PATCH` : lit `$_SERVER['HTTP_X_CSRF_TOKEN']` et compare à `$_SESSION['csrf_token']` — retourne HTTP 403 si invalide ou absent
- Toutes les réponses sont en `Content-Type: application/json`
- N'accède jamais à PDO directement — passe exclusivement par `ConversationManager`
- Valide et nettoie `id` : `filter_var($id, FILTER_VALIDATE_INT)` — retourne HTTP 400 si invalide

**Méthodes / fonctions clés** : fichier procédural.
```
validateCsrf(): void        // Lit header, compare session, envoie 403 + exit si invalide
sendJson(mixed $data, int $status = 200): void   // json_encode + Content-Type + http_response_code
getIntParam(string $key): int   // Lit GET/POST, valide int, HTTP 400 si absent/invalide
```

**Dépendances** : `config/config.php` · `src/ConversationManager.php`

**Points de sécurité** :
- Le CSRF token est en session PHP (côté serveur) — inviolable depuis le JS malveillant d'un autre onglet
- Même avec HTTP Basic Auth, la protection CSRF reste nécessaire si l'utilisateur visite un site malveillant depuis un navigateur déjà authentifié (attack vector : formulaire caché sur autre domaine)

---

### `config/config.php`

**Rôle** : Bootstrap PHP — premier fichier inclus via `require_once` dans tous les fichiers PHP.

**Responsabilités** :
- Charge `.env` : `$env = parse_ini_file(__DIR__ . '/../.env')` — si le retour est `false` (fichier absent ou malformé), lève une `RuntimeException` explicite avec un message clair plutôt que de continuer silencieusement avec des constantes nulles qui causeraient des erreurs cryptiques plus loin
- Définit les constantes globales :
  - `MISTRAL_API_URL` = `'https://api.mistral.ai/v1/chat/completions'`
  - `MISTRAL_MODEL`, `MISTRAL_TEMPERATURE`, `MISTRAL_MAX_TOKENS`
  - `DB_PATH` = `__DIR__ . '/../storage/mistralchat.db'`
  - `SYSTEM_PROMPT_PATH` = `__DIR__ . '/system_prompt.txt'`
  - `APP_ENV` = `'dev'` ou `'prod'`
- Configure `error_reporting` selon `APP_ENV` : tout afficher en `dev`, `0` en `prod`
- Démarre la session PHP : `session_start()` si pas déjà démarrée
- Génère et stocke le CSRF token : `if (empty($_SESSION['csrf_token'])) { $_SESSION['csrf_token'] = bin2hex(random_bytes(32)); }`
- Initialise `Database::getInstance()` qui crée le schéma SQLite si la base est nouvelle

**Méthodes / fonctions clés** : aucune (fichier de bootstrap procédural)

**Dépendances** : `.env` · `src/Database.php`

**Points de sécurité** :
- `parse_ini_file()` sans vérification du retour est un bug silencieux courant — toujours vérifier `!== false`
- `bin2hex(random_bytes(32))` produit un CSRF token cryptographiquement sûr (64 chars hex, entropie 256 bits)
- Ne jamais exposer de constante sensible (clé API) dans un message d'erreur

---

### `config/system_prompt.txt`

**Rôle** : Instructions permanentes du bot injectées en position 0 (rôle `system`) à chaque requête Mistral.

**Responsabilités** :
- Définit le persona : ton, langue, limites, comportements spéciaux
- Chargé par `ConversationManager::loadSystemPrompt()` et mis en cache dans `$systemPromptCache`
- Modifiable sans toucher au code — c'est intentionnel : permet de changer le comportement du bot sans redéploiement

**Contenu d'exemple** :
```
Tu es un assistant IA concis et précis. Tu réponds toujours en français.
Tu es honnête sur tes limites et tu ne génères pas de contenu nuisible.
Tu formmates tes réponses en Markdown quand c'est pertinent.
```

**Points de sécurité** :
- Ce fichier est lu par PHP mais ne doit pas être exposé en HTTP (inclus dans le dossier `config/` qui n'est pas directement accessible via Apache si le DocumentRoot pointe sur la racine du projet)
- Valider que le fichier existe et est lisible dans `loadSystemPrompt()` — lever une exception si absent plutôt que d'envoyer des requêtes Mistral sans system prompt

---

### `src/Database.php`

**Rôle** : Singleton PDO SQLite — connexion unique, mode WAL, initialisation du schéma.

**Responsabilités** :
- Implémente le pattern Singleton : `private static ?PDO $instance = null` — garantit une seule connexion PDO par requête PHP
- Configure la connexion immédiatement après instanciation :
  - `PRAGMA journal_mode = WAL` — **critique sur Laragon** : sans WAL, une écriture en fin de stream (`chat.php` sauvegarde la réponse) bloque toute lecture simultanée (`history.php` qui charge la sidebar), causant des erreurs `SQLITE_BUSY`. WAL permet des lectures concurrentes pendant l'écriture
  - `PRAGMA foreign_keys = ON` — active les contraintes FK et le `ON DELETE CASCADE`
  - `PRAGMA busy_timeout = 5000` — attend jusqu'à 5 secondes si la DB est verrouillée avant de lever une exception
- Configure PDO : `ERRMODE_EXCEPTION`, `DEFAULT_FETCH_MODE = FETCH_ASSOC`
- Crée le schéma au premier lancement si les tables n'existent pas

**Méthodes / fonctions clés** :
```php
public static function getInstance(): PDO
// Pattern singleton : crée la connexion si nulle, retourne l'instance existante sinon

private static function connect(): PDO
// Crée la connexion PDO, active les PRAGMAs, appelle initSchema()

private static function initSchema(PDO $pdo): void
// Exécute le DDL de création des tables, index et trigger (voir section Schéma SQLite)
```

**Dépendances** : constante `DB_PATH` de `config/config.php`

**Points de sécurité** :
- Le fichier `.db` est créé dans `storage/` — ce dossier est bloqué par `.htaccess` et exclu du git
- En cas d'échec de connexion (dossier `storage/` absent ou non-accessible en écriture), PDO lève une `PDOException` attrapée dans `config.php` avec un message explicite
- `PRAGMA foreign_keys = ON` doit être exécuté à chaque nouvelle connexion PDO (les PRAGMAs ne persistent pas entre connexions)

---

### `src/MistralClient.php`

**Rôle** : Wrapper API Mistral — deux modes d'opération (streaming et synchrone).

**Responsabilités** :
- Encapsule toute la logique cURL d'appel à l'API Mistral
- **Mode stream** : ouvre une connexion cURL avec `CURLOPT_WRITEFUNCTION`, parse les chunks SSE reçus ligne par ligne, extrait le token de `choices[0].delta.content` et appelle le callback
- **Mode synchrone** : requête cURL classique, retourne le contenu complet — utilisé pour la génération de titre
- Gère la spécificité Windows : si `APP_ENV === 'dev'`, peut désactiver `CURLOPT_SSL_VERIFYPEER` ou pointer `CURLOPT_CAINFO` vers le bundle CA de Laragon (`C:/laragon/etc/ssl/cacert.pem`)
- Gère les retries : si Mistral retourne 429 (rate limit) ou 503, attend et retente 1 fois
- Parser SSE côté PHP : lit `data: {...}` lignes, ignore `data: [DONE]`, extrait `choices[0].delta.content` des lignes JSON valides

**Méthodes / fonctions clés** :
```php
public function __construct(
    string $apiKey,
    string $model,
    float  $temperature = 0.7,
    int    $maxTokens   = 2048
)

public function streamMessage(array $messages, callable $onChunk): void
// $onChunk(string $token): void — appelé pour chaque token reçu
// Lève MistralException si la connexion échoue ou si Mistral retourne une erreur HTTP

public function sendMessage(array $messages): string
// Retourne le contenu complet de la réponse
// Lève MistralException en cas d'erreur

private function buildPayload(array $messages, bool $stream): array
// Construit le corps JSON de la requête

private function buildCurlHandle(array $payload): \CurlHandle
// Configure cURL : headers, timeout, SSL, POST body

private function parseSseChunk(string $line): ?string
// Extrait le token d'une ligne "data: {...}"
// Retourne null si la ligne n'est pas un événement de token (ex: data: [DONE])

private function buildHeaders(): array
// Retourne ['Authorization: Bearer ...', 'Content-Type: application/json']
```

**Dépendances** : aucune dépendance interne (constantes lues dans le constructeur depuis `config.php`)

**Points de sécurité** :
- La clé API ne doit jamais apparaître dans les logs — ne pas la logger, même partiellement
- `CURLOPT_SSL_VERIFYPEER = false` est acceptable en dev local uniquement — en prod, utiliser `CURLOPT_CAINFO` avec le bundle CA correct
- Le parser SSE doit ignorer silencieusement les lignes malformées (ne pas lever d'exception sur un chunk partiel)

---

### `src/ConversationManager.php`

**Rôle** : Couche d'accès aux données conversationnelles — aucun autre fichier n'accède à PDO directement.

**Responsabilités** :
- Centralise toutes les requêtes SQL relatives aux conversations et messages
- Injecte le system prompt en position 0 de chaque tableau de messages envoyé à Mistral
- Gère le budget de tokens pour éviter de dépasser la fenêtre de contexte de Mistral
- Génère automatiquement le titre d'une nouvelle conversation après le premier échange

**Méthodes / fonctions clés** :
```php
public function __construct(
    PDO           $pdo,
    MistralClient $client,
    string        $systemPromptPath
)

public function createConversation(): int
// INSERT conversations (title = 'Nouvelle conversation')
// Retourne l'id auto-incrémenté
// NE génère PAS le titre ici (pour ne pas bloquer le premier token SSE)

public function generateTitle(int $convId, string $firstUserMessage): string
// Appel MistralClient::sendMessage() non-streamé avec le prompt :
//   "Résume ce message en 5 mots maximum, sans ponctuation finale : {firstUserMessage}"
// UPDATE conversations SET title = '{generatedTitle}' WHERE id = {convId}
// Retourne le titre généré

public function getMessages(int $convId): array
// Retourne [{role:'system', content:'...'}, {role:'user',...}, {role:'assistant',...}, ...]
// Le system prompt (chargé depuis system_prompt.txt) est TOUJOURS en index 0
// Les messages sont ordonnés par created_at ASC

public function getConversationMessages(int $convId): array
// Retourne [{id, role, content, created_at}, ...] pour l'affichage côté client
// Sans le system prompt (pas pertinent pour l'UI)

public function saveMessage(int $convId, string $role, string $content): void
// INSERT messages (conversation_id, role, content)
// Le trigger SQL met à jour automatiquement conversations.updated_at

public function listConversations(): array
// SELECT id, title, created_at, updated_at FROM conversations ORDER BY updated_at DESC

public function deleteConversation(int $convId): void
// DELETE FROM conversations WHERE id = ?
// ON DELETE CASCADE supprime les messages associés automatiquement

public function renameConversation(int $convId, string $title): void
// UPDATE conversations SET title = ?, updated_at = CURRENT_TIMESTAMP WHERE id = ?
// Valide que title n'est pas vide et <= 100 caractères

public function truncateToTokenBudget(array $messages, int $maxTokens): array
// Estime le budget : estimateTokenCount($messages) → sum(strlen($m['content'])) / 4
// INVARIANT CRITIQUE : ne retire JAMAIS $messages[0] (le system prompt)
// Retire les messages depuis l'index 1 (les plus anciens) jusqu'à rentrer dans le budget
// Log un warning si le system prompt seul dépasse le budget

private function loadSystemPrompt(): string
// Lecture et mise en cache de config/system_prompt.txt
// Lève RuntimeException si le fichier est absent ou vide

private function estimateTokenCount(array $messages): int
// sum(strlen($m['content']) / 4) pour tous les messages
// Approximation : 4 caractères ≈ 1 token pour le français/anglais
```

**Dépendances** : `src/Database.php` (PDO injecté) · `src/MistralClient.php` (injecté) · `config/system_prompt.txt`

**Points de sécurité** :
- Toutes les requêtes SQL utilisent des `prepare()` + `execute()` avec paramètres bindés — aucune interpolation de variables dans le SQL
- `truncateToTokenBudget()` ne doit jamais supprimer le message système — le casser entraînerait un comportement non défini du bot

---

### `assets/css/style.css`

**Rôle** : Feuille de style unique — layout, composants UI, thèmes et animation streaming.

**Responsabilités** :
- **Variables CSS** : définit le design system complet via custom properties — `--bg-primary`, `--bg-surface`, `--text-primary`, `--text-secondary`, `--accent`, `--accent-hover`, `--border`, `--danger`, `--bubble-user`, `--bubble-bot` — le thème sombre est par défaut (`:root`), le thème clair est activé par `[data-theme="light"]` sur `<html>`
- **Layout** : CSS Grid sur `<body>` — colonne sidebar fixe 260px + zone chat flexible ; la zone chat est elle-même en Flexbox colonne (messages + formulaire d'envoi en bas)
- **Composants** :
  - `.message-bubble` avec variantes `.user` et `.assistant` : padding, border-radius, avatar
  - `pre code` : fond distinct, scrollbar horizontale, `white-space: pre`, sélectionnable
  - `#message-input` : `resize: none` + auto-resize via JS
  - Sidebar : liste `#conversation-list` avec `.conversation-item` et état `.active`
- **Animation streaming** : `.streaming::after { content: '▋'; animation: blink 1s step-end infinite; }` — le curseur clignotant apparaît sur la bulle en cours de réception, disparaît automatiquement quand `ui.js` retire la classe `.streaming`
- **Responsive** : sidebar cachée (`display: none`) sous 640px, bouton hamburger `#sidebar-toggle` pour l'afficher en overlay

**Dépendances** : aucune

---

### `assets/js/app.js`

**Rôle** : Orchestrateur — state management, event listeners, coordination des modules `api.js` et `ui.js`.

**Responsabilités** :
- Maintient le state local : `let currentConversationId = null` et `let isStreaming = false`
- Au `DOMContentLoaded` : appelle `refreshSidebar()` puis, si un `id` est présent en query string (`?conv=42`), charge cette conversation
- Gère le submit du formulaire : `Enter` soumet (sauf `Shift+Enter` qui insère un saut de ligne), `Ctrl+Enter` soumet également
- Appelle `api.streamChat()` en transmettant les quatre callbacks : `onInit`, `onToken`, `onTitle`, `onDone`, `onError`
- Sur `onInit(id)` : met à jour `currentConversationId`, appelle `refreshSidebar()`, met en surbrillance la nouvelle conversation
- Sur `onTitle(title)` : met à jour le texte de l'élément de sidebar correspondant à `currentConversationId` sans rechargement complet
- Sur `onDone()` : appelle `setStreaming(false)`, réactive le formulaire
- Sur `onError(message)` : appelle `ui.errorAssistantBubble(element, message)`, appelle `setStreaming(false)`
- Gère le clic "Nouvelle conversation" : `currentConversationId = null`, `ui.clearChat()`, `ui.showWelcomeMessage()`
- Gère le toggle thème : `document.documentElement.setAttribute('data-theme', ...)`

**Méthodes / fonctions clés** (module ES2020) :
```javascript
function init()
// DOMContentLoaded handler — attache tous les event listeners

async function handleSubmit(e)
// Validation, setStreaming(true), ui.appendUserMessage(), ui.createAssistantBubble()
// puis api.streamChat()

async function loadConversation(id)
// api.getConversation(id), ui.clearChat(), affiche les messages historiques
// currentConversationId = id, ui.setActiveConversation(id)

async function refreshSidebar()
// api.listConversations() → ui.renderSidebar(conversations)

function setStreaming(bool)
// isStreaming = bool, désactive/réactive le bouton submit et le textarea

function startNewConversation()
// currentConversationId = null, ui.clearChat(), ui.showWelcomeMessage()

function toggleTheme()
// Toggle data-theme sur <html>, persist dans localStorage
```

**Dépendances** : `api.js` · `ui.js`

---

### `assets/js/api.js`

**Rôle** : Module réseau exclusif — `streamChat` via `fetch`+`ReadableStream` et endpoints REST.

**Responsabilités** :
- **`streamChat()`** : utilise `fetch` + `response.body.getReader()` + `TextDecoder` pour lire le stream SSE. `EventSource` natif n'est pas utilisé car il ne supporte pas `POST` — la seule option pour envoyer un corps JSON est `fetch`. Le reader consomme les chunks binaires, les décode en UTF-8, les accumule dans un buffer ligne par ligne, puis parse chaque ligne selon le format SSE :
  - Ligne `event: init` → lit la ligne `data:` suivante → appelle `onInit(id)`
  - Ligne `data: {"token":"..."}` → extrait le token → appelle `onToken(token)`
  - Ligne `event: title` → lit la ligne `data:` suivante → appelle `onTitle(title)`
  - Ligne `data: [DONE]` → appelle `onDone()`
  - Ligne `event: error` → lit la ligne `data:` suivante → appelle `onError(message)`
- **REST** : fonctions `fetch` classiques vers `history.php` avec `X-CSRF-Token` sur les mutantes

**Méthodes / fonctions clés** :
```javascript
export async function streamChat(
  conversationId,    // int | null
  message,           // string
  { onInit, onToken, onTitle, onDone, onError }
)
// fetch POST api/chat.php + ReadableStream reader + parseur SSE ligne par ligne

export async function listConversations()
// GET api/history.php?action=list → Array<{id, title, updated_at}>

export async function getConversation(id)
// GET api/history.php?action=get&id={id} → Array<{role, content, created_at}>

export async function deleteConversation(id)
// DELETE api/history.php?action=delete&id={id} + header X-CSRF-Token

export async function renameConversation(id, title)
// PATCH api/history.php?action=rename&id={id} + body {title} + header X-CSRF-Token

function getCsrfToken()
// Lit document.querySelector('meta[name="csrf-token"]').content
// Cache la valeur en variable module pour éviter de re-querier le DOM à chaque appel
```

**Dépendances** : `index.php` (meta tag CSRF token)

**Points de sécurité** :
- En cas d'erreur réseau (`fetch` rejeté, `response.ok === false`), appelle directement `onError()` avec un message explicite
- Le CSRF token est lu depuis le meta tag injecté par PHP — jamais stocké dans `localStorage` (accessible à tout JS de la page)

---

### `assets/js/ui.js`

**Rôle** : Module DOM pur — aucun appel réseau, aucun state global.

**Responsabilités** :
- Crée et insère les éléments DOM des messages dans `#chat-messages`
- Accumule les tokens en texte brut pendant le stream, puis transforme en HTML sécurisé à la fin
- Gère les états d'erreur mi-stream sans laisser de bulle bloquée en `.streaming`
- Applique `DOMPurify.sanitize()` sur tout HTML injecté dans le DOM

**Méthodes / fonctions clés** :
```javascript
export function appendUserMessage(content)
// Crée <div class="message-bubble user"> avec texte échappé (pas de Markdown pour l'utilisateur)
// Appelle scrollToBottom()

export function createAssistantBubble()
// Crée <div class="message-bubble assistant streaming"> vide, avec avatar
// Stocke le buffer texte brut dans element.dataset.rawContent = ''
// Retourne l'élément DOM (app.js le conserve pour les appels suivants)

export function appendTokenToBuffer(element, token)
// element.dataset.rawContent += token
// Insère également le token en texte brut dans le DOM pour la visibilité immédiate
// Appelle scrollToBottom()

export function finalizeAssistantBubble(element)
// Récupère element.dataset.rawContent (texte brut complet)
// renderMarkdown(rawText) → marked.parse() avec options sécurisées
//   (mangle: false, headerIds: false, gfm: true, breaks: true)
// DOMPurify.sanitize(html, {USE_PROFILES: {html: true}})
// element.innerHTML = safeHtml
// element.classList.remove('streaming') → stoppe l'animation curseur
// Appelle highlight.js sur les blocs <pre><code> si disponible

export function errorAssistantBubble(element, message)
// element.classList.remove('streaming')
// Affiche <div class="error-state">⚠ {message}</div> dans la bulle
// Les tokens déjà affichés restent visibles (ne pas effacer le contenu partiel)

export function renderSidebar(conversations)
// Reconstruit #conversation-list depuis le tableau
// Chaque item : <li class="conversation-item" data-id="{id}"> avec titre et date relative

export function setActiveConversation(id)
// Retire .active de tous les items, ajoute .active sur l'item avec data-id={id}

export function scrollToBottom(force = false)
// Scroll seulement si l'utilisateur est dans les 80px du bas (ne pas forcer si l'utilisateur a remonté)
// Si force = true, scroll toujours (utilisé au chargement d'une conversation)

export function clearChat()
// Vide #chat-messages

export function showWelcomeMessage()
// Insère un message de bienvenue / état vide
```

**Dépendances** : `DOMPurify` (CDN) · `marked.js` (CDN) · `highlight.js` (CDN, optionnel)

**Points de sécurité** :
- **`DOMPurify.sanitize()` est obligatoire** avant tout `innerHTML` — une réponse Mistral contenant `<script>` ou `<img onerror="...">` causerait un XSS stocké dans SQLite puis exécuté à chaque chargement de la conversation
- `appendUserMessage()` utilise `textContent` (pas `innerHTML`) pour le texte de l'utilisateur — échappement natif du navigateur
- `appendTokenToBuffer()` insère via `textContent` pendant le stream, pas `innerHTML` — seulement `finalizeAssistantBubble()` utilise `innerHTML` après sanitisation

---

### `assets/img/avatar-bot.svg`

**Rôle** : Icône vectorielle du bot dans les bulles de réponse.

**Points d'implémentation** : SVG inline dans `ui.js` ou dans le template HTML recommandé pour éviter une requête HTTP supplémentaire et permettre la colorisation via `currentColor` (s'adapte automatiquement au thème).

---

### `storage/mistralchat.db`

**Rôle** : Fichier de base de données SQLite — créé automatiquement par `Database.php` au premier lancement.

**Points de sécurité** :
- Bloqué par `.htaccess` (`Require all denied` sur `storage/`)
- Exclu du git
- Sauvegardable avec un simple `cp` — pas besoin d'outil spécial
- Les fichiers `-wal` et `-shm` (journaux WAL) sont également exclus du git via `.gitignore`

---

## Flux de données

### Cas A — Message dans une conversation existante

```
[Utilisateur appuie sur Enter ou clique Envoyer]
        │
        │  currentConversationId = 42
        │  isStreaming = false → vérifié par app.js::handleSubmit()
        ▼
[app.js::setStreaming(true)]
        │  bouton désactivé, textarea désactivée
        ▼
[ui.js::appendUserMessage("Quel est le sens de la vie ?")]
[ui.js::createAssistantBubble()] ─────────────────────── retourne <element>
        │
        ▼
[api.js::streamChat(42, message, {onInit, onToken, onTitle, onDone, onError})]
        │
        │  fetch POST api/chat.php
        │  body: { "conversation_id": 42, "message": "Quel est le sens de la vie ?" }
        │  headers: { "Content-Type": "application/json" }
        ▼
┌─────────────────────────────────────────────────────────────────┐
│ api/chat.php                                                    │
│                                                                 │
│  ob_get_level() > 0 → ob_end_clean()                           │
│  set_time_limit(0)                                              │
│  ignore_user_abort(true)                                        │
│  conversation_id = 42 (non null → pas de createConversation())  │
│  saveMessage(42, 'user', message)                               │
│  getMessages(42) → [system_prompt, msg1, msg2, ..., newMsg]     │
│  truncateToTokenBudget(messages, 2048)                          │
│  Headers SSE émis                                               │
│  MistralClient::streamMessage(messages, $onChunk)               │
└──────────────────────┬──────────────────────────────────────────┘
                       │ cURL CURLOPT_WRITEFUNCTION
                       ▼
          [API Mistral] ──── stream SSE ────
                       │
              ┌────────▼────────┐
              │ chunk reçu      │  (boucle)
              │ parseSseChunk() │
              │ token extrait   │
              └────────┬────────┘
                       │
                       │  echo "data: {\"token\":\"La\"}\n\n"
                       │  ob_flush() + flush()
                       │
                       ▼ (ReadableStream JS)
              [api.js] onToken("La")
              [ui.js::appendTokenToBuffer(element, "La")]
              [ui.js::scrollToBottom()]
                       │
               ... (répété pour chaque token) ...
                       │
              [API Mistral] : data: [DONE]
                       │
                       │  echo "data: [DONE]\n\n" + flush()
                       │  saveMessage(42, 'assistant', fullResponse)
                       │
                       ▼
              [api.js] onDone()
              [ui.js::finalizeAssistantBubble(element)]
                  ├── renderMarkdown(rawContent) → marked.parse()
                  ├── DOMPurify.sanitize(html)
                  ├── element.innerHTML = safeHtml
                  └── element.classList.remove('streaming')
              [app.js::setStreaming(false)]
```

---

### Cas B — Premier message (nouvelle conversation, `conversation_id = null`)

```
[Utilisateur tape son premier message dans un chat vide]
        │
        │  currentConversationId = null
        ▼
[app.js::handleSubmit()]
[ui.js::appendUserMessage(message)]
[ui.js::createAssistantBubble()] → <element>
        │
        ▼
[api.js::streamChat(null, message, callbacks)]
        │
        │  fetch POST api/chat.php
        │  body: { "conversation_id": null, "message": "..." }
        ▼
┌─────────────────────────────────────────────────────────────────┐
│ api/chat.php                                                    │
│                                                                 │
│  conversation_id === null détecté                               │
│  ConversationManager::createConversation() → id = 43            │
│  saveMessage(43, 'user', message)                               │
│                                                                 │
│  ── Émis AVANT les premiers tokens ──                           │
│  echo "event: init\n"                                           │
│  echo "data: {\"conversation_id\":43}\n\n"                      │
│  flush()                                                        │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼ (ReadableStream JS — reçu avant les tokens)
              [api.js] onInit(43)
              [app.js] currentConversationId = 43
              [app.js::refreshSidebar()] → api.listConversations() → ui.renderSidebar()
              [ui.js::setActiveConversation(43)]
                       │
                       ▼
              [suite identique au Cas A — stream des tokens]
                       │
                       ▼ (après data: [DONE])
┌─────────────────────────────────────────────────────────────────┐
│ api/chat.php — après saveMessage(assistant)                     │
│                                                                 │
│  $isNew === true → ConversationManager::generateTitle(43, msg)  │
│    └─ MistralClient::sendMessage([{role:'user',                 │
│         content:'Résume en 5 mots max : {message}'}])           │
│    └─ UPDATE conversations SET title = 'Titre généré' WHERE id=43│
│                                                                 │
│  echo "event: title\n"                                          │
│  echo "data: {\"title\":\"Titre généré\"}\n\n"                  │
│  flush()                                                        │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
              [api.js] onTitle("Titre généré")
              [app.js] met à jour le texte de l'item sidebar id=43
              (sans rechargement complet de la liste)
```

---

### Cas C — Erreur mi-stream

```
[Scénario A : réseau coupé côté client]
        │
        ▼
[fetch ReadableStream] → reader.read() reject ou stream termine prématurément
        │
        ▼
[api.js] catch(error) → onError("Connexion interrompue")
        │
        ▼
[ui.js::errorAssistantBubble(element, "Connexion interrompue")]
        ├── element.classList.remove('streaming') → curseur arrêté
        └── insère "⚠ Connexion interrompue" dans la bulle (tokens partiels conservés)
[app.js::setStreaming(false)] → formulaire réactivé
[app.js] enregistre l'état partiel : l'utilisateur peut renvoyer son message

┌─ côté serveur (ignore_user_abort = true) ──────────────────────┐
│  chat.php continue jusqu'à la fin du stream Mistral            │
│  saveMessage(43, 'assistant', fullResponse) est exécuté        │
│  La conversation est sauvegardée en SQLite même si le client   │
│  est parti — cohérence des données garantie                    │
└────────────────────────────────────────────────────────────────┘

[Scénario B : erreur API Mistral (quota, timeout, 5xx)]
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│ api/chat.php                                                    │
│                                                                 │
│  catch(MistralException $e)                                     │
│    echo "event: error\n"                                        │
│    echo "data: {\"message\":\"" . addslashes($e->getMessage())  │
│         . "\"}\n\n"                                             │
│    flush()                                                      │
│    exit                                                         │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
              [api.js] onError("Quota dépassé — réessayez plus tard")
              [ui.js::errorAssistantBubble(element, message)]
              [app.js::setStreaming(false)]
```

---

## Sécurité

| Risque | Mitigation | Fichier responsable |
|---|---|---|
| Accès non authentifié | HTTP Basic Auth Apache sur tout le projet | `.htaccess` (généré par `setup.php`) |
| Exposition clé API Mistral | `.env` bloqué par FilesMatch + absent du git | `.htaccess` · `.gitignore` |
| Accès direct à `.htpasswd` | FilesMatch `Require all denied` | `.htaccess` |
| Accès direct à la base SQLite | DirectoryMatch `Require all denied` sur `storage/` | `.htaccess` |
| XSS via tokens Mistral (stockés en SQLite) | `DOMPurify.sanitize()` obligatoire avant tout `innerHTML` | `assets/js/ui.js` |
| XSS via input utilisateur | `textContent` au lieu de `innerHTML` pour les bulles utilisateur | `assets/js/ui.js` |
| CSRF sur endpoints REST mutants | Header `X-CSRF-Token` validé contre `$_SESSION['csrf_token']` | `api/history.php` · `config/config.php` |
| Injection SQL | PDO `prepare()` + `execute()` exclusivement, zéro interpolation | `src/ConversationManager.php` |
| Compression gzip cassant le SSE | `SetEnvIf no-gzip` pour `api/chat.php` | `.htaccess` |
| Timeout Apache sur streams longs | `set_time_limit(0)` | `api/chat.php` |
| Perte données si onglet fermé | `ignore_user_abort(true)` | `api/chat.php` |
| Conflit output buffer (mod_php Windows) | `ob_get_level() > 0 → ob_end_clean()` | `api/chat.php` |
| Lock SQLite concurrent (read + write) | Mode WAL + `PRAGMA busy_timeout = 5000` | `src/Database.php` |
| Certificat SSL absent Windows | `CURLOPT_CAINFO` vers le bundle CA de Laragon en dev | `src/MistralClient.php` |
| `.env` silencieusement absent | Vérification du retour de `parse_ini_file()` + exception explicite | `config/config.php` |
| `system_prompt.txt` absent | Vérification + exception dans `loadSystemPrompt()` | `src/ConversationManager.php` |
| `setup.php` accessible après installation | FilesMatch dans `.htaccess` généré | `.htaccess` |

---

## Installation

### Prérequis
- Laragon installé (Apache 2.4, PHP 8.1+)
- Extensions PHP actives : `pdo_sqlite`, `curl` (vérifiables via `php -m` dans le terminal Laragon)
- Clé API Mistral (obtenue sur console.mistral.ai)

### Étapes

**1. Cloner le projet**
```
C:/laragon/www/mistralchat/
```

**2. Créer le fichier `.env`**

Copier `.env.example` → `.env` et renseigner toutes les valeurs :
```ini
MISTRAL_API_KEY=sk-votre-cle-ici
MISTRAL_MODEL=mistral-large-latest
MISTRAL_TEMPERATURE=0.7
MISTRAL_MAX_TOKENS=2048
AUTH_USER=admin
AUTH_PASS=votre_mot_de_passe_fort
APP_ENV=dev
```

**3. Activer les modules Apache dans Laragon**

Dans Laragon → Menu → Apache → `httpd.conf`, vérifier que ces lignes sont décommentées :
```apache
LoadModule rewrite_module modules/mod_rewrite.so
LoadModule headers_module  modules/mod_headers.so
LoadModule deflate_module  modules/mod_deflate.so
```

Redémarrer Laragon après modification.

**4. Exécuter `setup.php`**

Ouvrir `http://mistralchat.test/setup.php` dans le navigateur (ou `php setup.php` dans le terminal Laragon).

Le script :
- Vérifie PHP 8.1+, `pdo_sqlite`, `curl`
- Génère `.htpasswd` avec le hash APR1-MD5 de `AUTH_PASS` (sans CLI)
- Génère `.htaccess` avec le chemin absolu correct pour `AuthUserFile` (ex: `C:/laragon/www/mistralchat/.htpasswd`)
- Crée le dossier `storage/`
- Affiche un rapport ✓/✗ pour chaque étape

En cas d'erreur sur une étape, le rapport indique la correction précise à effectuer.

**5. Vérifier l'installation**

Accéder à `http://mistralchat.test` → une popup de login apparaît → saisir `AUTH_USER` / `AUTH_PASS` → l'interface du chatbot s'affiche.

La base SQLite `storage/mistralchat.db` est créée automatiquement au premier appel API.

**6. Personnaliser le persona du bot**

Éditer `config/system_prompt.txt` avec les instructions souhaitées. Aucun redémarrage nécessaire — lu à chaque requête.

> **Note** : `setup.php` est automatiquement bloqué par `.htaccess` après l'installation. Pour le ré-exécuter (changement de mot de passe), supprimer temporairement la règle FilesMatch correspondante dans `.htaccess`.

---

## Dépendances

| Couche | Technologie | Version minimale | Justification |
|---|---|---|---|
| Serveur | PHP | 8.1+ | Types natifs, `readonly`, `enum` — aucun polyfill nécessaire |
| Serveur web | Apache via Laragon | 2.4+ | `mod_rewrite`, `mod_headers`, `mod_deflate` requis |
| Base de données | SQLite via PDO | 3.35+ | Embarquée dans PHP, mode WAL stable, zéro configuration serveur |
| IA | API Mistral | — | Endpoint `/v1/chat/completions` compatible OpenAI, supporte SSE natif |
| Frontend | HTML/CSS/JS vanilla | ES2020+ | Modules natifs (`type="module"`), `ReadableStream`, `TextDecoder` — zéro bundler |
| Sanitisation HTML | DOMPurify | 3.x | Référence XSS côté client, maintenu activement, CDN cdnjs |
| Rendu Markdown | marked.js | 9.x | Rapide, léger, configurable, CDN cdnjs |
| Coloration code | highlight.js | 11.x | Optionnel — détecte le langage automatiquement, CDN cdnjs |

> Aucun framework PHP (Laravel, Symfony, Slim). Aucun bundler JS (Webpack, Vite). Aucune dépendance Composer. L'application démarre avec PHP + Apache natifs de Laragon, sans configuration supplémentaire au-delà des étapes d'installation.

---

## Schéma SQLite

Géré par `src/Database.php::initSchema()`. Exécuté une seule fois à la création de la base.

```sql
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;
PRAGMA busy_timeout = 5000;

CREATE TABLE IF NOT EXISTS conversations (
    id         INTEGER  PRIMARY KEY AUTOINCREMENT,
    title      TEXT     NOT NULL DEFAULT 'Nouvelle conversation',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS messages (
    id              INTEGER  PRIMARY KEY AUTOINCREMENT,
    conversation_id INTEGER  NOT NULL,
    role            TEXT     NOT NULL CHECK(role IN ('user', 'assistant', 'system')),
    content         TEXT     NOT NULL,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (conversation_id)
        REFERENCES conversations(id)
        ON DELETE CASCADE
);

-- Index sur la FK pour accélérer getMessages() et deleteConversation()
CREATE INDEX IF NOT EXISTS idx_messages_conv_id
    ON messages(conversation_id);

-- Index sur updated_at pour accélérer listConversations() ORDER BY updated_at DESC
CREATE INDEX IF NOT EXISTS idx_conversations_updated
    ON conversations(updated_at);

-- Trigger : met à jour conversations.updated_at automatiquement
-- à chaque insertion de message utilisateur ou assistant
-- (pas pour role='system' qui n'est jamais inséré en base)
CREATE TRIGGER IF NOT EXISTS trg_conv_updated
    AFTER INSERT ON messages
    WHEN NEW.role IN ('user', 'assistant')
BEGIN
    UPDATE conversations
    SET    updated_at = CURRENT_TIMESTAMP
    WHERE  id = NEW.conversation_id;
END;
```

### Notes sur le schéma

- **Le trigger** `trg_conv_updated` remplace la nécessité d'appeler manuellement `UPDATE conversations SET updated_at = ...` après chaque `saveMessage()` — la sidebar (`ORDER BY updated_at DESC`) reste triée correctement automatiquement
- **`ON DELETE CASCADE`** sur `messages.conversation_id` : `deleteConversation()` n'a besoin que d'un seul `DELETE FROM conversations WHERE id = ?` — SQLite supprime les messages associés automatiquement
- **`CHECK(role IN (...))`** : contrainte au niveau DB — un `role` invalide levait déjà une exception PDO avant même d'atteindre la couche applicative
- **WAL** (`journal_mode = WAL`) : les lectures (sidebar, historique) ne bloquent pas les écritures (fin de stream). Sans WAL en mode journal par défaut (`DELETE`), `history.php` et `chat.php` se bloqueraient mutuellement sur Laragon où plusieurs requêtes HTTP peuvent s'exécuter en parallèle
