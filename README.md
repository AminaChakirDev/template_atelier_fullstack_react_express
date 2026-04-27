# Atelier – Portfolio Full-Stack
## Express + MySQL + React + Tailwind · 2 jours

---

> **Objectif pédagogique**
> Concevoir et développer une application web full-stack de type portfolio personnel.
> L'admin peut se connecter, gérer ses projets (CRUD), et les visiteurs peuvent envoyer un message de contact.
> Cet atelier mobilise l'ensemble de la stack DWWM : MVC Express, JWT, MySQL, React, Tailwind, formulaires, fetch authentifié.

---

## Stack technique

| Côté | Technologie |
|---|---|
| Back-end | Node.js · Express 5 (ES Modules) · MySQL2 |
| Auth | JWT + bcrypt |
| Mail | Mailjet (API REST) |
| Front-end | React 19 (Vite) · Tailwind CSS v4 |
| BDD | MySQL / MariaDB |

---

## Rendu attendu

- Dépôt GitHub avec deux dossiers : `backend/` et `frontend/`
- Un fichier `README.md` à la racine expliquant comment lancer le projet
- Variables d'environnement dans `.env.example` (jamais le `.env` réel)
- Code propre, MVC respecté côté back, composants organisés côté front

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
npm install express mysql2 bcrypt jsonwebtoken dotenv cors nodemailer
npm install -D @eslint/js
```

### 1.3 Structure de dossiers à créer

```
portfolio-backend/
├── src/
│   ├── config/
│   │   └── db.js
│   ├── controllers/
│   │   ├── auth.controller.js
│   │   ├── project.controller.js
│   │   └── contact.controller.js
│   ├── middlewares/
│   │   ├── auth.middleware.js
│   │   └── errorHandler.js
│   ├── models/
│   │   ├── user.model.js
│   │   └── project.model.js
│   ├── routes/
│   │   ├── auth.routes.js
│   │   ├── project.routes.js
│   │   └── contact.routes.js
│   └── server.js
├── .env
├── .env.example
└── package.json
```

> 💡 Créer tous les dossiers et fichiers vides dès maintenant. Cela donne une vision d'ensemble avant de coder.

### 1.4 Fichier `.env`

```env
PORT=3001
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=
DB_NAME=portfolio_db
JWT_SECRET=un_secret_tres_long_et_aleatoire
MJ_APIKEY_PUBLIC=ta_cle_publique_mailjet
MJ_APIKEY_PRIVATE=ta_cle_privee_mailjet
MAIL_FROM=tonemail@domaine.com
MAIL_TO=destinataire@domaine.com
```

Créer aussi `.env.example` avec les mêmes clés mais sans les valeurs.

---

## Étape 2 · Base de données _(~30 min)_

### 2.1 Créer la base et les tables (dans phpMyAdmin ou le terminal MySQL)

```sql
CREATE DATABASE IF NOT EXISTS portfolio_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

USE portfolio_db;

CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  password VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE projects (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  description TEXT,
  tech_stack VARCHAR(255),
  github_url VARCHAR(500),
  demo_url VARCHAR(500),
  image_url VARCHAR(500),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### 2.2 Insérer l'utilisateur admin (mot de passe hashé)

Pour générer un hash bcrypt en Node :

```bash
node -e "import('bcrypt').then(b => b.default.hash('monMotDePasse', 10).then(console.log))"
```

Puis insérer en SQL :

```sql
INSERT INTO users (email, password) VALUES ('admin@portfolio.fr', '$2b$10$...');
```

> ⚠️ Il n'y aura **pas** de route `/register` publique. L'admin est créé une seule fois en base directement.

### 2.3 Configurer la connexion MySQL

**`src/config/db.js`**

```js
import mysql from 'mysql2/promise';
import 'dotenv/config';

const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
});

export default pool;
```

---

## Étape 3 · Serveur Express et middlewares globaux _(~20 min)_

**`src/server.js`**

```js
import 'dotenv/config';
import express from 'express';
import cors from 'cors';

import authRoutes from './routes/auth.routes.js';
import projectRoutes from './routes/project.routes.js';
import contactRoutes from './routes/contact.routes.js';
import errorHandler from './middlewares/errorHandler.js';

const app = express();
const PORT = process.env.PORT || 3001;

// Middlewares globaux
app.use(cors({ origin: 'http://localhost:5173' }));
app.use(express.json());

// Routes
app.use('/api/auth', authRoutes);
app.use('/api/projects', projectRoutes);
app.use('/api/contact', contactRoutes);

// Gestionnaire d'erreurs (toujours en dernier)
app.use(errorHandler);

app.listen(PORT, () => {
  console.log(`Serveur démarré sur http://localhost:${PORT}`);
});
```

**`src/middlewares/errorHandler.js`**

```js
const errorHandler = (err, req, res, next) => {
  const status = err.status || 500;
  const message = err.message || 'Erreur interne du serveur';
  res.status(status).json({ error: message });
};

