---
title: HAProxy sous Debian 10
publishDate: 2019-10-02 00:00:00
img: /assets/HAproxy.png
img_alt: Soft pink and baby blue water ripples together in a subtle texture.
description: |
  HAProxy sous Debian 10
tags:
 - Disponibilité
---

<h2> Présentation </h2>
<p>
HAProxy est un service de répartition d’un ensemble de tâches sur un ensemble de ressources, dans le but d’en rendre le traitement global plus efficace. Par exemple, cela permet de répartir la charge d’un seul applicatif http sur plusieurs serveurs http tout en étant transparent pour le client. En cas de défaillance de l’un, l’autre prendra entièrement la charge sur lui. On parle de Haute Disponibilité.
</p>
<p>
On fait référence à Frontend comme la partie accessible au public, et au Backend comme l'ensemble des serveurs qui se partagent la charge.
</p>
<p>
Ce type d'infrastructure est également très bénéfique pour le déploiement lors de la phase de production, en remplaçant progressivement les serveurs par la nouvelle application, une transition qui est bien sûr totalement invisible pour le client. Nous prévoyons de décharger un serveur pour effectuer une mise à jour de l'application, puis de le remettre en service une fois la mise à jour effectuée. Si un souci de mise à jour survient, si l'application présente une erreur ou si un problème quelconque se manifeste, la production ne s'arrête pas et nous pouvons prendre le temps nécessaire pour analyser et remédier à la situation. Cette tâche sera réalisée progressivement jusqu'à la mise à jour intégrale du backend.
</p>
<div>
<table>
  <thead>
    <tr>
      <th>Machine</th>
      <th>OS</th>
      <th>Distribution</th>
      <th>Version</th>
      <th>Rôle</th>
      <th>Nom d’hôte</th>
      <th>IP</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Machine Virtuelle Virtual Box</td>
      <td>GNU / Linux</td>
      <td>Debian</td>
      <td>10.5</td>
      <td>Serveur Load Balancer</td>
      <td>lb01</td>
      <td>Frontend 172.16.200.10  Backend 10.0.0.1</td>
    </tr>
    <tr>
      <td>Machine Virtuelle Virtual Box</td>
      <td>GNU / Linux</td>
      <td>Debian</td>
      <td>10.5</td>
      <td>Serveur Web Backend</td>
      <td>web01</td>
      <td>10.10.10.2</td>
    </tr>
    <tr>
      <td>Machine Virtuelle Virtual Box</td>
      <td>GNU / Linux</td>
      <td>Debian</td>
      <td>10.5</td>
      <td>Serveur Web Backend</td>
      <td>web02</td>
      <td>10.10.10.3</td>
    </tr>
    <tr>
      <td>Dell Latitude 3500</td>
      <td>Windows</td>
      <td>10 Entreprise</td>
      <td>1903</td>
      <td>Client Web</td>
      <td>L019-163</td>
      <td>172.16.1.68</td>
    </tr>
  </tbody>
