

## 📝 **Plan du Rapport Global du Démonstrateur**

---

### **1. Introduction générale**

Ce document a pour objectif de présenter en détail le fonctionnement du démonstrateur d’erreurs conçu autour de la transmission d’images via un système émetteur-canal-récepteur. Ce projet a été implémenté sur des cartes FPGA Nexys 4 basées sur des puces Artix-7, en exploitant des modules PMOD, notamment l’écran RGB OLED, pour la visualisation des résultats.

L’architecture repose sur la transmission d’une image depuis un émetteur vers un récepteur en passant par un canal de communication pouvant introduire des erreurs. L’image est tout d’abord affichée sur le PMOD OLED. Par la suite, un codage de vitterbi est appliqué à l’image avant sa transmission. Après le passage par un canal bruité (modélisé par un code vhdl de SNR ), l’image est décodée à l’aide de l’algorithme de Viterbi, puis de nouveau affichée pour visualiser l’effet de la correction d’erreurs sur notre PMOD.

La communication série entre les différents composants (émetteur, canal, récepteur) est assurée via UART. L’ensemble du système est composé des modules VHDL pour le traitement temps réel (affichage, encodage, décodage) et des scripts Python pour l'envoi d'image et le pilotage des transferts. Ce travail a été réalisé dans le cadre du projet de démonstration d’un système de transmission avec détection et correction d’erreurs.



### **2. Architecture globale du démonstrateur**

#### **Figure schématique globale du système**

Le démonstrateur repose sur une architecture en trois parties principales réparties sur trois plateformes FPGA Nexys 4, chacune connectée à un PC. La communication entre les différentes cartes se fait via l’interface UART. L’image transmise traverse successivement les étapes d’émission, de bruitage (canal) et de réception avec correction d’erreurs, chacune étant visualisable à l’écran PMOD OLED connecté à chaque carte.

![Description de l'image](images/Screenshot 2025-05-07 at 15.53.45.png)



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






### **3. Fonctionnalité globale du démonstrateur**

* Description du **flux de données** de bout en bout :

  1. Lecture image
  2. Envoi UART
  3. Ajout de bruit
  4. Décodage
  5. Affichage image sur PMOD RGB OLED
* Mise en œuvre réelle sur les cartes Nexys A7

---

### **4. Description des modules VHDL**

#### 4.1 Modules principaux

* `bitmap.vhd` : Lecture des données d'image
* `image_display.vhd` : Envoi vers l’OLED (affichage)
* `uart_tx.vhd` / `uart_rc.vhd` : Transmission série UART
* `encoder.vhd` : Codage (ex: convolutionnel)
* `viterbi_decoder.vhd` : Décodage


ici on doit citer quand on va utiliser deux uart rx, et quand on va utiliser les deux uart tx


#### 4.2 Modules de testbench

* Objectif des testbenchs
* Ce qui a été simulé :

  * UART
  * Affichage image
  * Encodeur/décodeur (1 bit, puis 8 bits)
* Limitations visualisées (latence, erreurs, limites de correction)


on a tester la transmissions des bits pour la ram
---

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
