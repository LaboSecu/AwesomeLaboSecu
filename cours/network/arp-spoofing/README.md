## Principe de l'arp spoofing

L’ARP est un protocole qui, de par sa conception, expose les réseaux informatiques et leurs composants à des vulnérabilités et des dangers qui sont faciles à exploiter lorsque l’on connaît bien son fonctionnement. Nous allons ici étudier le fonctionnement des attaques utilisant le protocole ARP puis essayer de donner des pistes pour s’en protéger.

Nous allons ici voir en quoi l’ARP est exploitable sur les réseaux afin d’effectuer des attaques de type *MITM* (“*Man in the Middle*“) ainsi que des attaques *DOS* (“*Denial Of Service*“). 

### Le point faible du fonctionnement d’ARP et d’Ethernet

L’ARP est un protocole de couche 2 du modèle OSI et TCP/IP), lors des échanges sur un réseau, les adresses MAC sont utilisées pour la formation des trames Ethernet et le rôle de l’ARP est donc de fournir, à partir d’une adresse IP, l’adresse MAC correspondante. Par défaut, une carte réseau refuse les paquets reçus n’ayant pas son adresse MAC comme adresse MAC de destination. On pourrait comparer cela au fait de recevoir une lettre alors que le nom du destinataire n’est pas le notre, nos cartes réseaux, en individus sages et réfléchis, refusent donc d’ouvrir ces lettres.

Une trame Ethernet complète doit donc forcément avoir un couple IP-MAC correcte dans les champs destination pour transiter sans encombre d’un hôte A à un hôte B.
Cependant, le fonctionnement d’ARP sur les ordinateurs est le suivant  :  à chaque réception d’un paquet, la carte réseau va vérifier le couple IP-MAC qu’il contient et mettre à jour sa table ARP (aussi appelée cache ARP) si le couple trouvé n’est pas enregistré. Ceci dans le but de ne pas faire de requête ARP à chaque échange et de remplir son cache ARP de façon dynamique. Partant de ce principe, on peut très bien imaginer que si l’on a trois hôtes, un hôte A pourrait très bien informer un deuxième hôte B via une requête ARP qu’il dispose de l’adresse MAC de l’hôte C sans que cela soit vrai.
Le principe de *l’ARP* ou du *MAC spoofing* (spoofing voulant dire *“parodier”, “usurper”)* est d’**envoyer des informations à un système afin de lui faire enregistrer des informations qui ne sont pas les bonnes** **et qui usurpent l’identité (la relation IP-MAC) d’un autre système**. Partons du schéma suivant :

