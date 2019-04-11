## Le DNS Spoofing
### Qu'est-ce que c'est ?

L'objectif de cette attaque est de rediriger, à leur insu, des Internautes vers des sites pirates. Pour la mener à bien, le pirate utilise des faiblesses du protocole DNS (Domain Name System) et/ou de son implémentation au travers des serveurs de nom de domaine. A titre de rappel, le protocole DNS met en oeuvre les mécanismes permettant de faire la correspondance entre une adresse IP et un nom de machine (ex.: www.truc.com). Il existe deux principales attaques de type DNS Spoofing : le DNS ID Spoofing et le DNS Cache Poisoning. Concrètement, le but du pirate est de faire correspondre l'adresse IP d'une machine qu'il contrôle à un nom réel et valide d'une machine publique. 
Description de l'attaque


### DNS ID Spoofing


Si une machine A veut communiquer avec une machine B, la machine A a obligatoirement besoin de l'adresse IP de la machine B. Cependant, il se peut que A possède uniquement le nom de B. Dans ce cas, A va utiliser le protocole DNS pour obtenir l'adresse IP de B à partir de son nom.
Une requête DNS est alors envoyée à un serveur DNS, déclaré au niveau de A, demandant la résolution du nom de B en son adresse IP. Pour identifier cette requête une numéro d'identification (en fait un champs de l'en-tête du protocole DNS) lui est assigné. Ainsi, le serveur DNS enverra la réponse à cette requête avec le même numéro d'identification. L'attaque va donc consister à récupérer ce numéro d'identification (en sniffant, quand l'attaque est effectuée sur le même réseau physique, ou en utilisant une faille des systèmes d'exploitation ou des serveurs DNS qui rendent prédictibles ces numéros) pour pouvoir envoyer une réponse falsifiée avant le serveur DNS. Ainsi, la machine A utilisera, sans le savoir, l'adresse IP du pirate et non celle de la machine B initialement destinatrice. Le schéma ci-dessous illustre simplement le principe du DNS ID Spoofing.



### DNS Cache Poisoning


Les serveurs DNS possèdent un cache permettant de garder pendant un certain temps la correspondance entre un nom de machine et son adresse IP. En effet, un serveur DNS n'a les correspondances que pour les machines du domaine sur lequel il a autorité. Pour les autres machines, il contacte le serveur DNS ayant autorité sur le domaine auquel appartiennent ces machines. Ces réponses, pour éviter de sans cesse les redemander aux différents serveurs DNS, seront gardées dans ce cache. Le DNS Cache Poisoning consiste à corrompre ce cache avec de fausses informations. Pour cela le pirate doit avoir sous son contrôle un nom de domaine (par exemple fourbe.com) et le serveur DNS ayant autorité sur celui-ci ns.fourbe.com. L'attaque se déroule en plusieurs étapes :
Le pirate envoie une requête vers le serveur DNS cible demandant la résolution du nom d'une machine du domaine fourbe.com (ex.: www.fourbe.com)
Le serveur DNS cible relaie cette requête à ns.fourbe.com (puisque c'est lui qui a autorité sur le domaine fourbe.com)
Le serveur DNS du pirate (modifié pour l'occasion) enverra alors, en plus de la réponse, des enregistrements additionnels (dans lesquels se trouvent les informations falsifiées à savoir un nom de machine publique associé à une adresse IP du pirate)
Les enregistrements additionnels sont alors mis dans le cache du serveur DNS cible
Une machine faisant une requête sur le serveur DNS cible demandant la résolution d'un des noms corrompus aura pour réponse une adresse IP autre que l'adresse IP réelle associée à cette machine.

### Comment s'en protéger ?

Mettre à jour les serveurs DNS (pour éviter la prédictibilité des numéros d'identification et les failles permettant de prendre le contrôle du serveur)
Configurer le serveur DNS pour qu'il ne résolve directement que les noms des machines du domaine sur lequel il a autorité
Limiter le cache et vérifier qu'il ne garde pas les enregistrements additionnels.
Ne pas baser de systèmes d'authentifications par le nom de domaine : Cela n'est pas fiable du tout.