# KlassBox PRO — Architecture & Documentation Technique Complète

---

Ce document récapitule la feuille de route technique, l'architecture logicielle, les schémas de base de données, ainsi que la logique métier de l'ensemble des modules développés pour la plateforme **KlassBox PRO**.

## 1. Aperçu Général de la Plateforme

---

KlassBox PRO est un système de gestion intégré dédié à la restauration rapide et au service traiteur. Il réunit l'enregistrement HACCP, l'impression d'étiquettes thermiques conformes aux normes sanitaires, la gestion automatique des stocks, la planification des réapprovisionnements, l'optimisation des tournées de livraison, la suite de communication omnicanale ainsi que l'automatisation autonome des réseaux sociaux.

| **ModuleComposants ClésTechnologies Utilisées**<br>  |                                                                                                  |                                                          |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------ | -------------------------------------------------------- |
| **PWA & Synchronisation Offline**                    | Service Worker, IndexedDB Sync Queue, Background Sync API                                        | JavaScript (ES6+), Workbox                               |
| **Traçabilité HACCP & Températures**                 | Relevés critiques, Alertes temps réel, Logs d'audit                                              | Express.js, PostgreSQL, WebSockets                       |
| **Étiquetage & Calcul Nutritionnel**                 | Générateur HTML 77x51mm, Déduction des allergènes, Calcul macro                                  | Express.js, Node.js HTML Renderer                        |
| **Gestion des Stocks & Réapprovisionnement**         | Déduction atomique par lot, Alertes seuil minimum, Auto-Génération de bons de commande           | PostgreSQL Transactions, Express.js, Socket.io           |
| **Optimisation des Tournées**                        | Clustering géographique, Respect des fenêtres horaires, Dispatching livreur                      | Node.js, OSRM API, PostgreSQL PostGIS                    |
| **Marketing & Communication Omnicanale**             | Capture automatique de commandes, Envois groupés (Email & WhatsApp), Base CRM unifiée            | Meta Cloud API (WhatsApp), SendGrid, Node.js, PostgreSQL |
| **Automation Social Media & IA**                     | Planning hebdomadaire, Génération de réels/posts, Validation & Publication auto (IG, FB, TikTok) | Meta Graph API, TikTok Content API, Node.js, Cron Queue  |

## 2. Moteur de Synchronisation Hors-Ligne (PWA)

---

Pour garantir un fonctionnement continu en cuisine ou lors des livraisons même en cas d'interruption réseau, KlassBox PRO s'appuie sur une architecture PWA avec stockage local dans IndexedDB et re-synchronisation prioritaire.

### 2.1 Logiciel Service Worker

// service-worker.js
const CACHE\_NAME = 'klassbox-v1';
const OFFLINE\_URLS = ['/', '/index.html', '/styles.css', '/app.js'];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE\_NAME).then((cache) => cache.addAll(OFFLINE\_URLS))
  );
});

self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-haccp-logs') {
    event.waitUntil(flushPendingHaccpLogs());
  }
});

async function flushPendingHaccpLogs() {
  const db = await openIndexedDB();
  const logs = await db.getAll('pending\_logs');
  for (const log of logs) {
    try {
      await fetch('/api/haccp/logs', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(log)
      });
      await db.delete('pending\_logs', log.id);
    } catch (err) {
      console.error('Échec de synchronisation:', err);
    }
  }
}

## 3. Module HACCP & Relevés de Température

---

Chaque prise de température (cuisson, refroidissement, chambre froide) est soumise à une validation automatique. Si une valeur sort de l'intervalle critique, une alerte est immédiatement générée et notifiée à l'équipe.

### 3.1 Schéma de Base de Données

CREATE TABLE haccp\_logs (
    id SERIAL PRIMARY KEY,
    batch\_id INT REFERENCES production\_batches(id),
    type VARCHAR(30) NOT NULL, -- 'cooking', 'cooling', 'storage'
    temperature\_c NUMERIC(4, 1) NOT NULL,
    is\_compliant BOOLEAN NOT NULL,
    operator\_id INT NOT NULL,
    recorded\_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT\_TIMESTAMP
);

## 4. Calcul Nutritionnel et Impression d'Étiquettes Thermiques

---

Le module génère un flux HTML/CSS au format exact 77x51mm compatible avec les imprimantes thermiques de cuisine. Il calcule la valeur énergétique et la teneur en macronutriments en fonction du poids net de la portion et fait ressortir dynamiquement les allergènes majeurs.

### 4.1 Contrôleur Express.js d'Étiquetage

