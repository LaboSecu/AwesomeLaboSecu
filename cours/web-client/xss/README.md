# Cross-Site Scripting (XSS)



## Attaque

#### Exploit

Contexte : 
Le serveur délivre une page HTML simple pour envoyer un message. Elle contient une zone de texte et un bouton pour envoyer le message.

<div style="margin:auto; width:50%; display:flex; flex-direction:column;">
    <h5>Envoyer un message</h5>
    <textarea placeholder="enter your message"></textarea>
    <button>
        Send
    </button>
</div>

Ceci est schema simplifié, d'une communication client / serveur normale


```sequence
Alice -> Server : Post input data
Server -> Server : Parse input data
Server -> Storage : Store input
Storage -> Server : Access stored input
Server -> Bob : Redirect to another page
```



Maintenant un example d'attaque XSS 

avec le Payload suivant : 

```html
<img src="malicious.site.com"/>
```



<div style="margin:auto; width:50%; display:flex; flex-direction:column;">
    <textarea placeholder="enter your message">XSS Test<img src="malicious.site.com"/>
    </textarea>
    <button>
        Post
    </button>
</div>


```sequence
Alice -> Server : Post *Malicious* input data
Server -> Server : Parse input data
Server -> Storage : Store input
Storage -> Server : Access stored input
Server -> Bob : Redirect to another page
Bob -> Bob : Render Malicious input
Bob -> Outside : Implicit Request Sended with client cookies
```



Ici Alice envoit dans son message une balise HTML image.
En temps normale la balise image contient un lien vers la source de l'image. Ici, son usage est détourné et contient un lien vers un site malicieux.
Le message est par la suite envoyée sur le serveur, et stocké sur celui ci. 
Puis est reçu par Bob. 
A la réception du message, le moteur de rendu HTML contenu dans son navigateur va tenter de charger l'image et effectuer une requête à son insu vers le site malicieux d'Alice.

**Bob à été victime d'une attaque XSS ! **

Cette attaque est plutôt limité mais peut permettre à un attaquant d'aller beaucoup plus loin, en injectant des scripts javascript qui peuvent remplir automatiquement des formulaires avec les informations de session de la victime par exemple.

Ici l'attaque exploite une vulnérabilité du serveur qui ne sécurise aucune entrée utilisateur. 

## Remédiation

Pour remédier aux failles XSS au sein de son application, il est nécessaire de verifier et transformer **TOUTES** les entrées utilisateurs avant de les stocker en base de données.

#### Encoding

Le code HTML, Javascript et charactères spéciaux doivent être encodés

Example :

Ce code HTML malicieux est passe dans un formulaire

```html
<div>
    <img src="attacker.site.com"/>
</div>
```

Le serveur devrait le transformer en :

```
&ltdiv&gt
	&ltimg src="attacker.site.com"/&gt
&lt/div&gt
```

Avec ca, le navigateur ne devrait pas interprêter le code HTML et l'affichera donc comme du texte.

#### Liste noire des balises HTML 

Parfois, l'on veut permettre à l'utilisateur d'utiliser du HTML pour ajouter une signature a son profil sur un forum par exemple. 
La saisie devra être interpreter par le navigateur web, nous ne pouvons donc pas utiliser l'encoding qui rend le code html inutilisable.

Il faut mettre en place un systeme de listes noire/blanche pour certaines balises HTML que nous souhaitons autoriser et effectuer des vérifications sur les différentes balises pour être certain qu'aucun code malicieux n'as être inséré.

Ces differents check sont effectués par un outil appelé HTML sanatizer 



## Protection

En tant qu'utilisateur, il est necessaire de pouvoir se proteger contre des failles XSS deja exploites. 
Malheuresement peu de solutions existent pour se protéger d'attaques XSS. 
A savoir : 

- désactiver Javascript dans le navigateur 
- bloquer les requêtes effectuées sur des domaines différents de celui sur lequel on navigue.

**Note** : Bloquer le Javascript n'est pas une très bonne solution, non pas parce que celà ne fonctionne pas mais car sa désactivation, empêche l'accès à certains sites, particulièrement ceux utilisant des technologies modernes comme React et vuejs qui utilise Javascript pour rendre la page HTML

## Reference

https://en.wikipedia.org/wiki/Cross-site_scripting

https://en.wikipedia.org/wiki/HTML_sanitization