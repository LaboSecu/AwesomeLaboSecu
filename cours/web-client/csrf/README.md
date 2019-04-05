# Cross-Site Request Forgery (CSRF)

##  Attaque

### Contexte

L'administrateur d'un site web est connecté à celui-ci par un système de sessions.
Un membre de ce même site web veut supprimer un des messages du site web. Mais il n'a pas les droits nécessaires avec son compte pour le supprimer.

### Exploit

Le membre arrive à connaître le lien qui permet de supprimer le message en question.
Le membre envoie un message avec un lien vers une page web contenant une balise image avec le lien de suppression du message à la place du lien vers un image.

Exemple de balise image malicieuse :

```html
<img src="http://website.com/api.php?action=delete&target=msg-42" />
```

L'administrateur va ouvrir le lien envoyé par le membre. Le navigateur de l'administrateur va vouloir appeler l'image, ce qui va envoyer l'odre au site web de supprimer le message avec les droits de l'administrateur, sans que celui-ci ne se rend compte de rien.

### Diagramme de séquence

```sequence
Admin -> Website : Authenticate on website
Member -> Admin : Send link of page with malicious HTML image
Admin -> Admin : Open link
Admin -> Website : Call hidden request
Website -> Website : Delete meesage with Victim's credentials
```

## Protection

- Demander des confirmations à l'utilisateur pour les actions critiques, au risque d'alourdir l'enchaînement des formulaires.
- Demander une confirmation de l'ancien mot de passe à l'utilisateur pour changer celui-ci ou changer l'adresse mail du compte.
- Utiliser des jetons de validité (ou *Token*)  dans les formulaires via le chiffrement d'un identifiant utilisateur, un nonce et un horodatage. Le  serveur doit vérifier la correspondance du jeton envoyé en recalculant  cette valeur et en la comparant avec celle reçue.
- Éviter d'utiliser des requêtes HTTP GET pour effectuer des actions :  cette technique va naturellement éliminer des attaques simples basées  sur les images, mais laissera passer les attaques fondées sur  JavaScript, lesquelles sont capables très simplement de lancer des  requêtes HTTP POST.
- Effectuer une vérification du référent dans les pages sensibles : connaître la provenance du client permet de  sécuriser ce genre d'attaques. Ceci consiste à bloquer la requête du  client si la valeur de son référent est différente de la page d'où il  doit théoriquement provenir.

## Référence

- https://en.wikipedia.org/wiki/Cross-site_request_forgery