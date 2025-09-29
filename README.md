# SUMMIT : Surchauffe Urbaine : approche Multi-échelles par Mesures Infrarouge Thermique

Ce projet a pour objectif de mettre en place une chaîne de traitement complète pour les images infrarouge thermique collectées à bord d'un véhicule. Le processus couvre l'organisation des données brutes, le prétraitement, la segmentation sémantique des images, la correction des masques, le calcul des températures de surface, et la génération de rapports finaux.

---

## ⚙️ Installation

1.  **Cloner le dépôt :**
    ```bash
    git clone [URL_DU_DEPOT]
    cd [NOM_DU_DEPOT]
    ```

2.  **Installer les dépendances Python :**
    Il est recommandé d'utiliser un environnement virtuel.
    ```bash
    python -m venv venv
    source venv/bin/activate  # Sur Windows: venv\Scripts\activate
    pip install -r requirements.txt
    ```

3.  **Installer les outils externes :**
    Les modèles et outils externes doivent être placés dans le dossier `External/`.
    * **ExifTool :** Téléchargez et décompressez ExifTool. Le chemin vers l'exécutable doit correspondre à `exiftool_path` dans `Config/config.yaml`.
    * **FTNet :** Clonez ou copiez le dépôt du modèle FTNet dans `External/FTNet/`.
    * **Segment Anything (SAM) :** Clonez ou copiez le dépôt et téléchargez le poids du modèle (`.pth`) requis. Le chemin doit correspondre à `sam_checkpoint` dans `Config/config.yaml`.

4.  **Configuration :**
    Le fichier principal de configuration est `Config/config.yaml`. Avant de lancer les scripts, assurez-vous de renseigner les chemins et les paramètres, notamment le `session_id` que vous souhaitez traiter.

---

## 📂 Organisation des Données et Fichiers

La structure du projet est organisée comme suit pour séparer clairement les données, les scripts et les résultats.

* `Config/` : Contient le fichier de configuration central `config.yaml`.
* `Data/` : Toutes les données liées au projet. Chaque sous-dossier est généralement organisé par identifiant de session (`session_id`).
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
* `External/` : Contient les outils et modèles externes comme ExifTool, FTNet et Segment Anything.
* `Models/` : Contient les modèles de deep learning.
    * `finetuned_models/` : Modèles ré-entraînés sur nos données spécifiques et logs d'entraînement.
* `Results/` : Contient les sorties finales du traitement.
    * `thermal_results/` : Cartes de températures finales en format image (`.png`).
    * `thermal_results_raw_npy/` : Données brutes des cartes de températures (`.npy`).
    * `final_report/` : Rapports CSV contenant des statistiques par image.
    * `master_table.csv` : Un fichier CSV global qui agrège les informations et métriques de toutes les images de toutes les sessions traitées.
* `Scripts/` : Contient tous les scripts Python pour exécuter la chaîne de traitement.



---

## 🚀 Description des Scripts

Chaque script est conçu pour être lancé depuis la racine du projet. La configuration de chaque script est gérée via le fichier `Config/config.yaml`.

### 1. `data_organization.py`

* **Objectif :** Prend les données brutes d'une session et les organise dans la structure de dossiers standardisée du projet (`Data/raw_sessions/session_id`).
* **Lancement :**
    ```bash
    python Scripts/data_organization.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    data_organization:
      file_patterns: { ... }
      standard_names: { ... }
    ```
* **Résultats :** Un nouveau dossier `Data/raw_sessions/[session_id]` est créé et peuplé avec les images et les fichiers de métadonnées renommés.

### 2. `preprocessing.py`

* **Objectif :** Prétraite les images thermiques brutes : conversion en niveaux de gris et normalisation par percentiles pour améliorer le contraste.
* **Lancement :**
    ```bash
    python Scripts/preprocessing.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    session_id: "session_570"
    preprocessing:
      lower_percentile: 1.0
      upper_percentile: 99.0
    ```
* **Résultats :** Les images prétraitées sont sauvegardées dans `Data/preprocessed_images/[session_id]`.

### 3. `segmentation.py`

* **Objectif :** Applique le modèle de segmentation sémantique FTNet sur les images prétraitées.
* **Lancement :**
    ```bash
    python Scripts/segmentation.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    session_id: "session_570"
    segmentation:
      template_toml: "external/FTNet/ftnet/cfg/infer_images.toml"
      model_weights: "models/finetuned_models/..."
      output_dir_name: "ftnet"
    ```
* **Résultats :** Les masques de segmentation sont générés dans `Data/segmentation_results/[session_id]/ftnet/`.

### 4. `correction.py`

