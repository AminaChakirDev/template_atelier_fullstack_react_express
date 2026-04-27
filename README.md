# Atelier – Portfolio Full-Stack
## Express + MySQL + React + Tailwind · 2 jours

---

> **Objectif pédagogique**
> Concevoir et développer une application web full-stack de type portfolio personnel.
> L'admin peut se connecter, gérer ses projets (CRUD), et les visiteurs peuvent envoyer un message de contact.
> Cet atelier mobilise l'ensemble de la stack DWWM : architecture en couches (routes → controller → service → model), JWT avec rôle, validation back (`express-validator`) et front (React Hook Form), MySQL, React, Tailwind.

---

## Stack technique

| Côté | Technologie |
|---|---|
| Back-end | Node.js · Express 5 (ES Modules) · MySQL2 |
| Validation back | `express-validator` |
| Auth | JWT (`jsonwebtoken`) + `bcrypt` |
| Mail | Nodemailer + Mailjet SMTP |
| Front-end | React 19 (Vite) · Tailwind CSS v4 |
| Validation front | React Hook Form |
| Routing front | React Router v6 |
| BDD | MySQL / MariaDB |

---

## Rendu attendu

- **2 dépôts GitHub distincts** : `portfolio-backend` et `portfolio-frontend`
- Un fichier `README.md` dans chaque repo expliquant comment installer et lancer le projet
- Variables d'environnement documentées dans `.env.example` (jamais le `.env` réel)
- Architecture en couches respectée côté back : routes → controller → **service** → model
- Validation obligatoire : `express-validator` côté back, React Hook Form côté front
- `async/await` utilisé **partout** (aucun `.then()` dans le code rendu)
- Code propre, composants organisés côté front

---

# JOUR 1 — Back-end

---

## Étape 1 · Mise en place du projet back-end _(~30 min)_

### 1.1 Initialisation

```bash
mkdir portfolio-backend && cd portfolio-backend
npm init -y
```

Modifier `package.json` pour activer les ES Modules et ajouter les scripts :

```json
{
  "type": "module",
  "scripts": {
    "dev": "node --watch src/server.js",
    "start": "node src/server.js"
  }
}
```

### 1.2 Installation des dépendances

```bash
npm install express mysql2 bcrypt jsonwebtoken dotenv cors nodemailer express-validator
```

### 1.3 Structure de dossiers à créer

L'architecture utilisée est en **4 couches** :

```
portfolio-backend/
├── src/
│   ├── config/
│   │   └── db.js                    ← Pool de connexion MySQL
│   ├── controllers/
│   │   ├── auth.controller.js        ← Reçoit req/res, délègue au service
│   │   ├── project.controller.js
│   │   └── contact.controller.js
│   ├── services/                     ← ★ Logique métier (nouvelle couche)
│   │   ├── auth.service.js           ← Vérif identifiants, génération JWT
│   │   ├── project.service.js        ← Logique CRUD, règles métier
│   │   └── contact.service.js        ← Envoi d'email
│   ├── models/
│   │   ├── user.model.js             ← Requêtes SQL utilisateurs
│   │   └── project.model.js          ← Requêtes SQL projets
│   ├── middlewares/
│   │   ├── auth.middleware.js         ← Vérifie le JWT
│   │   ├── authorize.middleware.js    ← Vérifie le rôle (admin, etc.)
│   │   ├── validate.middleware.js     ← Récupère les erreurs express-validator
│   │   └── errorHandler.js            ← Gestionnaire d'erreurs centralisé
│   ├── validators/
│   │   ├── project.validator.js       ← Règles de validation des projets
│   │   └── contact.validator.js       ← Règles de validation du contact
│   ├── routes/
│   │   ├── auth.routes.js
│   │   ├── project.routes.js
│   │   └── contact.routes.js
│   └── server.js
├── .env
├── .env.example
├── .gitignore
└── package.json
```

> 💡 **Créer tous les dossiers et fichiers vides dès maintenant**, avant d'écrire la moindre ligne de code. Cela donne une vision d'ensemble de l'architecture.

**Rappel du rôle de chaque couche :**

