# Diamant-gemme-
Somaliland lab
Code complet du système gemmologique + veille scientifique

Voici un socle complet, prêt à cloner et étendre. Il couvre: frontend PWA (Next.js + Tailwind), backend REST (Node/Express), base de données (PostgreSQL), notifications (FCM), offline (IndexedDB + service worker), et intégration de sources (Europe PMC).

---

Arborescence du projet

`
gemmology-system/
├─ README.md
├─ docker-compose.yml
├─ .env.example
├─ backend/
│  ├─ package.json
│  ├─ src/
│  │  ├─ server.ts
│  │  ├─ config.ts
│  │  ├─ db.ts
│  │  ├─ fcm.ts
│  │  ├─ routes/
│  │  │  ├─ veille.ts
│  │  │  ├─ favorites.ts
│  │  │  ├─ auth.ts
│  │  ├─ middleware/authGuard.ts
│  │  ├─ models/
│  │  │  ├─ users.sql
│  │  │  ├─ favorites.sql
│  │  │  ├─ history.sql
├─ frontend/
│  ├─ package.json
│  ├─ next.config.mjs
│  ├─ postcss.config.cjs
│  ├─ tailwind.config.cjs
│  ├─ public/
│  │  ├─ manifest.json
│  │  ├─ icons/
│  │  │  ├─ icon-192.png
│  │  │  ├─ icon-512.png
│  │  ├─ sw.js
│  ├─ src/
│  │  ├─ pages/
│  │  │  ├─ _app.tsx
│  │  │  ├─ index.tsx
│  │  │  ├─ dashboard.tsx
│  │  │  ├─ favorites.tsx
│  │  │  ├─ api/
│  │  │  │  ├─ subscribe-push.ts
│  │  ├─ lib/indexeddb.ts
│  │  ├─ lib/api.ts
│  │  ├─ components/
│  │  │  ├─ Layout.tsx
│  │  │  ├─ Card.tsx
│  │  │  ├─ Metric.tsx
`

---

Configuration d’environnement

`bash

.env.example (copie -> .env pour backend)
PORT=4000
DATABASE_URL=postgres://postgres:postgres@db:5432/gemmology
JWTSECRET=changeme
FCMSERVERKEY=yourfirebaseserver_key
EUROPEPMCBASE=https://www.ebi.ac.uk/europepmc/webservices/rest/search
CORS_ORIGIN=http://localhost:3000
`

---

Docker compose (DB + services)

`yaml

docker-compose.yml
version: "3.9"
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_USER: postgres
      POSTGRES_DB: gemmology
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
`

---

Backend (Node/Express, TypeScript)

`json
// backend/package.json
{
  "name": "gemmology-backend",
  "private": true,
  "scripts": {
    "dev": "ts-node-dev --respawn src/server.ts",
    "start": "node dist/server.js",
    "build": "tsc"
  },
  "dependencies": {
    "axios": "^1.7.2",
    "cors": "^2.8.5",
    "express": "^4.19.2",
    "jsonwebtoken": "^9.0.2",
    "pg": "^8.12.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/jsonwebtoken": "^9.0.6",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.4.5"
  }
}
`

`ts
// backend/src/config.ts
import 'dotenv/config';
export const config = {
  port: Number(process.env.PORT || 4000),
  dbUrl: process.env.DATABASE_URL!,
  jwtSecret: process.env.JWT_SECRET!,
  fcmKey: process.env.FCMSERVERKEY!,
  corsOrigin: process.env.CORS_ORIGIN!,
  europePmcBase: process.env.EUROPEPMCBASE!
};
`

`ts
// backend/src/db.ts
import { Pool } from 'pg';
import { config } from './config';
export const pool = new Pool({ connectionString: config.dbUrl });
export async function query<T = any>(text: string, params?: any[]) {
  const res = await pool.query<T>(text, params);
  return res.rows;
}
`

`ts
// backend/src/fcm.ts
import axios from 'axios';
import { config } from './config';

export async function sendPush(token: string, title: string, body: string) {
  const url = 'https://fcm.googleapis.com/fcm/send';
  const payload = {
    to: token,
    notification: { title, body }
  };
  await axios.post(url, payload, {
    headers: {
      'Authorization': key=${config.fcmKey},
      'Content-Type': 'application/json'
    }
  });
}
`