</table>
</div>
<h2>
Préalable réseau et installation
</h2>
<section>
  <h3>
  Configuration du réseau sur Virtual Box
  </h3>
  <p>
  Pour assurer un fonctionnement optimal de notre backend, toutes les machines backend devront être connectées au même réseau. Dans le cas où VirtualBox est utilisé, comme c'est le cas ici, il faudra définir dans la configuration réseau de nos machines le mode d'accès en réseau interne et lui attribuer un nom, que nous appellerons backend.
  </p>
  <p>
  On va ajouter une 2ème carte réseau au serveur HAProxy qui sera dans le réseau interne backend, on fera de même pour les serveurs webs. Comme ceci : 
  </p>
  <br></br>
  <img src="/public/assets/photo_1-HAproxy.jpg" alt="Configuration réseau VBox" width="804" height="578" loading="lazy" decoding="async">
  <br></br>
  <p>
  Afin d'établir l'infrastructure du réseau, deux cartes réseau sont nécessaires pour le serveur HAProxy, étant donné que nous prévoyons de séparer notre infrastructure entre le frontend qui se verra attribuer l'adresse <em>172.16.200.10 </em> et le backend qui se verra attribuer l'adresse <em>10.0.0.1</em> Nous allons installer une seconde carte réseau.
  </p>
  <div class="mdx-ap mb-6 rounded-xl bg-[#212121] flex relative items-center overflow-hidden">
    <div class="bg-gray-700 h-full px-4 py-2 text-black whitespace-nowrap">$</div>
    <code class="px-2 whitespace-nowrap overflow-x-auto"><strong>nano /etc/network/interfaces</strong></code>
  </div>
  <pre class="astro-code github-dark" style="background-color:#24292e;overflow-x:auto" tabindex="0"><code><span class="line"><span style="color:#6A737D"># The loopback network interface  </span></span>
  <span class="line"><span style="color:#E1E4E8">auto lo  </span></span>
  <span class="line"><span style="color:#E1E4E8">iface lo inet loopback  </span></span>
  <span class="line"></span>
  <span class="line"><span style="color:#6A737D"># Frontend  </span></span>
  <span class="line"><span style="color:#E1E4E8">allow-hotplug enp0s3  </span></span>
  <span class="line"><span style="color:#E1E4E8">auto enp0s3  </span></span>
  <span class="line"><span style="color:#E1E4E8">iface enp0s3 inet static  </span></span>
  <span class="line"><span style="color:#E1E4E8">address 172.16.200.10/16  </span></span>
  <span class="line"><span style="color:#E1E4E8">gateway 172.16.0.1  </span></span>
  <span class="line"></span>
  <span class="line"><span style="color:#6A737D"># Backend  </span></span>
  <span class="line"><span style="color:#E1E4E8">allow-hotplug enp0s8  </span></span>
  <span class="line"><span style="color:#E1E4E8">auto enp0s8  </span></span>
  <span class="line"><span style="color:#E1E4E8">iface enp0s8 inet static  </span></span>
  <span class="line"><span style="color:#E1E4E8">address 10.0.0.1/8</span></span></code></pre>
  <p>On peut installer HAProxy :</p>
  <div class="mdx-ap mb-6 rounded-xl bg-[#212121] flex relative items-center overflow-hidden">
    <div class="bg-gray-700 h-full px-4 py-2 text-black whitespace-nowrap">$</div>
    <code class="px-2 whitespace-nowrap overflow-x-auto"><strong>apt update</strong></code>
  </div>
  <div class="mdx-ap mb-6 rounded-xl bg-[#212121] flex relative items-center overflow-hidden">
    <div class="bg-gray-700 h-full px-4 py-2 text-black whitespace-nowrap">$</div>
    <code class="px-2 whitespace-nowrap overflow-x-auto"><strong>apt install haproxy</strong></code>
  </div>
  <p>Pour connaître la version :</p>
  <div class="mdx-ap mb-6 rounded-xl bg-[#212121] flex relative items-center overflow-hidden">
    <div class="bg-gray-700 h-full px-4 py-2 text-black whitespace-nowrap">$</div>
    <code class="px-2 whitespace-nowrap overflow-x-auto"><strong>haproxy -v</strong></code>
  </div>
</section> 
<section>
<h2>Configuration de HAProxy</h2>
<p>Le service ne démarre pas car il faut obligatoirement renseigner la configuration. Le fichier de configuration principal se trouve dans /etc/haproxy/haproxy.cfg. Il se décompose en 4 sections :</p>
<br></br>
<div>
  <p>
  <em>Global</em>
  <br>
  En haut du fichier de configuration HAProxy se trouve la section globale. Les paramètres définissent la sécurité à l’échelle du processus et les réglages des performances qui affectent HAProxy à un bas niveau.
  </p>
  <br></br>
  <p>
  <em>Defaults</em>
  <br>
  Au fur et à mesure que la configuration se développe, l’utilisation d’une section par défaut aidera à réduire la duplication. Ses paramètres s’appliquent à toutes les sections frontend et backend qui viennent après. On est toujours libre de remplacer ces paramètres dans les sections suivantes.
  </p>
  <br></br>
  <p>
  <em>Frontend</em>
  </br>
  Lorsqu'HAProxy est utilisé comme reverse proxy devant les serveurs principaux, une section frontend spécifie les adresses IP et les ports accessibles aux clients. Il est possible d'ajouter plusieurs sections frontend afin d'exposer différents sites Web sur Internet. Chaque mot-clé frontend est suivi d'une étiquette, comme www.monsite.com, permettant de le distinguer des autres.
  </p>
  <br></br>
  <p>
  <em>Backedn</em>
  Une section backend définit un groupe de serveurs chargés de traiter les requêtes en répartissant la charge. Chaque backend reçoit une étiquette personnalisée, comme web_servers, pour l'identifier. Cette configuration reste généralement simple, nécessitant peu de paramètres dans la plupart des cas.
  </p>