| Couche | Responsabilité |
|---|---|
| `routes` | Déclare les URL et les middlewares associés |
| `controllers` | Reçoit `req`/`res`, extrait les données, appelle le service, renvoie la réponse |
| `services` | Contient toute la logique métier (pas de `req`/`res` ici) |
| `models` | Exécute les requêtes SQL |

### 1.4 Fichier `.env`

```env
PORT=3001
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=
DB_NAME=portfolio_db
JWT_SECRET=un_secret_tres_long_et_aleatoire_a_remplacer
MJ_APIKEY_PUBLIC=ta_cle_publique_mailjet
MJ_APIKEY_PRIVATE=ta_cle_privee_mailjet
MAIL_FROM=tonemail@domaine.com
MAIL_TO=destinataire@domaine.com
```

Créer également `.env.example` avec les mêmes clés, valeurs vides, et ajouter `.env` dans `.gitignore`.

---

## Étape 2 · Base de données _(~30 min)_

### 2.1 Créer la base et les tables

Dans phpMyAdmin ou le terminal MySQL :

```sql
CREATE DATABASE IF NOT EXISTS portfolio_db
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

USE portfolio_db;

-- La colonne role permet la protection par rôle
CREATE TABLE users (
  id         INT AUTO_INCREMENT PRIMARY KEY,
  email      VARCHAR(255) NOT NULL UNIQUE,
  password   VARCHAR(255) NOT NULL,
  role       VARCHAR(50)  NOT NULL DEFAULT 'admin',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE projects (
  id          INT AUTO_INCREMENT PRIMARY KEY,
  title       VARCHAR(150)  NOT NULL,
  description TEXT,
  tech_stack  VARCHAR(255),
  github_url  VARCHAR(500),
  demo_url    VARCHAR(500),
  image_url   VARCHAR(500),
  created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### 2.2 Insérer l'utilisateur admin

Il n'y aura **pas** de route `/register` publique. L'admin est créé une seule fois, directement en base, avec un mot de passe **pré-haché**.

Pour générer le hash, utilisez l'un de ces deux outils au choix :

**Option A — outil en ligne :**
👉 [https://bcrypt-generator.com](https://bcrypt-generator.com) (cost factor : 10)

**Option B — commande Node dans le terminal :**
```bash
node -e "import('bcrypt').then(b => b.default.hash('VotreMotDePasse', 10).then(console.log))"
```

Puis insérer en SQL (remplacer le hash par celui que vous avez généré) :

```sql
INSERT INTO users (email, password, role)
VALUES ('admin@portfolio.fr', '$2b$10$VOTRE_HASH_ICI', 'admin');
```

### 2.3 Configurer la connexion MySQL

Dans `src/config/db.js`, créer et exporter un **pool** de connexions en utilisant `mysql2/promise` et les variables d'environnement.

> 📖 Vous avez déjà fait cette configuration dans le cours myblog. Retrouvez votre cours et adaptez-le.

---

## Étape 3 · Serveur Express _(~20 min)_

### 3.1 Structure minimale de `server.js`

Voici la structure de base attendue. À vous de la compléter au fur et à mesure des étapes :

```js
import 'dotenv/config';
import express from 'express';
import cors from 'cors';
// TODO : importer vos routes au fur et à mesure

import errorHandler from './middlewares/errorHandler.js';

const app = express();
const PORT = process.env.PORT || 3001;

// Middlewares globaux
app.use(cors({ origin: 'http://localhost:5173' }));
app.use(express.json());

// Exemple avec une route — à dupliquer pour chaque groupe de routes
app.use('/api/auth', authRoutes);
// TODO : brancher les autres routes ici

// Gestionnaire d'erreurs — toujours EN DERNIER
app.use(errorHandler);

app.listen(PORT, () => {
  console.log(`Serveur démarré sur http://localhost:${PORT}`);
});
```

### 3.2 Middleware `errorHandler`

Créer `src/middlewares/errorHandler.js`. Ce middleware intercepte toutes les erreurs et renvoie une réponse JSON propre.

> Sa signature est particulière : il prend **4 paramètres** `(err, req, res, next)`. Retrouvez le pattern dans votre cours sur la gestion d'erreurs Express.

### 3.3 Middleware `validate`

Créer `src/middlewares/validate.middleware.js`. Ce middleware est appelé **après** les règles `express-validator` dans une route, pour vérifier s'il y a des erreurs et renvoyer un `400` si c'est le cas.

```js
import { validationResult } from 'express-validator';

