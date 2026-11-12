---
title: "Des props qui ne doivent jamais tomber : notre socle firmware ESP32"
date: 2026-11-12
lang: fr
tags: [esp32, esp-idf, firmware, c, freertos, mqtt, ota, electronique, escape-game]
---

> **Titres alternatifs** (à choisir, puis supprimer ce bloc)
>
> 1. Le matériel d'abord, le réseau ensuite : notre socle ESP-IDF pour accessoires d'escape game
> 2. La carte qui répondait au ping : leçons de firmware ESP32 en salle
> 3. Mode sans échec, OTA signée et polling assumé : du firmware pour des salles hostiles

# Des props qui ne doivent jamais tomber : notre socle firmware ESP32

Dans un escape game, les *props* sont l'endroit où le logiciel touche le monde physique : un digicode, un coffre à électroaimant, des vases canopes qui reconnaissent les statuettes posées dessus, un canon, un kraken qui cligne des yeux. Chez Team Break, la plupart de ces accessoires sont pilotés par un **ESP32** programmé en C avec **ESP-IDF**. Pour qu'ils se ressemblent tous (même configuration, même page web, même comportement en cas de panne), ils partagent un **socle firmware** commun.

Dans le [premier article](./team-break-architecture.fr.md) de cette série, j'ai survolé l'électronique. Ici, j'entre dans le détail : pourquoi ce socle existe, comment il est construit, et ce que le terrain nous a appris.

## L'environnement : hostile, et sans personne sur place

Un accessoire d'escape game vit dans des conditions qu'on ne retrouve pas souvent en informatique :