// labelController.js
export const getThermalLabelHtml = async (req, res) => {
  const { batchId } = req.params;
  const batchQuery = \`
    SELECT b.batch\_number, b.production\_date, b.expiry\_date, r.name AS recipe\_name, r.portion\_weight\_g
    FROM production\_batches b
    JOIN recipes r ON b.recipe\_id = r.id
    WHERE b.id = $1;
  \`;
  const result = await pool.query(batchQuery, [batchId]);
  const batch = result.rows[0];

  const html = \`
    \<!DOCTYPE html>
    \<html>
    \<head>
      \<style>
        @page { size: 77mm 51mm; margin: 0; }
        body { font-family: Arial; font-size: 7pt; width: 77mm; height: 51mm; padding: 2mm; }
        .header { font-weight: bold; font-size: 9pt; border-bottom: 1pt solid #000; }
      \</style>
    \</head>
    \<body>
      \<div class="header">KlassBox PRO - ${batch.recipe\_name}\</div>
      \<div>Lot: ${batch.batch\_number} | DLC: ${batch.expiry\_date}\</div>
    \</body>
    \</html>
  \`;
  res.setHeader('Content-Type', 'text/html');
  res.send(html);
};

## 5. Gestion des Stocks, Alertes & Bons de Commande Automatiques

---

Lors de la validation d'un lot de production, les ingrédients requis sont immédiatement déduits du stock. Si le stock restant est inférieur au seuil de sécurité, une alerte WebSocket est diffusée et un bon de commande fournisseur au format BROUILLON est créé ou complété.

### 5.1 Transaction SQL de Déduction de Stock

BEGIN;

INSERT INTO production\_batches (batch\_number, recipe\_id, portions\_produced, expiry\_date, operator\_id)
VALUES ('LOT-20260912-01', 1, 50, CURRENT\_DATE + INTERVAL '4 days', 102);

UPDATE ingredients
SET current\_stock = current\_stock - (ri.quantity\_per\_portion \* 50),
    updated\_at = CURRENT\_TIMESTAMP
FROM recipe\_ingredients ri
WHERE ingredients.id = ri.ingredient\_id AND ri.recipe\_id = 1;

COMMIT;

## 6. Optimisation des Tournées de Livraison

---

L'algorithme de dispatching regroupe les commandes clients par secteur géographique tout en respectant les fenêtres horaires de livraison spécifiées lors de la commande.

### 6.1 Schéma des Tournées

CREATE TABLE delivery\_runs (
    id UUID PRIMARY KEY DEFAULT gen\_random\_uuid(),
    driver\_id UUID REFERENCES users(id),
    status VARCHAR(20) DEFAULT 'draft',
    departure\_time TIMESTAMPTZ,
    total\_distance\_km NUMERIC(6, 2)
);

CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen\_random\_uuid(),
    customer\_name VARCHAR(255) NOT NULL,
    address TEXT NOT NULL,
    latitude NUMERIC(10, 8) NOT NULL,
    longitude NUMERIC(11, 8) NOT NULL,
    delivery\_window\_end TIMESTAMPTZ NOT NULL,
    delivery\_run\_id UUID REFERENCES delivery\_runs(id),
    stop\_sequence INT
);

## 7. Module Marketing Commercial & Communication Omnicanale (WhatsApp / Email)

---

Ce module regroupe la gestion administrative et commerciale, la prise de commande automatisée multi-canal et l'envoi groupé de mailings et plats du jour en un seul clic.

### 7.1 Schéma de Base de Données CRM & Campagnes

\-- Base de données unifiée des clients
CREATE TABLE clients (
    id UUID PRIMARY KEY DEFAULT gen\_random\_uuid(),
    first\_name VARCHAR(100),
    last\_name VARCHAR(100),
    email VARCHAR(255) UNIQUE,
    phone\_number VARCHAR(30) UNIQUE,
    preferred\_channel VARCHAR(20) DEFAULT 'whatsapp', -- 'whatsapp', 'email', 'both'
    tags TEXT[], -- ex: ['corporate', 'daily\_menu\_subscriber', 'vip']
    opt\_in\_marketing BOOLEAN DEFAULT TRUE,
    created\_at TIMESTAMPTZ DEFAULT CURRENT\_TIMESTAMP
);

\-- Suivi des campagnes de diffusion
CREATE TABLE campaigns (
    id UUID PRIMARY KEY DEFAULT gen\_random\_uuid(),
    title VARCHAR(255) NOT NULL,
    campaign\_type VARCHAR(50) NOT NULL, -- 'daily\_special', 'promo', 'event'
    subject\_email VARCHAR(255),
    content\_text TEXT NOT NULL,
    media\_url TEXT, -- Lien vers l'image du plat du jour ou flyer
    target\_tags TEXT[],
    scheduled\_at TIMESTAMPTZ,
    status VARCHAR(20) DEFAULT 'draft', -- 'draft', 'scheduled', 'sent'
    sent\_count INT DEFAULT 0
);

## 8. Module d'Automation Réseaux Sociaux (Instagram, Facebook, TikTok)

---

Ce système prépare automatiquement la communication quotidienne sur les réseaux sociaux d'après les menus et événements prévus. Il génère les visuels, vidéos et textes, puis les soumet à la validation avant publication programmée.

### 8.1 Schéma de Base de Données Social Media

CREATE TYPE social\_platform AS ENUM ('instagram', 'facebook', 'tiktok', 'google\_my\_business');
CREATE TYPE content\_format AS ENUM ('post\_image', 'story', 'reel\_short', 'carousel');
CREATE TYPE post\_status AS ENUM ('draft\_generated', 'pending\_approval', 'scheduled', 'published', 'failed');

\-- Calendrier Éditorial Récurrent
CREATE TABLE editorial\_schedules (
    id UUID PRIMARY KEY DEFAULT gen\_random\_uuid(),
    day\_of\_week INT NOT NULL CHECK (day\_of\_week BETWEEN 1 AND 7),
    target\_time TIME NOT NULL,
    theme VARCHAR(100) NOT NULL,
    preferred\_format content\_format NOT NULL,
    platforms social\_platform[] NOT NULL,
    ai\_prompt\_template TEXT NOT NULL,
    is\_active BOOLEAN DEFAULT TRUE
);

\-- File d'Attente des Publications Générées
CREATE TABLE social\_posts (
    id UUID PRIMARY KEY DEFAULT gen\_random\_uuid(),
    schedule\_id UUID REFERENCES editorial\_schedules(id) ON DELETE SET NULL,
    recipe\_id INT REFERENCES recipes(id) ON DELETE SET NULL,
    title VARCHAR(255) NOT NULL,
    caption\_text TEXT NOT NULL,
    hashtags TEXT[],
    media\_urls TEXT[] NOT NULL,
    video\_script TEXT,
    status post\_status DEFAULT 'draft\_generated',
    scheduled\_for TIMESTAMPTZ NOT NULL,
    approved\_by INT REFERENCES users(id),
    created\_at TIMESTAMPTZ DEFAULT CURRENT\_TIMESTAMP
);

\-- Historique et Logs de Publication
CREATE TABLE publication\_logs (
    id UUID PRIMARY KEY DEFAULT gen\_random\_uuid(),
    post\_id UUID REFERENCES social\_posts(id) ON DELETE CASCADE,
    platform social\_platform NOT NULL,
    external\_post\_id VARCHAR(255),
    published\_at TIMESTAMPTZ,
    error\_message TEXT,
    reach\_count INT DEFAULT 0,
    engagement\_count INT DEFAULT 0
);

### 8.2 Service d'Exécution Quotidien et Publication Automatique

// socialAutomationService.js
import { pool } from '../config/db.js';
import { generateCaptionAndVisuals } from '../services/aiGenerator.js';
import { publishToMeta, publishToTikTok } from '../services/socialPublisher.js';

export const prepareDailySocialContent = async () => {
  const todayIndex = new Date().getDay() || 7;

  const schedules = await pool.query(
    \`SELECT \* FROM editorial\_schedules WHERE day\_of\_week = $1 AND is\_active = TRUE\`,
    [todayIndex]
  );

  const dailyRecipe = await pool.query(
    \`SELECT r.id, r.name, r.description FROM daily\_menus dm JOIN recipes r ON dm.recipe\_id = r.id WHERE dm.menu\_date = CURRENT\_DATE\`
  );
  const recipeInfo = dailyRecipe.rows[0];

  for (const schedule of schedules.rows) {
    const generatedContent = await generateCaptionAndVisuals({
      theme: schedule.theme,
      recipe: recipeInfo,
      promptTemplate: schedule.ai\_prompt\_template,
      format: schedule.preferred\_format
    });

    await pool.query(
      \`INSERT INTO social\_posts 
       (schedule\_id, recipe\_id, title, caption\_text, hashtags, media\_urls, video\_script, scheduled\_for, status)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8, 'pending\_approval')\`,
      [
        schedule.id,
        recipeInfo?.id || null,
        \`Post ${schedule.theme} - ${new Date().toLocaleDateString('fr-FR')}\`,
        generatedContent.caption,
        generatedContent.hashtags,
        generatedContent.mediaUrls,
        generatedContent.videoScript,
        \`${new Date().toISOString().split('T')[0]}T${schedule.target\_time}\`
      ]
    );
  }
};

## 9. Perspectives de Perfectionnement et Fonctionnalités Futures

---

**Pistes d'Amélioration & Modules Optionnels Recommandés :**

- **Suivi GPS & Notification SMS Client :** Envoi d'un lien de géolocalisation en direct pour le suivi du livreur et notification automatique 10 minutes avant l'arrivée.
- **Facturation & Comptabilité Intégrée :** Export comptable automatique des achats et ventes (format CSV/XLSX) et synchronisation directe avec vos outils de gestion.
- **Terminal de Paiement Embarqué :** Intégration de solutions de paiement sans contact (SoftPOS) sur l'application mobile des livreurs.
- **Planification Prédictive des Stocks :** Utilisation de l'historique des ventes pour estimer les besoins en matières premières d'une semaine sur l'autre et réduire le gaspillage.
- **Analyse de Sentiment & Réponses Auto aux Commentaires :** Traitement automatique des commentaires et messages privés sur Instagram/Facebook/TikTok pour répondre aux demandes d'information ou de réservation.