const validate = (req, res, next) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ errors: errors.array() });
  }
  next();
};

export default validate;
```

> Ce middleware est le même pour toutes les routes. Il est branché entre les règles de validation et le contrôleur.

---

## Étape 4 · Authentification JWT _(~45 min)_

### 4.1 Modèle utilisateur

Dans `src/models/user.model.js`, écrire une fonction `findByEmail(email)` qui interroge la table `users` avec une requête paramétrée et retourne l'utilisateur trouvé (ou `null`).

> ⚠️ Toutes les fonctions doivent utiliser `async/await`. Aucun `.then()` n'est accepté dans le code rendu.

### 4.2 Service auth

Dans `src/services/auth.service.js`, écrire la fonction `loginUser({ email, password })` qui :

1. Appelle `userModel.findByEmail(email)`
2. Renvoie une erreur `401` si l'utilisateur n'existe pas
3. Compare le mot de passe avec `bcrypt.compare`
4. Renvoie une erreur `401` si le mot de passe est incorrect
5. Génère un JWT signé avec `{ id, email, role }` dans le payload (durée : `24h`)
6. Retourne le token

> Le service ne connaît pas `req` ni `res`. Il travaille avec des données brutes et lance des erreurs si nécessaire.

### 4.3 Contrôleur auth

Dans `src/controllers/auth.controller.js`, écrire `login(req, res, next)` qui :

1. Extrait `email` et `password` de `req.body`
2. Appelle `authService.loginUser(...)` dans un bloc `try/catch`
3. Renvoie `res.json({ token })` en cas de succès
4. Passe l'erreur à `next(err)` en cas d'échec

### 4.4 Middleware `authenticate`

Dans `src/middlewares/auth.middleware.js`, créer le middleware `authenticate` qui :

1. Lit le header `Authorization`
2. Vérifie qu'il commence par `'Bearer '`
3. Extrait et vérifie le token avec `jwt.verify`
4. Stocke le payload décodé dans `req.user`
5. Appelle `next()` ou renvoie une erreur `401`

### 4.5 Middleware `authorize`

Dans `src/middlewares/authorize.middleware.js`, créer une fonction `authorize(...roles)` qui retourne un middleware vérifiant que `req.user.role` fait partie des rôles autorisés.

```js
// Exemple d'usage attendu dans une route :
router.post('/', authenticate, authorize('admin'), createProject);
```

> `authorize` est une **factory** : elle prend des rôles en paramètre et retourne un middleware. Si le rôle de l'utilisateur ne correspond pas, répondre `403 Forbidden`.

### 4.6 Routes auth

Dans `src/routes/auth.routes.js`, déclarer `POST /login` et la brancher sur le contrôleur.

### ✅ Test Étape 4

Tester dans Thunder Client ou Postman :

```
POST http://localhost:3001/api/auth/login
Body JSON : { "email": "admin@portfolio.fr", "password": "VotreMotDePasse" }
```

Résultats attendus :
- Bons identifiants → `200` + `{ "token": "eyJ..." }`
- Mauvais mot de passe → `401`
- Email absent → `400`

---

## Étape 5 · Validation back avec express-validator _(~20 min)_

Avant de développer le CRUD projets, mettre en place la validation.

### 5.1 Règles de validation des projets

Dans `src/validators/project.validator.js`, exporter un tableau de règles `validateProject` couvrant :

| Champ | Règles |
|---|---|
| `title` | Obligatoire · chaîne · min 2 caractères · max 150 caractères |
| `description` | Optionnel · si présent, chaîne · max 2000 caractères |
| `tech_stack` | Optionnel · si présent, chaîne · max 255 caractères |
| `github_url` | Optionnel · si présent, doit être une URL valide (`isURL`) |
| `demo_url` | Optionnel · si présent, doit être une URL valide (`isURL`) |
| `image_url` | Optionnel · si présent, doit être une URL valide (`isURL`) |

> 📖 Syntaxe de référence dans votre cours sur `express-validator` : `body('champ').notEmpty().isLength(...)` etc.

### 5.2 Règles de validation du contact

Dans `src/validators/contact.validator.js`, exporter `validateContact` couvrant :

| Champ | Règles |
|---|---|
| `name` | Obligatoire · chaîne · min 2 · max 100 caractères |
| `email` | Obligatoire · format email valide (`isEmail`) |
| `message` | Obligatoire · chaîne · min 10 · max 2000 caractères |

---

## Étape 6 · CRUD Projets _(~60 min)_

### 6.1 Modèle projet

Dans `src/models/project.model.js`, écrire les fonctions suivantes (toutes en `async/await`) :

| Fonction | Requête SQL |
|---|---|
| `findAll()` | `SELECT * FROM projects ORDER BY created_at DESC` |
| `findById(id)` | `SELECT * FROM projects WHERE id = ?` |
| `create(data)` | `INSERT INTO projects (title, ...) VALUES (?, ...)` puis `findById(insertId)` |
| `update(id, data)` | `UPDATE projects SET ... WHERE id = ?` puis `findById(id)` |
| `remove(id)` | `DELETE FROM projects WHERE id = ?` → retourne `affectedRows > 0` |

> ⚠️ Toutes les requêtes doivent être **paramétrées** avec `?`. Jamais de concaténation de chaînes.

### 6.2 Service projet

Dans `src/services/project.service.js`, écrire les fonctions métier :

| Fonction service | Ce qu'elle fait |
|---|---|
| `getAllProjects()` | Appelle `model.findAll()` |
| `getProjectById(id)` | Appelle `model.findById(id)`, lance une erreur `404` si `null` |
| `createProject(data)` | Appelle `model.create(data)` |
| `updateProject(id, data)` | Vérifie que le projet existe (via `findById`), appelle `model.update` |
| `deleteProject(id)` | Appelle `model.remove(id)`, lance une erreur `404` si `false` |

### 6.3 Contrôleur projet

Dans `src/controllers/project.controller.js`, écrire une fonction pour chaque action (`getAllProjects`, `getProjectById`, `createProject`, `updateProject`, `deleteProject`). Chaque contrôleur :
- Extrait les données nécessaires de `req.body` ou `req.params`
- Appelle la fonction de service correspondante
- Renvoie la réponse JSON avec le bon code HTTP
- Attrape les erreurs avec `try/catch` et les passe à `next(err)`

### 6.4 Routes projet

Dans `src/routes/project.routes.js`, déclarer les routes en appliquant les middlewares appropriés :

| Route | Middlewares |
|---|---|
| `GET /` | — |
| `GET /:id` | — |
| `POST /` | `authenticate` · `authorize('admin')` · `validateProject` · `validate` |
| `PUT /:id` | `authenticate` · `authorize('admin')` · `validateProject` · `validate` |
| `DELETE /:id` | `authenticate` · `authorize('admin')` |

### ✅ Tests Étape 6

```
GET    /api/projects              → 200 []
GET    /api/projects/999          → 404
POST   /api/projects (sans token) → 401
POST   /api/projects (token admin, title manquant) → 400 + erreurs de validation
POST   /api/projects (token admin, données valides) → 201 + projet créé
PUT    /api/projects/1 (token admin) → 200 + projet modifié
DELETE /api/projects/1 (token admin) → 204
```

---

## Étape 7 · Formulaire de contact _(~30 min)_

### 7.1 Service contact

Dans `src/services/contact.service.js`, écrire `sendContactEmail({ name, email, message })` qui :
- Crée un transporteur Nodemailer configuré avec les variables d'environnement Mailjet SMTP
- Envoie un email formaté à l'adresse `MAIL_TO`
- Lance une erreur en cas d'échec d'envoi

> Le transporteur Nodemailer pour Mailjet utilise le host `in-v3.mailjet.com` sur le port `587`.

### 7.2 Contrôleur et route contact

- Dans `src/controllers/contact.controller.js` : `sendContact(req, res, next)` qui extrait les données et appelle le service.
- Dans `src/routes/contact.routes.js` : `POST /` avec les middlewares `validateContact` et `validate`.

### ✅ Test Étape 7

```
POST /api/contact
Body : { "name": "Alice", "email": "alice@test.fr", "message": "Bonjour, je vous contacte !" }
→ 200 { message: "Message envoyé avec succès" }

