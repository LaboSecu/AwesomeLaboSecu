# Cross-Site Scripting (XSS)



<div style="margin:auto; width:50%; display:flex; flex-direction:column;">
    <textarea placeholder="enter your message"></textarea>
    <button>
        Post
    </button>
</div>

```sequence
Client -> Server : Post input data
Server -> Server : Parse input data
Server -> Storage : Store input
Storage -> Server : Access stored input
Server -> Client : Redirect to another page
```



Payload : 

```html
<img src="malicious.site.com"/>
```



<div style="margin:auto; width:50%; display:flex; flex-direction:column;">
    <textarea placeholder="enter your message">
		XSS Test<img src="malicious.site.com"/></textarea>
    <button>
        Post
    </button>
</div>

```sequence
Client -> Server : Post *Malicious* input data
Server -> Server : Parse input data
Server -> Storage : Store input
Storage -> Server : Access stored input
Server -> Client : Redirect to another page
Client -> Client : Render Malicious input
Client -> Outside : Implicit Request Sended with client cookies
```





## Attaque



## Remediation

Pour remedier aux failles XSS au sein de son application, il est necessaire de verifier et transformer **TOUTES** les entrees utilisateurs avant de les stocker en base de donnees.

#### Encoding

Le code HTML, Javascript et characteres speciaux doivent etre encodes

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

Avec ca, le navigateur ne devrait pas interpreter le code HTML et l'affichera donc comme du texte.

#### Liste noire des balises HTML 

Parfois, l'on veut permettre a l'utilisateur d'utiliser du HTML pour ajouter une signature a son profil sur un forum par exemple. 
La saisie devra etre interpreter par le navigateur web, nous ne pouvons donc pas utiliser l'encoding qui rend le code html inutilisable.

Il faut mettre en place un systeme de listes noire/blanche pour certaines balises HTML que nous souhaitons autoriser et effectuer des verifications sur les differentes balises pour etre certain qu'aucun code malicieux n'as ete insere.

Ces differents check sont effectues par un outil appele HTML sanatizer 



## Protection

En tant qu'utilisateur, il est necessaire de pouvoir se proteger contre des failles XSS deja exploites. 
Lors de l'arrive sur un site 



## Reference

https://en.wikipedia.org/wiki/Cross-site_scripting

https://en.wikipedia.org/wiki/HTML_sanitization