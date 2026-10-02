# Remonter les données de son Xpeng dans Home Assistant

**Sans cloud, sans abonnement, sans téléphone dans la voiture.**

Retour d'expérience complet sur un XPENG G6 Performance AWD 2026 (pack LFP 80,8 kWh), avec un dongle OBD WiCAN Pro qui publie directement en MQTT vers Home Assistant.

Tout ce qui suit a été testé et fonctionne. Je signale explicitement ce qui reste incertain ou non résolu — il y en a, et j'ai besoin d'aide dessus (voir la dernière section).

---

## 1. Le résultat

Voici ce qui remonte dans Home Assistant, en local, avec un rafraîchissement de quelques secondes :

| Donnée | Statut |
|---|---|
| État de charge (SOC) | ✅ exact au dixième près |
| Odomètre | ✅ |
| Courant batterie haute tension (signé) | ✅ — positif en charge, négatif en décharge |
| Température batterie min / max | ✅ |
| Tension cellule min / max | ✅ |
| Tension batterie 12 V | ✅ (donnée du dongle) |
| État de santé (SOH) | ❌ pas de réponse sur pack LFP 2026 |
| Tension pack | ❌ idem |
| Portes, vitres, allumage, prise VE | ❌ non exposés |
| Verrouillage, climatisation à distance | ❌ impossible (lecture seule) |

Le courant HV est la donnée la plus intéressante et la moins évidente : son signe permet de savoir si la voiture charge, ce qu'aucune autre source ne donne. Multiplié par la tension pack, il donnerait la puissance réelle aux bornes de la batterie.

---

## 2. Pourquoi cette solution plutôt qu'une autre

XPENG ne publie aucune API publique. J'ai exploré toutes les pistes avant d'arriver là.

**ABRP Premium + Enode.** XPENG a développé une API tierce pour Enode, qu'ABRP consomme, et une intégration HACS ramène ça dans HA. Ça donne le SOC et la position. Trois défauts rédhibitoires : 15 à 30 minutes de latence cumulée, aucun état de charge, aucun odomètre, et 5 €/mois. J'ai testé pendant l'essai gratuit — quand l'abonnement expire, les entités se figent silencieusement sur leur dernière valeur, ce qui est pire que rien.

**Enode en direct.** Un compte développeur gratuit existe, mais il ne crée que des clients *sandbox*, incapables de lier un vrai véhicule. L'accès production passe par leur service commercial, avec plusieurs jours de délai et aucune garantie pour un particulier. Le périmètre officiel pour le G6 se limite de toute façon à : état de charge, informations, localisation, démarrage et arrêt de la charge. Pas d'odomètre, pas de verrouillage, pas de clim.

**EVLinkHA.** Mort. Le domaine renvoie une page de parking. L'intégration GitHub existe encore mais tape sur une API disparue.

**xpcardata + AI Box Android.** Le projet `stevelea/xpcardata` est le plus riche en données (126 tensions de cellules, historique de charge). Mais il exige un dongle BLE *plus* une AI Box Android à demeure dans la voiture, plus Tailscale. Beaucoup de matériel, beaucoup de points de panne, et un dongle BLE laissé branché en permanence est un vrai trou de sécurité.

**WiCAN Pro.** Un seul objet, en WiFi authentifié, qui parle MQTT nativement. C'est ce que j'ai retenu. Détail amusant : l'auteur de xpcardata utilise lui-même un WiCAN Pro sur son G6, et c'est lui qui a contribué le profil PID XPENG au firmware.

**EU Data Act.** Le règlement (UE) 2023/2854 est applicable depuis le 12 septembre 2025 et donne un **droit** d'accès aux données de son véhicule connecté. XPENG s'est engagé auprès de propriétaires à le respecter. J'ai envoyé une demande, sans retour à ce jour. Ça ne coûte rien et ça mûrit en arrière-plan — je conseille à tout le monde de le faire, l'effet de masse aidera. Attention : ce droit porte sur les *données*, pas sur la commande à distance.

---

## 3. Le matériel

**WiCAN Pro** — référence `MP-WICAN-PRO`, MeatPi Electronics. Environ **76 € HT chez Mouser France** (port gratuit au-delà de 75 €, facturation UE, pas de douane), ou 89 $ + 18 $ de port chez Crowd Supply. Les stocks sont tendus, pensez au backorder.

⚠️ **Ne prenez pas le WiCAN classique** (`WICAN-OBD-C3`), moins cher mais limité en mémoire. Le fabricant recommande explicitement la version PRO, et surtout la fonction de réveil périodique — essentielle pour capter les charges nocturnes — est arrivée sur la branche PRO en premier.