export default errorHandler;
```

---

## Étape 4 · Authentification JWT _(~45 min)_

### 4.1 Modèle utilisateur

**`src/models/user.model.js`**

```js
import pool from '../config/db.js';

const findByEmail = async (email) => {
  const [rows] = await pool.query('SELECT * FROM users WHERE email = ?', [email]);
  return rows[0] || null;
};

export default { findByEmail };
```

### 4.2 Contrôleur auth

**`src/controllers/auth.controller.js`**

```js
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';
import userModel from '../models/user.model.js';

export const login = async (req, res, next) => {
  try {
    const { email, password } = req.body;

    if (!email || !password) {
      const err = new Error('Email et mot de passe requis');
      err.status = 400;
      return next(err);
    }

    const user = await userModel.findByEmail(email);
    if (!user) {
      const err = new Error('Identifiants invalides');
      err.status = 401;
      return next(err);
    }

    const isValid = await bcrypt.compare(password, user.password);
    if (!isValid) {
      const err = new Error('Identifiants invalides');
      err.status = 401;
      return next(err);
    }

    const token = jwt.sign(
      { id: user.id, email: user.email },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );

    res.json({ token });
  } catch (err) {
    next(err);
  }
};
```

### 4.3 Middleware d'authentification

**`src/middlewares/auth.middleware.js`**

```js
import jwt from 'jsonwebtoken';

const authenticate = (req, res, next) => {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    const err = new Error('Token manquant');
    err.status = 401;
    return next(err);
  }

  const token = authHeader.split(' ')[1];

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch {
    const err = new Error('Token invalide ou expiré');
    err.status = 401;
    next(err);
  }
};

export default authenticate;
```

### 4.4 Routes auth

**`src/routes/auth.routes.js`**

```js
import { Router } from 'express';
import { login } from '../controllers/auth.controller.js';

const router = Router();

router.post('/login', login);

export default router;
```

### ✅ Test Étape 4

Tester dans Thunder Client ou Postman :

```
POST http://localhost:3001/api/auth/login
Body JSON : { "email": "admin@portfolio.fr", "password": "monMotDePasse" }
```

Résultat attendu : `{ "token": "eyJ..." }`

---

## Étape 5 · CRUD Projets _(~60 min)_

### 5.1 Modèle projet

**`src/models/project.model.js`**

```js
import pool from '../config/db.js';

const findAll = async () => {
  const [rows] = await pool.query('SELECT * FROM projects ORDER BY created_at DESC');
  return rows;
};

const findById = async (id) => {
  const [rows] = await pool.query('SELECT * FROM projects WHERE id = ?', [id]);
  return rows[0] || null;
};

const create = async ({ title, description, tech_stack, github_url, demo_url, image_url }) => {
  const [result] = await pool.query(
    'INSERT INTO projects (title, description, tech_stack, github_url, demo_url, image_url) VALUES (?, ?, ?, ?, ?, ?)',
    [title, description, tech_stack, github_url, demo_url, image_url]
  );
  return findById(result.insertId);
};

