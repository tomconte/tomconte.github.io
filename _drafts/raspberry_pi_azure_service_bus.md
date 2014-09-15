---
layout: post
date: 2014-07-15
title: Connecter votre Raspberry Pi au Cloud grace au Windows Azure Service Bus
---

# Connecter votre Raspberry Pi au Cloud grâce au Windows Azure Service Bus

Votre Raspberry Pi peut rendre de grands services en tant que contrôleur ou intermédiaire pour connecter divers senseurs au Cloud. Voici un exemple utilisant le Windows Azure Service Bus et la librairie [Apache Qpid](http://qpid.apache.org/) "Proton" pour connecter votre Pi au Cloud afin d'y envoyer très facilement des données collectées via un senseur, dans notre exemple une carte Arduino Esplora.

![Arduino Esplora relié au Raspberry Pi](/images/azure_pi/EsploraPI.JPG)

## Le Raspberry Pi et l'Internet des Choses

Connecter des senseurs au Cloud, voici quelque chose devenu monnaie courante : pèse-personnes, accessoires fitness divers et variés, ... Tous ces appareils utilisent souvent du Bluetooth, mais également parfois une connexion Wi-Fi pour envoyer directement leurs données dans un référentiel applicatif permettant d'offrir des services à l'utilisateur. Dans notre approche, nous allons utiliser le Pi pour implémenter une architecture similaire: notre senseur (à base d'Arduino) n'utilisera pas de connexion réseau directement, mais plutôt un port série, ce qui va considérablement alléger sa tâche et le besoin de le connecter au réseau (protocoles Wi-Fi, DHCP, etc.) Les informations seront transmises à un Raspberry Pi via USB, et c'est lui qui se chargera d'envoyer les informations à notre application hébergée dans le Cloud. Le Pi disposant nativement d'un OS Linux capable de se connecter au réseau via Wi-Fi, nous allons très facilement pouvoir nous connecter à l'extérieur afin de transmettre des messages.

## AMQP

Reste à choisir un protocole de communication qui permettra d'interagir le plus efficacement possible avec notre application Cloud: nous avons besoin d'un protocole simple, léger, mais néanmoins sécurisé et prenant en considération notre environnement réseau qui peut être légèrement aléatoire, même dans un environnement fixe à domicile.

Le Service Bus Azure nous propose d'utiliser le protocole [AMQP](http://www.amqp.org/), pour Advanced Message Queuing Protocol. Il s'agit d'un protocole d'échange de messages [standardisé par le consortium OASIS](https://www.oasis-open.org/news/pr/iso-and-iec-approve-oasis-amqp-advanced-message-queuing-protocol), et qui a pour but de pouvoir couvrir de très nombreux cas d'usage, depuis les échanges inter-bancaires jusqu'aux objets connectés, en passant bien entendu par le Cloud et les architectures hybrides.

L'on trouvera de nombreuses librairies AMQP pour divers environnements d'exécution, et nous pourrons sans problème utiliser AMQP depuis le Raspberry Pi. Mon choix s'est porté sur la librairie Apache Qpid "Proton" qui semble être la plus interopérable et qui est également très compacte car conçue pour être déployée sur des systèmes embarqués.

Il nous faut donc commencer par l'installer sur le Raspberry Pi. Vous trouverez un lien pour télécharger le code source sur la page [Qpid Proton](http://qpid.apache.org/proton/). La compilation et l'installation sont simples, les commandes ci-dessous devraient faire l'affaire. Pour que les bindings Python soient compilés, il vous faudra préalablement installer les packages [SWIG](http://www.swig.org/) et `python-dev`.

	sudo apt-get install swig python-dev
	wget http://apache.mirrors.multidist.eu/qpid/proton/0.7/qpid-proton-0.7.tar.gz
	tar xzf qpid-proton-0.7.tar.gz
	cd qpid-proton-0.7/
	mkdir build
	cd build
	cmake ..
	make
	sudo make install

Vous devriez maintenant avoir la librairie AMQP installée et prête à utiliser depuis Python.

## Configurer Azure Service Bus

Avant de pouvoir envoyer des données, il va bien sûr que je commence par créer le service qui va les recevoir. Nous allons utiliser la fonction Rubriques du Service Bus Azure, qui est tout-à-fait adaptée à la réception d'informations de type télémétrie en provenance d'objets connectés:

- Les Rubriqes ("Topics" en anglais) peuvent être assimilées à des files d'attentes, dans lesquels les objets connectés peuvent envoyer des messages.
- Chaque Rubrique va pouvoir conserver les messages pendant un certain temps (configurable), ce qui vous permet de les lire à votre rythme.
- Les Rubriques offrent des fonctions avancées, comme la détection de doublons.
- La lecture des messages se fait en établissant des Abonnements sur la Rubrique.
- Un Abonnement peut optionnellement spécifier un filtre, ce qui vous permet de ne recevoir qu'une partie des messages.

Le Service Bus va donc servir de frontal hautement disponible et va vous permettre d'absorber le flot de données de télémétrie et servir de "tampon" par exemple en cas de pic de charge, ce qui va vous permettre de traiter les données à votre propre rythme en les lisant via un ou plusieurs abonnements.

Pour créer votre Rubrique, allez dans le portail Azure puis sélectionnez le modules Service Bus. Cliquez le bouton "Créer" en bas en nommez votre nouvel espace de noms: cela deviendra l'URL auquel nous nous connecterons pour envoyer les données. Une fois l'espace de noms créé, cliquez dessus pour accéder à sa page d'administration, puis cliquez sur "Rubriques". Le bouton "créer une rubrique" vous tend les bras, choisissez "création rapide", ce qui vous permet de créer une Rubrique en lui donnant simplement un nom, par exemple "telemetrie.

Nous avons presque tout ce qu'il nous faut, il nous manque les informations de connexion: retournez sur le tableau de bord de l'espace de nommage Service Bus, et cliquez sur "informations de connexion" en bas: le dialogue qui s'affiche vous donnera l'"émetteur par défaut", ou login (généralement "owner") ainsi que sa clé d'accès. Ces deux informations vous seront nécessaires pour configurer la connexion plus bas.

## La carte Arduino Esplora

Passons maintenant au côté client!

Il existe de nombreux type de cartes de prototypage dans le catalogue Arduino. La carte [Esplora](http://arduino.cc/en/Main/arduinoBoardEsplora) est un peu spéciale pour plusieurs raisons: tout d'abord elle a la forme d'une manette de console de jeux, elle est donc très adaptée pour créer des contrôleurs spécifiques qui pourront être branchés en USB sur un ordinateur classique. Par ailleurs, elle est équipée de tout une sélection de senseurs intégrés, ce qui permet de commencer à lire des valeurs immédiatement, sans même avoir à faire de branchements. 

La carte Esplora étant plus riche que les cartes standard type Uno, elle est accompagnée d'une [librairie spécifique](http://arduino.cc/en/Guide/ArduinoEsploraExamples) exposant toutes les fonctions nécessaires pour accéder à ses divers senseurs.

Nous allons donc commencer par écrire un petit programme pour l'Esplora qui va utiliser différents capteurs (lumière, son, température) et transmettre leurs valeurs au Raspberry Pi. La communication se fera tout simplement par le biais des ports USB: la carte Esplora sera connectée au Pi, ce qui permettra de l'alimenter et de lire les valeurs sur le port série. Il nous suffit d'écrire les valeurs sous forme de texte, et l'on pourra les extraire facilement coté Raspberry Pi.

<script src="https://gist.github.com/tomconte/ab041e5af66d8d6d5942.js">
</script>

## Lecture et envoi des valeurs

Côté Raspberry Pi, je vais utiliser un petit script en Python pour lire les valeurs sur le port série puis les envoyer au Service Bus Azure en AMQP.

Vous aurez peut-être à changer le port série à utiliser (`/dev/ttyACM0` dans l'exemple), et bien entendu il vous faudra entrer l'adresse complète de votre Rubrique Service Bus dans la variable `address`, incluant l'utilisateur ("owner" par défaut), sa clé, l'adresse de votre Service Bus, et le nom de la rubrique (ici, "esplora").

<script src="https://gist.github.com/tomconte/441eee047b3f26810b58.js">
</script>

Le script n'a rien de bien mystérieux, l'API Proton étant plutôt simple. La majeure partie code concerne l'assemblage des propriétés du message AMQP. Tout d'abord j'utilise une propriété `did` (comme "device ID") pour identifier mon client; je l'initiatise avec `uuid.getnode()`, qui renvoie un identifiant unique basé sur l'adresse .MAC.

	message.properties[symbol("did")] = symbol(id)

Puis je vais créer un ensemble de propriétés reprenant les valeurs émises par la carte Arduino. Si vous regardez le code Arduino, les lignes émises ressemblent à ceci:

	T:123_M:456_L:789

Je veux donc ajouter à mon message trois propriétés T, M et L reprenant ces valeurs. Les trois lignes suivantes se chargent de ce travail:

	pairs = map(lambda x:x.split(':'), temp.split('_'))
	symbols = map(lambda x:(symbol(x[0]),int(x[1])), pairs)
	message.properties.update(dict(symbols))

Je commence par séparer les valeurs en paires en les splittant; j'utilise la fonction `map` qui est pratique pour appliquer une même opération (sous forme de `lambda`) à tous les éléments d'un tableau. Puis un autre `map` permet de transformer toutes les valeurs en `symbol`, et finalement un `dict` pour transformer les paires en dictionnaire.

Une fois l'Arduino Esplora branchée au Raspberry Pi, vous pouvez lancer le script Python et vous devriez voir le compteur de données s'incrémenter dans le portail.
