# SUMMIT : Surchauffe Urbaine : approche Multi-échelles par Mesures Infrarouge Thermique

Ce projet a pour objectif de mettre en place une chaîne de traitement complète pour les images infrarouge thermique collectées à bord d'un véhicule. Le processus couvre l'organisation des données brutes, le prétraitement, la segmentation sémantique des images, la correction des masques, le calcul des températures de surface, et la génération de rapports finaux.

---

## Installation

1.  **Cloner le dépôt :**
    ```bash
    git clone [URL_DU_DEPOT]
    cd [NOM_DU_DEPOT]
    ```

2.  **Installer les dépendances Python :**
    Il est recommandé d'utiliser un environnement virtuel avec Python 3.10.
    ```bash
    python -m venv venv
    source venv/bin/activate
    pip install -r requirements.txt
    ```

3.  **Configuration :**
    Le fichier principal de configuration est `config/config.yaml`. Avant de lancer les scripts, il faut renseigner les chemins et les paramètres, notamment le `session_id` à traiter.

---

## Organisation des Données et Fichiers

La structure du projet est organisée comme suit pour séparer clairement les données, les scripts et les résultats.

* `config/` : Contient le fichier de configuration central `config.yaml`.
* `data/` : Toutes les données liées au projet. Chaque sous-dossier est généralement organisé par identifiant de session (`session_id`).
    * `raw_sessions/` : Données brutes de chaque session de mesure.
        * `flir_images/` : Images thermiques (`.JPG`).
        * `aux_data/` : Fichiers texte (`.txt`) contenant les métadonnées (GPS, météo, etc.).
    * `preprocessed_images/` : Images normalisées et converties en niveaux de gris, prêtes pour la segmentation.
    * `segmentation_results/` : Résultats des différentes étapes de segmentation.
        * `ftnet/` : Masques de segmentation bruts générés par le modèle FTNet.
        * `shadows/` : Masques des ombres détectées sur la route.
        * `sam/` : Visualisations des masques générés par SAM lors de la correction.
        * `final_corrected/` : Masques de segmentation finaux après correction par SAM.
    * `annotation_data/` : Fichiers intermédiaires pour aider à l'annotation manuelle.
        * `sam_previews/`, `sam_npy_maps/`, `sam_class_maps/`.
    * `manual_annotations/` : Images annotées manuellement pour le fine-tuning.
    * `annotated_dataset/` : Le jeu de données finalisé pour le fine-tuning, séparé en `image`, `mask` et `edges`.
* `external/` : Contient les outils et modèles externes comme ExifTool, FTNet et Segment Anything.
* `models/` : Contient les modèles de deep learning.
    * `finetuned_models/` : Modèles ré-entraînés sur nos données spécifiques et logs d'entraînement.
* `results/` : Contient les sorties finales du traitement.
    * `thermal_results/` : Cartes de températures finales en format image (`.png`).
    * `thermal_results_raw_npy/` : Données brutes des cartes de températures (`.npy`).
    * `final_report/` : Rapports CSV contenant des statistiques par image.
    * `master_table.csv` : Un fichier CSV global qui agrège les informations et métriques de toutes les images de toutes les sessions traitées.
* `scripts/` : Contient tous les scripts Python pour exécuter la chaîne de traitement.

---

## Description des Scripts

Chaque script est conçu pour être lancé depuis la racine du projet. La configuration de chaque script est gérée via le fichier `config/config.yaml`.

### 0. `config.yaml`

* **Important:** Variable commune à modifer pour chaque script suivant la session utilisée.
    ```yaml
    session_id : session_XXX
  ```

### 1. `data_organization.py`

* **Objectif :** Ce script est le point d'entrée pour une nouvelle session de données. Il localise les fichiers d'images thermiques (`.JPG`) et les fichiers de logs auxiliaires (GPS, météo, etc.) dans un dossier source externe, puis les copie et les renomme dans la structure de dossiers standardisée du projet (`Data/raw_sessions/`).
* **Lancement :** Le script requiert deux arguments : l'identifiant de la session et le chemin vers le dossier contenant les données brutes.
    ```bash
    python scripts/data_organization.py --session-id [session_XXX] --source-dir [CHEMIN_VERS_LE_DOSSIER_SOURCE]
    ```
* **Configuration (`config.yaml`) :** A ne modifier qu'en cas de changements de formats sur les données récoltées.
    ```yaml
    data_organization:
      file_patterns:
        flir_folder_pattern: "FIR*"
        flir_image_subfolder_pattern: "APOFIR*.ME0_"
        aux_data_folder_pattern: "APO*.SES"
        aux_files:
          image_log: "APOFIR*.me0"
          meteo_sensor: "APOC3S*.me0"
          gps_sensor: "APOLOC*.ME0"
          surface_temp: "APOTHR*.re1"
      standard_names:
        image_log: "image_log.txt"
        meteo_sensor: "meteo_sensor.txt"
        gps_sensor: "gps_sensor.txt"
        surface_temp: "surface_temp.txt"
    ```
