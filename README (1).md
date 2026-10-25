# Dalil

Un employé veut savoir combien de jours de congé il a par an, comment déclarer une note de frais ou s'il peut déployer un vendredi. Il pose la question à Dalil (« guide » en arabe), qui cherche dans le règlement intérieur, les procédures et les notes RH, puis répond en citant la page. S'il ne trouve rien, il le dit au lieu d'inventer. Il comprend le français, l'arabe et la darja.

Les questions restées sans réponse servent aussi. Dalil les regroupe par sujet dans le tableau de bord « Trous de connaissance », et les RH voient quel document manque et combien de personnes l'ont cherché.

Projet de fin d'études.

## Comment ça marche

```
                 Indexation (à l'ajout d'un document)
 PDF / Word / texte ──▶ extraction ──▶ découpage en passages ──▶ embeddings (bge-m3) ──▶ PostgreSQL + pgvector

                 Question d'un employé
 question ──▶ embedding ──▶ recherche des 5 passages les plus proches
                            (filtrés selon le département de l'employé)
          ──▶ score trop faible ? ──▶ « je ne sais pas » + question enregistrée comme trou
          ──▶ sinon LLM avec les passages numérotés ──▶ réponse citée [1] [2] + sources
```

C'est le principe du RAG (*Retrieval-Augmented Generation*). Le modèle répond avec les passages retrouvés dans les documents de l'entreprise, pas avec ce qu'il a appris pendant son entraînement.

## Fonctionnalités

- Ajout de documents PDF, Word, texte ou Markdown, indexés en arrière-plan.
- Chaque information de la réponse porte le numéro de sa source. Un clic ouvre le document.
- Un document peut être réservé à un département. Les autres employés ne le voient ni dans la liste, ni dans les réponses.
- Sous un seuil de similarité, le modèle n'est pas appelé du tout : pas de passage pertinent, pas de réponse inventée.
- Les employés indiquent si une réponse les a aidés.
- Le tableau de bord regroupe les questions sans réponse ou jugées inutiles, même quand elles sont formulées autrement ou dans une autre langue, et les classe par nombre de demandes.
- Statistiques : taux de réponse, satisfaction, documents indexés.
- Comptes administrateur et employé, authentification JWT.
- Fonctionne en local et gratuitement avec Ollama. L'API Claude donne de meilleures réponses si on en a besoin.

## Démarrage rapide

Il faut Docker et Docker Compose, environ 6 Go de disque et 8 Go de RAM.

```bash
cp .env.example .env          # changer les mots de passe et JWT_SECRET
docker compose up -d --build
docker compose logs -f ollama  # attendre la fin du téléchargement des modèles (~3 Go, une seule fois)
```

Ouvrir http://localhost:3000 et se connecter avec `ADMIN_EMAIL` / `ADMIN_PASSWORD` du fichier `.env`.

Un document ajouté avant la fin du téléchargement passe en « Échec ». Le bouton **Relancer** l'indexe à nouveau.

### Utiliser Claude à la place du modèle local

Les petits modèles locaux s'en sortent bien en français, moins en darja. Pour de meilleures réponses, dans `.env` :

```
CHAT_PROVIDER=anthropic
ANTHROPIC_API_KEY=sk-ant-...
```

Les embeddings restent calculés en local par Ollama. Seuls la question et les passages retrouvés partent vers l'API.

## Démo pour la soutenance

Le dossier `samples/` contient trois documents d'une entreprise fictive.

1. En admin, ajouter `reglement-interieur.pdf` (toute l'entreprise), `note-de-frais.docx` (Finance) et `procedure-deploiement.md` (Technique).
2. Créer deux employés, un en Finance et un en Technique.
3. Avec l'employé Finance : « Combien de jours de congé ai-je par an ? » La réponse cite la page 2 du règlement.
4. Toujours en Finance : « Peut-on déployer le vendredi ? » Pas de réponse, la procédure est réservée à la Technique. L'employé Technique, lui, l'obtient.
5. Poser plusieurs variantes d'une question que les documents ne couvrent pas : « Le télétravail est-il autorisé ? », « 9addech men nhar télétravail ? », « Remote work policy? ».
6. En admin, ouvrir **Trous de connaissance**. Les trois variantes forment un seul sujet.

## Réglages

| Variable | Défaut | Effet |
|---|---|---|
| `MIN_SCORE` | 0.45 | Similarité minimale d'un passage pour être utilisé. Plus haut, Dalil dit plus souvent « je ne sais pas » et se trompe moins. |
| `GAP_SIMILARITY` | 0.78 | Au-dessus de cette similarité, deux questions sans réponse comptent comme le même sujet. |
| `TOP_K` | 5 | Nombre de passages envoyés au modèle. |
| `CHAT_MODEL` | qwen2.5:3b | Modèle Ollama. `qwen2.5:7b` répond mieux avec 16 Go de RAM. |

Ces valeurs sont réglées pour `bge-m3`. Pour les ajuster, regarder le score des sources dans les réponses de l'API.

## Développement

```bash
# Base de test (PostgreSQL + pgvector)
docker run -d --name dalil-db -p 5432:5432 \
  -e POSTGRES_USER=dalil -e POSTGRES_PASSWORD=dalil -e POSTGRES_DB=dalil_test pgvector/pgvector:pg16

cd backend
pip install -r requirements.txt pytest
pytest                        # les modèles sont remplacés par des faux, pas besoin d'Ollama

docker exec dalil-db createdb -U dalil dalil
uvicorn app.main:app --reload # API sur :8000, documentation sur /docs

cd ../frontend
npm install
npm run dev                   # interface sur :5173
```

Les tests tournent sur une vraie base pgvector, parce que c'est la recherche filtrée par département qu'il faut vérifier. À chaque push, GitHub Actions lance une base pgvector, les tests, le build du frontend et celui des images Docker.

## API principale

| Méthode | Route | Description |
|---|---|---|
| POST | `/api/auth/login` | Connexion, renvoie un jeton JWT |
| POST | `/api/chat` | Poser une question |
| GET | `/api/chat/history` | Historique de l'utilisateur |
| POST | `/api/questions/{id}/feedback` | Réponse utile (1) ou non (-1) |
| GET | `/api/documents` | Documents visibles par l'utilisateur |
| POST | `/api/documents` | Ajouter un document (admin) |
| GET | `/api/documents/{id}/file` | Ouvrir le fichier original |
| POST | `/api/documents/{id}/reindex` | Relancer l'indexation (admin) |
| GET | `/api/admin/gaps?days=30` | Trous de connaissance (admin) |
| GET | `/api/admin/stats` | Statistiques (admin) |
| GET/POST | `/api/users`, `/api/departments` | Gestion des comptes (admin) |

## Structure

```
backend/app/
  ingest.py     extraction PDF/Word/texte, découpage, indexation
  ai.py         appels aux modèles (Ollama, Claude)
  rag.py        recherche filtrée par département, prompt, citations
  gaps.py       regroupement des questions sans réponse
  auth.py       JWT, mots de passe, rôles
  routers/      accounts, documents, chat (+ admin)
frontend/src/pages/
  Chat, Documents, Gaps, Users, Login
samples/        documents de démonstration
```

## Pistes pour la suite

- Conversations à plusieurs tours, pour les questions de suivi.
- Recherche hybride mots-clés + vecteurs, utile pour les numéros d'articles et les références.
- Réponses affichées mot par mot (streaming).
- Synchronisation avec Google Drive, SharePoint ou Confluence.
- OCR des documents scannés.
- Un jeu de questions de référence pour mesurer la qualité des réponses à chaque changement.
