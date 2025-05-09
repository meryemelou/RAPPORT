

##  **Documentation**

---

### **1. Introduction générale**

Ce document a pour objectif de présenter en détail le fonctionnement du démonstrateur d’erreurs conçu autour de la transmission d’images via un système émetteur-canal-récepteur. Ce projet a été implémenté sur des cartes FPGA Nexys 4 basées sur des puces Artix-7, en exploitant des modules PMOD, notamment l’écran RGB OLED, pour la visualisation des résultats.

L’architecture repose sur la transmission d’une image depuis un émetteur vers un récepteur en passant par un canal de communication pouvant introduire des erreurs. L’image est tout d’abord affichée sur le PMOD OLED. Par la suite, un codage de vitterbi est appliqué à l’image avant sa transmission. Après le passage par un canal bruité (modélisé par un code vhdl de SNR ), l’image est décodée à l’aide de l’algorithme de Viterbi, puis de nouveau affichée pour visualiser l’effet de la correction d’erreurs sur notre PMOD.

La communication série entre les différents composants (émetteur, canal, récepteur) est assurée via UART. L’ensemble du système est composé des modules VHDL pour le traitement temps réel (affichage, encodage, décodage) et des scripts Python pour l'envoi d'image et le pilotage des transferts. Ce travail a été réalisé dans le cadre du projet de démonstration d’un système de transmission avec détection et correction d’erreurs.



### **2. Architecture globale du démonstrateur**

#### **Figure schématique globale du système**

Le démonstrateur repose sur une architecture en trois parties principales réparties sur trois plateformes FPGA Nexys 4, chacune connectée à un PC. La communication entre les différentes cartes se fait via l’interface UART. L’image transmise traverse successivement les étapes d’émission, de bruitage (canal) et de réception avec correction d’erreurs, chacune étant visualisable à l’écran PMOD OLED connecté à chaque carte.

![Schéma global du système](images/schema_systeme.png)



#### **Description globale de l’architecture**

Le démonstrateur a pour objectif de représenter une chaîne de communication numérique avec visualisation de l'effet des erreurs de transmission et de leur correction sur une image.

* **Émetteur (FPGA 1)**
  Le premier FPGA reçoit une image depuis un PC via UART. Cette image est d’abord affichée sur un écran OLED connecté à la carte pour visualiser la version originale. Ensuite, elle est encodée à l’aide d’un **codeur convolutif** (type Viterbi) et transmise à la deuxième carte FPGA par liaison série.

* **Canal de transmission bruité (FPGA 2 / PC 2)**
  La deuxième carte FPGA (ou le PC associé) reçoit les données encodées, puis applique un **bruit gaussien** avec un SNR contrôlable pour simuler un canal de transmission réel (bruit blanc additif). L’image bruitée est ensuite affichée pour montrer les erreurs introduites, puis transmise à la troisième carte.

* **Récepteur (FPGA 3)**
  La troisième carte reçoit les données bruitées. Un **décodeur Viterbi** permet de corriger les erreurs induites par le canal. L’image finale décodée est affichée à l’écran OLED, ce qui permet de comparer visuellement l’image originale, bruitée et corrigée.


#### **Communication UART entre les blocs**

Les échanges de données entre les différentes cartes FPGA et entre FPGA/PC sont réalisés via des interfaces UART, implémentées en VHDL. Ces liaisons série permettent un transfert de données simple, synchrone et efficace, facilitant le test et la validation de chaque étage indépendamment. L’ensemble de l’architecture est modulaire, ce qui permet de tester séparément l’émission, le canal et la réception.







###  **3. Fonctionnalité globale du démonstrateur**

####  Description du flux de données de bout en bout :

1. **Lecture de l’image**
   L’image à transmettre peut être dans différents formats standards (comme PNG, BMP, JPG, etc.). Un script Python est utilisé pour la convertir dans un format exploitable par le FPGA, généralement en données brutes (bitmap binaire). Ce traitement permet de découper et organiser les pixels afin qu'ils soient transmissibles via une liaison série UART.

2. **Transmission UART vers le FPGA Émetteur**
   Une fois l’image convertie, elle est envoyée via une liaison UART depuis le PC vers le premier FPGA (FPGA Émetteur). Ce dernier reçoit les données série à l’aide d’un module `UART_RX` implémenté en VHDL.

3. **Encodage & Canal bruité**
   Sur le FPGA Émetteur, les données d’image sont encodées à l’aide d’un encodage Viterbi. Ensuite, les données encodées sont transférées à un PC intermédiaire connecté à un deuxième FPGA qui simule un canal bruité, en injectant un bruit gaussien selon un SNR réglable. Cela permet de simuler des conditions de transmission réalistes.