const update = async (id, { title, description, tech_stack, github_url, demo_url, image_url }) => {
  await pool.query(
    'UPDATE projects SET title=?, description=?, tech_stack=?, github_url=?, demo_url=?, image_url=? WHERE id=?',
    [title, description, tech_stack, github_url, demo_url, image_url, id]
  );
  return findById(id);
};

const remove = async (id) => {
  const [result] = await pool.query('DELETE FROM projects WHERE id = ?', [id]);
  return result.affectedRows > 0;
};

export default { findAll, findById, create, update, remove };
```

### 5.2 Contrôleur projet

**`src/controllers/project.controller.js`**

```js
import projectModel from '../models/project.model.js';

export const getAllProjects = async (req, res, next) => {
  try {
    const projects = await projectModel.findAll();
    res.json(projects);
  } catch (err) {
    next(err);
  }
};

export const getProjectById = async (req, res, next) => {
  try {
    const project = await projectModel.findById(req.params.id);
    if (!project) {
      const err = new Error('Projet introuvable');
      err.status = 404;
      return next(err);
    }
    res.json(project);
  } catch (err) {
    next(err);
  }
};

export const createProject = async (req, res, next) => {
  try {
    const { title } = req.body;
    if (!title) {
      const err = new Error('Le titre est obligatoire');
      err.status = 400;
      return next(err);
    }
    const project = await projectModel.create(req.body);
    res.status(201).json(project);
  } catch (err) {
    next(err);
  }
};

export const updateProject = async (req, res, next) => {
  try {
    const project = await projectModel.findById(req.params.id);
    if (!project) {
      const err = new Error('Projet introuvable');
      err.status = 404;
      return next(err);
    }
    const updated = await projectModel.update(req.params.id, req.body);
    res.json(updated);
  } catch (err) {
    next(err);
  }
};

export const deleteProject = async (req, res, next) => {
  try {
    const deleted = await projectModel.remove(req.params.id);
    if (!deleted) {
      const err = new Error('Projet introuvable');
      err.status = 404;
      return next(err);
    }
    res.status(204).send();
  } catch (err) {
    next(err);
  }
};
```

### 5.3 Routes projet

**`src/routes/project.routes.js`**

```js
import { Router } from 'express';
import {
  getAllProjects,
  getProjectById,
  createProject,
  updateProject,
  deleteProject,
} from '../controllers/project.controller.js';
import authenticate from '../middlewares/auth.middleware.js';

const router = Router();

// Routes publiques
router.get('/', getAllProjects);
router.get('/:id', getProjectById);

// Routes protégées (admin uniquement)
router.post('/', authenticate, createProject);
router.put('/:id', authenticate, updateProject);
router.delete('/:id', authenticate, deleteProject);

export default router;
```

### ✅ Tests Étape 5

```
GET    http://localhost:3001/api/projects          → liste vide []
POST   http://localhost:3001/api/projects          → 401 sans token
POST   (avec header Authorization: Bearer <token>) → 201 + projet créé
GET    http://localhost:3001/api/projects/1        → le projet
PUT    http://localhost:3001/api/projects/1        → projet modifié
DELETE http://localhost:3001/api/projects/1        → 204
```

---

## Étape 6 · Formulaire de contact (Mailjet) _(~30 min)_

### 6.1 Contrôleur contact

**`src/controllers/contact.controller.js`**

```js
import nodemailer from 'nodemailer';

export const sendContact = async (req, res, next) => {
  try {
    const { name, email, message } = req.body;

    if (!name || !email || !message) {
      const err = new Error('Tous les champs sont obligatoires');
      err.status = 400;
      return next(err);
    }

    const transporter = nodemailer.createTransport({
      host: 'in-v3.mailjet.com',
      port: 587,
      auth: {
        user: process.env.MJ_APIKEY_PUBLIC,
        pass: process.env.MJ_APIKEY_PRIVATE,
      },
    });

    await transporter.sendMail({
      from: process.env.MAIL_FROM,
      to: process.env.MAIL_TO,
      subject: `[Portfolio] Message de ${name}`,
      text: `De : ${name} <${email}>\n\n${message}`,
      html: `<p><strong>De :</strong> ${name} &lt;${email}&gt;</p><p>${message}</p>`,
    });

    res.json({ message: 'Message envoyé avec succès' });
  } catch (err) {
    next(err);
  }
};
```

### 6.2 Routes contact

**`src/routes/contact.routes.js`**

```js
import { Router } from 'express';
import { sendContact } from '../controllers/contact.controller.js';

