# Transmission à Yavuz — KlassBox HTML et intégration serveur

## Ce dossier

Travail effectué sur le HTML autonome fourni par Emre. Le dépôt klassbox-back et le projet Vercel de production n’ont pas été consultés ni modifiés. Il ne faut pas remplacer le site ou sa base par ce dossier.

L’archive contient l’application complète, sa référence antérieure, les extensions séparées, les documents fournis et les tests. L’utilisateur peut continuer à travailler en local ; ses données de navigateur ne sont pas automatiquement incluses dans cette archive. Une sauvegarde JSON récente doit être exportée séparément pour préparer leur migration.

## Fichiers à examiner

| Fichier | Rôle |
|---|---|
| KlassBox_Manager.html | Application reconstruite pour utilisation locale |
| sources/base-reference.html | Base conservée avec modules historiques |
| manager.js | Manager, pôles métier et confirmations |
| operations.js | Lots, recettes, productions, relevés et documents |
| status.js | État documenté des fonctions et liens vers les modules |
| build.py | Reconstruction sans écraser la référence |
| documents/Passerelle-confirmations.md | Contrat de la passerelle e-mail, non déployée |
| documents/Integration-et-limites.md | Fonctions présentes, limites et erreurs des exemples serveur |
| tests-*.json et audit-routes.json | Résultats de simulation, pas une recette de production |

## Données locales ajoutées

Clé historique localStorage : klassbox5_db. Les extensions conservent les objets existants. db.centralManager contient tâches administratives, planning, rapports et confirmations ; db.operationsDocs contient recettes, lots, consommations et relevés. KlassGrowth utilise db.klassGrowth. Certaines tâches sont partagées par les pôles via cette dernière structure.

Les identifiants locaux sont des chaînes et ne doivent pas être convertis implicitement en identifiants PostgreSQL. Prévoir une table de correspondance, un import à blanc et un contrôle des quantités et montants avant migration. Les données médicales/allergènes et coordonnées clients ne doivent pas être publiées avec le site statique.

## Ordre de raccordement

1. Examiner les schémas, l’authentification, les établissements et les autorisations existants du serveur.
2. Définir la source de vérité des clients/commandes ; sauvegarder et importer sans doublons, avec gestion explicite des conflits.
3. Brancher les confirmations et webhooks du site avec idempotence ; distinguer reçu, validé, payé et livré.
4. Relier WhatsApp/e-mail entrants ; vérifier les signatures, rapprocher les clients et garder les commandes ambiguës à valider.
5. Relier les comptes marketing et sociaux, avec approbation, consentement et suivi par destinataire/canal.
6. Relier Stripe, statistiques et cartographie ; tester les échecs et les reprises avant activation.
7. Ajouter une file hors connexion côté client et son acquittement serveur. Le HTML local n’assure pas de synchronisation multi-utilisateurs.

## Points à tester avec le matériel et les comptes réels

- Deux appareils modifient la même commande sans perte silencieuse.
- Un webhook répété n’ajoute pas une deuxième commande ni un deuxième encaissement.
- Une confirmation acceptée n’est pas renvoyée après une panne ambiguë.
- Un livreur n’accède qu’aux données autorisées pour sa tournée/site.
- Une production concurrente ne consomme pas deux fois le même stock.
- Une feuille A4 et les étiquettes Zebra vont aux bonnes imprimantes ; une impression demandée n’est pas déclarée physiquement réussie sans confirmation.
- Les comptes, montants, soldes et historiques concordent avant/après migration.

Aucun secret de connexion n’est à transmettre dans ce document. Configurer les accès et variables dans les outils d’hébergement prévus à cet effet. Les rapports locaux et règles ne constituent pas des agents IA autonomes ni une certification HACCP.