**Rien d'autre à acheter.** Pas de téléphone, pas d'AI Box, pas de dongle BLE. Une carte microSD est optionnelle (log local).

**Prérequis côté maison :**
- Un réseau **WiFi 2,4 GHz** qui couvre le garage. Le WiCAN est un ESP32, il ne voit pas le 5 GHz. Vérifiez le signal *à l'emplacement de la voiture* avant de commander.
- Un broker MQTT dans Home Assistant. L'add-on **Mosquitto** suffit, et si vous avez déjà Zigbee2MQTT vous l'avez déjà.

**La prise OBD du G6** est sous le volant, accessible sans démonter de cache. Le dongle se branche et se débranche en deux secondes.

---

## 4. Installation pas à pas

### 4.1 Préparer Home Assistant

1. Installer l'add-on **Mosquitto broker** s'il n'est pas déjà là, et le démarrer.
2. Créer un utilisateur HA dédié (Réglages → Personnes → Utilisateurs), par exemple `wican`, coché **utilisateur local uniquement**, non administrateur. Notez le mot de passe.
3. Dans HACS, ajouter le dépôt personnalisé `https://github.com/jay-oswald/ha-wican`, catégorie *Integration*, puis l'installer.

### 4.2 Premier contact avec le dongle

Branchez le WiCAN sur la prise OBD. Il ne peut pas être alimenté par l'USB-C, qui ne sert qu'au flashage.

