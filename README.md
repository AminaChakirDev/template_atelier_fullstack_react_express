# Atelier – Portfolio Full-Stack
## Express + MySQL + React + Tailwind · 2 jours

---

## À quoi sert ce projet ?

Cet atelier est l'occasion de développer **votre propre portfolio de développeur**.

Une fois déployé, il vous servira de carte de visite auprès des recruteurs : vos visiteurs pourront y découvrir vos projets, et vous aurez la main sur tout le contenu depuis un espace d'administration sécurisé.

**Ce que les visiteurs verront :**
- La liste de vos projets avec image de couverture, description et technologies utilisées
- Le détail de chaque projet : lien GitHub, démo vidéo, stack complète
- Un formulaire de contact pour vous écrire directement

**Ce que vous gérerez en tant qu'administrateur :**
- Ajouter, modifier ou supprimer vos projets
- Renseigner pour chaque projet : titre, description, stack technique, lien GitHub, lien démo, image de couverture
- Accéder à votre espace admin via une connexion sécurisée par JWT

C'est un projet que vous pourrez continuer à faire évoluer après la formation — et que vous aurez tout intérêt à montrer en entretien.

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
| Mail | Nodemailer + Gmail SMTP |
| Front-end | React 19 (Vite) · Tailwind CSS v4 |
| Validation front | React Hook Form |
| Routing front | React Router v6 |
| BDD | MySQL / MariaDB |

---

## Rendu attendu

- **2 dépôts GitHub distincts** : `portfolio-backend` et `portfolio-frontend`
- Un fichier `README.md` dans chaque repo expliquant comment installer et lancer le projet
- Variables d'environnement documentées dans `.env.example` (jamais le `.env` réel)
- Architecture en couches respectée côté back : routes → controller → service → model
- Validation obligatoire : `express-validator` côté back, React Hook Form côté front
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

### 1.2 Installation des dépendances

```bash
npm install express mysql2 bcrypt jsonwebtoken dotenv cors nodemailer express-validator
npm install --save-dev nodemon
```

Modifier `package.json` pour activer les ES Modules et ajouter les scripts :

```json
{
  "type": "module",
  "scripts": {
    "dev": "nodemon src/server.js",
    "start": "node src/server.js"
  }
}
```

### 1.3 Structure de dossiers à créer

L'architecture utilisée est en couches :

```
portfolio-backend/
├── src/
│   ├── config/
│   │   └── db.js                     ← Pool de connexion MySQL
│   ├── controllers/
│   │   ├── auth.controller.js        ← Reçoit req/res, délègue au service
│   │   ├── project.controller.js
│   │   └── contact.controller.js
│   ├── services/                     ← Logique métier
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
│   │   ├── auth.validator.js          ← Règles de validation du login
│   │   ├── project.validator.js       ← Règles de validation des projets
│   │   └── contact.validator.js       ← Règles de validation du contact
│   ├── routes/
│   │   ├── auth.routes.js
│   │   ├── project.routes.js
│   │   └── contact.routes.js
│   ├── errors/
│   │   └── AppError.js               ← Classe d'erreur personnalisée avec status HTTP
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
MAIL_USER=laplateforme.io.lyon@gmail.com
MAIL_PASS=xxxx xxxx xxxx xxxx
MAIL_TO=laplateforme.io.lyon@gmail.com
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

CREATE TABLE users (
  id         INT AUTO_INCREMENT PRIMARY KEY,
  email      VARCHAR(255) NOT NULL UNIQUE,
  password   VARCHAR(255) NOT NULL,
  role       VARCHAR(50)  NOT NULL,
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
VALUES ('admin@portfolio.fr', 'VOTRE_HASH_ICI', 'admin');
```

### 2.3 Configurer la connexion MySQL

Dans `src/config/db.js`, créer et exporter un **pool** de connexions en utilisant `mysql2/promise` et les variables d'environnement.
 
<details>
<summary>💡 Aide — configuration du pool MySQL</summary>
  
```js
import mysql from 'mysql2/promise';
 
const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
});
 
export default pool;
```
 
</details>

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
 