* **Résultats :** Le script peuple la structure de données du projet. À la fin de l'exécution, le dossier `Data/raw_sessions/flir_images/[session_id]` contient les images et le dossier `Data/raw_sessions/aux_data/[session_id]` contient les fichiers de logs renommés.

### 2. `preprocessing.py`

* **Objectif :** Ce script prépare les images thermiques brutes pour la segmentation. Il convertit les images radiométriques en images en niveaux de gris 8 bits. Le processus se déroule en deux étapes : le calcul de seuils de normalisation globaux sur toute la session, puis la normalisation de chaque image individuellement pour garantir une cohérence de contraste.
* **Lancement :** Le script utilise le `session_id` spécifié dans le fichier de configuration.
    ```bash
    python scripts/preprocessing.py
    ```
* **Configuration (`config.yaml`) :** 
    ```yaml
    preprocessing:
      lower_percentile: 1.0
      upper_percentile: 99.0
    ```
* **Résultats :** Un nouveau dossier est créé à l'emplacement `Data/preprocessed_images/[session_id]`. Ce dossier contient les images thermiques normalisées au format `.png`.

### 3. `segmentation.py`

* **Objectif :** Ce script lance le processus de segmentation sémantique en utilisant le modèle externe **FTNet**. Il agit comme un wrapper qui génère une configuration temporaire pour FTNet, puis exécute le script d'inférence pour assigner une étiquette de classe (ex: `route`, `bâtiment`) à chaque pixel.
* **Lancement :** Le script est piloté par le fichier `config.yaml`.
    ```bash
    python scripts/segmentation.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    session_id: "session_570"
    segmentation:
      template_toml: "external/FTNet/ftnet/cfg/infer_images.toml"
      model_weights: "models/finetuned_models/Test_4/ckpt/epoch=46-val_mIOU=0.4421.ckpt"
      output_dir_name: "ftnet"
    ```
* **Résultats :** Un nouveau dossier est créé à l'emplacement `Data/segmentation_results/ftnet/[session_id]` contenant les masques de prédiction bruts générés par FTNet.

### 4. `correction.py`

* **Objectif :** Ce script a un double objectif : la **correction automatique** des masques de FTNet en utilisant les contours précis du modèle **Segment Anything (SAM)**, et la **préparation à l'annotation** en générant des cartes de segments avec des identifiants uniques.
* **Lancement :** Le script est piloté par le `session_id` dans le fichier de configuration.
    ```bash
    python scripts/correction.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    session_id: "session_570"
    correction:
      ftnet_input_dir: "ftnet"
      ftnet_results_subpath: "Evaluation/infer/Predictions"
      sam_checkpoint: "external/segment-anything/sam_vit_h_4b8939.pth"
      sam_model_type: "vit_h"
      output_refined_dir: "final_corrected"
      output_sam_viz_dir: "sam"
      coverage_thresholds:
        route: 0.7
        building: 0.6
      generator_params:
        points_per_side: 32
    ```
* **Résultats :** Le script génère quatre sorties principales : les masques corrigés (`final_corrected`), les visualisations SAM (`sam`), et les cartes d'IDs et aperçus pour l'annotation (`annotation_data`).

### 5. `thermal_calculate.py`