</div>
<h3>Exemple de fichier de configuration</h3>
<p>Voici le fichier de configuration qui est utilisé dans notre exemple :</p>
<pre class="astro-code github-dark" style="background-color:#24292e;overflow-x:auto" tabindex="0"><code><span class="line"><span style="color:#E1E4E8">global  </span></span>
<span class="line"><span style="color:#E1E4E8">log /dev/log local0  </span></span>
<span class="line"><span style="color:#E1E4E8">log /dev/log local1 notice  </span></span>
<span class="line"><span style="color:#E1E4E8">chroot /var/lib/haproxy  </span></span>
<span class="line"><span style="color:#E1E4E8">stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners  </span></span>
<span class="line"><span style="color:#E1E4E8">stats timeout 30s  </span></span>
<span class="line"><span style="color:#E1E4E8">user haproxy  </span></span>
<span class="line"><span style="color:#E1E4E8">group haproxy  </span></span>
<span class="line"><span style="color:#E1E4E8">daemon  </span></span>
<span class="line"></span>
<span class="line"><span style="color:#E1E4E8">defaults  </span></span>
<span class="line"><span style="color:#E1E4E8">log global  </span></span>
<span class="line"><span style="color:#E1E4E8">mode http  </span></span>
<span class="line"><span style="color:#E1E4E8">option httplog  </span></span>
<span class="line"><span style="color:#E1E4E8">option dontlognull  </span></span>
<span class="line"><span style="color:#E1E4E8">timeout connect 5000  </span></span>
<span class="line"><span style="color:#E1E4E8">timeout client 50000  </span></span>
<span class="line"><span style="color:#E1E4E8">timeout server 50000  </span></span>
<span class="line"><span style="color:#E1E4E8">errorfile 400 /etc/haproxy/errors/400.http  </span></span>
<span class="line"><span style="color:#E1E4E8">errorfile 403 /etc/haproxy/errors/403.http  </span></span>
<span class="line"><span style="color:#E1E4E8">errorfile 408 /etc/haproxy/errors/408.http  </span></span>
<span class="line"><span style="color:#E1E4E8">errorfile 500 /etc/haproxy/errors/500.http  </span></span>
<span class="line"><span style="color:#E1E4E8">errorfile 502 /etc/haproxy/errors/502.http  </span></span>
<span class="line"><span style="color:#E1E4E8">errorfile 503 /etc/haproxy/errors/503.http  </span></span>
<span class="line"><span style="color:#E1E4E8">errorfile 504 /etc/haproxy/errors/504.http  </span></span>
<span class="line"></span>
<span class="line"><span style="color:#E1E4E8">frontend test  </span></span>
<span class="line"><span style="color:#B392F0">bind</span><span style="color:#E1E4E8"> </span><span style="color:#B392F0">172.16.200.10</span><span style="color:#E1E4E8">:80  </span></span>
<span class="line"><span style="color:#E1E4E8">default_backend web_servers  </span></span>
<span class="line"><span style="color:#E1E4E8">option httpclose  </span></span>
<span class="line"></span>
<span class="line"><span style="color:#E1E4E8">backend web_servers  </span></span>
<span class="line"><span style="color:#E1E4E8">balance roundrobin  </span></span>
<span class="line"><span style="color:#E1E4E8">option httpchk HEAD / HTTP/1.0  </span></span>
<span class="line"><span style="color:#B392F0">server</span><span style="color:#E1E4E8"> </span><span style="color:#B392F0">ha-proxy1</span><span style="color:#E1E4E8"> </span><span style="color:#B392F0">10.10.10.2</span><span style="color:#E1E4E8">:80 check  </span></span>
<span class="line"><span style="color:#B392F0">server</span><span style="color:#E1E4E8"> </span><span style="color:#B392F0">ha-proxy2</span><span style="color:#E1E4E8"> </span><span style="color:#B392F0">10.10.10.3</span><span style="color:#E1E4E8">:80 check  </span></span>
<span class="line"><span style="color:#E1E4E8">stats uri /statshaproxy  </span></span>
<span class="line"><span style="color:#B392F0">stats</span><span style="color:#E1E4E8"> </span><span style="color:#B392F0">auth</span><span style="color:#E1E4E8"> </span><span style="color:#B392F0">admin</span><span style="color:#E1E4E8">:admin  </span></span>
<span class="line"><span style="color:#E1E4E8">stats refresh 30s</span></span></code></pre>
<div>
<h3>Description global</h3>
<br></br>
<p>
<em><strong>log</strong></em> assure l'enregistrement des avertissements émis au démarrage ainsi que des problèmes survenant pendant l'exécution dans syslog. Il consigne également les requêtes au fur et à mesure de leur arrivée. Vous pouvez choisir de cibler le socket UNIX traditionnel où syslog ou journald écoutent (/dev/log), ou spécifier un serveur rsyslog distant pour stocker les journaux en externe sur votre serveur d'équilibrage de charge.
</p>
<br></br>
<em><strong>chroot</strong></em> permet de changer la racine du processus après son lancement, empêchant ainsi ce dernier et ses processus descendants d'accéder au système de fichiers en dehors de cet environnement restreint. Cela améliore la sécurité en limitant les actions possibles en cas de compromission.
<p>
<br></br>
<em><strong>stats socket</strong></em> active l’API Runtime, permettant d’effectuer des ajustements dynamiques, tels que la désactivation de serveurs, la gestion des vérifications d’état ou la modification des poids d’équilibrage de charge. Cette fonctionnalité offre un contrôle en temps réel sur HAProxy pour optimiser ses performances et sa gestion.
</p>
<br></br>
<p>
<em><strong>user / group </strong></em>indiquent à HAProxy de réduire ses privilèges après l’initialisation. Sous Linux, l’accès aux ports inférieurs à 1024 nécessite les droits root, et les clés privées TLS sont généralement protégées pour être lisibles uniquement par root. Sans spécifier un utilisateur et un groupe, HAProxy conserverait ses privilèges root, ce qui constitue une mauvaise pratique en matière de sécurité. Il est important de noter qu’HAProxy ne crée pas ces utilisateurs et groupes ; ils doivent être définis au préalable.
</p>
</div>
<div>
<h3>Description de defaults</h3>
<p>
<em><strong>log global</strong></em> indique aux différentes sections (frontend, backend, etc.) d'utiliser la configuration de journalisation définie dans la section global. Bien que chaque section puisse avoir ses propres directives de journalisation, cette option est pratique lorsqu’un seul serveur syslog est utilisé. Elle permet d'assurer une cohérence dans la collecte des logs sans devoir répéter la configuration dans chaque bloc.
</p>
<br></br>
<p>
<em><strong>mode </strong></em>Le paramètre mode détermine le fonctionnement d'HAProxy en tant que proxy TCP ou HTTP.