📖 [Voir le cours sur la gestion d'erreurs Express](https://drive.google.com/file/d/1ysyCJEZS3aKuQpJ-Yw4Fx36dcSAiVsmW/view?usp=sharing)

### 3.3 Classe `AppError`
 
Créer `src/errors/AppError.js`. Cette classe étend `Error` et ajoute un code HTTP `status`, ce qui permet aux services de lancer des erreurs précises sans connaître `req`/`res`.
 
```js
class AppError extends Error {
  constructor(message, status) {
    super(message);
    this.status = status;
  }
}
 
export default AppError;
```
 
**Utilisation dans un service :**
```js
import AppError from '../errors/AppError.js';
 
// Exemple : ressource introuvable
throw new AppError('Projet introuvable', 404);
 
// Exemple : accès refusé
throw new AppError('Identifiants invalides', 401);
```
 
`errorHandler` récupère automatiquement `err.status` et `err.message` pour construire la réponse JSON. Si `status` est absent (erreur inattendue), il retourne `500`.

### 3.4 Middleware `validate`

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

### 4.2 Service auth

Dans `src/services/auth.service.js`, écrire la fonction `loginUser({ email, password })` qui :

1. Appelle `userModel.findByEmail(email)`
2. Renvoie une erreur `401` si l'utilisateur n'existe pas
3. Compare le mot de passe avec `bcrypt.compare`
4. Renvoie une erreur `401` si le mot de passe est incorrect
5. Génère un JWT signé avec `{ id, email, role }` dans le payload (durée : `24h`)
6. Retourne le token

> Le service ne connaît pas `req` ni `res`. Il travaille avec des données brutes et lance des erreurs si nécessaire.

<details>
<summary>💡 Aide — service loginUser</summary>
  
```js
export const loginUser = async (email, password) => {
  const user = await UserModel.findByEmail(email);
  if (!user) {
    throw new AppError("Email ou mot de passe incorrect", 401 );
  }

  const valid = await bcrypt.compare(password, user.password);
  if (!valid) {
    throw new AppError("Email ou mot de passe incorrect", 401);
  }

  const token = jwt.sign(
    { id: user.id, email: user.email, role: user.role },
    process.env.JWT_SECRET,
    { expiresIn: "24h" },
  );

  return token;
};
```
 
</details>

📖 [Voir le cours sur la génération de JWT](https://drive.google.com/file/d/1xXKUI71qdmV8cHF-faq4zOXQ3b_ViYwG/view?usp=drive_link)

### 4.3 Contrôleur auth

Dans `src/controllers/auth.controller.js`, écrire `login(req, res, next)` qui :
 
1. Extrait `email` et `password` de `req.body`
2. Appelle `authService.loginUser(...)`
3. Renvoie `res.json({ token })` en cas de succès

> 💡 Pas de `try/catch` — Express 5 propage automatiquement les erreurs async vers `errorHandler`.

### 4.4 Middleware `authenticate`

Dans `src/middlewares/auth.middleware.js`, créer le middleware `authenticate` qui :

1. Lit le header `Authorization`
2. Vérifie qu'il commence par `'Bearer '`
3. Extrait et vérifie le token avec `jwt.verify`
4. Stocke le payload décodé dans `req.user`
5. Appelle `next()` ou renvoie une erreur `401`

<details>
<summary>💡 Aide — middleware authenticate</summary>
  
```js
export const authenticate = (req, res, next) => {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return next(new AppError('Token manquant', 401));
  }

  const token = authHeader.split(' ')[1];

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    req.user = payload;
    next();
  } catch {
    next(new AppError('Token invalide', 401));
  }
};
```
 
</details>

### 4.5 Middleware `authorize`

Dans `src/middlewares/authorize.middleware.js`, créer une fonction `authorize(role)` qui retourne un middleware vérifiant que `req.user.role` fait partie des rôles autorisés.

```js
// Exemple d'usage attendu dans une route :
router.post('/', authenticate, authorize('admin'), createProject);
```

> `authorize` est une **factory** : elle prend un rôle en paramètre et retourne un middleware. Si le rôle de l'utilisateur ne correspond pas, répondre `403 Forbidden`.

<details>
<summary>💡 Aide — middleware authorize</summary>
  
```js
import AppError from "./AppError.js";

export const authorize = (allowedRole) => (req, res, next) => {
  if (!allowedRole === req.user.role) {
    return next(new AppError(403, "Action non autorisée"));
  }
  next();
};
```
 
</details>

### 4.6 Validator auth

Dans `src/validators/auth.validator.js`, exporter `validateAuth` couvrant :

| Champ | Règles |
|---|---|
| `email` | Obligatoire · format email valide (`isEmail`) |
| `password` | Obligatoire |

<details>
  
<summary>💡 Aide — syntaxe express-validator</summary>

```js
import { body } from 'express-validator';
 
export const validateAuth = [
  body('email').notEmpty().withMessage('Le champ email est requis').isEmail().withMessage('Email invalide'),
  body('password').notEmpty().withMessage('Mot de passe requis'),
];

```
 
</details>


### 4.7 Routes auth

Dans `src/routes/auth.routes.js`, déclarer `POST /login` avec les middlewares `validateAuth` et `validate` · puis le contrôleur, et brancher le fichier dans `server.js`.

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
- Email invalide (pas un format email) → `400`

---

## Étape 5 · Validation back avec express-validator _(~20 min)_

Avant de développer le CRUD projets, mettre en place la validation.

### 5.1 Règles de validation des projets et de l'id
 
Dans `src/validators/project.validator.js`, exporter deux tableaux de règles :

**`validateId`** — valide le paramètre `:id` des routes :
 
| Paramètre | Règles |
|---|---|
| `id` | Entier · valeur minimale : 1 |

**`validateProject`** — valide le corps de la requête :

| Champ | Règles |
|---|---|
| `title` | Obligatoire · chaîne · min 2 caractères · max 150 caractères |
| `description` | Optionnel · si présent, chaîne · max 2000 caractères |
| `tech_stack` | Optionnel · si présent, chaîne · max 255 caractères |
| `github_url` | Optionnel · si présent, doit être une URL valide (`isURL`) |
| `demo_url` | Optionnel · si présent, doit être une URL valide (`isURL`) |
| `image_url` | Optionnel · si présent, doit être une URL valide (`isURL`) |

📖 [Voir le cours sur express-validator](https://drive.google.com/file/d/1kagNSyo4q9WzSTZ1jbatxyiZuJL0pf0F/view?usp=drive_link)

### 5.2 Règles de validation du contact

Dans `src/validators/contact.validator.js`, exporter `validateContact` couvrant :

| Champ | Règles |
|---|---|
| `name` | Obligatoire · chaîne · min 2 · max 100 caractères |
| `email` | Obligatoire · format email valide (`isEmail`) |
| `message` | Obligatoire · chaîne · min 10 · max 2000 caractères |

---

## Étape 6 · CRUD Projets _(~60 min)_
 
> 🔁 **Méthode de travail** : Pour chaque feature, implémenter les 3 couches (model → service → controller) puis déclarer la route et tester immédiatement avant de passer à la suivante. N'hésite pas à revoir ce qui a été fait dans l'[exercice de debug BeatList](https://github.com/AminaChakirDev/cdpi-2025-debug-express-js-beatlist/tree/correction)
 
---
 
### Feature 6.1 — Récupérer tous les projets
 
**Model**
— dans `src/models/project.model.js`, écrire `findAll()` :
- Requête : `SELECT * FROM projects ORDER BY created_at DESC`
- Retourne le tableau de résultats (vide si aucun projet)
  
**Service**
— dans `src/services/project.service.js`, écrire `getAllProjects()` :
- Appelle `model.findAll()` et retourne le résultat
  
**Controller**
— dans `src/controllers/project.controller.js`, écrire `getAllProjects(req, res)` :
- Appelle le service et renvoie `res.json(projects)`
- Rappel : pas de `try/catch` (Express 5 propage automatiquement les erreurs async)
  
**Route** — dans `src/routes/project.routes.js`, déclarer `GET /` sans middleware d'auth, et brancher le fichier dans `server.js`.
 
**✅ Test**
```
GET /api/projects → 200 []
```
 
---
 
### Feature 6.2 — Récupérer un projet par son id
 
**Model**
— écrire `findById(id)` :
- Requête paramétrée : `SELECT * FROM projects WHERE id = ?`
- Retourne l'objet projet ou `null`
  
**Service**
— écrire `getProjectById(id)` :
- Appelle `model.findById(id)`
- Lance une `AppError('Projet introuvable', 404)` si le résultat est `null`
  
**Controller**
— écrire `getProjectById(req, res)` :
- Extrait `req.params.id`
- Appelle le service et renvoie `res.json(project)`
  
**Route**
— déclarer `GET /:id` avec les middlewares : `validateId` · `validate`
 
**✅ Tests**
```
GET /api/projects/1   → 404 (pas encore de projet en base)
GET /api/projects/abc → 400 (id invalide)
GET /api/projects/0   → 400 (id invalide)
```
 
---
 
### Feature 6.3 — Créer un projet
 
**Model**
— écrire `create(data)` :
- Requête `INSERT INTO projects (title, description, tech_stack, github_url, demo_url, image_url) VALUES (?, ?, ?, ?, ?, ?)`
- Appelle `findById(result.insertId)` pour retourner le projet complet
  
**Service**
— écrire `createProject(data)` :
- Appelle `model.create(data)` et retourne le projet créé
  
**Controller**
— écrire `createProject(req, res)` :
- Extrait les champs de `req.body`
- Appelle le service et renvoie `res.status(201).json(project)`
  
**Route**
— déclarer `POST /` avec les middlewares : `authenticate` · `authorize('admin')` · `validateProject` · `validate`
 
**✅ Tests**
```
POST /api/projects (sans token)                        → 401
POST /api/projects (token admin, title manquant)       → 400 + erreurs de validation
POST /api/projects (token admin, github_url invalide)  → 400 + erreur sur le champ url
POST /api/projects (token admin, données valides)      → 201 + projet créé avec son id
```
 
---
 
### Feature 6.4 — Modifier un projet
 
**Model**
— écrire `update(id, data)` :
- Requête `UPDATE projects SET title=?, description=?, tech_stack=?, github_url=?, demo_url=?, image_url=? WHERE id=?`
- Appelle `findById(id)` pour retourner le projet mis à jour
  
**Service**
— écrire `updateProject(id, data)` :
- Vérifie que le projet existe avec `findById(id)` → `AppError(404)` si absent
- Appelle `model.update(id, data)` et retourne le résultat
  
**Controller**
— écrire `updateProject(req, res)` :
- Extrait `req.params.id` et `req.body`
- Appelle le service et renvoie `res.json(project)`
  
**Route**
— déclarer `PUT /:id` avec les middlewares : `authenticate` · `authorize('admin')` · `validateId` · `validateProject` · `validate`
 
**✅ Tests**
```
PUT /api/projects/1 (token admin, données valides) → 200 + projet modifié
PUT /api/projects/999 (token admin)                → 404
PUT /api/projects/abc (token admin)                → 400 (id invalide)
PUT /api/projects/1 (sans token)                   → 401
```
 
---
 
### Feature 6.5 — Supprimer un projet
 
**Model**
— écrire `remove(id)` :
- Requête `DELETE FROM projects WHERE id = ?`
- Retourne `result.affectedRows > 0` (booléen)
  
**Service**
— écrire `deleteProject(id)` :
- Appelle `model.remove(id)`
- Lance une `AppError('Projet introuvable', 404)` si le résultat est `false`
  
**Controller**
— écrire `deleteProject(req, res)` :
- Extrait `req.params.id`
- Appelle le service et renvoie `res.status(204).send()`
  
**Route**
— déclarer `DELETE /:id` avec les middlewares : `authenticate` · `authorize('admin')` · `validateId` · `validate`
 
**✅ Tests**
```
DELETE /api/projects/1 (token admin)   → 204
DELETE /api/projects/1 (token admin)   → 404 (déjà supprimé)
DELETE /api/projects/abc (token admin) → 400 (id invalide)
DELETE /api/projects/1 (sans token)    → 401
```
 
---

## Étape 7 · Formulaire de contact _(~30 min)_

### 7.1 Service contact
 
Dans `src/services/contact.service.js`, écrire `sendContactEmail({ name, email, message })` qui :
- Crée un transporteur Nodemailer configuré avec Gmail SMTP
- Envoie un email formaté à l'adresse `MAIL_TO`
- Lance une erreur en cas d'échec d'envoi
  
<details>
  
<summary>💡 Aide — configuration du transporteur Nodemailer Gmail</summary>

```js
import nodemailer from 'nodemailer';
 
const transporter = nodemailer.createTransport({
  host: 'smtp.gmail.com',
  port: 587,
  auth: {
    user: process.env.MAIL_USER,
    pass: process.env.MAIL_PASS, // mot de passe d'application Google
  },
});
```
 
</details>

> ⚠️ `MAIL_PASS` est un **mot de passe d'application** généré depuis les paramètres de sécurité Google, pas le mot de passe du compte.

### 7.2 Contrôleur et route contact

Dans `src/controllers/contact.controller.js`, écrire `sendContact(req, res, next)` qui :
- Extrait `name`, `email` et `message` de `req.body`
- Appelle `contactService.sendContactEmail(...)`
- Renvoie `res.json({ message: 'Message envoyé avec succès' })`

### 7.3 Route contact
 
Dans `src/routes/contact.routes.js`, déclarer `POST /` avec les middlewares `validateContact` · `validate` · puis le contrôleur, et brancher le fichier dans `server.js`.

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
npm install tailwindcss @tailwindcss/vite react-router-dom react-hook-form jwt-decode
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
│   ├── hooks/
│   │   └── apiFetch.js              ← Fonction centralisée pour tous les appels API
│   ├── components/
│   │   ├── Navbar.jsx
│   │   ├── ProjectCard.jsx
│   │   └── ContactForm.jsx          ← Utilise React Hook Form
│   ├── context/
│   │   ├── AuthContext.js           ← Création et export du contexte
│   │   └── AuthProvider.jsx         ← Provider
│   ├── pages/
│   │   ├── HomePage.jsx
│   │   ├── ProjectsPage.jsx
│   │   ├── ProjectDetailPage.jsx    ← Page détail d'un projet
│   │   ├── LoginPage.jsx            ← Utilise React Hook Form
│   │   └── admin/
│   │       ├── AdminPage.jsx            ← Dashboard : liste + suppression
│   │       ├── CreateProjectPage.jsx    ← Formulaire de création
│   │       └── EditProjectPage.jsx      ← Formulaire d'édition pré-rempli
│   ├── utils/
│   │   └── jwt.utils.js             ← Fonction isTokenValid
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

📖 Lien vers la [documentation Vite sur les variables d'environnements](https://vite.dev/guide/env-and-mode#env-variables)

---

## Étape 9 · Utilitaire API _(~15 min)_

### 9.1 Fonction `apiFetch`

Dans `src/hooks/apiFetch.js`, écrire une fonction `apiFetch(endpoint, options)` qui :

1. Lit `import.meta.env.VITE_API_URL` pour construire l'URL complète
2. Récupère le token depuis le `localStorage`
3. Construit les headers : `Content-Type: application/json` + `Authorization: Bearer <token>` si token présent
4. Appelle `fetch` avec `async/await`
5. Si la réponse n'est pas `ok`, extrait le JSON de l'erreur et lance une `Error`
6. Si le statut est `204` (No Content), retourne `null`
7. Sinon, retourne `response.json()`

> 📖 Cette fonction est similaire à celle que vous avez développée dans le cours myblog. Retrouvez-la et adaptez-la à cette architecture.

📖 Voir le cours sur [apiFetch](https://drive.google.com/file/d/15NSDTT_1s7tLkWhCYKbsCLjvdiKhMvbJ/view?usp=sharing) et [Gestion des erreurs](https://drive.google.com/file/d/1vigS_HOuO2F51GAVIyz9lbNQ4Vwk3j7e/view?usp=drive_link)

---

## Étape 10 · Contexte d'authentification _(~20 min)_

### 10.1 `jwt.utils.js`
 
Dans `src/utils/jwt.utils.js`, créer et exporter la fonction `isTokenValid(token)` qui vérifie qu'un token JWT est présent et non expiré :
 
```js
import { jwtDecode } from 'jwt-decode';
 
export function isTokenValid(token) {
  try {
    const decodedToken = jwtDecode(token);
    if (!decodedToken.exp) return false;
    return decodedToken.exp * 1000 > Date.now();
  } catch {
    return false;
  }
}
```
 
> 💡 `jwtDecode` ne vérifie pas la signature du token — il se contente de le décoder. C'est suffisant côté front pour savoir si le token est expiré avant d'envoyer une requête.

### 10.2 `AuthContext.js`
 
Dans `src/context/AuthContext.js`, créer et exporter uniquement le contexte React :
 
```js
import { createContext } from "react";

export const AuthContext = createContext();
```
 
### 10.3 `AuthProvider.jsx`
 
Dans `src/context/AuthProvider.jsx`, créer et exporter :
 
- `AuthProvider` : le composant provider qui expose via la valeur du contexte :
  - `isAuthenticated` (booléen)
  - `login(token)` : stocke le token dans `localStorage` et met à jour le state
  - `logout()` : supprime le token de `localStorage` et remet le state à `null`

> 💡 Au montage, si un token expiré est présent dans le `localStorage`, il est supprimé immédiatement, l'utilisateur ne reste pas bloqué dans un état "connecté" avec un token invalide.
  
📖 [Voir les cours sur AuthContext et AuthProvider](https://drive.google.com/file/d/1509-pSq2q3uMV1ofKsgMwymuSphe2ofB/view?usp=drive_link)

<details>
  
<summary>💡 Aide — structure de AuthProvider</summary>
  
```jsx
import { useState } from "react";
import { AuthContext } from "./AuthContext";
import { isTokenValid } from "../utils/jwt.utils";

export function AuthProvider({ children }) {
  const storedToken = localStorage.getItem("token");
  
  if (storedToken && !isTokenValid(storedToken)) {
    localStorage.removeItem("token");
  }

  const [isAuthenticated, setIsAuthenticated] = useState(
    !!storedToken && isTokenValid(storedToken),
  );

  function login(token) {
    localStorage.setItem("token", token);
    setIsAuthenticated(true);
  }

  function logout() {
    localStorage.removeItem("token");
    setIsAuthenticated(false);
  }

  return (
    <AuthContext.Provider
      value={{
        isAuthenticated,
        login,
        logout,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
}
```
 
</details>

### 10.4 Brancher le provider dans `main.jsx`

```jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import { AuthProvider } from './context/AuthProvider.jsx';
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
- Lit `isAuthenticated` depuis `AuthContext`
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
| `/admin/projects/new` | `CreateProjectPage` | `PrivateRoute` |
| `/admin/projects/:id/edit` | `EditProjectPage` | `PrivateRoute` |
 
📖 [Voir le cours sur React Router et PrivateRoute](https://drive.google.com/file/d/15NSDTT_1s7tLkWhCYKbsCLjvdiKhMvbJ/view?usp=drive_link)

### 11.2 `Navbar`

Le composant `Navbar` doit :
- Afficher les liens de navigation (Accueil, Projets)
- Utiliser `AuthContext` pour afficher "Connexion" ou "Déconnexion" selon l'état
- Appeler `logout()` puis rediriger vers `/` au clic sur "Déconnexion"

---

## Étape 12 · Page de connexion _(~20 min)_

### Consignes

Dans `src/pages/LoginPage.jsx`, utiliser **React Hook Form** pour gérer le formulaire de connexion.

Règles de validation RHF à appliquer :

| Champ | Règles |
|---|---|
| `email` | Requis · format email valide |
| `password` | Requis |

**Comportement attendu :**
- `handleSubmit` de RHF appelle `apiFetch('/auth/login', { method: 'POST', body: ... })`
- En cas de succès : appeler `login(data.token)` puis rediriger vers `/admin`
- En cas d'erreur : afficher le message d'erreur retourné par l'API
- Les messages d'erreur RHF s'affichent sous chaque champ en temps réel

📖 [Voir le cours sur React Hook Form](https://drive.google.com/file/d/1MbF4b6s79RSiGETwPCNuCMZ9xqB2UZVJ/view?usp=drive_link)
 
<details>
  
<summary>💡 Aide — structure de base avec React Hook Form</summary>
  
```jsx
const { register, handleSubmit, formState: { errors } } = useForm();
 
const onSubmit = async (data) => {
  // appel apiFetch ici
};
 
return (
  <form onSubmit={handleSubmit(onSubmit)}>
    <input {...register('email', { required: true })} />
    {errors.email && <p>Email requis</p>}
  </form>
);
```
 
</details>

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

> 💡 Pour tronquer la description, utiliser la classe Tailwind `line-clamp-3`.

### 13.2 `ProjectsPage`

Dans `src/pages/ProjectsPage.jsx` :
- Charger les projets au montage avec `useEffect` + `apiFetch('/projects')` en `async/await`
- Gérer les états `loading`, `error`, `projects`
- Afficher la grille de `ProjectCard`

> ⚠️ Penser à gérer les 3 états : afficher un message de chargement, un message d'erreur, et la liste quand les données sont disponibles.
 
<details>
  
<summary>💡 Aide — structure useEffect + fetch</summary>

```jsx
const [projects, setProjects] = useState([]);
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);
 
useEffect(() => {
  const fetchProjects = async () => {
    try {
      const data = await apiFetch('/projects');
      setProjects(data);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };
 
  fetchProjects();
}, []);
```
 
</details>

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

### 15.1 `AdminPage` — Dashboard
 
Dans `src/pages/admin/AdminPage.jsx` :
- Charger tous les projets au montage avec `apiFetch('/projects')`
- Afficher la liste avec pour chaque projet un bouton **Modifier** et un bouton **Supprimer**
- Le bouton **Modifier** redirige vers `/admin/projects/:id/edit`
- Le bouton **Supprimer** demande une confirmation (`window.confirm`) puis appelle `DELETE /api/projects/:id` et recharge la liste
> 💡 Après une suppression, il est possible soit de rappeler `apiFetch('/projects')`, soit de filtrer le state local avec `.filter()` pour éviter un rechargement réseau.
 
### 15.2 `CreateProjectPage` — Formulaire de création
 
Dans `src/pages/admin/CreateProjectPage.jsx`, utiliser **React Hook Form**.
 
**Règles de validation RHF :**
 
| Champ | Règles |
|---|---|
| `title` | Requis · min 2 · max 150 caractères |
| `description` | Optionnel · max 2000 caractères |
| `tech_stack` | Optionnel · max 255 caractères |
| `github_url` | Optionnel · si renseigné, doit commencer par `https://` |
| `demo_url` | Optionnel · si renseigné, doit commencer par `https://` |
| `image_url` | Optionnel · si renseigné, doit commencer par `https://` |
 
**Comportement attendu :**
- `handleSubmit` appelle `POST /api/projects` via `apiFetch`
- En cas de succès : rediriger vers `/admin`
- En cas d'erreur : afficher le message retourné par l'API
 
### 15.3 `EditProjectPage` — Formulaire d'édition
 
Dans `src/pages/admin/EditProjectPage.jsx` :
- Récupérer l'`id` depuis `useParams()`
- Charger le projet via `apiFetch('/projects/' + id)` au montage
- Pré-remplir le formulaire avec `reset(project)` de React Hook Form
- Mêmes règles de validation que la création
- `handleSubmit` appelle `PUT /api/projects/:id` via `apiFetch`
- En cas de succès : rediriger vers `/admin`
> 💡 `useForm` expose la méthode `reset(values)` pour pré-remplir le formulaire avec les données d'un projet existant. L'appeler dans le `useEffect` après avoir chargé le projet.
 
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
| Dashboard admin | `/admin` | Privé — role `admin` |
| Créer un projet | `/admin/projects/new` | Privé — role `admin` |
| Modifier un projet | `/admin/projects/:id/edit` | Privé — role `admin` |

---

# Checklist de rendu

Avant de soumettre, vérifier chaque point :

- [ ] 2 repos GitHub distincts (`portfolio-backend` et `portfolio-frontend`)
- [ ] `.env.example` présent dans chaque repo (`.env` dans `.gitignore`)
- [ ] `README.md` avec instructions d'installation dans chaque repo
- [ ] Architecture 4 couches respectée côté back (routes → controller → service → model)
- [ ] Validation `express-validator` sur les routes `POST /projects`, `PUT /projects/:id`, `POST /contact`
- [ ] Middleware `authorize('admin')` sur toutes les routes d'écriture
- [ ] React Hook Form utilisé sur : `LoginPage`, `CreateProjectPage`, `EditProjectPage`, `ContactForm`
- [ ] Utilisation de `async/await`
- [ ] Pages détail `/projects/:id` et édition `/admin/projects/:id/edit` fonctionnelles
- [ ] Formulaire de contact envoie un email réel via Gmail SMTP