`ts
// backend/src/middleware/authGuard.ts
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { config } from '../config';

export function authGuard(req: Request, res: Response, next: NextFunction) {
  const hdr = req.headers.authorization;
  if (!hdr) return res.status(401).json({ error: 'missing auth header' });
  const token = hdr.replace('Bearer ', '');
  try {
    const payload = jwt.verify(token, config.jwtSecret) as any;
    (req as any).user = payload;
    next();
  } catch {
    res.status(401).json({ error: 'invalid token' });
  }
}
`

`ts
// backend/src/routes/auth.ts
import { Router } from 'express';
import jwt from 'jsonwebtoken';
import { config } from '../config';
import { query } from '../db';

const router = Router();

router.post('/signup', async (req, res) => {
  const { email, name } = req.body;
  const [user] = await query('INSERT INTO users(email, name) VALUES($1,$2) RETURNING id,email,name', [email, name]);
  const token = jwt.sign({ id: user.id, email: user.email }, config.jwtSecret, { expiresIn: '7d' });
  res.json({ token, user });
});

router.post('/login', async (req, res) => {
  const { email } = req.body;
  const [user] = await query('SELECT id,email,name FROM users WHERE email=$1', [email]);
  if (!user) return res.status(404).json({ error: 'not found' });
  const token = jwt.sign({ id: user.id, email: user.email }, config.jwtSecret, { expiresIn: '7d' });
  res.json({ token, user });
});

export default router;
`

`ts
// backend/src/routes/favorites.ts
import { Router } from 'express';
import { query } from '../db';
import { authGuard } from '../middleware/authGuard';

const router = Router();
router.use(authGuard);

router.get('/', async (req, res) => {
  const userId = (req as any).user.id;
  const rows = await query('SELECT id, keyword FROM favorites WHERE user_id=$1 ORDER BY id DESC', [userId]);
  res.json(rows);
});

router.post('/', async (req, res) => {
  const userId = (req as any).user.id;
  const { keyword } = req.body;
  const [fav] = await query('INSERT INTO favorites(user_id, keyword) VALUES($1,$2) RETURNING id, keyword', [userId, keyword]);
  res.json(fav);
});

router.delete('/:id', async (req, res) => {
  const userId = (req as any).user.id;
  const { id } = req.params;
  await query('DELETE FROM favorites WHERE id=$1 AND user_id=$2', [id, userId]);
  res.json({ ok: true });
});

export default router;
`

`ts
// backend/src/routes/veille.ts
import { Router } from 'express';
import axios from 'axios';
import { config } from '../config';
import { authGuard } from '../middleware/authGuard';

const router = Router();
router.use(authGuard);

router.get('/', async (req, res) => {
  const q = (req.query.q as string) || 'gemology';
  const pageSize = Number(req.query.pageSize || 10);
  // Europe PMC REST search, JSON
  const url = ${config.europePmcBase}?query=${encodeURIComponent(q)}&format=json&pageSize=${pageSize};
  const { data } = await axios.get(url);
  res.json(data);
});

export default router;
`

`ts
// backend/src/server.ts
import express from 'express';
import cors from 'cors';
import { config } from './config';
import authRoutes from './routes/auth';
import favoritesRoutes from './routes/favorites';
import veilleRoutes from './routes/veille';

const app = express();
app.use(express.json());
app.use(cors({ origin: config.corsOrigin }));

app.get('/health', (_req, res) => res.json({ ok: true }));
app.use('/api/auth', authRoutes);
app.use('/api/favorites', favoritesRoutes);
app.use('/api/veille', veilleRoutes);

app.listen(config.port, () => {
  console.log(Backend running on :${config.port});
});
`

`sql
-- backend/src/models/users.sql
CREATE TABLE IF NOT EXISTS users (
  id SERIAL PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  name TEXT,
  created_at TIMESTAMP DEFAULT now()
);
`

`sql
-- backend/src/models/favorites.sql
CREATE TABLE IF NOT EXISTS favorites (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  keyword TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT now()
);
`

`sql
-- backend/src/models/history.sql
CREATE TABLE IF NOT EXISTS history (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
  query TEXT NOT NULL,
  source TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT now()
);
`

---

Frontend (Next.js + Tailwind + PWA)

`json
// frontend/package.json
{
  "name": "gemmology-frontend",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build && next export",
    "start": "next start"
  },
  "dependencies": {
    "next": "^14.2.5",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "axios": "^1.7.2",
    "idb": "^7.1.1",
    "tailwindcss": "^3.4.4"
  },
  "devDependencies": {
    "autoprefixer": "^10.4.19",
    "postcss": "^8.4.39",
    "typescript": "^5.4.5"
  }
}
`

`js
// frontend/tailwind.config.cjs
module.exports = {
  content: ["./src//*.{js,ts,jsx,tsx}"],
  theme: { extend: {} },
  plugins: []
};
`