mode http : permet à HAProxy d’analyser et de manipuler les requêtes HTTP, offrant des fonctionnalités avancées comme l’équilibrage de charge basé sur les en-têtes, la réécriture d’URL ou la gestion des cookies.
</p>

<p>
mode tcp : fonctionne au niveau bas, sans interprétation du contenu, utile pour des protocoles non HTTP comme SSH, SMTP ou MySQL.
</p>
<p>
Le choix du mode dépend des besoins spécifiques du service à proxyfier.
</p>
<br></br>
<p>
<em><strong>option httplog</strong></em> ou plus rarement tcplog, configure HAProxy pour utiliser un format de journalisation plus détaillé lors de l’envoi des logs à Syslog. En règle générale, vous préférerez l’option httplog dans la section defaults, car elle permet de collecter des informations détaillées sur les requêtes HTTP. Si HAProxy rencontre un frontend en mode tcp, il émettra un avertissement et rétrogradera automatiquement vers l'option tcplog, qui est plus adaptée aux connexions TCP, sans analyse du contenu.
</p>
<br></br>
<p>
<em><strong>option dontlognull </strong></em>HAProxy effectue régulièrement des sondages vers les systèmes pour vérifier leur disponibilité. Par défaut, même une simple sonde ou une analyse de port génère une entrée dans les logs. Si cela encombre trop les journaux, vous pouvez activer l'option dontlognull, qui empêche l'enregistrement des connexions où aucune donnée n'a été échangée, typiquement celles liées aux sondes. Cependant, les erreurs seront toujours retournées au client et prises en compte dans les statistiques. Si vous souhaitez éviter que ces sondes soient même considérées dans les logs, vous pouvez utiliser l'option http-ignore-probes à la place.
</p>
<br></br>
<p>
<em><strong>timeout connect / timeout client / timeout server</strong></em>
<p>permettent de configurer des délais d'expiration spécifiques pour les connexions et la communication entre HAProxy, le client et les serveurs principaux :
</p>
    
  <ul>
      <li><strong><code>timeout connect</code></strong> : définit la durée maximale que HAProxy attendra pour établir une connexion TCP avec un serveur principal. Ce délai est exprimé en secondes (avec le suffixe "s"), ou en millisecondes si aucun suffixe n'est utilisé.</li>
      <li><strong><code>timeout client</code></strong> : mesure l'inactivité pendant laquelle HAProxy attend que le client envoie des données (segments TCP). Si cette période d'inactivité dépasse le délai défini, la connexion est fermée.</li>
      <li><strong><code>timeout server</code></strong> : mesure l'inactivité pendant laquelle HAProxy attend que le serveur principal envoie des données. Si cette période d'inactivité est trop longue, la connexion est également fermée.</li>
  </ul>
  <p>Des délais d'expiration bien définis permettent d'éviter que des processus bloqués n'occupent indéfiniment des connexions, ce qui peut aider à améliorer la réutilisation des connexions et à réduire les risques de surcharge.</p>
