# Intégration des documents — état réel

Les deux documents fournis sont conservés intégralement dans l’application et dans ce dossier. Ils décrivent une architecture cible ; leur emploi du mot « développé » n'est pas une preuve de déploiement.

## Disponible dans ce HTML

- Manager central et huit pôles : tâches, brouillons marketing liés au catalogue, suivi RH, suggestions d’achat, rapports locaux, propositions de réponses aux avis saisis.
- Confirmations : préparation depuis les commandes éligibles, validation, export EML, suivi des tentatives. Contact et Reply-To : contact@klassbox.fr.
- Envoi transactionnel optionnel via une passerelle HTTPS à fournir. Désactivé initialement, limité aux nouvelles commandes après activation. Un accusé du service ne prouve pas la livraison dans la boîte du client.
- Documents techniques consultables, recherchables et téléchargeables.
- Réceptions et référencement des stocks existants sans double augmentation ; unités explicites.
- Fiches recettes versionnées liées aux plats et produits existants.
- Production saisie manuellement : contrôle des disponibilités, consommation des lots non périmés par date la plus proche, déduction du stock et journal. En cas d’échec de sauvegarde, restauration de la base en mémoire.
- Relevés par lot et opérateur ; comparaison aux seuils saisis, action corrective pour un écart. Sans seuil : à vérifier.
- Étiquette HTML 77 × 51 mm par lot : plat, lot, production, DLC, poids, allergènes et conservation renseignés. Impression depuis le navigateur, sans pilotage silencieux de Zebra.

## Non déployé, à connecter ou développer

| Élément des documents | État / besoin restant |
|---|---|
| Express / PostgreSQL / PostGIS | Aucun serveur ni schéma réel du dépôt klassbox-back disponible. Les exemples ne sont pas exécutés contre une base de production. |
| PWA / IndexedDB / Background Sync | Non activés dans le HTML ouvert comme fichier. Nécessitent hébergement, stratégie de cache, endpoint et authentification. |
| HACCP WebSockets / alertes multi-appareils | Local seulement ; pas de transmission serveur. |
| Déduction atomique multi-utilisateurs | La transaction locale ne remplace pas une transaction PostgreSQL avec verrouillage concurrent. |
| Nutrition calculée | Pas de données nutritionnelles fournisseur complètes ajoutées ; le poids ne permet pas de les inventer. |
| OSRM / optimisation avec créneaux | Les tournées existantes restent disponibles ; pas de solveur routier connecté. |
| WhatsApp / e-mail entrants | Pas de webhook fournisseur ni vérification de signature configurés. |
| Diffusion campagnes | Brouillons et validation locale ; pas de diffusion en masse configurée. |
| Génération IA de médias / publication sociale | Catalogue et textes préparatoires ; pas de génération vidéo ni API de publication connectées. |
| GPS / SMS / SoftPOS | Perspectives conservées ; non implémentées ici. |
| Prévisions financières | Estimation simple sur historique disponible, pas de modèle entraîné. |
| Permissions et sécurité multi-sites | À implémenter côté serveur ; une sélection locale de rôle ne sécurise pas les données. |

## Corrections indispensables au code serveur fourni

1. Créer les tables référencées avant leurs clés étrangères : users/sites/suppliers/clients/recipes avant lots, relevés et commandes.
2. Uniformiser les identifiants utilisateurs en fonction de la base réelle : UUID et INT ne sont pas interchangeables.
3. Ne pas créer une seconde table orders ou clients en production. Préparer des migrations additives après inspection du schéma réel et de ses données.
4. Unifier ingredients et raw_materials ; ajouter les tables daily_menus et les colonnes de description utilisées par les services.
5. Une commande reçue sans adresse ni géocodage reste en attente. Ne pas lui imposer immédiatement les champs NOT NULL nécessaires à une livraison confirmée.
6. Distinguer « message reçu, à valider » et « commande confirmée ». La détection par deux mots-clés ne constitue ni une compréhension fiable ni une transmission cuisine.
7. Exiger une identité d’événement fournisseur unique, vérifier sa signature et gérer les collisions téléphone/e-mail avant de créer un client.
8. Garder les relevés en attente sur échec réseau, HTTP non-2xx ou accusé invalide ; acquitter un identifiant idempotent confirmé par le serveur.
9. Isoler les données par site, dériver l’opérateur de l’utilisateur authentifié et calculer les règles de contrôle côté serveur, sans faire confiance à is_compliant reçu du navigateur.
10. Déduire les lots dans une transaction verrouillée, refuser stocks négatifs et lots périmés, tracer chaque consommation et empêcher un même lot de production d’être validé deux fois.
11. Garder l’opt-in marketing désactivé sans consentement enregistré ; gérer le désabonnement par canal.
12. Suivre chaque destinataire/canal séparément : accepté, échoué, résultat inconnu, livré. Ne pas compter tous les destinataires comme envoyés après une boucle partiellement échouée.
13. Paramétrer la version de l’API Meta et vérifier les modèles approuvés lors de la connexion. Ne pas figer l’ancienne URL copiée dans le document.
14. Planifier par fuseau du site et date locale ; dédupliquer chaque génération quotidienne, gérer l’absence de menu, exiger la validation avant publication.
15. Échapper les données d’étiquettes et gérer les lots inexistants. Ne pas remplacer allergènes inconnus par « Aucun », ni inventer DLC, température ou certification de conformité.

## Validation réalisée

Les parcours sont testés dans jsdom (DOM JavaScript simulé), avec e-mails simulés. Les résultats JSON sont inclus. Pas de test physique d’imprimante, pas de navigateur mobile réel, pas d’API fournisseur ni de PostgreSQL. L’affichage 77 × 51 doit être vérifié à 100 % sur le pilote et l’imprimante, notamment pour les textes longs.