`js
// frontend/postcss.config.cjs
module.exports = { plugins: { tailwindcss: {}, autoprefixer: {} } };
`

`json
// frontend/public/manifest.json
{
  "name": "Gemmology PWA",
  "short_name": "Gemmology",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#0f172a",
  "theme_color": "#1e3a8a",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
`

`js
// frontend/public/sw.js
self.addEventListener('install', (event) => {
  event.waitUntil(caches.open('v1').then((cache) => cache.addAll(['/','/manifest.json'])));
});
self.addEventListener('fetch', (event) => {
  event.respondWith(caches.match(event.request).then((res) => res || fetch(event.request)));
});
`

`tsx
// frontend/src/pages/_app.tsx
import type { AppProps } from 'next/app';
import '../styles.css';

export default function App({ Component, pageProps }: AppProps) {
  return <Component {...pageProps} />;
}
`

`tsx
// frontend/src/pages/index.tsx
import Link from 'next/link';

export default function Home() {
  return (
    <main className="min-h-screen bg-slate-900 text-white p-8">
      <h1 className="text-3xl font-bold">Gemmology + Veille scientifique</h1>
      <p className="mt-2 opacity-80">Dashboard, favoris, notifications, offline.</p>
      <div className="mt-6 flex gap-4">
        <Link className="px-4 py-2 bg-blue-600 rounded" href="/dashboard">Dashboard</Link>
        <Link className="px-4 py-2 bg-emerald-600 rounded" href="/favorites">Favoris</Link>
      </div>
    </main>
  );
}
`

`tsx
// frontend/src/components/Layout.tsx
export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <div className="min-h-screen bg-slate-900 text-white">
      <header className="p-4 border-b border-slate-700">Gemmology</header>
      <main className="p-6">{children}</main>
    </div>
  );
}
`

`ts
// frontend/src/lib/api.ts
import axios from 'axios';
const BASE = process.env.NEXTPUBLICAPI_BASE || 'http://localhost:4000';
let token: string | null = null;

export function setToken(t: string) { token = t; }
function auth() { return token ? { Authorization: Bearer ${token} } : {}; }

export async function signup(email: string, name: string) {
  const { data } = await axios.post(${BASE}/api/auth/signup, { email, name });
  setToken(data.token); return data;
}
export async function login(email: string) {
  const { data } = await axios.post(${BASE}/api/auth/login, { email });
  setToken(data.token); return data;
}
export async function search(q: string, pageSize = 10) {
  const { data } = await axios.get(${BASE}/api/veille, { params: { q, pageSize }, headers: auth() });
  return data;
}
export async function listFavorites() {
  const { data } = await axios.get(${BASE}/api/favorites, { headers: auth() }); return data;
}
export async function addFavorite(keyword: string) {
  const { data } = await axios.post(${BASE}/api/favorites, { keyword }, { headers: auth() }); return data;
}
export async function deleteFavorite(id: number) {
  const { data } = await axios.delete(${BASE}/api/favorites/${id}, { headers: auth() }); return data;
}
`

`ts
// frontend/src/lib/indexeddb.ts
import { openDB } from 'idb';
const DB_NAME = 'gemmology';
const STORE = 'favorites';

export async function db() {
  return openDB(DB_NAME, 1, {
    upgrade(db) {
      if (!db.objectStoreNames.contains(STORE)) db.createObjectStore(STORE, { keyPath: 'id', autoIncrement: true });
    }
  });
}
export async function saveLocalFavorite(keyword: string) {
  const d = await db(); await d.put(STORE, { keyword, createdAt: Date.now() });
}
export async function listLocalFavorites() {
  const d = await db(); return d.getAll(STORE);
}
`

`tsx
// frontend/src/pages/dashboard.tsx
import { useEffect, useState } from 'react';
import Layout from '../components/Layout';
import { search, signup } from '../lib/api';

export default function Dashboard() {
  const [q, setQ] = useState('gemology');
  const [results, setResults] = useState<any>(null);

  useEffect(() => { signup('demo@example.com', 'Demo'); }, []);

  async function run() {
    const data = await search(q, 10);
    setResults(data);
  }

  return (
    <Layout>
      <div className="flex gap-2">
        <input className="px-3 py-2 rounded bg-slate-800 border border-slate-700"
               value={q} onChange={e=>setQ(e.target.value)} placeholder="Mot-clé (ex: beryl)" />
        <button className="px-4 py-2 bg-blue-600 rounded" onClick={run}>Rechercher</button>
      </div>

      <div className="mt-6">
        {!results ? <p className="opacity-70">Aucun résultat…</p> :
          <ul className="space-y-3">
            {(results.resultList?.result || []).map((r: any) => (
              <li key={r.id} className="p-4 bg-slate-800 rounded">
                <div className="font-semibold">{r.title}</div>
                <div className="text-sm opacity-70">{r.authorString}</div>
                <a className="text-blue-300 text-sm" href={r.link ?? '#'} target="_blank">Ouvrir</a>
              </li>
            ))}
          </ul>}
      </div>
    </Layout>
  );
}
`

