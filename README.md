# HireLoop — AI-Powered Human Task Marketplace

> La couche "monde réel" pour les agents IA. Publiez une tâche, un humain l'accomplit.

---

## Table des matières

- [Vue d'ensemble](#vue-densemble)
- [Fonctionnalités](#fonctionnalités)
- [Architecture](#architecture)
- [Stack technique](#stack-technique)
- [Modèle de données](#modèle-de-données)
- [Installation](#installation)
- [Variables d'environnement](#variables-denvironnement)
- [API Reference](#api-reference)
- [Intégration IA / MCP](#intégration-ia--mcp)
- [Flux de paiement (escrow)](#flux-de-paiement-escrow)
- [Roadmap](#roadmap)
- [Contribution](#contribution)
- [Licence](#licence)

---

## Vue d'ensemble

**HireLoop** est une marketplace bidirectionnelle où des humains et des agents IA peuvent publier des tâches physiques ou digitales, et des exécutants humains vérifiés les accomplissent contre rémunération.

Le projet est inspiré de [rentahuman.ai](https://rentahuman.ai) mais conçu pour être **self-hostable**, **API-first**, et adapté au marché francophone / africain (support Wave, Orange Money, etc.).

```
Demandeur (humain ou bot IA)
        │
        │  POST /api/tasks/
        ▼
   Plateforme HireLoop
        │   ┌──────────────────┐
        │   │  Gestion tâches  │
        │   │  Escrow / paiement│
        │   │  Matching        │
        │   │  Validation      │
        │   └──────────────────┘
        │
        ▼
   Exécutant humain vérifié
```

---

## Fonctionnalités

### Pour les demandeurs (clients)
- Publier une tâche avec budget fixe ou horaire
- Définir des critères de preuve (photo, vidéo, GPS, rapport écrit)
- Consulter les candidatures et sélectionner un exécutant
- Suivre l'avancement en temps réel (WebSocket)
- Libérer ou disputer le paiement escrow
- Laisser une évaluation bidirectionnelle

### Pour les exécutants (workers)
- Créer un profil avec compétences, localisation et portfolio
- Postuler à des tâches matchant son profil
- Soumettre des preuves de complétion
- Recevoir le paiement automatiquement à validation
- Construire une réputation (note, reviews, badge vérifié)

### Pour les agents IA (intégration API)
- Authentification par API key
- Endpoints REST complets (publier, assigner, valider, payer)
- Webhooks pour recevoir les événements en temps réel
- Serveur MCP natif pour intégration dans des workflows Claude/GPT

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                        Clients                           │
│   Next.js 14 Web App      React Native Mobile App        │
└────────────────────┬─────────────────────────────────────┘
                     │ HTTPS / WebSocket
┌────────────────────▼─────────────────────────────────────┐
│                    API Gateway (Nginx)                    │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│               Django REST Framework                      │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │  Tasks   │  │  Users   │  │ Payments │  │  MCP    │ │
│  │  app     │  │  app     │  │  app     │  │ server  │ │
│  └──────────┘  └──────────┘  └──────────┘  └─────────┘ │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │          Celery Workers (tâches async)            │   │
│  │   Notifications · Matching · Paiements · Expiry  │   │
│  └──────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
         │                │                │
┌────────▼───┐   ┌────────▼──┐   ┌────────▼───┐
│ PostgreSQL │   │   Redis   │   │   S3 / R2  │
│ (données) │   │ (cache +  │   │  (médias,  │
│            │   │  broker)  │   │   preuves) │
└────────────┘   └───────────┘   └────────────┘
```

---

## Stack technique

| Couche        | Technologie                                      |
|---------------|--------------------------------------------------|
| Backend       | Python 3.12 · Django 5 · Django REST Framework  |
| Base de données | PostgreSQL 16 · pgvector (matching sémantique) |
| Cache / Queue | Redis 7 · Celery 5                              |
| Frontend web  | Next.js 14 · TypeScript · Tailwind CSS          |
| Mobile        | React Native · Expo                             |
| Stockage      | Cloudflare R2 (compatible S3)                   |
| Paiements     | Stripe Connect · Wave · Orange Money (plugin)   |
| Notifications | FCM · APNs · SendGrid · WebSocket (channels)   |
| Infra         | Docker · Docker Compose · GitHub Actions        |
| Monitoring    | Sentry · Prometheus · Grafana                   |

---

## Modèle de données

### Entités principales

```
User
├── id (UUID)
├── email
├── role[] → [client, worker, bot]
├── is_verified (bool)
├── stripe_account_id
├── wave_phone
└── rating_avg

Task
├── id (UUID)
├── title, description
├── category → [delivery, research, marketing, events, digital, other]
├── budget (Decimal)
├── budget_type → [fixed, hourly]
├── location_type → [remote, onsite]
├── location (PointField, nullable)
├── status → [draft, open, assigned, in_review, completed, paid, cancelled, disputed]
├── proof_requirements (JSONField)
├── deadline (DateTimeField)
├── client_id → FK(User)
└── worker_id → FK(User, nullable)

Application
├── id (UUID)
├── task_id → FK(Task)
├── worker_id → FK(User)
├── cover_note (Text)
├── proposed_price (Decimal)
└── status → [pending, accepted, rejected, withdrawn]

Escrow
├── id (UUID)
├── task_id → FK(Task, unique)
├── amount (Decimal)
├── currency (CharField)
├── status → [pending, held, released, refunded, disputed]
├── payment_provider → [stripe, wave, orange_money]
└── provider_ref (CharField)

Proof
├── id (UUID)
├── task_id → FK(Task)
├── worker_id → FK(User)
├── type → [photo, video, text, gps, document]
├── file_url (URLField)
├── gps_coords (PointField, nullable)
├── submitted_at
└── approved_by → FK(User, nullable)

Review
├── id (UUID)
├── task_id → FK(Task)
├── reviewer_id → FK(User)
├── reviewee_id → FK(User)
├── rating (1-5)
└── comment (Text)

Webhook
├── id (UUID)
├── user_id → FK(User)
├── url (URLField)
├── events[] → [task.created, task.assigned, proof.submitted, payment.released, ...]
├── secret (CharField)  ← HMAC signing
└── is_active (bool)
```

### Transitions de statut d'une tâche

```
draft ──► open ──► assigned ──► in_review ──► completed ──► paid
                     │                │
                     │           ◄────┘ (preuve rejetée)
                     │
                     └──► cancelled
                     └──► disputed ──► (résolution manuelle)
```

---

## Installation

### Prérequis

- Docker & Docker Compose v2+
- Python 3.12+
- Node.js 20+

### 1. Cloner le repo

```bash
git clone https://github.com/yourname/hireloop.git
cd hireloop
```

### 2. Configurer l'environnement

```bash
cp .env.example .env
# Éditer .env avec vos clés (voir section Variables d'environnement)
```

### 3. Lancer avec Docker Compose

```bash
docker compose up -d
```

Cela démarre : PostgreSQL, Redis, le serveur Django, Celery worker, Celery beat, et Nginx.

### 4. Initialiser la base de données

```bash
docker compose exec api python manage.py migrate
docker compose exec api python manage.py createsuperuser
docker compose exec api python manage.py loaddata initial_categories
```

### 5. Lancer le frontend

```bash
cd frontend
npm install
npm run dev
```

L'app est disponible sur `http://localhost:3000`, l'API sur `http://localhost:8000`.

### Développement local sans Docker

```bash
# Backend
python -m venv venv && source venv/bin/activate
pip install -r requirements/dev.txt
python manage.py migrate
python manage.py runserver

# Worker Celery (terminal séparé)
celery -A config worker -l info

# Frontend
cd frontend && npm run dev
```

---

## Variables d'environnement

```env
# Django
SECRET_KEY=your-secret-key
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1
DATABASE_URL=postgresql://postgres:password@localhost:5432/hireloop

# Redis
REDIS_URL=redis://localhost:6379/0

# Stockage
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_STORAGE_BUCKET_NAME=hireloop-media
AWS_S3_ENDPOINT_URL=https://<account>.r2.cloudflarestorage.com

# Paiements
STRIPE_SECRET_KEY=sk_test_...
STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Notifications
SENDGRID_API_KEY=SG....
FCM_SERVER_KEY=

# CORS
CORS_ALLOWED_ORIGINS=http://localhost:3000
```

---

## API Reference

L'API suit les conventions REST. Toutes les réponses sont en JSON. L'authentification se fait via **JWT** (sessions utilisateur) ou **API Key** (agents IA).

### Authentification

```http
# JWT (humains)
POST /api/auth/token/
Content-Type: application/json
{ "email": "user@example.com", "password": "..." }

# API Key (agents IA) — header sur toutes les requêtes
X-API-Key: hl_live_xxxxxxxxxxxx
```

### Tâches

```http
# Lister les tâches ouvertes
GET /api/tasks/?status=open&category=delivery&location=remote

# Publier une tâche
POST /api/tasks/
{
  "title": "Livrer un colis à Cotonou centre",
  "category": "delivery",
  "budget": 5000,
  "budget_type": "fixed",
  "location_type": "onsite",
  "location": { "lat": 6.3654, "lng": 2.4183 },
  "deadline": "2026-06-10T18:00:00Z",
  "proof_requirements": {
    "types": ["photo"],
    "instructions": "Photo avec le destinataire et le colis visible"
  }
}

# Détail d'une tâche
GET /api/tasks/{id}/

# Assigner un candidat
POST /api/tasks/{id}/assign/
{ "application_id": "uuid" }

# Valider la preuve et libérer le paiement
POST /api/tasks/{id}/approve/

# Disputer une tâche
POST /api/tasks/{id}/dispute/
{ "reason": "Le colis n'a pas été livré à la bonne adresse." }
```

### Candidatures

```http
# Postuler à une tâche
POST /api/tasks/{task_id}/applications/
{ "cover_note": "...", "proposed_price": 4500 }

# Lister mes candidatures
GET /api/applications/?status=pending
```

### Preuves

```http
# Soumettre une preuve
POST /api/tasks/{task_id}/proofs/
Content-Type: multipart/form-data
file=<binary>  &  type=photo
```

### Webhooks

```http
# Enregistrer un webhook
POST /api/webhooks/
{
  "url": "https://myagent.example.com/hooks/hireloop",
  "events": ["task.assigned", "proof.submitted", "payment.released"]
}
```

Chaque événement est signé avec un header `X-HireLoop-Signature: sha256=...` (HMAC-SHA256 du payload avec votre secret).

---

## Intégration IA / MCP

HireLoop expose un **serveur MCP** (Model Context Protocol) permettant à des agents Claude, GPT ou LangGraph de piloter la plateforme nativement.

### Connexion MCP

```json
{
  "mcpServers": {
    "hireloop": {
      "type": "url",
      "url": "https://api.hireloop.io/mcp/v1/sse",
      "headers": {
        "X-API-Key": "hl_live_xxxxxxxxxxxx"
      }
    }
  }
}
```

### Outils MCP disponibles

| Outil | Description |
|-------|-------------|
| `search_workers` | Cherche des exécutants par compétence et localisation |
| `create_task` | Publie une nouvelle tâche |
| `list_applications` | Liste les candidatures reçues pour une tâche |
| `assign_worker` | Sélectionne un candidat |
| `get_task_status` | Récupère le statut et les preuves soumises |
| `approve_task` | Valide la complétion et libère le paiement |
| `dispute_task` | Ouvre un litige |

### Exemple d'usage (LangGraph)

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

async with MultiServerMCPClient({
    "hireloop": {
        "url": "https://api.hireloop.io/mcp/v1/sse",
        "transport": "sse",
        "headers": {"X-API-Key": os.environ["HIRELOOP_API_KEY"]}
    }
}) as client:
    tools = client.get_tools()
    # L'agent peut maintenant appeler create_task, assign_worker, etc.
```

---

## Flux de paiement (escrow)

```
1. Client publie la tâche + bloque le montant
   POST /api/tasks/ → Stripe PaymentIntent créé → status: held

2. Worker accomplit la tâche + soumet la preuve
   POST /api/tasks/{id}/proofs/ → status: in_review

3a. Client approuve
    POST /api/tasks/{id}/approve/
    → Stripe Transfer vers le compte connecté du worker
    → Frais plateforme prélevés (10%)
    → status: paid

3b. Client dispute
    POST /api/tasks/{id}/dispute/
    → Escrow gelé, équipe support notifiée
    → Résolution sous 48h
    → Remboursement ou paiement selon décision

3c. Auto-validation
    Si le client ne répond pas sous 72h après soumission de preuve
    → Validation automatique (Celery beat)
```

---

## Roadmap

### v0.1 — MVP (en cours)
- [x] Auth JWT + API Key
- [x] CRUD tâches et candidatures
- [x] Upload de preuves (photo)
- [x] Escrow Stripe (marchés internationaux)
- [ ] Système de reviews
- [ ] Notifications email

### v0.2 — Intégration marché local
- [ ] Paiement Wave (Sénégal, Côte d'Ivoire, Burkina)
- [ ] Paiement MTN Mobile Money / Orange Money
- [ ] Support multilingue (FR, EN)
- [ ] App mobile React Native

### v0.3 — Intelligence
- [ ] Matching sémantique avec pgvector
- [ ] Serveur MCP complet
- [ ] Scoring de fiabilité des workers
- [ ] Détection de fraude basique (preuves dupliquées)

### v1.0 — Production
- [ ] Vérification d'identité (NIN, CNI)
- [ ] Dashboard analytics demandeur
- [ ] Programme de parrainage
- [ ] API publique documentée (Swagger)

---

## Contribution

Les contributions sont les bienvenues. Pour contribuer :

1. Forker le repo
2. Créer une branche : `git checkout -b feat/ma-feature`
3. Commiter : `git commit -m "feat: ajouter le matching par localisation"`
4. Pousser : `git push origin feat/ma-feature`
5. Ouvrir une Pull Request

### Standards de code

- Python : PEP 8, formatté avec `ruff`
- TypeScript : ESLint + Prettier
- Commits : [Conventional Commits](https://www.conventionalcommits.org/)
- Tests : couverture minimale de 80% sur les apps `tasks` et `payments`

### Lancer les tests

```bash
# Backend
docker compose exec api pytest --cov=apps --cov-report=term-missing

# Frontend
cd frontend && npm run test
```

---

## Licence

MIT © 2026 HireLoop — Voir [LICENSE](./LICENSE) pour les détails.

---

> Construit avec Django, Next.js et beaucoup de café ☕