const router = Router();

router.post('/', sendContact);

export default router;
```

### ✅ Test Étape 6

```
POST http://localhost:3001/api/contact
Body : { "name": "Alice", "email": "alice@test.fr", "message": "Bonjour !" }
```

---

## ✅ Récap fin Jour 1

À ce stade, votre API doit exposer :

| Méthode | Route | Auth | Description |
|---|---|---|---|
| POST | `/api/auth/login` | ✗ | Connexion admin |
| GET | `/api/projects` | ✗ | Liste des projets |
| GET | `/api/projects/:id` | ✗ | Un projet |
| POST | `/api/projects` | ✓ | Créer un projet |
| PUT | `/api/projects/:id` | ✓ | Modifier un projet |
| DELETE | `/api/projects/:id` | ✓ | Supprimer un projet |
| POST | `/api/contact` | ✗ | Envoyer un message |

---

# JOUR 2 — Front-end React + Tailwind

---

## Étape 7 · Mise en place du projet front-end _(~20 min)_

### 7.1 Initialisation

```bash
npm create vite@latest portfolio-frontend -- --template react
cd portfolio-frontend
npm install
npm install tailwindcss @tailwindcss/vite react-router-dom
```

### 7.2 Configurer Tailwind dans Vite

**`vite.config.js`**

```js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  plugins: [react(), tailwindcss()],
});
```

**`src/index.css`** (remplacer tout le contenu)

```css
@import "tailwindcss";
```

### 7.3 Structure de dossiers

```
portfolio-frontend/
├── src/
│   ├── api/
│   │   └── apiFetch.js
│   ├── components/
│   │   ├── Navbar.jsx
│   │   ├── ProjectCard.jsx
│   │   └── ContactForm.jsx
│   ├── context/
│   │   └── AuthContext.jsx
│   ├── pages/
│   │   ├── HomePage.jsx
│   │   ├── ProjectsPage.jsx
│   │   ├── LoginPage.jsx
│   │   └── AdminPage.jsx
│   ├── App.jsx
│   └── main.jsx
└── .env
```

### 7.4 Variables d'environnement front

**`.env`**

```env
VITE_API_URL=http://localhost:3001/api
```

---

## Étape 8 · Utilitaire API et contexte Auth _(~30 min)_

### 8.1 Fonction apiFetch

**`src/api/apiFetch.js`**

```js
const BASE_URL = import.meta.env.VITE_API_URL;

const apiFetch = async (endpoint, options = {}) => {
  const token = localStorage.getItem('token');

  const headers = {
    'Content-Type': 'application/json',
    ...(token && { Authorization: `Bearer ${token}` }),
    ...options.headers,
  };

  const response = await fetch(`${BASE_URL}${endpoint}`, {
    ...options,
    headers,
  });

  if (!response.ok) {
    const error = await response.json().catch(() => ({ error: 'Erreur inconnue' }));
    throw new Error(error.error || `Erreur ${response.status}`);
  }

  if (response.status === 204) return null;
  return response.json();
};

export default apiFetch;
```

### 8.2 Contexte d'authentification

**`src/context/AuthContext.jsx`**

```jsx
import { createContext, useContext, useState } from 'react';

const AuthContext = createContext(null);

