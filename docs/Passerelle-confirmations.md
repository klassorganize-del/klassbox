# Contrat de connexion des confirmations

État : client HTML implémenté et testé avec réponse simulée. Serveur à fournir ; aucun prestataire réel configuré.

L’URL HTTPS se configure dans Manager & assistants → Paramètres. Le navigateur transmet un POST JSON avec :

```json
{
  "type": "order_confirmation",
  "idempotencyKey": "klassbox:identifiant-commande:revision",
  "orderId": "identifiant-commande",
  "to": "adresse-client",
  "from": "contact@klassbox.fr",
  "replyTo": "contact@klassbox.fr",
  "subject": "Confirmation de votre commande KlassBox",
  "text": "Détail de la commande"
}
```

En-tête : `Idempotency-Key` identique au champ JSON.

Réponse attendue après acceptation du prestataire :

```json
{"accepted": true, "messageId": "reference-prestataire"}
```

Le serveur doit authentifier l’utilisateur, autoriser le site et la commande, vérifier le destinataire et le contenu depuis les données autorisées, limiter les demandes et conserver les clés idempotentes. Ne jamais transformer cette route en relais d’e-mail public acceptant n’importe quel destinataire. Les secrets du prestataire restent côté serveur. L’interface locale ne fournit pas cette authentification.

Un résultat incertain ne doit pas déclencher un nouvel envoi aveugle : consulter le prestataire et reprendre avec la même clé. La réception effective nécessite les événements de livraison du prestataire ; le HTML indique seulement « Acceptée par service ».

L’expéditeur contact@klassbox.fr doit être configuré auprès du prestataire. Une URL saisie ne configure pas le domaine d’envoi.

Les messages marketing ne suivent pas ce circuit : validation de campagne, consentement par canal et désabonnement doivent être traités séparément.