</p>
<br></br>
<p>
<em><strong>errorfile </strong></em>permet de spécifier les messages d'erreur personnalisés à retourner en cas d'erreur HTTP 400 ou 500. Cela permet de définir des réponses spécifiques pour ces erreurs, offrant ainsi une meilleure expérience utilisateur avec des messages plus clairs ou adaptés, au lieu des réponses d'erreur par défaut.
</p>
<br></br>
<h3>Description de Frontend</h3>
<p>
<em><strong>bind </strong></em>permet d'associer un écouteur à une adresse IP et un port spécifiques. L'adresse IP peut être omise pour écouter sur toutes les interfaces du serveur. Le port peut être défini comme une valeur unique, une plage ou une liste séparée par des virgules.
</p>
<p>
Il est courant d'utiliser les options ssl et crt avec bind pour gérer la terminaison SSL/TLS directement via HAProxy, évitant ainsi de déléguer cette tâche aux serveurs Web en backend. Cela améliore les performances et la sécurité en centralisant le chiffrement.
</p>
<br></br>
<p>
<em><strong>use_backend </strong></em><p>Le paramètre permet de sélectionner dynamiquement un <em>backend</em> spécifique pour traiter les requêtes entrantes, en fonction de conditions prédéfinies. Il est généralement utilisé avec des règles <strong>ACL</strong> (Access Control List) pour router le trafic selon certains critères, comme :</p>
    
  <ul>
        <li>Le chemin de l'URL (ex. : <code>if path_beg /api/</code> pour diriger les requêtes commençant par <code>/api/</code> vers un backend particulier).</li>
        <li>L'adresse IP du client.</li>
        <li>La présence d'un certain en-tête HTTP.</li>
  </ul>

  <p>Si aucune condition spéciale n'est nécessaire, une simple directive <code>default_backend</code> peut suffire pour définir le <em>backend</em> utilisé par défaut.</p>