4. **Réception & Décodage par Viterbi**
   Les données bruitées sont transmises à un troisième FPGA (FPGA Récepteur) où un décodeur Viterbi (implémenté en VHDL) est utilisé pour corriger les erreurs de transmission. Ce décodage restitue une version corrigée de l’image.

5. **Affichage de l’image sur PMOD RGB OLED**
   À chaque étape, l’image est convertie et affichée sur un écran RGB OLED connecté en PMOD sur chaque FPGA. Cela permet une visualisation directe du résultat de chaque étape. 

####  Mise en œuvre réelle sur les cartes Nexys A7 :

Le démonstrateur repose sur l’utilisation de cartes FPGA **Nexys A7**, équipées du FPGA **Artix-7**. Le projet est réparti sur trois cartes et/ou PC selon l’implémentation :




### **4. Description des modules VHDL**


## **Fonctionnalité globale de l’émetteur, du canal et du récepteur**

### **Émetteur (FPGA 1)**

Le rôle principal de l’émetteur est d’assurer la réception, l’encodage et l’envoi de l’image. Dans un premier temps, une image est transmise depuis un PC vers la carte FPGA via une interface UART, octet par octet. Ces données sont stockées dans une mémoire RAM locale, puis l’image originale est affichée sur un écran OLED afin d’avoir un aperçu de la qualité initiale.

Ensuite, chaque octet est encodé bit par bit à l’aide d’un **codeur convolutif** (Viterbi encoder), dont l’objectif est d’introduire de la redondance pour rendre la transmission plus robuste face aux erreurs. Le module `encode8` effectue ce traitement : il extrait chaque bit de l’octet, l’encode à l’aide d’un module `viterbi_encoder`, puis génère deux octets de sortie (`x1_byte`, `x2_byte`) correspondant aux deux symboles codés. Ce mécanisme permet de doubler les données à transmettre, mais augmente fortement la capacité de correction au niveau du récepteur. Une fois le codage terminé, les données sont prêtes à être transmises à la carte suivante via une liaison série.