- **Tout est en 12 V**, rien en USB. Les alimentations sont partagées avec des électroaimants, des moteurs et des relais.
- **Les câbles sont longs** et passent à côté de charges inductives qui commutent en permanence. Le moindre front parasite est une entrée fantôme potentielle.
- **La salle est pleine de métal** (et d'autres accessoires qui émettent) : le Wi-Fi n'y est pas toujours le bienvenu.
- **Le courant est coupé brutalement**, parfois plusieurs fois par jour.
- **Les joueurs manipulent tout**, et pas toujours avec délicatesse.
- **Il n'y a pas d'Internet** dans certaines salles, et **personne pour appuyer sur reset** au milieu d'une session.

Ajoutons une diversité de câblage qu'aucune documentation ne capture vraiment : selon les sites, le même capteur est monté à l'envers, ou en contact normalement fermé. Et quelques surprises mécaniques : à La Défense, les dagues d'une énigme sont de petits moteurs de fermeture centralisée automobile, qu'il ne faut *jamais* alimenter en continu.

Côté matériel, le standard est l'**ESP32-WROOM** avec 4 Mo de flash. Pour les besoins en relais, nous utilisons des cartes **Kincony KC868-A16** : un ESP32, 16 relais et 16 entrées sur des expandeurs I2C PCF8574, et de l'Ethernet via un PHY LAN8720. On trouve encore quelques Arduino Uno avec shield Ethernet sur des installations plus anciennes, et des panneaux d'E/S tiers.

## Pourquoi un socle

Au départ, chaque accessoire avait son firmware, copié depuis un gabarit. La configuration y était décrite par des **X-macros** :

```c
#define TB_CONFIG_CHAR_ITEMS                                   \
    TB_CONFIG_CHAR_ITEM(char *, ssid, "")                      \
    TB_CONFIG_CHAR_ITEM(char *, hostname, "New")               \
    TB_CONFIG_CHAR_ITEM(char *, mqtt_broker, "mqtt://mqtt.eclipse.org")
#define TB_CONFIG_ITEMS                                        \
    TB_CONFIG_ITEM(int, mqtt_port, 1883)                       \
    TB_CONFIG_ITEM(int, ma_nouvelle_valeur, 42)
```

Cela produisait une structure plate (`config.ssid`, `config.mqtt_port`), sans validation ni migration. Pour ajouter un champ à un accessoire, la documentation conseillait… de modifier le code partagé. Le composant réseau était devenu un « composant-dieu » qui embarquait le Wi-Fi, MQTT et le serveur HTTP, avec 11 dépendances et une dépendance circulaire avec le pilote Ethernet. Chaque copie divergeait, et chaque correction devait être reportée à la main.

Le socle actuel est né de ce constat. C'est un dépôt de composants ESP-IDF que chaque firmware d'accessoire inclut comme **sous-module Git**. Côté accessoire, il suffit d'une ligne dans le `CMakeLists.txt` racine :

```cmake
set(EXTRA_COMPONENT_DIRS ${CMAKE_SOURCE_DIR}/shared_components)
```

L'accessoire n'apporte que ce qui lui est propre : son brochage, sa logique de jeu, ses messages et sa section de configuration. Le découpage du composant réseau a retiré 1 052 lignes et ramené ses dépendances de 11 à 3.

Le socle compile avec **ESP-IDF 5.5 et 6.0**. La v6 a sorti plusieurs composants du framework (cJSON, le client MQTT, le pilote LAN87xx) vers le registre de composants : ils sont déclarés conditionnellement dans les `idf_component.yml` (`if: "idf_version >=6.0"`). Une règle est gravée dans chaque spécification : **mettre à jour le sous-module ne doit exiger aucune modification des firmwares hôtes**.

Il compte aujourd'hui une vingtaine de composants, environ 13 000 lignes de C et 6 900 lignes d'interface web :

| Domaine | Composants |
|---|---|
| Réseau | `tb_network` (pile TCP/IP, mDNS, interfaces), `tb_wifi`, `tb_lan8720`, `tb_mqtt`, `tb_ntp`, `tb_http_server` |
| Système | `tb_storage` (LittleFS), `tb_config`, `tb_log`, `tb_watchdog`, `tb_recovery`, `tb_infos` |
| Matériel | `tb_gpio`, `tb_i2c`, `tb_pcf8574`, `tb_sx1509`, `tb_neopixel`, `lib8tion`, `tb_rfid`, `tb_pn532` |

## Des interfaces en C, avec des pointeurs de fonctions

Pour qu'un accessoire n'ait pas à savoir si son réseau passe par le Wi-Fi ou l'Ethernet, si une broche est interne ou sur un expandeur I2C, ou quel modèle de lecteur RFID il utilise, le socle définit des **interfaces** à l'ancienne : des structures de pointeurs de fonctions.

```c
// Réseau : Wi-Fi ou Ethernet, même contrat
struct tb_network_interface_t {
  get_state get_state;   // NOT_CONFIGURED / DISCONNECTED / CONNECTING / CONNECTED
  get_infos get_infos;   // cJSON* pour la page web
  get_name  get_name;
  void *datas;           // données de l'implémentation
  void *netif;           // esp_netif_t*
};

// GPIO : broches internes, PCF8574, SX1509…
struct tb_gpio_interface_t {
  read_gpio read;              write_gpio write;
  get_gpio_type get_type;      set_gpio_type set_type;
  get_interface_name get_name; get_gpio_count get_count;
  set_interface_name set_name; void *datas;
};

// RFID : PN532 aujourd'hui, autre chose demain
struct tb_rfid_interface_t {
  tb_rfid_read_uid read_uid;   tb_rfid_get_reader_state get_state;
  tb_rfid_reset_reader reset_reader; tb_rfid_get_reader_count get_count;
  tb_rfid_get_interface_name get_name;
  void *datas;
};
```

Un expandeur d'E/S n'est qu'une implémentation de `tb_gpio_interface_t` : le reste du code (filtres, callbacks, page web) le traite exactement comme les broches de l'ESP32.

## L'ordre de démarrage est une décision de sécurité

L'ordre d'initialisation est fixé et documenté, avec une raison pour chaque étape :

1. **Le stockage** d'abord : la configuration, les logs, l'état de récupération et la page web sont des fichiers.
2. **La récupération** avant la configuration : elle peut remplacer une configuration cassée par la dernière configuration valide. Lire la configuration avant reviendrait à charger celle qui est cassée.
3. **La configuration, les logs, le watchdog.**
4. **Le matériel.**
5. **Le réseau.**
6. **NTP et MQTT**, sauf en mode sans échec.
7. **Le serveur HTTP** en dernier, puis on arme le marqueur de démarrage réussi.

La quatrième étape est la plus importante. La documentation l'explique en une phrase :

> **Hardware before network.** An output should reach its safe state (locks closed, motors off) as early as possible, not after a Wi-Fi connection that can take several seconds or never come.

Le firmware de la carte Kincony pousse la logique encore plus loin : il initialise les relais *avant même le stockage*, et les force à l'état repos dès les premières millisecondes.

```c
tb_gpio_init();
tb_i2c_init(&(tb_i2c_config_t){.sda = 4, .scl = 5});
in1  = init_pcf8574_with_retry(0x21, "in1");   in2  = init_pcf8574_with_retry(0x22, "in2");
out1 = init_pcf8574_with_retry(0x24, "out1");  out2 = init_pcf8574_with_retry(0x25, "out2");
configure_output_bank(out1, 0); configure_output_bank(out2, 8);   // écrire 1 = relais au repos
configure_input_bank(in1, 0);   configure_input_bank(in2, 8);
tb_storage_init_flash();
tb_recovery_init();  bool safe = tb_recovery_is_safe_mode_active();
tb_config_init(&config); tb_log_init(); tb_watchdog_init();
tb_network_init();
tb_network_add_interface(tb_lan8720_init(&eth_config));
tb_network_add_interface(tb_wifi_init());
if (!safe) { tb_ntp_init(); /* … puis MQTT et ses callbacks */ }
if (tb_http_server_init() == ESP_OK) tb_recovery_mark_boot_successful();
```

## Une configuration décrite par des schémas

Chaque composant déclare sa section de configuration avec des macros déclaratives. Voici celle du client MQTT :

```c
_Static_assert(MQTT_MAX_TOPIC_SIZE - 1 == 255, "sub_topic max length must follow MQTT_MAX_TOPIC_SIZE");
TB_CONFIG_GENERATE_PARSER(tb_mqtt, TB_MQTT, "tb_mqtt",
    TB_FIELD_STRING(tb_mqtt_config_t, broker, "localhost", 256),
    TB_FIELD_UINT32_RANGE(tb_mqtt_config_t, port, 1883, 1, 65535),
    TB_FIELD_STRING_LEN(tb_mqtt_config_t, sub_topic, "/sub_topic", 1, 255),
    TB_FIELD_STRING(tb_mqtt_config_t, pub_topic, "/pub_topic", 256)
);
TB_CONFIG_DEFINE_GETTER(tb_mqtt, "tb_mqtt", false);
```

Un accessoire déclare sa propre section de la même façon, dans son propre code, sans toucher au socle. Pour le kraken du *Trésor des pirates*, cela donne une fenêtre et un temps de recharge réglables :

```c
TB_CONFIG_GENERATE_PARSER(kraken, KRAKEN, "kraken",
    TB_FIELD_UINT32_RANGE(kraken_config_t, window,   5000, 500, 60000),
    TB_FIELD_UINT32_RANGE(kraken_config_t, cooldown, 3000, 100, 30000));
TB_CONFIG_DEFINE_GETTER(kraken, "kraken", true);
// usage : uint32_t window_ms = kraken_get_config()->window;
```

À la compilation, CMake découvre tous les fichiers `*_config_schema.h`, et des générateurs Python produisent deux choses : un en-tête de validation pour le firmware, et des **JSON Schemas** à partir desquels la page web de l'accessoire construit ses formulaires. Ajouter un réglage, c'est ajouter une ligne : le parsing, la validation, le getter typé et le champ de formulaire suivent. Le fichier de configuration final ressemble à ceci (valeurs d'exemple) :

```json
{
  "tb_mqtt":        { "broker": "mqtt://<broker>", "port": 1883,
                      "sub_topic": "/my_prop_sub", "pub_topic": "/my_prop_pub" },
  "tb_network":     { "hostname": "room-my-prop" },
  "tb_ntp":         { "enabled": true, "timezone": "CET-1CEST-2,M3.5.0/02:00:00,M10.5.0/03:00:00" },
  "log":            { "enabled": true, "max_file_size": 65536, "log_level": "warn" },
  "tb_recovery":    { "stable_uptime_s": 3600 },
  "my_prop":        { "tolerance_ms": 800 }
}
```

### Toujours avoir un plan B, et un plan C

Une configuration peut être perdue de plein de façons : une écriture interrompue par une coupure, un fichier tronqué envoyé depuis la page web, une erreur de saisie. Le chargement suit donc une chaîne de repli, décidée par une fonction pure :

1. le fichier courant, s'il est valide ;
2. sinon, la **dernière configuration valide connue** (`tb_config.last-known-good.json`) ;
3. sinon, les **valeurs d'usine** (`tb_config.factory-defaults.json`) ;
4. sinon, les valeurs par défaut des schémas, avec le point d'accès Wi-Fi de secours joignable.

Le fichier invalide n'est pas supprimé : il est renommé `tb_config.invalid.json` pour le diagnostic. Et un fichier présent mais *illisible* (mémoire insuffisante, erreur d'E/S) n'est jamais écrasé, car il est peut-être parfaitement valide.

Les modifications depuis la page web sont fusionnées avec la configuration courante, validées, puis écrites de façon atomique (fichier temporaire, puis renommage). Le bouton « Appliquer maintenant » réapplique le Wi-Fi, MQTT, le réseau, NTP et les logs à chaud ; si l'application échoue, la configuration revient à la dernière version valide.

## Ne jamais rester planté

### Le watchdog et le core dump

Le watchdog de tâches d'ESP-IDF est réglé à 10 s, avec panic, et surveille les tâches *idle* des deux cœurs ainsi que les tâches du socle (GPIO, MQTT, logs en direct). En cas de plantage, un core dump est écrit dans une partition dédiée de 64 Ko, lisible ensuite avec `idf.py coredump-info`.

### Le mode sans échec

Un watchdog qui redémarre la carte, c'est bien. Une carte qui redémarre en boucle parce que sa configuration la fait planter, c'est un accessoire perdu pour la journée. Le composant `tb_recovery` compte donc les démarrages ratés :

- comptent comme des échecs : panic, watchdogs, baisse de tension (*brownout*) et redémarrage logiciel **non annoncé** ;
- ne comptent pas : mise sous tension, reset externe, et redémarrages annoncés (reboot depuis la page web, OTA, retour aux valeurs d'usine) ;
- un démarrage n'est considéré comme réussi qu'après `stable_uptime_s`, **60 minutes par défaut**. C'est la durée d'une partie : un plantage au milieu d'une session compte comme un échec.

Au **deuxième échec consécutif**, la carte démarre en mode sans échec : configuration restaurée, Wi-Fi et page web actifs, mais **pas de MQTT**. Elle reste joignable pour le diagnostic, et surtout hors du jeu : un accessoire instable ne doit pas envoyer d'événements fantômes au moteur de la salle.

La décision est une fonction pure, sans aucun appel matériel :

```c
// A software restart announced by tb_recovery_note_intentional_restart()
// (reboot, OTA, factory reset from the web page) is not a crash: like a
// power-on, it resets the counter.
const bool announced_restart =
    previous->intentional_restart_pending && reason == ESP_RST_SW;
decision.user_requested = previous->safe_mode_requested;

if (previous->boot_marked_successful || announced_restart ||
    tb_recovery_reason_resets_counter(reason)) {
  next->failed_boot_count = 0;
} else if (!decision.user_requested && tb_recovery_reason_is_crash(reason)) {
  next->failed_boot_count = previous->failed_boot_count + 1;
  decision.counted_as_failure = true;
}
decision.repeated_failures = next->failed_boot_count >= TB_RECOVERY_SAFE_MODE_THRESHOLD;
decision.safe_mode_active  = decision.user_requested || decision.repeated_failures;
```

Parce qu'elle est pure, elle se teste sur PC, cas par cas, comme on le verra plus bas.

### Des logs qui survivent au redémarrage

Les macros `TB_LOGx` s'utilisent comme `ESP_LOGx`, mais poussent aussi chaque ligne dans un tampon de 8 Ko en RAM. Une tâche d'écriture de faible priorité vide ce tampon vers `app.log` toutes les 500 ms, ou dès qu'il est rempli au quart, avec rotation à une taille configurable. Juste avant un redémarrage volontaire, le tampon est vidé en au plus une seconde.

Cette architecture asynchrone vient d'un bug trouvé sur le banc : l'ancienne version écrivait en flash sous mutex, et sur une carte sans réseau, **65 lignes de log ont été perdues en 30 minutes** pour cause de contention. Aujourd'hui, l'appelant n'attend jamais plus de 10 ms, et les lignes perdues sont *comptées* : une ligne « N log line(s) lost » est écrite, et un compteur est exposé par l'API. On peut perdre des logs ; on ne doit pas perdre le fait qu'on en a perdu.

## Le réseau : Ethernet d'abord, Wi-Fi quand il le faut

### Ethernet

Dans une salle pleine de métal, l'Ethernet est le choix par défaut dès que la carte le permet. Le composant `tb_lan8720` pilote le MAC interne de l'ESP32 avec un PHY LAN8720 (Kincony, Olimex ESP32-POE, WT32-ETH01). ESP-IDF v6 a supprimé les options Kconfig de l'horloge RMII ; le composant les redéclare avec les mêmes noms, chargées uniquement en v6, pour que les `sdkconfig.defaults` des firmwares existants continuent de fonctionner sans modification.

### Wi-Fi, et un décalage de bits

Le Wi-Fi se reconnecte indéfiniment avec un backoff de 1, 2, 4, 8, 16 s, puis toutes les 30 s. La première version calculait le délai avec `BASE * (1 << retry_num)`, sans borne. Après environ 13 minutes sans point d'accès, le décalage débordait, le délai tombait à zéro, et la carte tentait de se reconnecter en boucle serrée. La logique est désormais une fonction pure et bornée :

```c
uint32_t tb_wifi_retry_delay_ms(int retry_num) {
  if (retry_num < 0) {
    retry_num = 0;
  }
  // 1000 << 5 = 32000 is already above the maximum: no shift beyond 4
  if (retry_num >= 5) {
    return TB_WIFI_RETRY_MAX_DELAY_MS;
  }
  uint32_t delay_ms = (uint32_t)TB_WIFI_RETRY_BASE_DELAY_MS << retry_num;
  return (delay_ms > TB_WIFI_RETRY_MAX_DELAY_MS) ? TB_WIFI_RETRY_MAX_DELAY_MS : delay_ms;
}
```

### MQTT, volontairement simple

Le client MQTT de l'accessoire fait peu de choses, mais les fait de façon prévisible : QoS 0, topics exacts (pas de jokers), au plus 10 callbacks, des messages jusqu'à 1 Ko. Une tâche surveillée par le watchdog vérifie la connexion toutes les 5 s et se réabonne à chaque reconnexion.

Quelques décisions de conception :

- **Une carte neuve n'a pas de broker.** L'initialisation réussit quand même, et le client démarre dès qu'un broker est enregistré depuis la page web, sans redémarrage.
- **Le topic d'abonnement peut changer à chaud.** Le client se désabonne de l'ancien et déplace tous les callbacks vers le nouveau.
- **Les callbacks s'exécutent dans la tâche MQTT.** C'est le piège numéro un, documenté en gras : pendant qu'un callback tourne, aucun autre message n'est lu. On y stocke la commande (une variable, une file FreeRTOS) et une tâche de l'accessoire agit ensuite.

La revue de code a trouvé deux bugs savoureux dans la réception. Le tampon de payload n'était pas terminé par un zéro, si bien que `"30"` pouvait être lu comme `"30_lights_on…"`, reste du message précédent. Et la comparaison des topics ne vérifiait pas les longueurs : un topic qui était le préfixe d'un autre le capturait.

### NTP, en local

Les logs ont besoin d'une heure réelle. Le socle utilise SNTP avec une chaîne de fuseau POSIX ; avant la synchronisation, les lignes sont horodatées `boot+<ms>ms`. Dans les salles sans Internet, il faut un serveur NTP local, et pour mesurer des durées, on utilise `esp_timer_get_time()`, jamais l'heure murale.

## Les entrées-sorties : le polling, assumé

### Pourquoi pas tout en interruptions ?

Le composant `tb_gpio` lit toutes les entrées déclarées depuis une tâche qui tourne toutes les 10 ms, puis leur applique un filtre : anti-rebond (50 ms par défaut, réglable par broche) ou moyenne glissante exponentielle pour les entrées analogiques. La question « pourquoi pas des interruptions ? » revient régulièrement. La réponse est écrite dans la feuille de route, et elle est entièrement dictée par le terrain :

> Escape-room wiring is long and sits next to switching relays and electromagnets. Polling plus debounce ignores glitches; an interrupt fires on each one, and a floating or noisy line can flood the CPU until the interrupt watchdog resets the board.

S'y ajoutent les contacts mécaniques qui rebondissent (il faudrait de toute façon un anti-rebond temporisé derrière l'interruption), les expandeurs et les entrées analogiques qui resteraient de toute façon en polling, et des accessoires en production validés ainsi. Les interruptions existent, mais en option, pour les impulsions trop courtes pour le polling : une bille qui traverse un faisceau infrarouge en 15 ms environ. Dans ce cas, l'ISR pousse des fronts horodatés dans une file, et tient un compteur de fronts par broche.

Pour la diversité de câblage évoquée plus haut, une option `invert_logic` inverse réellement les lectures, les callbacks et les écritures d'une broche. Un capteur monté à l'envers sur un site devient une ligne de configuration, pas une branche de code.

### Le relais qui claquait

Les cartes Kincony pilotent leurs relais via des PCF8574, des expandeurs 8 bits sur I2C. Le PCF8574 n'a pas de registre de direction : on écrit l'octet complet du port à chaque fois. La revue sur carte a trouvé deux défauts gênants dans une salle :

- **Un relais claquait à chaque écriture.** La fonction d'écriture repassait la broche en sortie, ce qui envoyait le bit à 0, *puis* la valeur réelle.
- **Tous les relais pulsaient au démarrage.** La copie locale du port démarrait à `0x00` : la première écriture tirait toutes les sorties vers le bas un instant.

La correction tient en peu de lignes : une copie locale (*shadow*) initialisée à `0xFF`, et **une seule transmission par écriture**, qui ne change que le bit visé.

```c
esp_err_t tb_pcf8574_port_write(uint8_t *port, uint8_t io, int32_t value,
                                tb_pcf8574_tx_t tx, void *ctx) {
  if (port == NULL || tx == NULL || io >= PCF8574_LOGIC_NB_PINS) {
    return ESP_ERR_INVALID_ARG;
  }
  uint8_t next;
  if (value) {
    next = *port | (uint8_t)(1U << io);
  } else {
    next = *port & (uint8_t)~(1U << io);
  }
  esp_err_t err = tx(ctx, next);
  if (err == ESP_OK) {
    *port = next;
  }
  return err;
}
```

La copie locale n'est mise à jour qu'après une transmission réussie. Cette logique est testée de façon exhaustive : 256 états de port, 8 broches et 2 valeurs.

### L'expandeur débranché qui redémarrait la carte

Autre trouvaille : débrancher un expandeur I2C faisait redémarrer la carte. Les transactions I2C utilisaient un timeout de `-1` (attente infinie) dans une tâche surveillée par le watchdog de 10 s. La documentation du composant le dit maintenant sans détour : « A `-1` timeout blocks forever… that reboots the board. » Les timeouts sont désormais de 50 ms, une lecture ratée renvoie `-1` (valeur inconnue), et un expandeur qui ne répond plus produit *une* ligne de log quand il tombe et une quand il revient, pas une par tick.

### RFID : un lecteur muet n'est pas un objet absent

Le composant RFID gère des lecteurs PN532 (jusqu'à 8 sur un même bus SPI), avec une réinitialisation automatique après 20 lectures ratées, qui réapplique le gain et le niveau RF. Deux choix méritent d'être soulignés :

- les tentatives passives du PN532 sont limitées à 1, parce que le comportement par défaut (réessayer indéfiniment) « makes an empty reader look like a dead one » ;
- côté API, **« pas de badge » n'est pas une erreur** : `ESP_OK` avec une longueur d'UID à zéro. Une erreur signifie que *le lecteur* ne répond pas. C'est ce qui permet à un accessoire de signaler un lecteur en panne au lieu d'attendre éternellement un objet.

Les UID attendus sont stockés sous forme de chaînes dans la configuration. Les vases canopes de notre salle égyptienne (quatre lecteurs RFID) fonctionnent ainsi : remplacer une statuette perdue est un changement de configuration, pas un nouveau firmware.

### Et la lumière

Le composant NeoPixel pilote des rubans WS2812 en RGB, hexadécimal ou HSV, avec la bibliothèque mathématique 8 bits de FastLED portée en C pour les animations. La luminosité est plafonnée par défaut à 50/255. Ce n'est pas qu'esthétique : à 60 mA par LED en blanc plein, un ruban qui s'allume d'un coup peut faire chuter une alimentation partagée avec un électroaimant.

## Une page web dans chaque accessoire

Chaque accessoire sert une page web depuis sa partition LittleFS, accessible par son nom mDNS (`http://<hostname>.local/`) :

- l'**état en direct** des entrées-sorties, en SSE (au plus 10 clients par flux, *keepalive* toutes les 25 s) ;
- l'**éditeur de configuration**, dont les formulaires sont générés à partir des schémas ;
- les **logs**, un explorateur de fichiers, le redémarrage, le mode sans échec, le retour aux valeurs d'usine ;
- la **mise à jour OTA** du firmware, et la mise à jour séparée de l'interface web (`www_update.tar`).

C'est l'outil de diagnostic de premier niveau : sur le réseau de la salle (ou via le VPN), un navigateur suffit pour voir ce que voit l'accessoire.

## OTA signée et releases reproductibles

La table de partitions est la même pour tous les accessoires : deux emplacements d'application de 1,25 Mo (`ota_0`, `ota_1`), 1,4 Mo de LittleFS pour la configuration, les logs et la page web, et 64 Ko pour le core dump.

La CI construit le firmware sur chaque tag, avec ESP-IDF épinglé, et publie l'ensemble des artefacts : application, bootloader, table de partitions, image de stockage, archive de l'interface web, paquet de flash complet. Elle produit aussi un **manifeste** (version, commit, taille et SHA-256 de chaque artefact), **signé en RSA-3072**. La clé publique est injectée à la compilation dans un en-tête généré, et l'accessoire vérifie la signature avec mbedTLS, puis la taille et l'empreinte de l'artefact, avant d'accepter une mise à jour. Les builds sont reproductibles (`CONFIG_APP_REPRODUCIBLE_BUILD=y`, pas de date de compilation dans le bootloader).

Sur le terrain, la mise à jour passe par la page web : le firmware est écrit dans l'emplacement inactif, et la configuration est conservée. Une règle de notre base de maintenance mérite d'être citée : **ne jamais reflasher les vases canopes par le port série**. `idf.py flash` réécrit aussi la partition de stockage, et efface donc les statuettes apprises. On passe par l'OTA, ou par `idf.py app-flash`, qui ne touche qu'à l'application.

Ce qui manque encore : le retour arrière automatique sur l'ancienne partition si le nouveau firmware ne démarre pas. C'est sur la feuille de route.

## Brancher un accessoire sur Maestro

Côté moteur de salle, un accessoire ESP32 est un **fournisseur électronique** de plus, avec son format de message (`tb_esp32`) et ses topics :

```json
{
  "key": "esp32_digicode",
  "message_format": "tb_esp32",
  "input_topics": [ "/games/chocolaterie/rooms/iceroom/digicode/state" ],
  "output_topic": "/games/chocolaterie/rooms/iceroom/digicode"
}
```

Quand le digicode publie `{"module":"in2","pin":0,"value":"off"}`, l'agent électronique de Maestro en déduit la clé `esp|min2|p0|voff`, la compare aux proxys déclarés dans le gameflow, et publie une notification métier (`digicode_unlocked`) à l'orchestrateur de la salle. Dans l'autre sens, la commande `trappe_unlock` devient `{"command":"set","data":[{"module":"out1","pin":0,"value":"on"}]}`. La même salle combine sans difficulté des panneaux d'E/S tiers et des accessoires ESP32 : chaque fournisseur a son broker, ses topics et son format.

Le protocole de la carte Kincony est volontairement minimal :

```json
{"command":"set","data":[{"module":"out1","pin":2,"value":"on"},{"module":"out2","pin":3,"value":"off"}]}
{"command":"get","data":[{"module":"in1","pin":3}]}
{"command":"get_all_infos"}
```

`get_all_infos` renvoie l'état complet de la carte : plateforme (version du SDK, mémoire libre, uptime), réseau, MQTT, stockage, et chaque GPIO avec son filtre, son type et sa valeur. Côté Maestro, la page technique de la console GM utilise `get` pour relire l'état des broches à la demande.

## Tester un firmware sans carte

Une bonne partie de la logique du socle se teste **sans ESP32**. ESP-IDF sait compiler pour une cible `linux`, et nous en profitons : 9 applications de test, **124 tests Unity**, compilés et exécutés sur PC par la CI.

| Composant | Tests |
|---|---|
| `tb_config` | 49 |
| `tb_gpio` | 21 |
| `tb_recovery` | 13 |
| `tb_mqtt` | 12 |
| `tb_pcf8574` | 8 (dont un test exhaustif) |
| `tb_neopixel` | 7 |
| `tb_log` | 6 |
| `tb_http_server` | 5 |
| `tb_wifi` | 3 |

La méthode est toujours la même : plutôt que de bouchonner un composant entier, on **isole la logique de décision dans un fichier pur** (`tb_wifi_logic.c`, `tb_recovery_decision.c`, `tb_pcf8574_logic.c`, `tb_config_source.c`) et on le teste directement. Les noms des tests citent les critères d'acceptation des spécifications (`test_crit10_safe_mode_at_two_failures`, `test_crit37_retry_delay_never_zero`).

La CI compile aussi un firmware qui utilise *tous* les composants, avec ESP-IDF 5.5.2 **et** 6.0.2, vérifie la découverte des schémas (y compris l'ajout et le retrait d'un schéma entre deux builds), et surveille le budget flash : le firmware de la carte Kincony doit garder au moins 3,5 % de marge.

## Une revue sur carte, et une règle de priorité

En octobre, nous avons passé le socle en revue complète : documentation, lecture de code et tests sur banc avec une carte de développement. Le résultat est une liste de défauts, dont certains sont décrits plus haut. Deux autres méritent d'être racontés :

- **Une configuration tronquée sortait la carte du réseau.** Envoyée depuis la page web, elle faisait retomber tous les composants sur leurs valeurs codées en dur. Pire : un enregistrement juste après écrasait la dernière configuration valide avec `{}`. D'où la chaîne de repli décrite plus haut, et la règle de ne jamais écrire la dernière configuration valide à partir d'une configuration non chargée.
- **« MQTT reconnected successfully » était journalisé trop tôt**, quand la demande de reconnexion était acceptée, pas quand la connexion était établie. Un log qui ment est pire que pas de log.

Les corrections ont été regroupées en lots, avec une règle d'ordonnancement qui résume assez bien notre métier :

> Ordering rule: what stops a prop in the middle of a session comes first, then what fails silently on props already in production (MQTT, network), then what breaks a build or a new project, then hardware drivers, then comfort and diagnostics.

La feuille de route compte 45 chantiers : 31 terminés, 2 en cours, 12 à faire (dont le retour arrière OTA et des tests d'intégration).

## Pour comparaison : l'Arduino qui répondait au ping

Tous nos accessoires ne sont pas sur ce socle. Les trappes de l'ascenseur de la Chocolaterie tournent sur un Arduino Uno avec un shield Ethernet W5500, et elles m'ont appris une leçon que je raconte dans le [premier article](./team-break-architecture.fr.md) : la carte se figeait de temps en temps, mais répondait toujours au ping, parce que le W5500 gère l'ICMP tout seul. La bibliothèque Ethernet attendait le contrôleur dans des boucles sans timeout, et les parasites d'un électroaimant suffisaient à la bloquer.

La solution ressemble, en miniature, à ce que fait le socle ESP32 : un watchdog matériel de 4 s, des timeouts réseau courts, plus aucune boucle bloquante, l'électroaimant alimenté dès le début de `setup()`. Et un fil d'Ariane qui survit au reset :

```c
struct Persist { uint16_t magic; uint16_t bootCount; uint8_t step; };
Persist persist __attribute__((section(".noinit")));  // survit au reset, pas à la coupure
// setup() : MCUSR = 0; wdt_disable(); warmBoot = (persist.magic == 0xC0C1);
// loop()  : wdt_reset(); persist.step = STEP_MQTT_LOOP; client.loop(); ...
```

À chaque connexion, la carte publie un message de diagnostic *retained* avec le nombre de démarrages, le type du dernier reset et l'étape où se trouvait le code juste avant (`reset_step`). Le tout tient dans 829 octets de RAM sur les 2 048 disponibles, grâce aux chaînes en `F()` et à l'abandon de `String`.

## Ce que j'en retiens

1. **Le matériel d'abord.** Un actionneur doit être dans un état sûr avant que la carte sache qu'elle a une adresse IP.
2. **Un accessoire instable sort du jeu.** Le mode sans échec garde la carte joignable mais coupe MQTT : mieux vaut un accessoire muet qu'un accessoire qui envoie des événements fantômes.
3. **Les décisions sont des fonctions pures.** Reconnexion, récupération, choix de configuration, écriture de port : tout ce qui décide se teste sur PC, cas par cas.
4. **La configuration est une donnée, avec un plan B et un plan C.** Elle est validée, versionnée par schémas, et ne doit jamais pouvoir rendre une carte injoignable.
5. **Le terrain décide.** Polling plutôt qu'interruptions, Ethernet plutôt que Wi-Fi, anti-rebond, logique inversable par configuration : chaque choix vient d'une salle réelle.
6. **Mettre à jour le socle ne casse aucun accessoire.** C'est une règle écrite dans chaque spécification, et la CI la vérifie.
7. **Un firmware se livre comme un logiciel.** Signé, reproductible, publié par la CI, mis à jour à distance sans toucher à la configuration.

Un bon accessoire, c'est celui dont personne ne parle : le coffre s'ouvre quand il faut, la statuette est reconnue du premier coup, et le jour où quelque chose ne va pas, la carte le dit au lieu de se taire.
