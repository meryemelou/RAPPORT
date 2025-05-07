

## 📝 **Plan du Rapport Global du Démonstrateur**

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


## **Fonctionnalité globale du démonstrateur**

Le démonstrateur implémente un système complet de lecture et d’affichage d’image sur un écran connecté via un module PMOD. Son objectif principal est de parcourir une mémoire RAM contenant une image encodée en 1, 2, 4 ou 16 bits par pixel (BPP), de convertir si nécessaire les données en un format compatible avec l’affichage (RGB565 - 16 bits), puis de les transmettre ligne par ligne à l'écran.

L'affichage est déclenché par un signal de contrôle (`enable_pmod`) qui active la lecture séquentielle des données. Pour chaque cycle d’horloge, l’adresse de lecture est générée automatiquement à partir des coordonnées de la matrice (`col_counter`, `row_counter`). Le système incrémente ces coordonnées pour parcourir l’image pixel par pixel, tout en envoyant la position (`pix_col`, `pix_row`) et la couleur du pixel (`pix_data_out`) à l’écran. Un signal `pix_write` valide chaque écriture.

Le démonstrateur est conçu pour être générique, avec des paramètres configurables comme la largeur et la hauteur de l’écran, ainsi que la profondeur de couleur (BPP). Il supporte plusieurs formats d’image grâce à un bloc de transcodage intégré, qui convertit automatiquement les données issues de la RAM vers le format RGB565 utilisé pour l’affichage.

Une logique de réinitialisation (`reset`) est également incluse, permettant de réinitialiser les compteurs et l’état interne du module à tout moment. À la fin de l’affichage complet de l’image (lorsque tous les pixels ont été envoyés), le module désactive automatiquement le signal de validation d’écriture et attend un nouveau déclenchement.





### **4. Description des modules VHDL**


## **Fonctionnalité globale de l’émetteur, du canal et du récepteur**

### **Émetteur (FPGA 1)**

Le rôle principal de l’émetteur est d’assurer la réception, l’encodage et l’envoi de l’image. Dans un premier temps, une image est transmise depuis un PC vers la carte FPGA via une interface UART, octet par octet. Ces données sont stockées dans une mémoire RAM locale, puis l’image originale est affichée sur un écran OLED afin d’avoir un aperçu de la qualité initiale.

Ensuite, chaque octet est encodé bit par bit à l’aide d’un **codeur convolutif** (Viterbi encoder), dont l’objectif est d’introduire de la redondance pour rendre la transmission plus robuste face aux erreurs. Le module `encode8` effectue ce traitement : il extrait chaque bit de l’octet, l’encode à l’aide d’un module `viterbi_encoder`, puis génère deux octets de sortie (`x1_byte`, `x2_byte`) correspondant aux deux symboles codés. Ce mécanisme permet de doubler les données à transmettre, mais augmente fortement la capacité de correction au niveau du récepteur. Une fois le codage terminé, les données sont prêtes à être transmises à la carte suivante via une liaison série.


### **Canal de transmission bruité (FPGA 2 / PC 2)**

Le canal représente une simulation d’un environnement de transmission réel où les données peuvent être altérées. Ce rôle peut être assuré soit par une deuxième carte FPGA, soit par un logiciel PC. Les octets encodés provenant de l’émetteur sont reçus, puis **un bruit gaussien additif (AWGN)** est appliqué pour simuler une dégradation du signal, typique d’un canal analogique.

Ce bruit est contrôlé par un paramètre SNR (Signal-to-Noise Ratio), qui permet de varier l’intensité du bruit injecté dans le signal. Une fois bruitées, les données sont affichées (image dégradée) puis renvoyées à la troisième carte (récepteur). Cette étape permet d’observer l’impact des perturbations sur une transmission non protégée.


### **Récepteur et Décodage (FPGA 3)**

Le récepteur a pour rôle de récupérer les données bruitées, de les décoder, puis de reconstruire l’image originale. Pour cela, il s’appuie sur un **décodage convolutif de type Viterbi**, capable de corriger les erreurs introduites par le bruit durant la transmission. Afin de permettre un traitement précis et contrôlé, une **ROM nommée `rom_sigma`** est utilisée. Cette mémoire contient des valeurs prédéfinies de sigma (écart-type du bruit), utilisées pour moduler la réponse du décodeur selon les conditions de bruit simulées. Elle permet d’ajuster dynamiquement la sensibilité du décodage en fonction du niveau de bruit reçu, en se basant sur une entrée `sb` (signal binaire) représentant un index de configuration.

En complément, un module `data_serializer` est utilisé pour **désérialiser les données** reçues. Les bits reçus en série sont regroupés en octets et stockés dans une mémoire tampon. Une fois reconstitués, ces octets sont présentés au décodeur Viterbi, qui applique son algorithme de correction basé sur les treillis de transition d’états. Ce processus permet de retrouver les données d’origine avec une grande fiabilité, malgré la dégradation subie par le canal.

Enfin, les octets corrigés sont envoyés vers un écran OLED connecté à la carte FPGA, permettant de visualiser l’image restaurée. Ce mécanisme de comparaison entre l’image originale, bruitée et décodée offre une démonstration concrète de l’efficacité du codage/décodage convolutif dans un environnement de transmission bruité.




### **5. Simulations**

* Résultats des TB :

  * UART : validité des trames
  * OLED : synchronisation des pixels
  * Encodeur : bits de sortie corrects
  * Viterbi : récupération de l’image
* Capture des formes d’ondes pertinentes (si disponible)
* Résumé des performances simulées

---

### **6. Description fonctionnelle des étapes**

#### 6.1 Étape 1 – Affichage de l’image sur PMOD RGB OLED

* Fichiers utilisés : `bitmap.vhd`, `image_display.vhd`, `uart_rc.vhd`, `uart_tx.vhd`
* Envoi image → réception via UART → affichage OLED
* Top-level intégré sur la carte Nexys A7

#### 6.2 Étape 2 – Transmission de l’image entre trois PC (UART)

* Rôle des 3 PC :

  * PC1 = Émetteur
  * PC2 = Canal (relai + bruit)
  * PC3 = Récepteur
* Scripts Python pour gérer l’UART :

  * Configuration ports série
  * Lecture/envoi/stockage
* Test de bout en bout avec affichage final

#### 6.3 Étape 3 – Transmission avec bruit, encodage et décodage

* **Ajout de l’encodeur VHDL dans l’émetteur**
* **Canal** : bruit simulé via script Python ou PC intermédaire
* **Décodage par Viterbi sur le FPGA du récepteur**
* **Affichage final de l’image débruitée sur OLED**

---

### **7. Rôle des scripts Python**

* Connexion UART entre les PC
* Envoi et réception de l’image en trames
* Ajout du bruit contrôlé (SNR configurable)
* Utilisés pour la simulation du canal entre FPGA ou PC

---

### **8. Résultats et observations**

* Affichage réussi avec et sans bruit
* Récupération correcte de l’image après décodage
* Comparaison visuelle entre image brute et image débruitée
* Cas 1 bit vs 8 bits : observations (qualité, fiabilité)

---

### **9. Conclusion**

* Synthèse du fonctionnement complet du démonstrateur
* Intégration hardware/software réussie
* Perspectives : amélioration du débit, autre type de codage, compression, etc.

---

IL faut ecrire la remarque  sur taux d'erreur binanire 