1. Il démarre en point d'accès WiFi nommé `WiCAN_xxxxxxx`, mot de passe par défaut `@meatpi#`.
2. Connectez-vous dessus, puis ouvrez **`http://192.168.0.10`** (attention : c'est `192.168.80.1` sur le modèle classique, pas sur le PRO).
3. **Changez immédiatement le mot de passe du point d'accès.** Sinon n'importe qui passant à proximité de la voiture peut se connecter au dongle.
4. **Mettez à jour le firmware** avant toute autre chose. Plusieurs bugs pénibles ont été corrigés récemment.

### 4.3 Réseau et MQTT

1. Passez le dongle en **mode Station** et connectez-le à votre WiFi 2,4 GHz.
2. Notez son nom mDNS dans l'onglet *Status* (forme `wican_xxxxxxxxxxxx.local`) — l'intégration HACS en a besoin.
3. Réservez-lui une IP fixe dans votre DHCP.
4. Onglet **Settings**, section CAN : mettez le protocole sur **AutoPID**. ⚠️ **Rien ne fonctionnera sans ça** — c'est la cause n°1 des tickets ouverts sur l'intégration.
5. Section MQTT : adresse du broker = l'IP locale de votre Home Assistant, **sans `http://` ni port**. Port 1883. Un Client ID unique. Les identifiants de l'utilisateur créé plus haut. Keep Alive 60 s.
6. *Store*, puis redémarrage depuis l'onglet *About*.

> **Note :** la documentation se contredit sur la découverte automatique. Une page dit d'activer *MQTT HA Discovery*, une autre indique que l'option est désactivée et qu'il faut passer par l'intégration HACS. Ça dépend de votre version de firmware — regardez ce que propose réellement votre interface.

---

## 5. Les PID — la partie délicate

### 5.1 Ne chargez pas le profil intégré tel quel

Le firmware embarque un profil `Xpeng: P5/P7/G6/G9/X9` sélectionnable en un clic. Il fonctionne, mais il interroge **234 paramètres**, dont les 192 tensions de cellules individuelles et 35 sondes de température.

Deux problèmes, constatés chez moi :

**Les tensions de cellules individuelles sont fausses.** Les trois premières sont correctes, les suivantes partent en vrille — j'ai relevé 1,54 V, 0 V, 3,70 V, 0,32 V sur des cellules voisines. C'est un bug d'offset sur les trames ISO-TP multi-frames, connu et documenté dans l'issue #514 du dépôt `meatpiHQ/wican-fw`. Idem pour les températures : une sonde à 191 °C et une dizaine bloquées à −40 °C.

**Le cycle sature.** Avec 234 paramètres interrogés en boucle, les PID en fin de file partent en timeout et ne remontent jamais.

### 5.2 Le profil allégé

Neuf paramètres, ceux qui servent réellement. Enregistrez ce contenu dans un fichier `.json` et importez-le via *Vehicle Profiles* dans l'onglet Automate.

Le fichier est dans ce dépôt : **[`xpeng_g6_wican.json`](xpeng_g6_wican.json)**

Téléchargez-le (bouton *Download raw file* en haut à droite du fichier), puis importez-le dans l'onglet *Automate* du dongle, section *Vehicle Profiles*.

> Il contient 8 paramètres : SOC, odomètre, tension pack, courant HV, températures batterie min/max, tensions cellule min/max. SOH est volontairement absent, voir la section 9.

### 5.3 Les quatre pièges à connaître

1. **La chaîne d'initialisation doit se terminer par un point-virgule.** Sans ça, le chargement depuis un fichier échoue silencieusement.
2. **`Destination Type` doit être sur `Default`** pour chaque paramètre. C'est ce qui décide si la valeur part vers MQTT ou reste interne au dongle.
3. **Ne mélangez jamais un profil chargé et des PID personnalisés ajoutés par-dessus.** L'issue #516 décrit exactement ça : les PID d'origine cessent de se mettre à jour et le nouveau n'apparaît pas. Soit l'un, soit l'autre.
4. **Si un PID ne remonte rien, retirez ses champs `Unit`, `Class`, `Min` et `Max`.** Le processus de chargement les ajoute parfois automatiquement et ça casse certaines lectures.

Le bouton **Test** à côté de chaque PID est votre meilleur ami : il envoie la requête et affiche la réponse brute immédiatement.

### 5.4 La voiture doit être réveillée

Le bus CAN ne répond pas sur une voiture endormie. Pour vos premiers essais, installez-vous dedans, contact mis. Un déverrouillage à distance depuis l'app suffit aussi à la réveiller quelques minutes.

Un dongle silencieux sur une voiture garée depuis deux heures **n'est pas une panne**, c'est le comportement normal.

---

## 6. Vérifier ses valeurs

Comparez le SOC au tableau de bord : la concordance doit être exacte. Vérifiez l'odomètre : c'est le piège classique, certains ont vu 25089 km s'afficher pour 1146 km réels.

**Sur les tensions de cellules**, sachez lire ce que vous voyez :

- **~3,3 V à mi-charge → pack LFP.** C'est le cas des G6 millésime 2026, qui abandonnent le NMC 87,5 kWh au profit d'un LFP 80,8 kWh, sur les versions RWD comme AWD.
- **~3,7 à 3,9 V → pack NMC**, millésimes antérieurs.

⚠️ Les fiches Wikipédia (FR et EN) sont périmées sur ce point et décrivent encore le NMC comme la grosse batterie. Ne vous y fiez pas.

**Si vous avez un pack LFP**, deux conséquences pratiques : la courbe de tension est très plate, donc le SOC estimé dérive plus facilement, et une charge à 100 % régulière (toutes les une à deux semaines) est recommandée pour recaler le BMS. C'est l'inverse de la consigne NMC.

Et sur le G6 2026, la charge alternative est limitée à **10,5 kW**, sans option 22 kW. Si vous avez une borne triphasée 22 kW, elle n'en donnera que la moitié.

---

## 7. Ce qu'on en fait dans Home Assistant

Les entités arrivent avec des identifiants illisibles du type `sensor.xxx_wican_14c19f49c72d_local_soc`. Renommez-les dès le départ (Réglages → Appareils → l'entité → engrenage) en `sensor.xpeng_g6_soc` et compagnie. Vous vous remercierez plus tard.

Ensuite, trois choses simples et utiles :

**Kilométrage périodique.** Deux helpers *Compteur de service* (utility_meter) sur l'odomètre, l'un en cycle mensuel, l'autre en cycle annuel. Ils affichent `unknown` jusqu'à la deuxième lecture, c'est normal.

**Consommation réelle.** Avec le kilométrage périodique d'un côté et l'énergie livrée par votre borne de l'autre, vous obtenez des kWh/100 km et un coût aux 100 km réels.

**Détection de charge.** Le signe de `HV_A` dit si la voiture charge. C'est bien plus fiable qu'une déduction basée sur la puissance de la borne, surtout si vous avez plusieurs véhicules qui se partagent le même point de charge.

Pour la comptabilité énergétique, préférez la mesure côté borne à `HV_V × HV_A` : elle inclut les pertes du chargeur embarqué, donc elle correspond à ce que vous payez.

---

## 8. Les limites, sans enjoliver

**La veille.** Le WiCAN surveille la tension 12 V et s'endort si elle reste sous 13 V pendant quelques minutes, tombant sous 1 mA. Voiture garée, la 12 V ne remonte pas forcément au-dessus du seuil — donc le dongle dort et rien ne remonte. La fonction de **réveil périodique** du WiCAN Pro est censée résoudre ça : il se réveille, publie, se rendort. À tester chez vous, je n'ai pas encore de recul.

**Le risque de décharge 12 V.** C'est le revers de la médaille. Un cas documenté sur BYD Atto 3 : un WiCAN qui maintenait l'ECU éveillé en permanence, avec **2 % de SOC haute tension perdus par nuit**. Le fabricant a d'ailleurs écarté l'option « rester éveillé sur activité CAN » comme dangereuse. Surveillez votre SOC au réveil pendant les premières nuits.

**Aucune commande à distance.** Le WiCAN en AutoPID est en lecture seule. Injecter des trames CAN pour actionner les serrures, ce serait de la rétro-ingénierie non documentée, sur un véhicule souvent en LLD — je le déconseille formellement.

Pour le préconditionnement et le verrouillage, la seule voie propre est l'app XPENG. Bonne nouvelle : elle expose des actions dans l'app **Raccourcis** d'iOS, donc une automatisation horaire ou géolocalisée est possible côté téléphone. Home Assistant peut même fournir la décision (« y a-t-il assez de surplus solaire ? ») via l'action *Rendre un modèle* de l'app compagnon, le raccourci se contentant d'interroger HA et d'agir en conséquence.

**En LLD**, jetez un œil à vos conditions générales avant de laisser un dongle branché en permanence. Ça se retire en deux secondes avant restitution.

---

## 9. Ce que je n'ai pas résolu — appel à contribution

Sur mon pack **LFP 2026**, deux PID répondent systématiquement `No response` :

- **SOH** — testé en `22011A1` (profil officiel) et `22110A1` (variante communautaire), aucun des deux
- **HV_V**, la tension pack — `2211011`

Ce ne sont pas des erreurs de formule : la requête part et rien ne revient. Le calculateur `704` répond bien pour le SOC et le courant, donc la voie de communication est bonne. Mon hypothèse est que le BMS du pack CALB LFP n'expose pas la même carte de PID que les packs NMC pour lesquels le profil a été écrit.

**Si vous avez un G6 2026 LFP et que vous trouvez ces PID, dites-le moi**, je mettrai le profil à jour et je le proposerai au dépôt.

La méthode recommandée par MeatPi pour un nouveau véhicule : passer le dongle en mode ELM327, s'y connecter avec **Car Scanner**, lancer une session, exporter les logs, et en déduire les initialisations et les formules. Des propriétaires français lisent bien la tension pack avec Car Scanner (588 V relevés sur un pack NMC), donc l'information existe quelque part.

**Si vous avez un P7, un G9 ou un X9**, tout ce tutoriel devrait s'appliquer — le profil couvre ces modèles — mais je ne peux rien confirmer. Vos retours sont les bienvenus.

---

## 10. Contribuer

Ce dépôt est ouvert. Trois façons d'aider :

- **Vous avez testé sur votre véhicule ?** Ouvrez une [issue](../../issues/new/choose) avec le modèle, le millésime, la chimie de batterie et ce qui remonte ou non. Même un « ça marche tel quel sur mon P7 » est utile.
- **Vous avez trouvé les PID SOH ou tension pack sur un pack LFP ?** C'est le trou principal, voir la section 9.
- **Vous avez corrigé ou enrichi le profil ?** Proposez une pull request.

Les contributions validées seront proposées au dépôt officiel `meatpiHQ/wican-fw` pour que tout le monde en profite.

---

## 11. Récapitulatif

| | |
|---|---|
| Coût | ~76 € (WiCAN Pro) |
| Abonnement | aucun |
| Matériel dans la voiture | un dongle |
| Latence | quelques secondes |
| Dépendance cloud | aucune |
| Temps d'installation | une soirée |

**Liens utiles**
- Firmware et profils : `github.com/meatpiHQ/wican-fw`
- Intégration Home Assistant : `github.com/jay-oswald/ha-wican`
- Discussion à l'origine du profil G6 : discussion #517 du dépôt wican-fw
- Alternative riche en données (BLE + AI Box) : `github.com/stevelea/xpcardata`

**Merci** à *stevelea*, qui a fait le gros du travail de rétro-ingénierie des PID XPENG et les a contribués au firmware, et à l'équipe MeatPi pour leur réactivité sur les issues.

---

*Testé sur XPENG G6 Performance AWD 2026, pack LFP 80,8 kWh, Home Assistant OS avec Mosquitto. Les valeurs et comportements peuvent différer sur d'autres millésimes ou chimies de batterie. Questions et corrections bienvenues.*
