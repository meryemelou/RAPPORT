#  **Tutoriel**



# Table des matières

- [1. Introduction](#1-introduction)
- [2. Configuration](#2-configuration)
- [3. Architecture de la chaîne de communication](#3-architecture-de-la-chaîne-de-communication)
- [4. Modélisation du canal bruité (avec SNR ajustable)](#4-modélisation-du-canal-bruité-avec-snr-ajustable)
- [5. Décodage et correction](#5-décodage-et-correction)
- [6. Communication entre les cartes](#6-communication-entre-les-cartes)
- [7. Utilisation de la simulation et des tests](#7-utilisation-de-la-simulation-et-des-tests)
- [8. Problèmes rencontrés et solutions](#8-problèmes-rencontrés-et-solutions)
- [9. Conclusion](#9-conclusion)




###  **1. Introduction**
Ce tutoriel présente la mise en place d'un démonstrateur pour des codes correcteurs d’erreurs dans une chaîne de communication numérique utilisant un FPGA Nexys 4 Artix-7. L’objectif est de démontrer l'impact des codes correcteurs d’erreurs, comme l'algorithme de Viterbi, sur la transmission d'images à travers un canal de communication bruité.

Le projet s'articule autour de l'implémentation d'un système de transmission d’image qui passe par plusieurs étapes : encodage de l’image, transmission à travers un canal bruité, puis décodage et correction des erreurs. Chaque étape est réalisée sur un ou plusieurs FPGA, et les images sont visualisées à chaque phase de la chaîne de transmission.

L’objectif de ce tutoriel est de guider l'utilisateur à travers chaque étape de l'implémentation, de la configuration des outils et du matériel, jusqu'à l’affichage et la correction des erreurs dans l’image.

###  **2. Configuration**

Dans ce projet, nous avons utilisé **Vivado 2022.2** pour la conception et la programmation du FPGA **Nexys 4 Artix-7**. Vivado est l'environnement de développement utilisé pour la synthèse, la simulation et la génération du bitstream nécessaire à la configuration du FPGA. Au début du projet, l'accent a été mis sur l'affichage d'images sur un écran **PMOD RGB OLED**, nécessitant la gestion de la mémoire, du bitmap de l'image, et des blocs de communication pour l'affichage (image display). L’interface de communication entre le FPGA et l'écran PMOD repose sur l'utilisation des blocs **UART RX** et **UART TX** pour la transmission des données.

Lorsque nous avons entamé la conception de l’émetteur, du récepteur et du canal bruité, nous avons utilisé trois modules FPGA. Toutefois, dans notre cas, le canal et le récepteur étant regroupés sur le même PC, seules deux cartes FPGA ont été nécessaires. Chaque PC, représentant une carte FPGA, a été relié aux autres par des câbles afin d’assurer la transmission des données entre les différents modules. Chaque FPGA possède un fichier de contrainte spécifique pour définir les pins utilisées dans la communication avec les périphériques. 

Le fichier de contrainte permet de spécifier les broches exactes de chaque carte FPGA utilisées pour ces connexions, et ces fichiers sont adaptés pour chaque PC en fonction des composants connectés. La configuration matérielle ainsi réalisée permet de relier les différents éléments du projet et de garantir la bonne communication et transmission de données entre les modules d'émetteur, récepteur, et le canal de transmission.


### **3. Architecture de la chaîne de communication**

La chaîne complète de transmission est implémentée à l’aide de **deux cartes FPGA Nexys A7**, connectées chacune à un PC, en suivant le schéma :
**PC1 + FPGA1 (Émetteur)** → **PC2 + FPGA2 (Récepteur + Canal)**


###  Etape 1 : PC 1:

Sur le premier PC, ouvrir vivado et ouvrir le fichier source nécessaire au projet :


1. **Définir le module top-level as top** :
   Une fois le module principal du design sélectionné (Top level), on le défini comme top-level en faisant clic droit → **"Set as Top"**.

2. **Configurer le fichier de contraintes** :
  
**Ajouter Le fichier `.xdc` via le menu** *"Add Sources" → "Add Constraints"*. **Une fois ouvert, seules les lignes correspondant aux signaux effectivement utilisés (horloge, reset, UART, etc.) ont été décommentées, afin d’associer les ports logiques du design aux broches physiques du FPGA. Dans notre cas, cela inclut :**(Pour la partie emetteur)

* **le signal d’horloge :** `clk`,
* **les interrupteurs :** `reset`, `enable`,
* **les LEDs :** `fifo_empty1`, `fifo_afull1`, `fifo_full1`, `fifo_empty2`, `fifo_full2`, `fifo_afull2`,
* **le connecteur Pmod Header JB :** `PMOD_CS`, `PMOD_MOSI`, `PMOD_SCK`, `PMOD_DC`, `PMOD_RES`, `PMOD_VCCEN`, `PMOD_EN`,
  Vérifier bien dans le fichier contraintes les lignes suivantes:
* **le connecteur Pmod Header JD :**TX1**,**TX2**
* **et l’interface USB-RS232 :**RX**.


4. **Générer le bitstream** :

   * Lancer la commande **"Generate Bitstream"** dans Vivado.
   * Cette étape lance automatiquement la **synthèse RTL**, l’**implémentation**, puis la **génération du fichier `.bit`**, nécessaire pour programmer le FPGA.
  

5. **Programmer la carte FPGA** :

   * Aller dans **"Open Hardware Manager"**, puis **"Open Target"** → **"Auto Connect"** pour établir la communication avec la carte.
   * Une fois la carte détectée, cliquer sur **"Program Device"**.
   * Sélectionner le fichier `.bit` généré, et lancer la programmation.
  


###   Etape 2 : PC 2:

Sur le deuxième PC, ouvrir vivado et ouvrir le fichier source nécessaire au projet :

on va suivre exactement les mêmes étapes que pour la première carte : 


1. **Définir le module top-level as top** :
   Une fois le module principal du design sélectionné (Top level), on le défini comme top-level en faisant clic droit → **"Set as Top"**.

2. **Configurer le fichier de contraintes** :
  
**Ajouter Le fichier `.xdc` via le menu** *"Add Sources" → "Add Constraints"*. 

  La différence va résider dans la configuration du fichier contrainte, Vérifier bien dans le fichier contraintes les lignes suivantes:
  
* **le connecteur Pmod Header JD : **RX1**, **RX2** .
* **et l’interface USB-RS232 : **TX**.


4. **Générer le bitstream** :

   * Lancer la commande **"Generate Bitstream"** dans Vivado.
   * Cette étape lance automatiquement la **synthèse RTL**, l’**implémentation**, puis la **génération du fichier `.bit`**, nécessaire pour programmer le FPGA.
  

5. **Programmer la carte FPGA** :

   * Aller dans **"Open Hardware Manager"**, puis **"Open Target"** → **"Auto Connect"** pour établir la communication avec la carte.
   * Une fois la carte détectée, cliquer sur **"Program Device"**.
   * Sélectionner le fichier `.bit` généré, et lancer la programmation.


###   Etape 3 : Cablâge des cartes:

   * **PMOD RGB OLED :** L'écran PMOD RGB OLED doit être connecté au port **PMOD Header JB** sur chacune des cartes
   * **Fils de connections entre les cartes :** On prend un fil de connexion et on branche TX1 avec RX1, tel que TX1 est sur le pin 1 du PMOD Header JD du FPGA1 sur le PC1 et RX1 est sur le même pin du même PMOD Header du FPGA2 sur le PC2.
     De même pour TX2 avec RX2, on prend un fil de connexion et on branche les pin 2 de chaque PMOD Header JD des FPGA.
     Puis prendre un autre fil de connexion et brancher les GND (ground) des deux PMOD Header JD.
     Voici une illustration du PMOD Header pour repérer les pins :
     ![PMOD Header](PMOD_Header.png)

###  Etape 4 : PC 2:

* Sur le deuxième PC, ouvrir un terminal et aller dans le dossier où se trouve le code python 2.py et exécuter la commande suivante : 

  ```bash
  minicom -D /dev/ttyUSB0 -b 921600 -C converted_image.txt
  ```

  > Le fichier `converted_image.txt` contiendra les données reçues.

  ###  Etape 5 : Sur les deux cartes:

  * Mettre le switch du reset (Switch J15) à 1 puis le remettre à 0
  * Mettre le switch du enable (Switch L16) à 1 sur les deux cartes
 
  ###  Etape 6 : Sur le PC 1:

  * Lancer l’envoi de l’image en exécutant 1.py :

  ```bash
  python 1.py
  ```
  On choisira quelle image et quel port de communication.

  Puis On attend de voir l'image affichée sur les PMOD OLED des deux cartes comme cet exemple:
   ![Image affichée](affichage_image.jpeg)

   ###  Etape 7 : Sur le PC 2:

  Pour reconstruire l'image envoyée sur le deuxième PC , exécuter 2.py:

   ```bash
  python 2.py
  ```

  Si on veut envoyer et afficher une deuxième image, fermer le terminal sur le PC 2 et supprimer le fichier `converted_image.txt` qui contient les données de l'image précédente, remettre le enable sur les deux cartes à 0 puis le remettre à 1, ensuite lancer l’envoi de la nouvelle image en exécutant 1.py sur le PC1.
  

###  **Conclusion**

Ce rapport de tutoriel nous a permis de mettre en œuvre de manière concrète l’ensemble des étapes d’une chaîne de communication numérique intégrant des codes correcteurs d’erreurs, depuis la transmission initiale d’une image jusqu’à sa réception corrigée après passage dans un canal bruité.