`tsx
// frontend/src/pages/favorites.tsx
import { useEffect, useState } from 'react';
import Layout from '../components/Layout';
import { addFavorite, listFavorites, login } from '../lib/api';
import { listLocalFavorites, saveLocalFavorite } from '../lib/indexeddb';

export default function Favorites() {
  const [remoteFavs, setRemoteFavs] = useState<any[]>([]);
  const [localFavs, setLocalFavs] = useState<any[]>([]);
  const [keyword, setKeyword] = useState('');

  useEffect(() => {
    (async () => {
      await login('demo@example.com');
      setRemoteFavs(await listFavorites());
      setLocalFavs(await listLocalFavorites());
    })();
  }, []);

  async function add() {
    if (!keyword) return;
    await addFavorite(keyword);
    await saveLocalFavorite(keyword);
    setRemoteFavs(await listFavorites());
    setLocalFavs(await listLocalFavorites());
    setKeyword('');
  }

  return (
    <Layout>
      <div className="flex gap-2">
        <input className="px-3 py-2 rounded bg-slate-800 border border-slate-700"
               value={keyword} onChange={e=>setKeyword(e.target.value)} placeholder="Mot-clé favori" />
        <button className="px-4 py-2 bg-emerald-600 rounded" onClick={add}>Ajouter</button>
      </div>

      <div className="mt-8 grid grid-cols-1 md:grid-cols-2 gap-6">
        <section>
          <h2 className="font-bold mb-2">Favoris (serveur)</h2>
          <ul className="space-y-2">
            {remoteFavs.map((f:any)=>(
              <li key={f.id} className="p-3 bg-slate-800 rounded">{f.keyword}</li>
            ))}
          </ul>
        </section>
        <section>
          <h2 className="font-bold mb-2">Favoris (offline)</h2>
          <ul className="space-y-2">
            {localFavs.map((f:any, i:number)=>(
              <li key={i} className="p-3 bg-slate-800 rounded">{f.keyword}</li>
            ))}
          </ul>
        </section>
      </div>
    </Layout>
  );
}
`

`css
/ frontend/src/styles.css /
@tailwind base;
@tailwind components;
@tailwind utilities;

html, body { height: 100%; }
`

`ts
// frontend/src/pages/api/subscribe-push.ts
// (placeholder: en vrai, stocker le token FCM côté backend)
export default function handler(_req: any, res: any) {
  res.status(200).json({ ok: true });
}
`

---

Initialisation de la base (exécution des scripts SQL)

`bash
psql "$DATABASE_URL" -f backend/src/models/users.sql
psql "$DATABASE_URL" -f backend/src/models/favorites.sql
psql "$DATABASE_URL" -f backend/src/models/history.sql
`

---

Lancement local

`bash

1) DB
docker-compose up -d

2) Backend
cd backend
cp ../.env.example .env
npm i
npm run dev

3) Frontend
cd ../frontend
npm i
npm run dev

Ouvre http://localhost:3000
`

---

Notes d’extension

- Sécurité: remplacer les endpoints d’auth par un vrai mot de passe + hashing (bcrypt) et rafraîchissement de tokens.
- FCM: collecter les tokens device (web push ou FCM JS SDK) et les stocker côté backend (table fcm_tokens).
- Sources: ajouter des adaptateurs pour Springer/Wiley (API ou scraping légal selon conditions).
- Monitoring: brancher les logs dans une stack ELK/Grafana et exposer /metrics (Prometheus) côté backend.
- PWA: améliorer le service worker (Workbox) pour cache fin + stratégie network-first sur veille.

---

Souhaites-tu que je te fournisse un dépôt Git minimal (tar structure) ou des scripts supplémentaires pour l’installation automatisée sur un VPS (Nginx + Systemd) ?![Screenshot_2025-11-23-23-57-20-740_com picturerock rock-edit](https://github.com/user-attachments/assets/f181721c-73da-4acd-8a60-67fb82eab05c)