* **Objectif :** Corrige et affine les masques de segmentation de FTNet en utilisant le modèle Segment Anything (SAM) pour améliorer la précision des contours des objets.
* **Lancement :**
    ```bash
    python Scripts/correction.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    session_id: "session_570"
    correction:
      ftnet_input_dir: "ftnet"
      sam_checkpoint: "external/segment-anything/..."
      output_refined_dir: "final_corrected"
      coverage_thresholds: { ... }
      generator_params: { ... }
    ```
* **Résultats :** Les masques corrigés sont sauvegardés dans `Data/segmentation_results/[session_id]/final_corrected/`. Des visualisations de SAM sont enregistrées dans `Data/segmentation_results/[session_id]/sam/`.

### 5. `road_shadow.py`

* **Objectif :** Détecte et segmente les zones d'ombre sur la surface de la route en se basant sur les masques de segmentation et les données de température brute.
* **Lancement :**
    ```bash
    python Scripts/road_shadow.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    session_id: "session_570"
    shadow_segmentation:
      min_pixels_for_analysis: 50
      ...
    ```
* **Résultats :** Les masques des ombres sont générés dans `Data/segmentation_results/[session_id]/shadows/`.

### 6. `thermal_calculate.py`

* **Objectif :** Calcule les cartes de température de surface (LST) en utilisant les données radiométriques des images, les masques de segmentation et les valeurs d'émissivité propres à chaque classe d'objet.
* **Lancement :**
    ```bash
    python Scripts/thermal_calculate.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    session_id: "session_570"
    dependencies:
      exiftool_path: "external/..."
    thermal_calculation:
      segmentation_mask_source_dir: "data/segmentation_results/final_corrected"
      emissivity_mapping: { ... }
      ...
    ```
* **Résultats :** Les cartes de températures sont sauvegardées au format `.npy` dans `Results/thermal_results_raw_npy/[session_id]/`.

### 7. `thermal_finalize.py`

* **Objectif :** Convertit les cartes de températures brutes (`.npy`) en images visualisables (`.png`) et génère un rapport CSV par image avec des statistiques (température min/max/moyenne par classe).
* **Lancement :**
    ```bash
    python Scripts/thermal_finalize.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    session_id: "session_570"
    ```
* **Résultats :** Les images PNG sont dans `Results/thermal_results/[session_id]/` et les rapports CSV dans `Results/final_report/[session_id]/`.

### 8. `master_table.py`

* **Objectif :** Agrège les résultats de toutes les images de la session traitée dans un unique fichier CSV (`master_table.csv`), ajoutant les nouvelles lignes à la suite des sessions précédentes.
* **Lancement :**
    ```bash
    python Scripts/master_table.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    session_id: "session_570"
    master_table:
      output_path: "results/master_table.csv"
      bgr_mapping: { ... }
    ```
* **Résultats :** Le fichier `Results/master_table.csv` est créé ou mis à jour.

---

### Scripts pour l'Annotation et le Fine-Tuning

Ces scripts sont utilisés pour améliorer le modèle de segmentation.

#### `annotation.py`

* **Objectif :** Prépare les données pour l'annotation manuelle en générant des aperçus basés sur SAM pour faciliter et accélérer le processus.
* **Lancement :**
    ```bash
    python Scripts/annotation.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    session_id: "session_570"
    annotation:
      sam_checkpoint: "external/..."
      generator_params: { ... }
    ```
* **Résultats :** Fichiers d'aide à l'annotation générés dans `Data/annotation_data/[session_id]`.

#### `prepare_fine_tuning.py`

* **Objectif :** Convertit les annotations manuelles au format requis par FTNet, en créant les masques, les images et les contours (`edges`) et en les divisant en ensembles d'entraînement et de validation.
* **Lancement :**
    ```bash
    python Scripts/prepare_fine_tuning.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    fine_tuning:
      dest_dataset_name: "data/annotated_dataset"
      train_split_ratio: 0.8
      edge_generation_script_path: "external/FTNet/..."
    ```
* **Résultats :** Le jeu de données prêt à l'emploi est créé dans le dossier spécifié par `dest_dataset_name`.

#### `fine_tuning.py`

* **Objectif :** Lance le processus de fine-tuning du modèle FTNet sur le jeu de données annoté.
* **Lancement :**
    ```bash
    python Scripts/fine_tuning.py
    ```
* **Configuration (`config.yaml`) :**
    ```yaml
    fine_tuning:
      template_toml: "external/FTNet/ftnet/cfg/finetune.toml"
      training_data_dir: "data/annotated_dataset_nuit"
      base_model_checkpoint: "external/FTNet/Trained_Models/..."
      output_model_dir: "models/finetuned_models/Test_4"
      trainer_params: { ... }
    ```
* **Résultats :** Un nouveau modèle entraîné est sauvegardé dans `Models/finetuned_models/`, avec ses logs et checkpoints. Vous pouvez ensuite mettre à jour le paramètre `model_weights` dans la section `segmentation` pour l'utiliser.