export const AuthProvider = ({ children }) => {
  const [token, setToken] = useState(localStorage.getItem('token'));

  const login = (newToken) => {
    localStorage.setItem('token', newToken);
    setToken(newToken);
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
  };

  const isAuthenticated = !!token;

  return (
    <AuthContext.Provider value={{ token, login, logout, isAuthenticated }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

### 8.3 Brancher le provider dans main.jsx

**`src/main.jsx`**

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

## Étape 9 · Routing et Navbar _(~20 min)_

### 9.1 App.jsx avec les routes

**`src/App.jsx`**

```jsx
import { Routes, Route, Navigate } from 'react-router-dom';
import { useAuth } from './context/AuthContext';
import Navbar from './components/Navbar';
import HomePage from './pages/HomePage';
import ProjectsPage from './pages/ProjectsPage';
import LoginPage from './pages/LoginPage';
import AdminPage from './pages/AdminPage';

const PrivateRoute = ({ children }) => {
  const { isAuthenticated } = useAuth();
  return isAuthenticated ? children : <Navigate to="/login" />;
};

export default function App() {
  return (
    <>
      <Navbar />
      <main className="max-w-5xl mx-auto px-4 py-8">
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/projects" element={<ProjectsPage />} />
          <Route path="/login" element={<LoginPage />} />
          <Route
            path="/admin"
            element={
              <PrivateRoute>
                <AdminPage />
              </PrivateRoute>
            }
          />
        </Routes>
      </main>
    </>
  );
}
```

### 9.2 Navbar

**`src/components/Navbar.jsx`**

```jsx
import { Link, useNavigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export default function Navbar() {
  const { isAuthenticated, logout } = useAuth();
  const navigate = useNavigate();

  const handleLogout = () => {
    logout();
    navigate('/');
  };

  return (
    <nav className="bg-gray-900 text-white px-6 py-4 flex items-center justify-between">
      <Link to="/" className="text-xl font-bold tracking-tight">
        Mon Portfolio
      </Link>
      <div className="flex gap-6 items-center">
        <Link to="/projects" className="hover:text-indigo-400 transition-colors">
          Projets
        </Link>
        {isAuthenticated ? (
          <>
            <Link to="/admin" className="hover:text-indigo-400 transition-colors">
              Admin
            </Link>
            <button
              onClick={handleLogout}
              className="bg-red-600 hover:bg-red-700 px-3 py-1 rounded text-sm transition-colors"
            >
              Déconnexion
            </button>
          </>
        ) : (
          <Link
            to="/login"
            className="bg-indigo-600 hover:bg-indigo-700 px-3 py-1 rounded text-sm transition-colors"
          >
            Connexion
          </Link>
        )}
      </div>
    </nav>
  );
}
```

---

## Étape 10 · Page de connexion _(~20 min)_

**`src/pages/LoginPage.jsx`**

```jsx
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';
import apiFetch from '../api/apiFetch';

export default function LoginPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);
  const { login } = useAuth();
  const navigate = useNavigate();

  const handleSubmit = async (e) => {
    e.preventDefault();
    setError('');
    setLoading(true);
    try {
      const data = await apiFetch('/auth/login', {
        method: 'POST',
        body: JSON.stringify({ email, password }),
      });
      login(data.token);
      navigate('/admin');
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="max-w-md mx-auto mt-16">
      <h1 className="text-2xl font-bold mb-6 text-center">Connexion Admin</h1>
      <form onSubmit={handleSubmit} className="bg-white shadow rounded-lg p-8 flex flex-col gap-4">
        {error && (
          <p className="bg-red-100 text-red-700 px-4 py-2 rounded text-sm">{error}</p>
        )}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-1">Email</label>
          <input
            type="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            required
            className="w-full border border-gray-300 rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500"
          />
        </div>
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-1">Mot de passe</label>
          <input
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
            className="w-full border border-gray-300 rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500"
          />
        </div>
        <button
          type="submit"
          disabled={loading}
          className="bg-indigo-600 text-white py-2 rounded hover:bg-indigo-700 disabled:opacity-50 transition-colors font-medium"
        >
          {loading ? 'Connexion...' : 'Se connecter'}
        </button>
      </form>
    </div>
  );
}
```

---

## Étape 11 · Page Projets publique _(~30 min)_

### 11.1 Composant carte projet

**`src/components/ProjectCard.jsx`**

```jsx
export default function ProjectCard({ project }) {
  return (
    <div className="bg-white rounded-xl shadow hover:shadow-md transition-shadow overflow-hidden">
      {project.image_url && (
        <img
          src={project.image_url}
          alt={project.title}
          className="w-full h-48 object-cover"
        />
      )}
      <div className="p-5">
        <h2 className="text-lg font-bold text-gray-900 mb-2">{project.title}</h2>
        {project.description && (
          <p className="text-gray-600 text-sm mb-3 line-clamp-3">{project.description}</p>
        )}
        {project.tech_stack && (
          <div className="flex flex-wrap gap-2 mb-4">
            {project.tech_stack.split(',').map((tech) => (
              <span
                key={tech}
                className="bg-indigo-100 text-indigo-700 text-xs px-2 py-1 rounded-full"
              >
                {tech.trim()}
              </span>
            ))}
          </div>
        )}
        <div className="flex gap-3">
          {project.github_url && (
            <a
              href={project.github_url}
              target="_blank"
              rel="noopener noreferrer"
              className="text-sm text-gray-700 hover:text-black underline"
            >
              GitHub
            </a>
          )}
          {project.demo_url && (
            <a
              href={project.demo_url}
              target="_blank"
              rel="noopener noreferrer"
              className="text-sm text-indigo-600 hover:text-indigo-800 underline"
            >
              Démo
            </a>
          )}
        </div>
      </div>
    </div>
  );
}
```

### 11.2 Page projets

**`src/pages/ProjectsPage.jsx`**

```jsx
import { useEffect, useState } from 'react';
import apiFetch from '../api/apiFetch';
import ProjectCard from '../components/ProjectCard';

export default function ProjectsPage() {
  const [projects, setProjects] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState('');

  useEffect(() => {
    apiFetch('/projects')
      .then(setProjects)
      .catch((err) => setError(err.message))
      .finally(() => setLoading(false));
  }, []);

  if (loading) return <p className="text-center text-gray-500 mt-16">Chargement...</p>;
  if (error) return <p className="text-center text-red-500 mt-16">{error}</p>;

  return (
    <div>
      <h1 className="text-3xl font-bold mb-8">Mes projets</h1>
      {projects.length === 0 ? (
        <p className="text-gray-500">Aucun projet pour l'instant.</p>
      ) : (
        <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
          {projects.map((project) => (
            <ProjectCard key={project.id} project={project} />
          ))}
        </div>
      )}
    </div>
  );
}
```

---

## Étape 12 · Page Admin (CRUD) _(~60 min)_

**`src/pages/AdminPage.jsx`**

```jsx
import { useEffect, useState } from 'react';
import apiFetch from '../api/apiFetch';

const emptyForm = {
  title: '',
  description: '',
  tech_stack: '',
  github_url: '',
  demo_url: '',
  image_url: '',
};

export default function AdminPage() {
  const [projects, setProjects] = useState([]);
  const [form, setForm] = useState(emptyForm);
  const [editingId, setEditingId] = useState(null);
  const [error, setError] = useState('');
  const [success, setSuccess] = useState('');

  const loadProjects = () => {
    apiFetch('/projects').then(setProjects).catch((err) => setError(err.message));
  };

  useEffect(() => {
    loadProjects();
  }, []);

  const handleChange = (e) => {
    setForm({ ...form, [e.target.name]: e.target.value });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setError('');
    setSuccess('');
    try {
      if (editingId) {
        await apiFetch(`/projects/${editingId}`, {
          method: 'PUT',
          body: JSON.stringify(form),
        });
        setSuccess('Projet modifié !');
      } else {
        await apiFetch('/projects', {
          method: 'POST',
          body: JSON.stringify(form),
        });
        setSuccess('Projet créé !');
      }
      setForm(emptyForm);
      setEditingId(null);
      loadProjects();
    } catch (err) {
      setError(err.message);
    }
  };

  const handleEdit = (project) => {
    setEditingId(project.id);
    setForm({
      title: project.title || '',
      description: project.description || '',
      tech_stack: project.tech_stack || '',
      github_url: project.github_url || '',
      demo_url: project.demo_url || '',
      image_url: project.image_url || '',
    });
    window.scrollTo({ top: 0, behavior: 'smooth' });
  };

  const handleDelete = async (id) => {
    if (!confirm('Supprimer ce projet ?')) return;
    try {
      await apiFetch(`/projects/${id}`, { method: 'DELETE' });
      loadProjects();
    } catch (err) {
      setError(err.message);
    }
  };

  const handleCancel = () => {
    setForm(emptyForm);
    setEditingId(null);
    setError('');
    setSuccess('');
  };

  const fields = [
    { name: 'title', label: 'Titre *', required: true },
    { name: 'description', label: 'Description', textarea: true },
    { name: 'tech_stack', label: 'Technologies (séparées par des virgules)' },
    { name: 'github_url', label: 'URL GitHub' },
    { name: 'demo_url', label: 'URL Démo' },
    { name: 'image_url', label: "URL de l'image" },
  ];

  return (
    <div>
      <h1 className="text-2xl font-bold mb-8">
        {editingId ? 'Modifier le projet' : 'Ajouter un projet'}
      </h1>

      {error && <p className="bg-red-100 text-red-700 px-4 py-2 rounded mb-4 text-sm">{error}</p>}
      {success && <p className="bg-green-100 text-green-700 px-4 py-2 rounded mb-4 text-sm">{success}</p>}

      <form onSubmit={handleSubmit} className="bg-white shadow rounded-lg p-6 flex flex-col gap-4 mb-12">
        {fields.map(({ name, label, required, textarea }) => (
          <div key={name}>
            <label className="block text-sm font-medium text-gray-700 mb-1">{label}</label>
            {textarea ? (
              <textarea
                name={name}
                value={form[name]}
                onChange={handleChange}
                rows={3}
                className="w-full border border-gray-300 rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500"
              />
            ) : (
              <input
                type="text"
                name={name}
                value={form[name]}
                onChange={handleChange}
                required={required}
                className="w-full border border-gray-300 rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500"
              />
            )}
          </div>
        ))}
        <div className="flex gap-3 mt-2">
          <button
            type="submit"
            className="bg-indigo-600 text-white px-5 py-2 rounded hover:bg-indigo-700 transition-colors font-medium"
          >
            {editingId ? 'Mettre à jour' : 'Ajouter'}
          </button>
          {editingId && (
            <button
              type="button"
              onClick={handleCancel}
              className="bg-gray-200 text-gray-700 px-5 py-2 rounded hover:bg-gray-300 transition-colors"
            >
              Annuler
            </button>
          )}
        </div>
      </form>

      <h2 className="text-xl font-bold mb-4">Projets existants</h2>
      {projects.length === 0 ? (
        <p className="text-gray-500">Aucun projet.</p>
      ) : (
        <div className="flex flex-col gap-3">
          {projects.map((project) => (
            <div
              key={project.id}
              className="bg-white shadow rounded-lg px-5 py-4 flex items-center justify-between"
            >
              <div>
                <p className="font-semibold text-gray-900">{project.title}</p>
                {project.tech_stack && (
                  <p className="text-xs text-gray-500 mt-1">{project.tech_stack}</p>
                )}
              </div>
              <div className="flex gap-2">
                <button
                  onClick={() => handleEdit(project)}
                  className="bg-yellow-100 text-yellow-800 px-3 py-1 rounded text-sm hover:bg-yellow-200 transition-colors"
                >
                  Modifier
                </button>
                <button
                  onClick={() => handleDelete(project.id)}
                  className="bg-red-100 text-red-700 px-3 py-1 rounded text-sm hover:bg-red-200 transition-colors"
                >
                  Supprimer
                </button>
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

---

## Étape 13 · Page d'accueil et formulaire de contact _(~30 min)_

### 13.1 Composant formulaire de contact

**`src/components/ContactForm.jsx`**

```jsx
import { useState } from 'react';
import apiFetch from '../api/apiFetch';

export default function ContactForm() {
  const [form, setForm] = useState({ name: '', email: '', message: '' });
  const [status, setStatus] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  const handleChange = (e) => {
    setForm({ ...form, [e.target.name]: e.target.value });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setStatus('');
    setError('');
    setLoading(true);
    try {
      await apiFetch('/contact', {
        method: 'POST',
        body: JSON.stringify(form),
      });
      setStatus('Message envoyé ! Je vous répondrai dès que possible.');
      setForm({ name: '', email: '', message: '' });
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="flex flex-col gap-4 max-w-lg">
      {status && <p className="bg-green-100 text-green-700 px-4 py-2 rounded text-sm">{status}</p>}
      {error && <p className="bg-red-100 text-red-700 px-4 py-2 rounded text-sm">{error}</p>}
      <div>
        <label className="block text-sm font-medium text-gray-700 mb-1">Nom</label>
        <input
          type="text"
          name="name"
          value={form.name}
          onChange={handleChange}
          required
          className="w-full border border-gray-300 rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500"
        />
      </div>
      <div>
        <label className="block text-sm font-medium text-gray-700 mb-1">Email</label>
        <input
          type="email"
          name="email"
          value={form.email}
          onChange={handleChange}
          required
          className="w-full border border-gray-300 rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500"
        />
      </div>
      <div>
        <label className="block text-sm font-medium text-gray-700 mb-1">Message</label>
        <textarea
          name="message"
          value={form.message}
          onChange={handleChange}
          required
          rows={5}
          className="w-full border border-gray-300 rounded px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500"
        />
      </div>
      <button
        type="submit"
        disabled={loading}
        className="bg-indigo-600 text-white py-2 rounded hover:bg-indigo-700 disabled:opacity-50 transition-colors font-medium"
      >
        {loading ? 'Envoi en cours...' : 'Envoyer'}
      </button>
    </form>
  );
}
```

### 13.2 Page d'accueil

**`src/pages/HomePage.jsx`**

```jsx
import { Link } from 'react-router-dom';
import ContactForm from '../components/ContactForm';

export default function HomePage() {
  return (
    <div>
      {/* Hero */}
      <section className="py-16 text-center">
        <h1 className="text-4xl font-extrabold text-gray-900 mb-4">
          Bonjour, je suis <span className="text-indigo-600">Votre Nom</span>
        </h1>
        <p className="text-xl text-gray-600 mb-8 max-w-xl mx-auto">
          Développeur Web & Web Mobile · Passionné par le code et la création d'interfaces.
        </p>
        <Link
          to="/projects"
          className="bg-indigo-600 text-white px-6 py-3 rounded-lg text-lg hover:bg-indigo-700 transition-colors font-medium"
        >
          Voir mes projets
        </Link>
      </section>

      {/* Contact */}
      <section className="mt-12 border-t pt-12">
        <h2 className="text-2xl font-bold mb-6">Me contacter</h2>
        <ContactForm />
      </section>
    </div>
  );
}
```

---

## ✅ Récap fin Jour 2

L'application complète est fonctionnelle :

| Page | URL | Accès |
|---|---|---|
| Accueil + contact | `/` | Public |
| Liste des projets | `/projects` | Public |
| Connexion | `/login` | Public |
| Dashboard admin | `/admin` | Privé (JWT) |

---

# Critères d'évaluation DWWM

| Compétence | Ce qui est évalué |
|---|---|
| CP1 – Maquetter une application | Structure des pages, cohérence UI |
| CP2 – Réaliser une IHM | Composants React, formulaires, gestion des états |
| CP3 – Développer une interface utilisateur web | Tailwind, responsive, accessibilité de base |
| CP5 – Créer une base de données | Schéma SQL, clés, types de données |
| CP6 – Développer les composants d'accès aux données | Modèles, requêtes SQL paramétrées |
| CP7 – Développer la partie back-end | MVC Express, routes REST, middlewares |
| CP8 – Élaborer et mettre en œuvre des composants dans une application | Auth JWT, gestion des erreurs, envoi d'email |

---

# Aller plus loin (bonus)

- Ajouter une validation côté back avec `express-validator`
- Paginer la liste des projets (`?page=1&limit=6`)
- Ajouter une page de détail projet `/projects/:id`
- Déployer le back sur Railway / Render et le front sur Vercel / Netlify
- Protéger les routes par rôle (`role` dans la table `users`)