</p>
<br></br>
<p>
<em><strong>default_backend </strong></em> spécifie le nom du <em>backend</em> vers lequel le trafic est acheminé si aucune règle <code>use_backend</code> ne l’a déjà redirigé. Si une requête n'est pas traitée par <code>use_backend</code> ou <code>default_backend</code>, HAProxy renverra une erreur <strong>503 Service Non Disponible</strong>.</p>
</p>
<br></br>
<p>
<em><strong>option httpclose </strong></em>    <p>Lorsque cette option est activée, HAProxy ferme immédiatement les connexions avec le serveur et le client après l’échange de la requête et de la réponse.</p>
    
  <p>Elle assure également que l'en-tête <code>Connection: close</code> est présent dans les requêtes et les réponses :</p>
  <ul>
        <li>Si l’en-tête est déjà défini, il est conservé.</li>
        <li>Si l’en-tête est absent, HAProxy l’ajoute.</li>
        <li>Tout en-tête <code>Connection</code> contenant une valeur autre que <code>close</code> est supprimé.</li>
  </ul>

  <p>Cette option peut être utile pour forcer la fermeture des connexions et éviter l'utilisation persistante des connexions HTTP, notamment pour des raisons de compatibilité avec certains serveurs ou clients.</p>
</p>
<br></br>
<h3>Description de Backend</h3>
<p>
<em><strong>balance </strong></em><p>Le paramètre <code>balance</code> contrôle la manière dont HAProxy sélectionne le serveur pour répondre à une requête lorsqu'aucune méthode de persistance ne s’applique. Il existe plusieurs stratégies d’équilibrage de charge :</p>

  <ul>
        <li><strong>roundrobin</strong> : sélectionne les serveurs à tour de rôle dans l’ordre.</li>
        <li><strong>leastconn</strong> : envoie la requête au serveur ayant le moins de connexions actives.</li>
        <li><strong>source</strong> : attribue toujours le même serveur à une adresse IP source donnée.</li>
        <li><strong>uri</strong> : sélectionne un serveur en fonction du hachage de l’URI demandée.</li>
  </ul>

  <p>Ces méthodes permettent d’optimiser la répartition du trafic en fonction des besoins de l’infrastructure.</p>
</p>
<br></br>
<p>
<em><strong>option httpchk </strong></em>active les vérifications de santé HTTP (couche 7) au lieu des vérifications TCP (couche 4). Un serveur qui ne répond pas correctement à une requête HTTP est retiré de la liste des serveurs actifs, même s'il est toujours accessible au niveau réseau. Cela permet d’éviter d’envoyer des requêtes à un serveur en panne.
</p>
<br></br>
<p>
<em><strong>server </strong></em>est utilisé dans la configuration du backend pour définir les serveurs disponibles. Il inclut un nom, une adresse IP ou un nom de domaine, ainsi qu’un port. Si un nom de domaine est utilisé, HAProxy peut le résoudre au démarrage ou en continu avec un résolveur DNS.
</p>
<br></br>
<p>
<em><strong>stats </strong></em> permet d’activer une page de statistiques web pour surveiller l’état de HAProxy. L’URI définit l’URL d’accès, protégée par un login et un mot de passe. L’option refresh permet d’actualiser automatiquement les statistiques à intervalle régulier.
</p>
<br></br>
<p>sources : <a href="https://www.haproxy.com/fr/blog/the-four-essential-sections-of-an-haproxy-configuration/" target="_blank" rel="noreferrer noopener">haproxy.com</a></p>
<h2>Configuration des serveurs web et test</h2>
<p>
Nous allons utiliser Apache2 pour nos deux serveurs web, en leur attribuant les adresses IP suivantes : 10.10.10.2 et 10.10.10.3. Il sera essentiel de vérifier qu’ils sont bien connectés au réseau interne backend sur VirtualBox, en procédant comme suit :
</p>
<br></br>
<img src="/public/assets/phot2-haproxy.jpg" alt="Configuration réseau VBox" width="804" height="578" loading="lazy" decoding="async">
<br></br>
<p>Nous allons modifier les fichiers index.html différemment sur chaque serveur afin de bien démontrer l’alternance entre eux. Bien sûr, dans un environnement de production, nos serveurs web devraient pointer vers la même ressource.</p>
<br></br>
  <div class="mdx-ap mb-6 rounded-xl bg-[#212121] flex relative items-center overflow-hidden">
    <div class="bg-gray-700 h-full px-4 py-2 text-black whitespace-nowrap">$</div>
    <code class="px-2 whitespace-nowrap overflow-x-auto">nano /var/www/html/index.html</code>
  </div>