POST /api/contact
Body : { "name": "A", "email": "pasunemail", "message": "x" }
→ 400 + tableau d'erreurs de validation
```

---

## ✅ Récap fin Jour 1

À ce stade, votre API expose :

| Méthode | Route | Auth | Rôle requis | Description |
|---|---|---|---|---|
| POST | `/api/auth/login` | ❌ | — | Connexion admin |
| GET | `/api/projects` | ❌ | — | Liste des projets |
| GET | `/api/projects/:id` | ❌ | — | Détail d'un projet |
| POST | `/api/projects` | ✅ | admin | Créer un projet |
| PUT | `/api/projects/:id` | ✅ | admin | Modifier un projet |
| DELETE | `/api/projects/:id` | ✅ | admin | Supprimer un projet |
| POST | `/api/contact` | ❌ | — | Envoyer un message |

---

# JOUR 2 — Front-end React + Tailwind

---

## Étape 8 · Mise en place du projet front-end _(~20 min)_

### 8.1 Initialisation

```bash
npm create vite@latest portfolio-frontend -- --template react
cd portfolio-frontend
npm install
npm install tailwindcss @tailwindcss/vite react-router-dom react-hook-form
```

### 8.2 Configurer Tailwind dans Vite

Dans `vite.config.js`, ajouter le plugin Tailwind :

```js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  plugins: [react(), tailwindcss()],
});
```

Dans `src/index.css`, remplacer tout le contenu par :

```css
@import "tailwindcss";
```

### 8.3 Structure de dossiers

```
portfolio-frontend/
├── src/
│   ├── api/
│   │   └── apiFetch.js              ← Fonction centralisée pour tous les appels API
│   ├── components/
│   │   ├── Navbar.jsx
│   │   ├── ProjectCard.jsx
│   │   └── ContactForm.jsx          ← Utilise React Hook Form
│   ├── context/
│   │   └── AuthContext.jsx          ← AuthContext + AuthProvider
│   ├── pages/
│   │   ├── HomePage.jsx
│   │   ├── ProjectsPage.jsx
│   │   ├── ProjectDetailPage.jsx    ← Page détail d'un projet
│   │   ├── LoginPage.jsx            ← Utilise React Hook Form
│   │   └── AdminPage.jsx            ← Utilise React Hook Form
│   ├── App.jsx
│   ├── main.jsx
│   └── index.css
├── .env
├── .env.example
├── vite.config.js
└── package.json
```

### 8.4 Variables d'environnement front

```env
VITE_API_URL=http://localhost:3001/api
```

---

## Étape 9 · Utilitaire API _(~15 min)_

### 9.1 Fonction `apiFetch`

Dans `src/api/apiFetch.js`, écrire une fonction `apiFetch(endpoint, options)` qui :

1. Lit `import.meta.env.VITE_API_URL` pour construire l'URL complète
2. Récupère le token depuis le `localStorage`
3. Construit les headers : `Content-Type: application/json` + `Authorization: Bearer <token>` si token présent
4. Appelle `fetch` avec `async/await`
5. Si la réponse n'est pas `ok`, extrait le JSON de l'erreur et lance une `Error`
6. Si le statut est `204` (No Content), retourne `null`
7. Sinon, retourne `response.json()`

> 📖 Cette fonction est similaire à celle que vous avez développée dans le cours myblog. Retrouvez-la et adaptez-la à cette architecture.

---

## Étape 10 · Contexte d'authentification _(~20 min)_

### 10.1 `AuthContext` avec `AuthProvider`

Dans `src/context/AuthContext.jsx`, créer et exporter :

- `AuthContext` : le contexte React
- `AuthProvider` : le composant provider qui wrape les enfants. Il expose via la valeur du contexte :
  - `token` (string ou null, initialisé depuis `localStorage`)
  - `isAuthenticated` (booléen dérivé de `token`)
  - `user` (payload JWT décodé — utilisez `JSON.parse(atob(token.split('.')[1]))` pour le décoder sans librairie)
  - `login(token)` : stocke le token dans `localStorage` et met à jour le state
  - `logout()` : supprime le token de `localStorage` et remet le state à `null`
- `useAuth` : hook personnalisé qui retourne `useContext(AuthContext)`

### 10.2 Brancher le provider dans `main.jsx`

```jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import { AuthProvider } from './context/AuthContext.jsx';
import App from './App.jsx';
import './index.css';

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <BrowserRouter>
      <AuthProvider>
        <App />
      </AuthProvider>
    </BrowserRouter>
  </StrictMode>
);
```

---

## Étape 11 · Routing et Navbar _(~20 min)_

### 11.1 `App.jsx` — routes et protection

Dans `App.jsx`, définir les routes avec React Router. Créer un composant `PrivateRoute` qui :
- Lit `isAuthenticated` depuis `useAuth()`
- Rend les enfants si authentifié
- Redirige vers `/login` sinon avec `<Navigate>`

Routes à déclarer :

| Path | Composant | Protection |
|---|---|---|
| `/` | `HomePage` | Public |
| `/projects` | `ProjectsPage` | Public |
| `/projects/:id` | `ProjectDetailPage` | Public |
| `/login` | `LoginPage` | Public |
| `/admin` | `AdminPage` | `PrivateRoute` |

### 11.2 `Navbar`

Le composant `Navbar` doit :
- Afficher les liens de navigation (Accueil, Projets)
- Utiliser `useAuth()` pour afficher "Connexion" ou "Déconnexion" selon l'état
- Appeler `logout()` puis rediriger vers `/` au clic sur "Déconnexion"

---

## Étape 12 · Page de connexion _(~20 min)_

### Consignes

Dans `src/pages/LoginPage.jsx`, utiliser **React Hook Form** pour gérer le formulaire de connexion.

Règles de validation RHF à appliquer :

| Champ | Règles |
|---|---|
| `email` | Requis · format email valide |
| `password` | Requis · min 6 caractères |

**Comportement attendu :**
- `handleSubmit` de RHF appelle `apiFetch('/auth/login', { method: 'POST', body: ... })`
- En cas de succès : appeler `login(data.token)` puis rediriger vers `/admin`
- En cas d'erreur : afficher le message d'erreur retourné par l'API
- Les messages d'erreur RHF s'affichent sous chaque champ en temps réel

> 📖 Syntaxe RHF : `const { register, handleSubmit, formState: { errors } } = useForm()`

---

## Étape 13 · Page Projets et carte projet _(~30 min)_

### 13.1 Composant `ProjectCard`

Créer `src/components/ProjectCard.jsx`. La carte affiche :
- L'image du projet (si `image_url` présent)
- Le titre
- La description tronquée
- Les badges technologies (splitter `tech_stack` sur `,`)
- Les liens GitHub et Démo (si présents)
- Un lien vers la page de détail `/projects/:id`

### 13.2 `ProjectsPage`

Dans `src/pages/ProjectsPage.jsx` :
- Charger les projets au montage avec `useEffect` + `apiFetch('/projects')` en `async/await`
- Gérer les états `loading`, `error`, `projects`
- Afficher la grille de `ProjectCard`

---

## Étape 14 · Page de détail d'un projet _(~20 min)_

Dans `src/pages/ProjectDetailPage.jsx` :
- Récupérer l'`id` depuis l'URL avec `useParams()`
- Charger le projet via `apiFetch('/projects/' + id)` au montage
- Afficher toutes les informations du projet (titre, description complète, technologies, liens, image)
- Afficher un bouton "← Retour aux projets" qui navigue vers `/projects`
- Gérer les cas `loading` et `error` (dont le `404`)

---

## Étape 15 · Page Admin — CRUD complet avec React Hook Form _(~60 min)_

### Consignes

Dans `src/pages/AdminPage.jsx`, utiliser **React Hook Form** pour le formulaire de création/modification de projet.

**Règles de validation RHF :**

| Champ | Règles |
|---|---|
| `title` | Requis · min 2 · max 150 caractères |
| `description` | Optionnel · max 2000 caractères |
| `tech_stack` | Optionnel · max 255 caractères |
| `github_url` | Optionnel · si renseigné, doit commencer par `https://` |
| `demo_url` | Optionnel · si renseigné, doit commencer par `https://` |
| `image_url` | Optionnel · si renseigné, doit commencer par `https://` |

