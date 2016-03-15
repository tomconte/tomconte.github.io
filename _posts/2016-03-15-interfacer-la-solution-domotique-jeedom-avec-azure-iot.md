---
layout: post
date: 2016-03-15
title: Interfacer la solution domotique Jeedom avec Azure IoT
---

Dans cet article je vais vous détailler plusieurs façons de relier votre installation Jeedom à Azure pour y exploiter vos données de domotique.

[Jeedom](https://www.jeedom.com/site/fr/) est un logiciel Open Source d'origine française, qui vous permet de créer votre propre centre de contrôle domotique. Le logiciel peut s'installer sur un Raspberry Pi (ce que je vais utiliser), mais vous pouvez aussi acheter du matériel Jeedom plus performant si vous le souhaitez. Par l'intermédiaire d'un système de plugins disponible sur une place de marché, Jeedom peut s'interfacer avec de nombreux matériels et capteurs de type Z-Wave, RFXCom, EnOcean, etc. 

Jeedom vous permet de contrôler et de visualiser vos appareils domotiques via une console Web, et il peut également historiser les valeurs reçues des capteurs dans une base MySQL locale. L'idée ici est de complémenter cette collecte standard en envoyant également des données dans [Azure IoT Hub](https://azure.microsoft.com/en-us/services/iot-hub/), ce qui pourra par la suite nous permettre d'utiliser les [autres services Azure IoT](https://azure.microsoft.com/en-us/solutions/iot-suite/) comme Stream Analytics ou Machine Learning pour traiter ces données dans le Cloud.

Commencez par suivre la [documentation Jeedom](https://www.jeedom.com/doc/documentation/installation/fr_FR/doc-installation.html) pour installer l'application sur votre Raspberry Pi ou sur une machine virtuelle. J'ai tendance à suivre les [étapes manuelles](https://www.jeedom.com/doc/documentation/installation/fr_FR/doc-installation.html#_autre), mais la version Docker fonctionne très bien aussi. 

## Configuration de Jeedom

Une fois Jeedom installé, l'on va y ajouter quelques objets histoire d'avoir un peu d'activité.

Aller dans la Gestion des Plugins, puis Accéder au Market. Installer le plugin Météo qui est simple à utiliser. N'oubliez pas de l'activer, puis aller dans la configuration du plugin et ajoutez une ville. Activez bien ce nouvel équipement, et vous pouvez le rendre visible pour qu'il s'affiche sur l'écran d'accueil.

Un autre plugin simple à utiliser pour faire des tests est le plugin Networks, que vous trouverez également dans le Market. Vous pouvez vous en servir pour mesure la latence et le status d'un équipement réseau ou d'un service Internet. Vous pouvez par exemple le configurer pour aller pinger le DNS Google (adresse IP 8.8.8.8), et utiliser le champ Auto-Actualisation pour mettre l'intervalle à une minute.

Maintenant, pour pouvoir regarder et capter tout ce qu'il se passe dans notre petite installation domotique en temps réel, il va nous falloir activer les logs de type "évènement". C'est sur ces logs temps réel que l'on va pouvoir se brancher pour collecter nos données dans Azure.

Aller dans Configuration puis Configuration des Logs & Messages, et dans le Niveau de Log, choisir "Info" et sauvegarder. Vous devriez maintenant voir des évènements arriver dans le log en temps réel, dans le menu Analyse.

## Installer ou mettre à jour Node.JS

Nous allons utiliser des utilitaires et librairies Node.JS pour nous connecter à IoT Hub, il nous faut donc installer les pré-requis nécessaires. En fonction de la version de Raspbian que vous utilisez, vous aurez soit une vieille version de Node.JS (celle provenant des dépôts Debian), soit pas de Node.JS du tout. Si il est déjà installé, retirez-le avec la commande suivante:

~~~
sudo apt-get remove nodejs nodejs-legacy
~~~

Il peut y avoir d'autres paquets qui dépendent de Node.JS (comme [Node-RED](http://nodered.org/docs/hardware/raspberrypi.html)), vous pourrez toujours les réinstaller par la suite.

Puis nous allons installer la dernière version disponible via le projet [node-arm](https://node-arm.herokuapp.com/), ce qui est à ce jour la façon la plus simple de procéder:

~~~
wget http://node-arm.herokuapp.com/node_latest_armhf.deb
sudo dpkg -i node_latest_armhf.deb
~~~

Vous voici avec un beau Node.JS tout neuf.

## Configuration de Azure IoT Hub

Nous allons utiliser le service [Azure IoT Hub](https://azure.microsoft.com/en-us/services/iot-hub/), qui est la porte d'entrée vers les autres service de traitements de données disponibles dans la plateforme. IoT Hub est un service clés en main qui permet d'établir une communication bi-directionnelle sécurisée entre vos appareils connectés et le Cloud. Dans notre cas, nous allons traiter notre boîte Jeedom comme un appareil qui serait muni d'un grand nombre de capteurs, pour IoT Hub cela correspondra donc à une Device. L'on peut imaginer dans un déploiement de Jeedom à grande échelle, par exemple dans un projet résidentiel, que chaque boîte Jeedom aurait sa propre identité dans IoT Hub.

Vous pouvez suivre [le paragraphe "Créer un Hub IoT" de la documentation Azure](https://azure.microsoft.com/fr-fr/documentation/articles/iot-hub-node-node-getstarted/#crer-un-hub-iot) pour créer votre service depuis le portail Web.

Une fois arrivé à la section "Création d’une identité d’appareil", nous allons par contre utiliser une autre méthode que d'écrire directement du code: nous allons utiliser l'utilitaire [iothub-explorer](https://github.com/Azure/azure-iot-sdks/tree/master/tools/iothub-explorer) qui va nous permettre de créer une Device en ligne de commande. Vous pouvez l'installer via `npm`:

~~~
sudo npm install -g iothub-explorer
~~~

Vous allez ensuite utiliser la chaîne de connexion IoT Hub, récupérée dans le portail à l'étape précédente, pour ouvrir une session:

~~~
iothub-explorer login "HostName=monhub.azure-devices.net;SharedAccessKeyName=iothubowner;SharedAccessKey=xxxxC9uLxxxxvFmtxxxxocnAxxxxtD00xxxxGLcXxxxx="
~~~

Et vous allez maintenant pouvoir créer une Device correspondant à votre boîte Jeedom:

~~~
iothub-explorer create jeedompi --connection-string
~~~

Vous pouvez bien entendu utiliser l'identifiant de votre choix à la place de "jeedompi". L'outil va vous renvoyer toutes les informations relatives à votre nouvel appareil, y compris ses clés de sécurité:

~~~
Created device jeedompi

-
  deviceId:                   jeedompi
  generationId:               635935562110536966
  etag:                       MA==
  connectionState:            Disconnected
  status:                     enabled
  statusReason:               null
  connectionStateUpdatedTime: 0001-01-01T00:00:00
  statusUpdatedTime:          0001-01-01T00:00:00
  lastActivityTime:           0001-01-01T00:00:00
  cloudToDeviceMessageCount:  0
  authentication:
    SymmetricKey:
      primaryKey:   xxxxRs0dxxxxxUwfxxxxqbpPxxxxW5yIxxxx13Wlxxx=
      secondaryKey: b0CVxxxxE8qTxxxxdOOhxxxxvfKfxxxxRUgyxxxxAmA=
-
  connectionString: HostName=monhub.azure-devices.net;DeviceId=jeedompi;SharedAccessKey=xxxx/SugxxxxC6yJxxxx32WRxxxxHKmuxxxxRvQ8xxx=
~~~

L'information qui va nous intéresser est la dernière, à savoir la chaîne de connexion de l'appareil (qu'il ne faut pas confondre avec la chaîne de connexion à l'IoT Hub). Dans une installation avec de multiples appareils, chacun d'entre eux aurait sa propre chaîne de connexion afin de sécuriser l'envoi de données.

## Création d'un script Node.JS pour dialoguer avec Azure IoT Hub

Nous allons maintenant créer un petit script Node.JS qui prendra une valeur en paramètre et enverra la donnée sous forme JSON à IoT Hub. Pourquoi JSON? Car c'est le format le plus universellement reconnu par les services que l'on pourrait vouloir utiliser en aval du flux de données, comme par exemple Azure Stream Analytics pour réaliser des analyses en temps réel.

~~~
mkdir iothub-scripts
cd iothub-scripts
npm init -y .
npm install azure-iot-device azure-iot-device-http --save
~~~

Comme vous le voyez, j'ai décidé ici d'utiliser le transport HTTP plutôt que les autres transports disponibles comme AMQP ou MQTT. Deux raisons à cela: la première est que ma boîte Jeedom était originellement derrière un pare-feu qui ne laissait passer que le HTTP et le HTTPS. L'autre étant quand dans notre premier exemple, nous allons communiquer en mode "intermittent", c'est-à-dire envoyer un petit paquet de données toute les minutes ou plus, ce qui ne nécessite pas forcément de garder une connexion établie en permanence. Nous verrons plus bas l'utilisation d'un transport AMQP.

Voici notre premier script, que j'ai nommé `send_one.js`:

~~~ javascript
var Http = require('azure-iot-device-http');
var Message = require('azure-iot-device').Message;

var connectionString = 'HostName=monhub.azure-devices.net;DeviceId=jeedompi;SharedAccessKey=xxxx';

var client = Http.clientFromConnectionString(connectionString);

var value = process.argv[2];
var data = JSON.stringify({ deviceId: 'jeedompi', value: value });
var message = new Message(data);

client.sendEvent(message, function(err, res) {
  if (err) {
    console.log(err.toString());
  } else {
    console.log(res.transportObj.statusCode);
  }
  process.exit();
});
~~~

Vous pouvez tester tout de suite ce script depuis la ligne de commande:

~~~
$ node send_one.js 42
204
~~~

Le résultat `204` est le code de retour HTTP qui signifie que la requête a bien été prise en compte. 

## Utilisation du plugin Script

Comment appeler notre script depuis Jeedom? Nous pouvons utiliser le bien nommé plugin officiel Script, présent également dans le Market. Lorsque vous l'activez, regardez la page de configuration: elle vous indique le "Chemin des scripts utilisateur", i.e. l'emplacement sur le disque où seront stockés vos scripts. Pour créer un nouveau script, allez sur le plugin Script via le menu Plugins, puis ajoutez un équipement. Donnons-lui le nom "Latence Google DNS" par exemple. Activez-le, mais inutile de le rendre visible étant donné qu'il ne nous servira qu'à envoyer des données.

Cliquez sur "Ajouter une commande script", puis dans la colonne Requête, cliquez sur Nouveau pour créer un nouveau script. J'ai nommé le mien "send_latence.sh". Puis dans la boîte texte qui apparaît, entrez le petit script Shell suivant:

~~~ sh
#!/bin/sh

node /home/pi/iothub-scripts/send_one.js $*
~~~

Bien entendu, changez le chemin de votre script Node.JS si nécessaire. Toujours dans la table de création du script, entrez un nom dans la colonne Nom, par exemple "send_latence".

Maintenant, si vous regardez la colonne Requête, vous verrez l'appel à votre script; il nous reste à passer une valeur intéressante en paramètre ! La [documentation du plugin Script](https://www.jeedom.com/doc/documentation/plugins/script/fr_FR/script) nous indique la marche à suivre: nous pouvons référencer les valeurs que nous voyons dans le log d'évènements dans les paramètres du script. Par exemple si vous voyez les lignes suivantes dans le log temps réel:

~~~
[2016-03-14 15:00:04][event][INFO] : Evènement sur la commande [Domicile][Paris][Température] valeur : 7
[2016-03-14 15:02:02][event][INFO] : Evènement sur la commande [Domicile][Google DNS][Latence] valeur : 20.9
~~~

Cela veut dire que vous pouvez utiliser `#[Domicile][Paris][Température]#` ou `#[Domicile][Google DNS][Latence]#` en paramètre de votre script pour passer la valeur correspondate. Au final votre Requête devrait donc ressembler à ceci:

~~~
/var/www/html/core/php/../../plugins/script/core/ressources/send_latence.sh #[Domicile][Google DNS][Latence]#
~~~

Cliquez donc sur Sauvegarder en bas de page pour sauver toute la configuration.

Vous pouvez alors cliquer sur le petit bouton Tester tout à droite du tableau, et Jeedom devrait donc vous afficher le code de retour `204`, signe que la donnée a bien été envoyée. Il ne reste plus qu'à configurer l'auto-actualisation pour le script en choisissant une exécution répétitive toutes les minutes, par exemple. N'oubliez pas de sauvegarder !

Votre Jeedom va maintenant transmettre la valeur à Azure IoT Hub toutes les minutes. Vous pouvez utiliser `iothub-explorer` pour lire ces valeurs et vérifier qu'elles sont bien transmises pour notre device:

~~~
$ iothub-explorer  "HostName=monhub.azure-devices.net;SharedAccessKeyName=iothubowner;SharedAccessKey=xxx" monitor-events jeedompi

Monitoring events from device jeedompi
Listening on endpoint tcontehub/ConsumerGroups/$Default/Partitions/0 start time: 1457965294655
Listening on endpoint tcontehub/ConsumerGroups/$Default/Partitions/1 start time: 1457965294655
Listening on endpoint tcontehub/ConsumerGroups/$Default/Partitions/2 start time: 1457965294655
Listening on endpoint tcontehub/ConsumerGroups/$Default/Partitions/3 start time: 1457965294655

Event received:
{ deviceId: 'jeedompi', value: '114' }
Event received:
{ deviceId: 'jeedompi', value: '21' }
~~~

Notez qu'à l'heure actuelle, vous devrez donner votre chaîne de connexion IoT Hub à la commande `monitor-events`. N'hésitez pas à utiliser le bouton Tester pour envoyer des données!

## Exploitation du log d'évènements

Avec le plugin Script, vous pouvez donc programmer des envois de données, jusqu'à une fois par minute, ou même déclencher un envoi sur évènement, via un Scénario par exemple.

Maintenant, imaginons que vous ayez beaucoup de capteurs et autres équipements reliés à votre Jeedom, et que vous vouliez envoyer toutes les données dans Azure IoT Hub pour les filtrer et les traiter tranquillement dans le Cloud: il serait bien fastidieux de créer autant de scripts que de source possibles de données! Un moyen simple de capter toutes les données collectées par Jeedom est de se brancher directement sur le log d'évènements qui est écrit sur disque; essayez par exemple cette commande:

~~~
tail -f /var/www/html/log/event
~~~

Vous verrez tous vos évènements défiler en temps réel. Il est donc trivial de passer ce log à un autre script Node.JS qui va se charger de le parser, d'extraire les informations et de tout envoyer dans IoT Hub.

Cette fois-ci, notre script tournant en permanence, cela a définitivement un intérêt d'utiliser AMQP: nous allons établir une connection au démarrage du script, puis la maintenir ouverte et la réutiliser à chaque fois que nous aurons des données à envoyer. Cela sera beaucoup plus efficace que de créer et d'envoyer une nouvelle requête HTTP à chaque fois.

Voici donc notre nouveau script, que j'ai nommé `stream_events.js`:

~~~ javascript
var Amqp = require('azure-iot-device-amqp');
var Message = require('azure-iot-device').Message;

var connectionString = 'HostName=tcontehub.azure-devices.net;DeviceId=jeedompi;SharedAccessKey=zbDRRs0dDiInxUwfDnfpqbpP6rjHW5yIBPkX13Wl1gE=';

var client = Amqp.clientFromConnectionString(connectionString);

client.open(function(err) {
  if (err) {
    console.err('Could not connect: ' + err.message);
  } else {

    var lineReader = require('readline').createInterface({
      input: process.stdin
    });

    lineReader.on('line', function (line) {
      if (!/sur la commande/.test(line)) return;

      var re = /commande \[(.+)\]\[(.+)\]\[(.+)\] valeur : (.+)/;
      var m = re.exec(line);

      var locationId = m[1];
      var sensorId = m[2];
      var valueType = m[3];
      var value = m[4];

      var data = JSON.stringify({ deviceId: 'jeedompi', locationId: locationId, sensorId: sensorId, timestamp: new Date(), valuetype: valueType, value: value });

      console.log(data);

      var message = new Message(data);

      client.sendEvent(message, function(err, res) {
        if (err) {
          console.log(err.toString());
        } else {
          console.log(res.transportObj.statusCode);
        }
      });
    });
  }
});
~~~

Avant de le lancer, ajoutez le module Azure IoT AMQP:

~~~
npm install azure-iot-device-amqp --save
~~~

Puis lancez-le comme suit:

~~~
tail -f /var/www/html/log/event | node ./stream_events.js
~~~

La plus grande partie du script est consacrée au parsing de la ligne émise par Jeedom à l'aide d'une expression régulière. Nous extrayons les détails de l'évènement et les transmettons sous forme de trois propriétés dans le JSON: `locationId`, `sensorId` et `valuetype`, plus bien entendu la valeur de l'évènement dans `value`. Avec cette méthode, tous vos évènements seront automatiquement envoyés à IoT Hub. Voici un petit exemple du JSON produit: 

~~~
{
	"deviceId": "jeedompi",
	"locationId": "Domicile",
	"sensorId": "Google DNS",
	"timestamp": "2016-03-14T15:00:09.747Z",
	"valuetype": "Latence",
	"value": "23.1"
}
{
	"deviceId": "jeedompi",
	"locationId": "Domicile",
	"sensorId": "Paris",
	"timestamp": "2016-03-14T15:00:09.895Z",
	"valuetype": "Température",
	"value": "8"
}
~~~

Cette fois aussi, vous pouvez utiliser `iothub-explorer monitor-events` pour visualiser les données au fur et à mesure qu'elle sont reçues par IoT Hub.

## Conclusion

Nous avons vu plusieurs façons de brancher la collecte de données de Jeedom dans Azure, sans avoir à modifier le code de Jeedom ni écrire de code bien complexe. Dans un futur billet j'explorerai des méthodes plus radicales, comme ajouter du code PHP dans Jeedom pour envoyer directement des données à IoT Hub, sans avoir à utiliser de script externe Node.JS.
