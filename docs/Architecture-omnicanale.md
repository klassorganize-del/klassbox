# KlassBox PRO — Architecture & Documentation Technique Complète

---

Ce document récapitule la feuille de route technique, l'architecture logicielle, les schémas de base de données, ainsi que la logique métier de l'ensemble des modules développés pour la plateforme **KlassBox PRO**.

## 1. Aperçu Général de la Plateforme

---

KlassBox PRO est un système de gestion intégré dédié à la restauration rapide et au service traiteur. Il réunit l'enregistrement HACCP, l'impression d'étiquettes thermiques conformes aux normes sanitaires, la gestion automatique des stocks, la planification des réapprovisionnements, l'optimisation des tournées de livraison et la suite de communication omnicanale & marketing.

| **ModuleComposants ClésTechnologies Utilisées**<br>  |                                                                                        |                                                          |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| **PWA & Synchronisation Offline**                    | Service Worker, IndexedDB Sync Queue, Background Sync API                              | JavaScript (ES6+), Workbox                               |
| **Traçabilité HACCP & Températures**                 | Relevés critiques, Alertes temps réel, Logs d'audit                                    | Express.js, PostgreSQL, WebSockets                       |
| **Étiquetage & Calcul Nutritionnel**                 | Générateur HTML 77x51mm, Déduction des allergènes, Calcul macro                        | Express.js, Node.js HTML Renderer                        |
| **Gestion des Stocks & Réapprovisionnement**         | Déduction atomique par lot, Alertes seuil minimum, Auto-Génération de bons de commande | PostgreSQL Transactions, Express.js, Socket.io           |
| **Optimisation des Tournées**                        | Clustering géographique, Respect des fenêtres horaires, Dispatching livreur            | Node.js, OSRM API, PostgreSQL PostGIS                    |
| **Marketing & Communication Omnicanale**             | Capture automatique de commandes, Envois groupés (Email & WhatsApp), Base CRM unifiée  | Meta Cloud API (WhatsApp), SendGrid, Node.js, PostgreSQL |

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

### 7.2 Service de Capture Automatique des Commandes (Webhook Ingestion)

// orderIngestionService.js
import { pool } from '../config/db.js';

export const handleIncomingMessage = async ({ senderPhone, senderEmail, textBody, source }) => {
  // 1. Identification ou création du client
  let clientResult = await pool.query(
    \`SELECT id, first\_name FROM clients WHERE phone\_number = $1 OR email = $2\`,
    [senderPhone, senderEmail]
  );

  let client = clientResult.rows[0];
  if (!client) {
    const newClient = await pool.query(
      \`INSERT INTO clients (phone\_number, email, preferred\_channel) 
       VALUES ($1, $2, $3) RETURNING id\`,
      [senderPhone, senderEmail, source]
    );
    client = newClient.rows[0];
  }

  // 2. Traitement du message & intégration dans KlassBox PRO
  const isOrder = textBody.toLowerCase().includes('commande') || textBody.toLowerCase().includes('plat du jour');

  if (isOrder) {
    const newOrder = await pool.query(
      \`INSERT INTO orders (customer\_id, raw\_order\_text, source, status) 
       VALUES ($1, $2, $3, 'pending\_validation') RETURNING id\`,
      [client.id, textBody, source]
    );

    // 3. Réponse automatique de confirmation
    await sendAutoReply(client, source, "Votre commande a bien été reçue et transmise à la cuisine !");
    return newOrder.rows[0];
  }
};

### 7.3 Service de Diffusion 1-Clic "Plat du Jour" (WhatsApp & Mail)

// broadcastService.js
import axios from 'axios';

export const sendDailySpecialBroadcast = async ({ campaignId, title, message, photoUrl, targetTag }) => {
  // Récupération des abonnés
  const recipients = await pool.query(
    \`SELECT email, phone\_number, preferred\_channel 
     FROM clients 
     WHERE opt\_in\_marketing = TRUE AND $1 = ANY(tags)\`,
    [targetTag]
  );

  for (const client of recipients.rows) {
    // Envoi via WhatsApp Cloud API
    if (client.preferred\_channel === 'whatsapp' || client.preferred\_channel === 'both') {
      await axios.post(
        \`https\://graph.facebook.com/v18.0/${process.env.WA\_PHONE\_NUMBER\_ID}/messages\`,
        {
          messaging\_product: 'whatsapp',
          to: client.phone\_number,
          type: 'template',
          template: {
            name: 'daily\_special\_alert',
            language: { code: 'fr' },
            components: [
              { type: 'header', parameters: [{ type: 'image', image: { link: photoUrl } }] },
              { type: 'body', parameters: [{ type: 'text', text: message }] }
            ]
          }
        },
        { headers: { Authorization: \`Bearer ${process.env.WA\_BEARER\_TOKEN}\` } }
      );
    }

    // Envoi par E-mail (SendGrid / Mailgun)
    if (client.preferred\_channel === 'email' || client.preferred\_channel === 'both') {
      await sendEmailNotification({
        to: client.email,
        subject: \`Plat du jour : ${title}\`,
        htmlContent: \`\<p>${message}\</p>\<img src="${photoUrl}" style="max-width:100%;"/>\`
      });
    }
  }

  await pool.query(\`UPDATE campaigns SET status = 'sent', sent\_count = $1 WHERE id = $2\`, [recipients.rowCount, campaignId]);
};

## 8. Perspectives de Perfectionnement et Fonctionnalités Futures

---

**Pistes d'Amélioration & Modules Optionnels Recommandés :**

- **Suivi GPS & Notification SMS Client :** Envoi d'un lien de géolocalisation en direct pour le suivi du livreur et notification automatique 10 minutes avant l'arrivée.
- **Facturation & Comptabilité Intégrée :** Export comptable automatique des achats et ventes (format CSV/XLSX) et synchronisation directe avec vos outils de gestion.
- **Terminal de Paiement Embarqué :** Intégration de solutions de paiement sans contact (SoftPOS) sur l'application mobile des livreurs.
- **Planification Prédictive des Stocks :** Utilisation de l'historique des ventes pour estimer les besoins en matières premières d'une semaine sur l'autre et réduire le gaspillage.