**Fonctionnalités à implémenter :**

1. **Lister** tous les projets au montage
2. **Créer** : le formulaire vide → `POST /api/projects` → recharge la liste
3. **Modifier** : clic "Modifier" → pré-remplir le formulaire avec `reset(project)` de RHF → `PUT /api/projects/:id` → recharge la liste
4. **Supprimer** : confirmation → `DELETE /api/projects/:id` → met à jour la liste
5. **Annuler** : bouton "Annuler" qui remet le formulaire à vide avec `reset()` et désactive le mode édition

> 💡 `useForm` expose la méthode `reset(values)` pour pré-remplir le formulaire avec les données d'un projet existant.

---

## Étape 16 · Page d'accueil et formulaire de contact _(~30 min)_

### 16.1 `ContactForm` avec React Hook Form

Dans `src/components/ContactForm.jsx`, utiliser **React Hook Form**.

**Règles de validation RHF :**

| Champ | Règles |
|---|---|
| `name` | Requis · min 2 caractères |
| `email` | Requis · format email valide |
| `message` | Requis · min 10 caractères |

**Comportement :**
- `handleSubmit` appelle `apiFetch('/contact', { method: 'POST', body: ... })`
- Succès : message de confirmation + `reset()` du formulaire
- Erreur : message d'erreur affiché