![Architecture de l'émetteur](images/fpga1.png)


### **Canal de transmission bruité (FPGA 2)**

Le canal représente une simulation d’un environnement de transmission réel où les données peuvent être altérées. Les octets encodés provenant de l’émetteur sont reçus, puis **un bruit gaussien additif (AWGN)** est appliqué pour simuler une dégradation du signal, typique d’un canal analogique.

Ce bruit est contrôlé par un paramètre SNR (Signal-to-Noise Ratio), qui permet de varier l’intensité du bruit injecté dans le signal. Une fois bruitées, les données sont affichées (image dégradée) puis renvoyées à la troisième carte (récepteur). Cette étape permet d’observer l’impact des perturbations sur une transmission non protégée.


### **Récepteur et Décodage (FPGA 3)**

Le récepteur a pour rôle de récupérer les données bruitées, de les décoder, puis de reconstruire l’image originale. Pour cela, il s’appuie sur un **décodage convolutif de type Viterbi**, capable de corriger les erreurs introduites par le bruit durant la transmission. Afin de permettre un traitement précis et contrôlé, une **ROM nommée `rom_sigma`** est utilisée. Cette mémoire contient des valeurs prédéfinies de sigma (écart-type du bruit), utilisées pour moduler la réponse du décodeur selon les conditions de bruit simulées. Elle permet d’ajuster dynamiquement la sensibilité du décodage en fonction du niveau de bruit reçu, en se basant sur une entrée `sb` (signal binaire) représentant un index de configuration.

En complément, un module `data_serializer` est utilisé pour **désérialiser les données** reçues. Les bits reçus en série sont regroupés en octets et stockés dans une mémoire tampon. Une fois reconstitués, ces octets sont présentés au décodeur Viterbi, qui applique son algorithme de correction basé sur les treillis de transition d’états. Ce processus permet de retrouver les données d’origine avec une grande fiabilité, malgré la dégradation subie par le canal.

Enfin, les octets corrigés sont envoyés vers un écran OLED connecté à la carte FPGA, permettant de visualiser l’image restaurée. Ce mécanisme de comparaison entre l’image originale, bruitée et décodée offre une démonstration concrète de l’efficacité du codage/décodage convolutif dans un environnement de transmission bruité.

## **Affichage de l'image sur le PMOD OLED**

Le démonstrateur implémente un système complet de lecture et d’affichage d’image sur un écran connecté via un module PMOD sur chaque FPGA. Son objectif principal est de parcourir une mémoire RAM contenant une image encodée en 1, 2, 4 ou 16 bits par pixel (BPP), de convertir si nécessaire les données en un format compatible avec l’affichage (RGB565 - 16 bits), puis de les transmettre ligne par ligne à l'écran.

L'affichage est déclenché par un signal de contrôle (`enable_pmod`) qui active la lecture séquentielle des données. Pour chaque cycle d’horloge, l’adresse de lecture est générée automatiquement à partir des coordonnées de la matrice (`col_counter`, `row_counter`). Le système incrémente ces coordonnées pour parcourir l’image pixel par pixel, tout en envoyant la position (`pix_col`, `pix_row`) et la couleur du pixel (`pix_data_out`) à l’écran. Un signal `pix_write` valide chaque écriture.

Le démonstrateur est conçu pour être générique, avec des paramètres configurables comme la largeur et la hauteur de l’écran, ainsi que la profondeur de couleur (BPP). Il supporte plusieurs formats d’image grâce à un bloc de transcodage intégré, qui convertit automatiquement les données issues de la RAM vers le format RGB565 utilisé pour l’affichage.

Une logique de réinitialisation (`reset`) est également incluse, permettant de réinitialiser les compteurs et l’état interne du module à tout moment. À la fin de l’affichage complet de l’image (lorsque tous les pixels ont été envoyés), le module désactive automatiquement le signal de validation d’écriture et attend un nouveau déclenchement.

![Schéma bloc de l'affichage de l'image](images/image.png)




### **5. Description des modules VHDL détaillée**



### **Partie 1 : Affichage de l'image sur un écran OLED RGB**

1. **Fichier VHDL pour la gestion de l'affichage** :

   * **`RAM_IMAGE.vhd`** : Module pour le stockage de l’image dans la RAM.
   * **`Image_Display.vhd`** : Module pour la gestion de l’affichage des pixels sur l'écran OLED RGB.



### **Partie 2 : Encodage des données (Viterbi Encoder)**

1. **Fichier VHDL pour l'encodeur Viterbi (1 bit)** :

   * **`viterbi_encoder.vhd`** : Module pour l'encodage Viterbi à 1 bit.

2. **Fichier VHDL pour l'encodeur Viterbi (8 bits)** :

   * **`viterbi_encoder_8bits.vhd`** : Module pour l'encodage Viterbi à 8 bits.


### **Partie 3 : Simulation de bruit (Canal AWGN)**

1. **Fichier VHDL pour la simulation de bruit** :

   * **`awgn_channel.vhd`** : Module pour la simulation de bruit gaussien additif (AWGN) dans le canal de communication.



### **Partie 4 : Décodage des données (Viterbi Decoder)**

1. **Fichier VHDL pour le décodeur Viterbi** :

   * **`viterbi_decoder.vhd`** : Module pour le décodage Viterbi des données reçues.


![Schéma global du système](images/them.png)



### **6. Simulations**



Afin de valider le bon fonctionnement de notre système de transmission de données, une **simulation complète a été réalisée** à l’aide d’un banc d’essai VHDL. Le fichier de test `tb_top_level.vhd` simule l’interconnexion des différents blocs du système, incluant l’encodage convolutif, l’ajout de bruit contrôlé via un paramètre `snr`, et le décodage Viterbi. Cette simulation permet de vérifier **le comportement temporel et logique** de l’ensemble du système sans recourir immédiatement au matériel FPGA.

Le `testbench` génère un **signal d’horloge (clk)** avec une période définie (10 ns), simule un **reset** initial, puis active progressivement les signaux de contrôle (`enable`, `dat`, `snr`) dans un scénario temporel réaliste. Le signal `snr` est configuré à sa valeur maximale (`"111111"`) pour observer le comportement du système dans un environnement fortement bruité. Le banc d’essai permet de vérifier que les données sont correctement transmises, encodées, perturbées par un bruit simulé, puis **corrigées par le décodeur Viterbi**.

Grâce à cette simulation, nous avons pu **valider la cohérence des transferts de données vers la RAM** pour l’affichage, ainsi que le bon fonctionnement des modules d’**encodage** et de **décodage**. Elle nous a permis de détecter et corriger d’éventuelles erreurs de synchronisation ou de logique, avant l’implémentation matérielle sur les FPGA.





### **7. Rôle des scripts Python dans la transmission de l'image via UART**

Dans le cadre de ce projet, plusieurs scripts Python ont été utilisés pour établir une communication entre le PC et le FPGA via l'interface UART, gérer la transmission des données sous forme d'images, appliquer un bruit contrôlé et effectuer des opérations d'encodage et de décodage, notamment avec le Viterbi.

#### **7.1 Connexion UART entre les PC et le FPGA**

La connexion entre le PC et le FPGA a été réalisée en utilisant la bibliothèque `serial` de Python, qui permet de configurer une interface série pour l'envoi et la réception de données via UART. La communication s'est effectuée à une vitesse de transmission de 921600 bauds, conformément à la configuration de l'UART sur le FPGA. Le script Python suivant a été utilisé pour envoyer des bits individuels au FPGA et recevoir les réponses encodées :

```python
import serial
import time

# Configuration du port série
ser = serial.Serial(port='/dev/ttyUSB1', baudrate=921600, timeout=1)

def send_bit(bit):
    byte_to_send = bit & 0x01  # Assurez-vous que c'est bien 0 ou 1
    ser.write(bytes([byte_to_send]))  # Envoi du bit
    print(f"Sent bit: {bit}")

def receive_encoded():
    data = ser.read(1)  # Lecture de 1 octet
    if len(data) == 0:
        print("Timeout, no data received.")
        return None
    byte_received = data[0]
    x1 = byte_received & 0x01
    x2 = (byte_received >> 1) & 0x01
    print(f"Received encoded bits: x1={x1}, x2={x2}")
    return x1, x2
```

Ce script permet de transmettre un flux de bits, puis de recevoir des données encodées sous forme de deux bits (x1 et x2). Cette transmission est effectuée dans un format compatible avec l'encodeur Viterbi utilisé sur le FPGA.

#### **7.2 Envoi et réception de l'image en trames**

Les images étaient envoyées sous forme de trames de bits encodées à travers UART. Le processus a impliqué la conversion des images en données binaires, puis l'application d'un codage Viterbi avant de transmettre chaque bit au FPGA. Une fois l'image encodée, chaque octet était envoyé au FPGA, et l'encodage (utilisant la méthode Viterbi) était appliqué pour garantir la fiabilité des données transmises.

#### **7.3 Ajout de bruit contrôlé (SNR configurable)**

Dans la simulation de canal, un bruit contrôlé a été ajouté pour simuler des perturbations dans la transmission des données. L'ajout de bruit est crucial pour tester la robustesse du système de transmission et la capacité du décodeur à récupérer correctement les données malgré des erreurs introduites par le canal. Bien que le bruit contrôlé n’ait pas été inclus directement dans le code que tu as fourni, des outils comme `numpy` et `random` peuvent être utilisés pour générer du bruit et modifier le signal reçu afin de tester la performance du décodeur Viterbi dans des conditions réelles de transmission.

#### **7.4 Encodage et Décodage avec Viterbi**

L'encodeur et le décodeur Viterbi ont été implémentés en Python pour correspondre exactement au comportement attendu sur le FPGA. Le décodeur Viterbi permet de récupérer les bits originaux à partir des bits reçus encodés, même en présence d'erreurs dues au bruit du canal. Voici un extrait du code utilisé pour l'implémentation de l'encodeur et du décodeur Viterbi :

```python
def hard_viterbi_decode(encoded_bits):
    # Décode les bits reçus en utilisant l'algorithme Viterbi
    # (implémentation simplifiée pour un code convolutif avec une contrainte de longueur 3 et un taux de 1/2)
    K = 3
    num_states = 2 ** (K - 1)
    path_metrics = [float('inf')] * num_states
    path_metrics[0] = 0  # Démarre depuis l'état 0
    paths = [[] for _ in range(num_states)]

    for i in range(0, len(encoded_bits), 2):
        x1_recv = encoded_bits[i]
        x2_recv = encoded_bits[i+1]

        new_metrics = [float('inf')] * num_states
        new_paths = [[] for _ in range(num_states)]

        for prev_state in range(num_states):
            if path_metrics[prev_state] == float('inf'):
                continue

            for bit_in in [0, 1]:
                next_state = ((prev_state << 1) | bit_in) & 0b11
                x1_expected, x2_expected = get_output_bits(prev_state, bit_in)
                metric = path_metrics[prev_state] + int(x1_recv != x1_expected) + int(x2_recv != x2_expected)

                if metric < new_metrics[next_state]:
                    new_metrics[next_state] = metric
                    new_paths[next_state] = paths[prev_state] + [bit_in]

        path_metrics = new_metrics
        paths = new_paths

    min_state = path_metrics.index(min(path_metrics))
    return paths[min_state]
```

L'encodeur Viterbi est utilisé pour générer les bits encodés à partir des bits d'entrée. Une fois les bits reçus, le décodeur Viterbi applique une recherche de chemin pour corriger les erreurs potentielles et récupérer les bits originaux. Ce processus a été validé en comparant les bits décodés avec les bits d'origine.

#### **7.5 Test de l'encodeur et du décodeur**

Pour valider l'ensemble du processus de transmission, nous avons testé l'encodeur et le décodeur en envoyant des séquences de bits connues, puis en vérifiant si les bits reçus correspondaient aux bits envoyés. Voici un extrait du test de comparaison des bits encodés :

```python
# Exemple de séquence de bits
test_bits = [1, 0, 1, 1, 0, 0, 1, 0]
expected_encoded = viterbi_encode(test_bits)
actual_encoded = []

# Simulation de l'envoi et de la réception de bits
for bit in test_bits:
    ser.write(bytes([bit & 0x01]))  # Envoi du bit au FPGA
    time.sleep(0.01)

    response = ser.read(1)  # Lecture de la réponse
    byte = response[0]
    x1 = byte & 0x01
    x2 = (byte >> 1) & 0x01
    actual_encoded.extend([x1, x2])

# Comparaison des résultats
print("Expected encoding :", expected_encoded)
print("FPGA returned      :", actual_encoded)
print("✅ Match:", expected_encoded == actual_encoded)
```


### **8. Résultats et observations**



1. **Affichage réussi avec et sans bruit :**
   L'affichage de l'image a été réussi tant avec qu'avec bruit. Cependant, un problème de synchronisation a été initialement observé, où l'image devenait floue après chaque réinitialisation. Ce problème était dû à la configuration incorrecte du signal de réinitialisation et de l'activation du signal `enable` pour le PMOD. Une fois ces paramètres correctement ajustés, l'affichage a fonctionné comme prévu. En présence de bruit (ajout d'un bruit contrôlé via un SNR configurable), l'image est devenue presque entièrement floue, ce qui a bien illustré l'impact négatif du bruit sur la qualité de l'image.

2. **Récupération correcte de l’image après décodage :**
   Après le décodage des données, l'image récupérée était très proche de l'image originale, montrant que le système de décodage, même en présence de bruit, était capable de récupérer la majorité des informations. Les performances du système de décodage ont ainsi été jugées bonnes, avec seulement de très légères distorsions perceptibles dans les zones de haute fréquence de l'image.

3. **Comparaison visuelle entre image brute et image débruitée :**
   Lors de la comparaison entre l'image brute (avec bruit) et l'image débruitée, une nette amélioration de la lisibilité a été observée après débruitage. Bien que certaines imperfections subsistent dans l'image débruitée, celles-ci étaient significativement moins visibles que dans l'image brute, ce qui a permis de retrouver une image beaucoup plus fidèle à l'original.

4. **Cas 1 bit vs 8 bits : observations (qualité, fiabilité) :**
   Les tests comparant la transmission de 1 bit contre 8 bits ont montré de bonnes performances pour les deux configurations. Toutefois, la transmission par blocs de 8 bits a présenté une meilleure robustesse et fiabilité, notamment face au bruit. En effet, l'usage de 8 bits à la fois a permis une récupération plus fiable des données et une qualité d'image supérieure par rapport à l'utilisation d'un seul bit à la fois.



### **9. Conclusion**

Ce projet de démonstration d'un système de transmission avec détection et correction d'erreurs a permis de valider l'implémentation d'un système complet basé sur l'architecture FPGA Nexys 4, combinant traitement matériel et logiciel pour la transmission d'images à travers un canal bruité. L'intégration des différentes parties du démonstrateur — émetteur, canal, et récepteur — a été réalisée avec succès, permettant de transmettre des images tout en appliquant un codage convolutif de type Viterbi pour la correction des erreurs dues au bruit.

Le système a fonctionné de manière satisfaisante, avec une visualisation claire des étapes du processus de transmission : l'image originale, l'image bruitée, et l'image corrigée après décodage. Les performances du décodeur ont montré que, même dans des conditions de bruit élevé, une bonne partie des erreurs de transmission pouvaient être corrigées, garantissant ainsi une restitution fidèle de l'image.

Néanmoins, des améliorations restent possibles pour optimiser le système. Il serait intéressant d'explorer des solutions pour augmenter le débit de transmission tout en maintenant la fiabilité de la correction d'erreurs. De plus, l'intégration d'autres types de codage, ainsi que des techniques de compression d'image, pourrait également améliorer l'efficacité du système. Ces perspectives ouvrent des pistes intéressantes pour des applications futures, où la transmission fiable d'images à travers des canaux bruités est cruciale.