[![attaque-arp-spoof-protection-01](https://ogma-sec.fr/wp-content/uploads/2015/06/attaque-arp-spoof-protection-01.jpg)](https://ogma-sec.fr/wp-content/uploads/2015/06/attaque-arp-spoof-protection-01.jpg)Schéma basique d’un LAN relié par un switch de niveau 2

Ici, le Pirate possède l’adresse MAC `00:0C:19:AC:44:CA` et le serveur l’adresse MAC `00:0C:29:4A:A4:41`, la table adresse MAC chez le client ressemble donc à cela (obtenue avec la commande `arp -a` sous Windows comme sous Linux) :

```
gateway.local (192.168.19.2) at 00:05:56:e9:52:3b [ether] on eth0
pirate.local (192.168.19.130) at 00:0c:19:ac:44:ca [ether] on eth0
serveur.local (192.168.19.132) at 00:0c:29:4a:a4:41 [ether] on eth0
```

Table ARP du client avant attaque

On voit donc ici les associations IP-MAC correspondent à notre schéma. Si le pirate envoie des paquets au client avec l’adresse IP du source du serveur mais en laissant son adresse MAC (ce qui est faisable si l’on construit nos paquets nous même plutôt que si on laisse notre carte réseau le faire), la table ARP de notre client va donc enregistrer le couple suivant :

```
192.168.19.132 – 00:0C:19:AC:44:CA
```

La table ARP de notre client va donc ressembler à cela :

```
gateway.local (192.168.19.2) at 00:0c:19:ac:44:ca [ether] on eth0
pirate.local (192.168.19.130) at 00:0c:19:ac:44:ca [ether] on eth0
serveur.local (192.168.19.132) at 00:0c:19:ac:44:ca [ether] on eth0
```

Table ARP après attaque

Cela vient du fait que l’enregistrement est marqué comme dynamique, donc volatile, et qu’il peut être mis à jour à chaque réception de paquet présentant des couples IP – MAC différents. Les paquets que le client va générer à destination du serveur vont donc à présent se former avec l’adresse IP destination du serveur et avec l’adresse MAC destination du pirate étant donné qu’il se base pour cela sur sa table ARP et que celle-ci est falsifiée.

Pour ceux qui ont du mal à comprendre comment exploiter ce fonctionnement, il faut se pencher sur le fonctionnement des switchs de niveau 2 qui est assez basique. Quand un swtich reçoit un paquet, il va observer l’adresse MAC destination (une trame Ethernet contenant un champ adresse destination et un champ adresse source, toutes deux des adresses MAC), comparer cette adresse avec sa table *CAM* (*Content Adressable Memory*) qui contient la correspondance MAC – port sur lequel elle est affectée. Si la MAC a été enregistrée précédemment sur une de ses interfaces, il envoie le paquet sur cette interface, sinon il émet une requête ARP afin de voir sur quelle interface est l’adresse MAC renseignée. Il est important de comprendre ici qu’un switch de niveau 2 ne regardera absolument pas les données concernant l’IP (source et destination), mais uniquement la couche 2 (contenant les adresses MAC).

On voit donc que si la table ARP de notre cible est falsifiée, il va former ces trames avec l’adresse IP du serveur mais va en fin de compte les envoyer au pirate car il formera ses requêtes avec comme adresse MAC de destination celle du pirate. Il est important de bien visualiser ces différentes couches et leur impact sur le chemin des paquets au travers un switch pour comprendre les attaques par ARP spoofing.

### Mise en œuvre de l’attaque avec arpspoof

Passons à la pratique, si vous avez encore du mal à saisir le principe, la mise en pratique d’une attaque ARP va vous aider à mieux comprendre son fonctionnement et ses possibilités. On parle d’ARP spoofing ou de MAC spoofing quand il s’agit de falsifier la table ou le cache ARP d’un système. Pour information, l’anglais *“spoof”* ou *“spoofing”* se traduit *“parodier”* ou *“usurper”* en français.

#### Attaque Man in the Middle

L’objectif de l’attaquant va ici être de se faire discret en faisant tout pour ne pas soupçonner une attaque, l’attaquant va donc rediriger les paquets détournés afin que les utilisateurs naviguent et utilisent le réseau sans perturbation ou coupure, et avec le moins de ralentissement réseau possible. On obtient ce genre de comportement lorsque l’on cherche à capturer des informations transitant entre deux systèmes par exemple, l’ARP spoof va alors être une étape de l’attaque *MITM* (elle ne constitue pas l’intégralité de l’attaque). Une attaque Man in the Middle permet à l’attaquant de se positionner au milieu d’une conversation ou d’un échange réseau pour intercepter, lire, supprimer ou modifier tout ou partie de cette communication.

Dans notre attaque, nous suivons le même schéma que celui exposé un peu plus haut. Nous allons ici essayer de capturer une session FTP via ARP spoofing. Plus clairement, nous allons faire en sorte d’intercepter les communications du client, initialement destinées au serveur sur lequel nous avons installé un service FTP, mais nous allons également avoir pour soucis de faire suivre les paquets afin que le client ne se doute de rien et accède bien au serveur.

Sur notre poste pirate (KaliLinux), nous activons dans un premier temps le forwarding de paquet :

```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

Sans cette commande, les paquets ne pourront pas transiter à travers le système de notre pirate et ainsi atteindre leur cible originelle. Dans un deuxième temps, nous allons utiliser *“arpspoof”* pour envoyer continuellement des paquets ARP qui vont falsifier la table ARP de notre client. On va donc indiquer que l’IP du serveur a notre adresse MAC :

```
arpspoof -t 192.168.19.131 192.168.19.132
```

On peut traduire cette ligne de commande par “*fait moi passer pour 192.168.19.132 auprès de 192.168.19.131*“. Pour information, on va envoyer des paquets de façon constante car si une communication est effectuée du serveur vers le client, ce dernier va à nouveau mettre à jour sa table ARP et cette fois-ci avec les informations correctes. Il faut donc remettre continuellement à jour la table ARP de notre client pour qu’elle reste falsifiée. Voici la sortie que nous aurons une fois la commande lancée :[![arp-spoof-attaque-protection-04](https://ogma-sec.fr/wp-content/uploads/2015/06/arp-spoof-attaque-protection-04.png)](https://ogma-sec.fr/wp-content/uploads/2015/06/arp-spoof-attaque-protection-04.png)
On remarque l’envoi de paquet ARP reply indiquant que l’IP du serveur (`192.168.19.132`) a maintenant l’adresse MAC du pirate (`00:c:29:ac:44:ca`), il s’agit ici d’une donnée qui est fausse et falsifiée par l’action du pirate.

Nous voyons donc clairement que l’outil envoie des paquets ARP reply (je vous renvoie vers le tutoriel sur l’ARP pour plus de précision), ce qui va avoir pour effet de falsifier la table ARP de la cible. Il faut savoir que dans une utilisation normale d’ARP, les ARP reply interviennent en réponse à une requête ARP diffusée le plus souvent en broadcast par un autre hôte ou alors lorsqu’un hôte arrive sur un nouveau réseau. Si l’on observe ces paquets avec Wireshark :
[![arp-spoof-attaque-protection-02](https://ogma-sec.fr/wp-content/uploads/2015/06/arp-spoof-attaque-protection-02.png)](https://ogma-sec.fr/wp-content/uploads/2015/06/arp-spoof-attaque-protection-02.png)Nous voyons ici très clairement les paquets successivement envoyés par notre pirate. On voit que ce sont des *ARP reply* et, chose intéressante, on remarque que le *Sender MAC address* est bien l’adresse MAC de notre pirate (conformément à notre schéma) mais que le *Sender IP address* est celle de notre serveur, ce qui montre bien le fonctionnement de l’ARP spoofing. Pendant l’envoi de ces informations, voici à quoi va ressembler la table ARP de notre cible :

```
gateway.local (192.168.19.2) at 00:05:56:e9:52:3b [ether] on eth0
pirate.local (192.168.19.130) at 00:0c:19:ac:44:ca [ether] on eth0
serveur.local (192.168.19.132) at 00:0c:19:ac:44:ca [ether] on eth0
```

Table ARP après attaque

Nous allons maintenant poursuivre notre attaque MITM en effectuant une requête FTP de la part de notre client vers notre serveur. En toute logique, les paquets vont donc atterrir chez notre pirate car le client va utiliser sa table ARP (falsifiée), le switch va donc envoyer ces paquets à notre pirate car il fait correctement son travail (un peu bêtement d’ailleurs), pirate qui va faire suivre les paquets vers le serveur car il a l’IP forwarding d’activé. On procède donc à une écoute réseau sur notre pirate avec Wireshark durant la requête :
[![arp-spoof-attaque-protection-01](https://ogma-sec.fr/wp-content/uploads/2015/06/arp-spoof-attaque-protection-01.png)](https://ogma-sec.fr/wp-content/uploads/2015/06/arp-spoof-attaque-protection-01.png)
On voit donc que les identifiants FTP passent par notre pirate alors que, normalement ils ne doivent pas y transiter, mais si l’on observe l’entête Ethernet de ces paquets FTP :

[![arp-spoof-attaque-protection-03](https://ogma-sec.fr/wp-content/uploads/2015/06/arp-spoof-attaque-protection-03.png)](https://ogma-sec.fr/wp-content/uploads/2015/06/arp-spoof-attaque-protection-03.png)Entête Ethernet des paquets FTP

Nous voyons que la destination des paquets FTP est bien l’IP du serveur pour le niveau 3 mais qu’elle est la MAC de notre piratep pour le niveau 2, ce qui fait qu’un switch de niveau 2 va diriger les paquets vers notre pirate plutôt que vers le serveur. Voici ce qu’il se passe actuellement au niveau réseau :
![img](https://ogma-sec.fr/wp-content/uploads/2015/06/arp-spoof-attaque-protection-04.jpg)
**Etape 1 :** Notre pirate usurpe donc l’adresse MAC du serveur auprès du client. La table ARP du client étant falsifiée, les paquets qu’il enverra seront dans un premier temps envoyés à notre pirate.

**Étape 2 :** Notre pirate intercepte les paquets envoyés par le client, puis les rejoue auprès du serveur, nous sommes donc dans un contexte d’attaque par « L’homme du milieu » (Man in the middle).

**Étape 3 :** Le serveur va répondre aux requêtes du clients en lui envoyant directement les paquets.

Il faut noter que pour que l’attaque soit complète, il faudrait également effectuer une attaque ARPspoof pour le chemin de retour, usurpant ainsi l’identité (l’adresse MAC) du client auprès du serveur. On pourrait alors récupérer par exemple les fichiers récupérés par le client sur le serveur et les reconstituer directement dans Wireshark.

Nous venons de voir ensemble comment un pirate peu effectuer une attaque MITM grâce à l’ARP spoofing. Nous allons également voir que l’ARP Spoofing permet d’effectuer une attaque par déni de service (DOS). Nous verrons ensuite, et c’est le but de l’article, comment essayer de se protéger de ces attaques.

#### Attaque DOS via ARP Spoofing

L’objectif d’un attaquant est est ici de mettre à mal un service informatique ou une partie de celui-ci, que ce soit une plage IP, un ou plusieurs hôtes ciblés. Pour ce faire, il faut suivre également le même principe de l’attaque MITM sauf que nous allons désactiver l’IP forwarding. Ainsi, les paquets ne seront pas retransmis à leur cible originelle et tomberont donc dans le vide, rendant ainsi la connectivité de notre ou nos cibles impossible car aucun paquet n’arrivera à destination. Pour ce faire, l’attaquant peur se faire passer pour la passerelle auprès du client, dès qu’il voudra aller sur Internet ou sur un autre réseau, les paquets destinés à la passerelle arriveront sur le pirate qui ne les fera pas suivre.

Nous n’avons dans notre exemple ciblé qu’un seul hôte, cependant, nous pouvons également opter pour la diffusion des messages ARP falsifiant les tables ARP des cibles en broadcast, ce qui aura pour effet d’affecter toutes les machines se situant sur la même plage IP que nous. On appelle ce mode de fonctionnement de l’attaque ARP spoofing le *gratuitous ARP*. Par exemple :

```
aprspoof -t 192.168.19.255 192.168.19.2
```

Cette commande, en suivant notre schéma d’origine, va nous faire passer pour la passerelle sur tout le réseau et va donc rediriger l’intégralité du trafic du réseau vers notre poste pirate. Nous pourrons alors au choix en activant l’IP forwarding capturer le trafic et le faire suivre pour que les postes ne se doutent de rien ou alors désactiver l’IP forwarding pour causer une attaque par déni de service.

### ARPspoof, comment se sécuriser ?

Nous avons vu le concept de l’ARP spoofing, comment cette attaque pouvait être effectuée par un attaquant et quels pouvaient être les impacts de cette attaque au sein d’un système d’information. Il n’est pas dans mes habitudes de faire un article détaillant une attaque pour l’attaque elle-même, nous allons donc maintenant voir ensemble des outils et des pistes de sécurisation afin de se protéger contre l’ARP spoofing. Le fait est que pour bien se défendre, il faut comprendre en détail comment on est attaqué, voila chose faite !

#### L’enregistrement ARP statique

Une des techniques, la plus simple mais la plus lourde dans un contexte d’entreprise, est d’utiliser la fonction « enregistrement statique » des tables ARP dans les OS Windows et Linux. Nous avons vu que le remplissage et la modification de la table ARP dans les OS était dynamique. Nous pouvons toutefois, pour les éléments critiques du système d’information (Gateway, AD, web intranet par exemple) forger une adresse ARP de façon statique dans la table ARP des machines, ces enregistrements statiques ne seront ainsi plus falsifiables via des requêtes ARP malicieuses. Cela peut se faire avec l’option `-s` de la commande ARP :

```
arp -s adresseIP adresse MAC
```

Ici, l’option « *-s* » indique le renseignement de l’association IP-MAC en *“static”,* une attaque par ARP spoofing ne pourra plus la modifier.
Cette protection comporte bien évidemment quelques inconvénients dans le cadre d’une maintenance quotidienne d’un parc client, si le système enregistré de façon statique (par exemple, la passerelle) est modifié, il faut aller changer l’enregistrement statique sur tous les hôtes client du parc. Cela peut être le cas par exemple lors du changement d’interface réseau, d’un remplacement de matériel ou encore de l’activation d’un système de Load Balancing ou de FailOver. Dans un parc informatique, on peut alors s’aider d’un script de démarrage ou du déploiement fait via les GPO, cela reste tout de même contraignant au quotidien.

#### Bloquer les paquets « gratuitous ARP » : L’approche Symantec et SonicWall

Lors de mes recherches sur les techniques de protection, je suis arrivé à la lecture d’une solution appelée « *Anti-MAC spoofing* » mise en place dans certains produits de Symantec comme « *Symantec Sygate Enterprise Protection*« . Le principe est en fait de n’autoriser les réponses ARP (ARP Reply) à transiter sur le réseau uniquement si une requête ARP (ARP request) a été enregistrée précédemment. Également, seules les réponses en rapport avec la requêtes seront autorisées à transiter. Il ne sera alors pas possible, après une requête de type « *quelle est l’adresse MAC de 192.168.0.1* » d’accepter une réponse « L’adresse MAC de 192.168.0.17 est `aa:bb:cc:dd:ee:ff` ».

En des termes plus techniques, on bloque le « *gratuitous ARP* » qui est le fait d’envoyer des réponses ARP à des machines n’ayant rien demandé dans le but de fausser leur table ARP, ce qui est la base de l’ARP spoofing.

Plus d’informations dans la documentation Symantec :<https://www.symantec.com/security_response/glossary/define.jsp?letter=a&word=anti-mac-spoofing>
Dans le même esprit, on retrouve la solution « *MAC-IP anti Spoof* » de SonicWall. Outre une base centralisée saisie par l’administrateur, la base DHCP peut ici également être une base de référence.

Plus d’informations dans la documentation SonicWall : <http://help.mysonicwall.com/sw/eng/7630/ui2/70/Policies_Network_MAC-IPAnti-Spoof_Snwls.html>
En théorie, cela peut être une protection efficace. Je n’ai en revanche pas plus de détail sur l’infrastructure que cela requiert au niveau du déploiement réseau. Il faut en effet que tout le réseau soit couvert par cette protection pour ne pas laisser de zones vulnérables.

#### Détection par les IDS

De manière plus classique, et sans avoir à acheter de solution révolutionnaire, il est également possible de détecter des attaques ARP spoofing via des IDS. Il est pour cela nécessaire d’avoir un IDS en place avec un port mirroring au niveau du cœur de réseau. En analysant le trafic ARP, on pourra ainsi détecter des comportements anormaux comme l’ARP spoofing.

En effet, nous avons vu que pour que l’ARP spoofing soit un succès, il faut flooder (envoyer continuellement) des paquets ARP-reply à notre ou nos cibles. Ce comportement ne s’observe pas dans un contexte normal de communication entre des machines. L’analyse d’un grand nombre de paquets ARP-reply envoyés peut donc être faite par des IDS qui enverrons alors une alerte aux équipes de sécurité.

C’est une configuration que je n’ai encore jamais mise en place, en connaissant le fonctionnement d’un IDS en port mirroring et de l’attaque ARP spoofing, je pense que l’on peut néanmoins s’attendre à des détections d’attaque par ARP spoofing via cette méthode, qui a le mérite de pouvoir être mise en place avec des logiciels libres !

#### DAI : Dynamic ARP Protection

Le DAI, ou « *Dynamic ARP Protection* » est une solution notamment employée dans les éléments actifs Cisco et HP dans le but de protéger le réseau des attaques par ARP spoofing. Le principe est différent de celui employé par Symantec. En effet, le DAI va, pour chaque réponse ARP envoyée depuis un port qui n’est pas de confiance, comparer les données qu’elle contient à une base de données de confiance pré-enregistrée dans le réseau et dropper les paquets ARP-reply contenant de fausses informations. Cela évite alors en principe la falsification d’une correspondance MAC – IP au sein d’un réseau et ainsi les attaques MITM via ARP spoofing.

Cette technique de protection est intéressante car on peut alors maintenir une table ARP statique centralisée qui sert donc de référence à toutes les réponses ARP.
Plus d’informations à ce sujet dans la documentation Cisco : <http://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst6500/ios/12-2SX/configuration/guide/book/dynarp.html#wp1082194>

Dans cet article, nous avons étudié le fonctionnement, l’exécution et les impacts des attaques Man in the Middle par ARP spoofing. Nous avons également étudié et vu plusieurs pistes de protection contre ces attaques. Il faut savoir que les attaques par l’homme du milieu sont la base de beaucoup d’autres attaques réseau comme SslStrip, DNS spoofing, etc. Il est donc crucial de sécuriser cette base dans un système d’information sensible.

### Référence

- https://ogma-sec.fr/attaque-man-in-the-middle-via-arp-spoofing/