### 16.2 `HomePage`

Dans `src/pages/HomePage.jsx` :
- Section hero avec nom, titre, description courte et bouton vers `/projects`
- Section contact avec le composant `ContactForm`

---

## ✅ Récap fin Jour 2

L'application complète est fonctionnelle :

| Page | URL | Accès |
|---|---|---|
| Accueil + contact | `/` | Public |
| Liste des projets | `/projects` | Public |
| Détail d'un projet | `/projects/:id` | Public |
| Connexion | `/login` | Public |
| Dashboard admin CRUD | `/admin` | Privé — role `admin` |

---

# Critères d'évaluation DWWM

| Compétence | Ce qui est évalué dans cet atelier |
|---|---|
| CP1 – Maquetter une application | Structure des pages, cohérence UI, arborescence respectée |
| CP2 – Réaliser une IHM | Composants React, formulaires RHF, gestion des états |
| CP3 – Développer une interface utilisateur web | Tailwind, responsive, accessibilité de base (labels, aria) |
| CP5 – Créer une base de données | Schéma SQL, types, colonne `role`, requêtes paramétrées |
| CP6 – Développer les composants d'accès aux données | Modèles, architecture en couches, `async/await` |
| CP7 – Développer la partie back-end | Routes REST, middlewares, validation `express-validator` |
| CP8 – Élaborer et mettre en œuvre des composants dans une application | Auth JWT, rôles, gestion d'erreurs centralisée, envoi d'email |