<br></br>
<p>Sur web01 :</p>
<br></br>
<pre class="astro-code github-dark" style="background-color:#24292e;overflow-x:auto" tabindex="0"><code><span class="line"><span style="color:#E1E4E8">&lt;!</span><span style="color:#85E89D">DOCTYPE</span><span style="color:#E1E4E8"> </span><span style="color:#B392F0">html</span><span style="color:#E1E4E8">&gt;  </span></span>
<span class="line"><span style="color:#E1E4E8">&lt;</span><span style="color:#85E89D">head</span><span style="color:#E1E4E8">&gt;  </span></span>
<span class="line"><span style="color:#E1E4E8">        &lt;</span><span style="color:#85E89D">meta</span><span style="color:#E1E4E8"> </span><span style="color:#B392F0">charset</span><span style="color:#E1E4E8">=</span><span style="color:#9ECBFF">"UTF-8"</span><span style="color:#E1E4E8">&gt;  </span></span>
<span class="line"><span style="color:#E1E4E8">&lt;/</span><span style="color:#85E89D">head</span><span style="color:#E1E4E8">&gt;  </span></span>
<span class="line"><span style="color:#E1E4E8">&lt;</span><span style="color:#85E89D">body</span><span style="color:#E1E4E8">&gt;  </span></span>
<span class="line"><span style="color:#E1E4E8">        &lt;</span><span style="color:#85E89D">h1</span><span style="color:#E1E4E8">&gt;HAPROXY BACKEND 1&lt;/</span><span style="color:#85E89D">h1</span><span style="color:#E1E4E8">&gt;  </span></span>
<span class="line"><span style="color:#E1E4E8">&lt;/</span><span style="color:#85E89D">body</span><span style="color:#E1E4E8">&gt;  </span></span>
<span class="line"><span style="color:#E1E4E8">&lt;/</span><span style="color:#85E89D">html</span><span style="color:#E1E4E8">&gt;</span></span></code></pre>
<br></br>
<p>Sur web02 :</p>
<br></br>
<pre class="astro-code github-dark" style="background-color:#24292e;overflow-x:auto" tabindex="0"><code><span class="line"><span style="color:#E1E4E8">&lt;!</span><span style="color:#85E89D">DOCTYPE</span><span style="color:#E1E4E8"> </span><span style="color:#B392F0">html</span><span style="color:#E1E4E8">&gt;  </span></span>
<span class="line"><span style="color:#E1E4E8">&lt;</span><span style="color:#85E89D">head</span><span style="color:#E1E4E8">&gt;  </span></span>
<span class="line"><span style="color:#E1E4E8">        &lt;</span><span style="color:#85E89D">meta</span><span style="color:#E1E4E8"> </span><span style="color:#B392F0">charset</span><span style="color:#E1E4E8">=</span><span style="color:#9ECBFF">"UTF-8"</span><span style="color:#E1E4E8">&gt;  </span></span>
<span class="line"><span style="color:#E1E4E8">&lt;/</span><span style="color:#85E89D">head</span><span style="color:#E1E4E8">&gt;  </span></span>
<span class="line"><span style="color:#E1E4E8">&lt;</span><span style="color:#85E89D">body</span><span style="color:#E1E4E8">&gt;  </span></span>
<span class="line"><span style="color:#E1E4E8">        &lt;</span><span style="color:#85E89D">h1</span><span style="color:#E1E4E8">&gt;HAPROXY BACKEND 2&lt;/</span><span style="color:#85E89D">h1</span><span style="color:#E1E4E8">&gt;  </span></span>
<span class="line"><span style="color:#E1E4E8">&lt;/</span><span style="color:#85E89D">body</span><span style="color:#E1E4E8">&gt;  </span></span>
<span class="line"><span style="color:#E1E4E8">&lt;/</span><span style="color:#85E89D">html</span><span style="color:#E1E4E8">&gt;</span></span></code></pre>
<br></br>
<p>On peut alors tester avec le client web l’adresse de notre frontend qui est <em>172.16.200.10</em> :</p>
<br></br>
<p><img src="/public/assets/backend1.webp" alt="backend1" width="538" height="151" loading="lazy" decoding="async">
<br></br>
<img src="/public/assets/backend2.webp" alt="backend2" width="535" height="150" loading="lazy" decoding="async"></p>
<br></br>
<p>HAProxy alterne bien entre nos deux serveurs web lorsque nous rafraîchissons la page.</p>
</div>
</section>