* **Objectif :** Ce script effectue le calcul de la **Température de Surface (LST)** pour chaque pixel. Il applique un modèle physique qui corrige les mesures en fonction des conditions environnementales (température de l'air, humidité, rayonnement réfléchi) et de l'**émissivité** de chaque type de surface (route, bâtiment, etc.).
* **Lancement :** Le script utilise un traitement parallèle pour accélérer les calculs.
    ```bash
    python scripts/thermal_calculate.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    session_id: "session_570"
    thermal_calculation:
      segmentation_mask_source_dir: "data/segmentation_results/final_corrected"
      master_table_path: "results/master_table.csv"
      emissivity_mapping:
        route: 0.92
        building: 0.92
        tree: 0.96
      bgr_mapping:
        route: [255, 0, 0]
      num_workers: 4
    ```
* **Résultats :** Le script crée un dossier `Results/thermal_results_raw_npy/[session_id]/` contenant des fichiers `.npy`. Chaque fichier est un tableau NumPy 2D des températures en degrés Celsius.

### 6. `road_shadow.py`

* **Objectif :** Ce script détecte et segmente les zones d'ombre **uniquement sur la surface de la route**. Il analyse la distribution de température des pixels de la route et utilise l'**algorithme de seuillage automatique d'Otsu** pour trouver la température optimale séparant les zones ensoleillées des zones ombragées.
* **Lancement :** Le script traite les images en parallèle.
    ```bash
    python scripts/road_shadow.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    session_id: "session_570"
    shadow_segmentation:
      min_std_dev_for_otsu: 3.0
      otsu_min_group_percentage: 0.05
      mean_temp_considered_hot: 50.0
      mean_temp_considered_cold: 30.0
    ```
* **Résultats :** Le script génère des masques binaires dans `Data/segmentation_results/shadows/[session_id]/`, où les pixels blancs représentent les ombres sur la route.

### 7. `thermal_finalize.py`

* **Objectif :** Ce script finalise la session en produisant des **rapports statistiques** et des **cartes de température visuelles**. Il calcule des métriques (température moyenne, etc.) par classe et pour les zones soleil/ombre de la route. Il normalise également toutes les cartes de température de la session sur une échelle globale pour assurer la cohérence visuelle.
* **Lancement :** Le script s'exécute pour la session définie dans `config.yaml`.
    ```bash
    python scripts/thermal_finalize.py
    ```
* **Configuration (`config.yaml`) :** Ce script n'a pas sa propre section mais réutilise les configurations des autres scripts.
* **Résultats :** Le script produit un rapport `rapport_thermique_final.csv` dans `Results/final_report/[session_id]/` et des images `.png` normalisées dans `results/thermal_results/[session_id]/`.

### 8. `table.py`

* **Objectif :** Ce script est l'outil de gestion de la base de données finale `master_table.csv`. Il fonctionne en **deux modes distincts** pour agréger toutes les informations de toutes les sessions.
* **Lancement :**
    1.  **Mode `meteo` (à lancer en premier pour une session) :** Synchronise et intègre les données brutes des capteurs (GPS, météo) de la session dans la table maître.
        ```bash
        python scripts/table.py --mode meteo
        ```
    2.  **Mode `temperatures` (à lancer à la fin) :** Met à jour la table maître avec les résultats des analyses (statistiques thermiques, métriques de performance IoU).
        ```bash
        python scripts/table.py --mode temperatures
        ```
* **Configuration (`config.yaml`) :**
    ```yaml
    session_id: "session_570"
    master_table:
      output_path: "results/master_table.csv"
      bgr_mapping:
        route: [255, 0, 0]
    ```
* **Résultats :** Le script crée ou met à jour le fichier `Results/master_table.csv`, qui est le livrable final consolidé du projet.

---

## Scripts pour l'Annotation et le Fine-Tuning

### 9. `annotation.py`

* **Objectif :** Ce script lance une **application web interactive** (avec Streamlit) pour l'annotation manuelle des images. Il permet à un utilisateur de créer des masques de **vérité terrain** en cliquant sur des segments pré-calculés par SAM et en leur assignant une classe, ce qui est beaucoup plus rapide que de dessiner des polygones.
* **Lancement :** Le lancement se fait avec la commande `streamlit run`.
    ```bash
    streamlit run scripts/annotation.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    session_id: "session_570"
    annotation:
      pre_annotated_dir: "final_corrected"
      output_bgr_colors:
        route: [255, 0, 0]
    ```
* **Résultats :** Le script génère les masques de vérité terrain, vérifiés par un humain, dans le dossier `Data/manual_annotations/[session_id]/`.

### 10. `prepare_fine_tuning.py`

* **Objectif :** Ce script prépare un **jeu de données pour le fine-tuning**. Il prend les annotations manuelles, les sépare en ensembles d'entraînement et de validation, convertit les masques colorés en **masques d'ID de classe** (1 canal), et génère les **cartes de contours (edges)** requises par le modèle FTNet.
* **Lancement :**
    ```bash
    python scripts/prepare_fine_tuning.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    session_id: "session_570"
    fine_tuning:
      dest_dataset_name: "data/annotated_dataset"
      train_split_ratio: 0.8
      edge_generation_script_path: "external/FTNet/ftnet/helper/edge_generation/generate_edges.py"
      model_classes:
        background: {id: 0, color_bgr: [0, 0, 0]}
        route: {id: 1, color_bgr: [255, 0, 0]}
    ```
* **Résultats :** Le script génère un dossier complet (défini par `dest_dataset_name`) contenant les sous-dossiers `image`, `mask`, `edges`, et les fichiers `train.txt`/`val.txt`, prêt pour l'entraînement.

### 11. `fine_tuning.py`

* **Objectif :** Ce script lance le **processus d'entraînement (fine-tuning)** du modèle FTNet. Il utilise le jeu de données préparé à l'étape précédente pour spécialiser un modèle de base sur les données thermiques annotées, dans le but d'améliorer ses performances.
* **Lancement :**
    ```bash
    python scripts/fine_tuning.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    fine_tuning:
      template_toml: "external/FTNet/ftnet/cfg/finetune.toml"
      training_data_dir: "data/annotated_dataset_nuit"
      base_model_checkpoint: "external/FTNet/Trained_Models/ftnet_resnext101_32x8d_filters_128_edge_3_blocks_2/rev-2/Fine Tuning/ckpt/1-epoch=37-val_mIOU=0.4305.ckpt"
      output_model_dir: "models/finetuned_models/Test_4"
      trainer_params:
        epochs: 50
        train_batch_size: 2
      optimizer_params:
        name: "SGD"
        lr: 0.001
    ```
* **Résultats :** Un nouveau modèle entraîné est sauvegardé dans le dossier spécifié par `output_model_dir`. Ce dossier contient les logs d'entraînement et, surtout, le sous-dossier `ckpt/` avec les poids du modèle (`.ckpt`). Le chemin vers le meilleur checkpoint peut ensuite être utilisé dans la section `segmentation` pour faire des inférences avec ce nouveau modèle amélioré.