---

# Checklist de rendu

Avant de soumettre, vérifier chaque point :

- [ ] 2 repos GitHub distincts (`portfolio-backend` et `portfolio-frontend`)
- [ ] `.env.example` présent dans chaque repo (`.env` dans `.gitignore`)
- [ ] `README.md` avec instructions d'installation dans chaque repo
- [ ] Architecture 4 couches respectée côté back (routes → controller → service → model)
- [ ] Validation `express-validator` sur les routes `POST /projects`, `PUT /projects/:id`, `POST /contact`
- [ ] Middleware `authorize('admin')` sur toutes les routes d'écriture
- [ ] React Hook Form utilisé sur : `LoginPage`, `AdminPage`, `ContactForm`
- [ ] `async/await` partout (aucun `.then()`)
- [ ] Page détail `/projects/:id` fonctionnelle
- [ ] Formulaire de contact envoie un email réel via Mailjet

---

# Aller plus loin (bonus)

- Pagination de la liste des projets (`?page=1&limit=6`)
- Ajout d'un champ `is_featured` (booléen) pour épingler des projets sur l'accueil
- Déploiement du back sur Railway ou Render, du front sur Vercel ou Netlify
- Protéger la route `DELETE /projects/:id` avec une confirmation côté serveur (vérifier que le projet appartient à l'admin connecté)
- Ajouter `helmet` pour les en-têtes de sécurité HTTP
