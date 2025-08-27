# Vague 1: Mobilité I

Cette page documente le preprocessing de la première vague du Panel Lémanique effectué dans le fichier `preprocess_wave1.R`.

## Fonction `recoder_adjectifs`
- Description: recode les questions 97 à 100 afin de les harmoniser avec les mêmes questions de la vague 3. Retourne pour chaque question trois variables (e.g.: `Q97_1`, `Q97_2`et `Q97_3`) avec les adjectifs labellés de 1 à 24
- Input: les données brutes de la vague 1, le suffixe, indiquant le mode de transport dont la question traite (e.g. `voiture`), ainsi que le nom complet de la questions (e.g.: `Q97`)
- Output: les données brutes de la vague 1 avec les questions 97 à 100 recodées
- Détail:
```r
pattern <- paste0("_", suffixe, "$")
texte_col <- paste0("Autre_TEXTE_", suffixe)
autre_col <- paste0("Autre_", suffixe)
possible_cols <- names(data)[stringr::str_detect(names(data), pattern)]
adjectif_cols <- possible_cols[!(possible_cols %in% c(texte_col, autre_col))]
```
Détecte toutes les colonnes traitant d'un mode de transport, et ne garde que les colonnes avec des valeurs numériques.
```r
long_data <- data %>%
  dplyr::select(IDNO, dplyr::all_of(adjectif_cols)) %>%
  tidyr::pivot_longer(
    cols = -IDNO,
    names_to = "adjectif_var",
    values_to = "selected"
  ) %>%
  dplyr::filter(selected == 1) %>%
  dplyr::mutate(adjectif = stringr::str_remove(adjectif_var, paste0("_", suffixe)))
```
Passage en format à l'aide de la fonction `pivot_longer` en ne gardant que les réponses cochées et suppression du suffixe du nom de la variable.
```r
dictionnaire <- long_data %>%
  dplyr::distinct(adjectif) %>%
  dplyr::arrange(adjectif) %>%
  dplyr::mutate(code = dplyr::row_number())
```
Création du dictionnaire des adjectifs
```r
coded_data <- long_data %>%
  dplyr::left_join(dictionnaire, by = "adjectif") %>%
  dplyr::select(IDNO, code)
  
grouped <- coded_data %>%
  dplyr::group_by(IDNO) %>%
  dplyr::summarise(codes = list(sort(code)), .groups = "drop")
```
Crée une liste des adjectifs choisis par chaque participant·e
```r
wide_data <- grouped %>%
  tidyr::unnest_wider(codes, names_sep = "_") %>%
  dplyr::rename_with(~ paste0(question_nom, "_", seq_along(.)), starts_with("codes_"))
```
Retour en format large, renommage des colonnes en `Q97_1`, `Q97_2`, etc ...
```r
adjective_list <- dictionnaire$adjectif[!is.na(dictionnaire$adjectif) & dictionnaire$adjectif != ""]
labels <- setNames(seq_along(adjective_list), adjective_list)
wide_data <- wide_data %>%
  dplyr::mutate(
    dplyr::across(
      -IDNO,
      ~ haven::labelled(.x, labels)
    )
  )
```
Les colonnes sont labellisées
```r
data_final <- data %>%
  dplyr::select(-dplyr::all_of(adjectif_cols)) %>%
  dplyr::select(-dplyr::all_of(paste0("Autre_", suffixe))) %>%
  dplyr::rename(!!paste0(question_nom, "_TEXTE") := dplyr::all_of(texte_col)) %>%
  dplyr::left_join(wide_data, by = "IDNO")
```
Les nouvelles colonnes sont rajoutées au database de base, les anciennes sont supprimées et les colonnes textes sont renommées en `Q97_TEXTE`, `Q98_TEXTE`, etc ...

## Fonction `get_section`

-   Description: retourne la section du questionnaire dont fait partie une questions
-   Input: le numéro de la question
-   Output: le nom de la section

## Fonction `documentation`

-   Description: permet d'obtenir une documentation de toutes les variables dans la vague 1
-   Input: les données brutes de la vague 1
-   Output: un fichier csv avec le nom de la variable, le texte de la variable et les labels de la variable

## Fonction `write_files_participants`

-   Description: permet d'obtenir toutes les informations relatives aux particpant·es de la vague 1
-   Input: les données brutes de la vague 1
-   Output: un fichier csv avec une ligne par participant·e et les informations relatives aux participant·es de la vague 1, ainsi qu'un fichier *labels* détaillant les labels des variables dans le premier fichier

## Fonction `write_files_questions`

-   Description: permet de lister les questions posées aux participant·es dans cette vague
-   Input: les données brutes de la vague 1
-   Output: un fichier csv contenant les variables des questions, le texte des questions et leur section dans le questionnaire (obtenue avec la fonction get_section, voir [`get_section`](#fonction-get_section)), ainsi qu'un fichier *labels*, détaillant les labels des questions

## Fonction `write_files_survey_completion`

-   Description: renvoie les métadonnées de l'enquête
-   Input: les données brutes de la vague 1
-   Output: un fichier csv contenant pour chaque participant·e la liste des métadonnées

## Fonction `write_label_file`

-   Description: permet d'écrire tout les csv *labels*. Retire les labels pour les colonnes indiquées à l'aide de la fonction `get_labels`
-   Input: les données brutes de la vague 1, les colonnes dont il faut retirer les labels, et le nom final du fichier csv
-   Output: un fichier csv contenant les labels des colonnes indiquées

## Fonction `write_file_section`

-   Description: permet d'obtenir les sections du questionnaire de la vague 1
-   Input: aucun
-   Output: un fichier csv contenant les sections du questionnaire

## Fonction `write_file_answers`

-   Description: permet d'obtenir les réponses aux questions par les participantes. Affiche une réponse par participant·es par ligne.
-   Input: les données brutes de la vague 1
-   Output: un fichier csv avec une réponse par question et par participant·e par ligne
-   Détail:

``` r
participants_colnames <- c("group","pays","gp_age_source","numero_insee","numero_ofs","CP_source","Localité_source")
  extra_colnames <- c("wgt_socio",  "wgt_cant_trim",    "wgt_agg_trim", "wgt_cant_trim_gps",    "wgt_agg_trim_gps", "wgt_cant_trim_v2", "wgt_agg_trim_v2")
  responses <- data |>
    dplyr::select(
      -all_of(c(participants_colnames, extra_colnames))
    ) |>
    zap_all()
```

Ne sélectionne que les colonnes avec les questions, et applique tout les fonctions de `haven::zap_*` avec `zap_all`

``` r
responses <- responses |> 
    dplyr::mutate(dplyr::across(tidyselect::where(is.character), ~ remove_whitespace(.x))) |>
    dplyr::mutate(dplyr::across(tidyselect::everything(), ~ readr::parse_guess(as.character(.x))))
```

Supprime tout les espaces au début d'une chaîne de caractère et cherche à obtenir le type de données

``` r
response_texts <- pivot_responses()
response_values <- pivot_responses()
```

Pivote les tableaux

## Fonction `main`

-   Description: nettoye les données et obtiens tout les csv grâce aux fonctions décrites plus haut
-   Input: aucun
-   Output: aucun
-   Détail:

``` r
get_data_folder <- function() {
    os <- Sys.info()[["sysname"]]
    
    if (os == "Linux") {
      return("/mnt/lasur/")  
    } else if (os == "Darwin") {
      return("/Volumes/LASUR/")  
    } else if (os == "Windows") {
      return("//enac1files.epfl.ch/LASUR")  
    } else {
      stop("Unsupported OS")
    }
  }
```

Détecte l'OS de l'utilisateur et essaye d'obtenir l'emplacement du drive lasur

``` r
# Suppress participants where data has to be erased
  id_suppress <- c("CH21089", "FR3456", "CH18003")
  wave1_data <- wave1_data %>% 
    dplyr::filter(!IDNO %in% id_suppress)
```

Supprime les participant·es ayant demandé la suppression de leurs données

``` r
  wave1_data <- wave1_data[wave1_data$Participe == 1,]
```

Ne garde que les participant·es ayant complété entièrement le questionnaire

``` r
  wave1_data <- recoder_adjectifs(wave1_data, suffixe = "voiture", question_nom = "Q97")
  wave1_data <- recoder_adjectifs(wave1_data, suffixe = "train", question_nom = "Q98")
  wave1_data <- recoder_adjectifs(wave1_data, suffixe = "autresTPs", question_nom = "Q99")
  wave1_data <- recoder_adjectifs(wave1_data, suffixe = "velo", question_nom = "Q100")
  
  labels_questions <- list(
    Q97 = "Pouvez-vous citer trois adjectifs qui vous paraissent les plus adaptés pour qualifier la voiture ?",
    Q98 = "Pouvez-vous citer trois adjectifs qui vous paraissent les plus adaptés pour qualifier le train (y compris Léman Express) ?",
    Q99 = "Pouvez-vous citer trois adjectifs qui vous paraissent les plus adaptés pour qualifier les autres transports publics (métro, tram, bus) ?",
    Q100 = "Pouvez-vous citer trois adjectifs qui vous paraissent les plus adaptés pour qualifier le vélo (conventionnel ou électrique) ?"
  )
  
  # Applique les labels aux questions 97 à 100
  for (q in names(labels_questions)) {
    vars <- grep(paste0("^", q, "_\\d+$"), names(wave1_data), value = TRUE)
    for (v in vars) {
      attr(wave1_data[[v]], "label") <- labels_questions[[q]]
    }

    texte_col <- paste0(q, "_TEXTE")
    if (texte_col %in% names(wave1_data)) {
      attr(wave1_data[[texte_col]], "label") <- paste0(labels_questions[[q]], " - TEXTE")
    }
  }
```

Recode les questions 97 à 100 pour qu'elle soient semblable au question 97 à 100 dans la vague 3 en utilisant la fonction `recoder_adjectifs` (voir [adjectifs](wave1.md#fonction-recoder_adjectifs)).

``` r
  wave1_data <- wave1_data |>
    dplyr::rename(
      participant_code = IDNO,
      pays = Pays,
      group = Groupe,
      gp_age_source = GP_Age_source,
      numero_insee = Numéro_INSEE,
      numero_ofs = Numéro_OFS,
      date_naissance = Q106,
      genre = Q107,
      Q31_jour = jourQ31,
      check_genre = Checksex,
      check_age = checkage
    )
  
  names(wave1_data) <- sub("^satisfaction", "Q130_satisfaction", names(wave1_data))
  names(wave1_data) <- sub("^latitude_(.*)", "\\1_latitude", names(wave1_data))
  names(wave1_data) <- sub("^longitude_(.*)", "\\1_longitude", names(wave1_data))
```

Renomme plusieurs variables afin d'harmoniser le nom de certaines variables récurrentes entre les vagues et afin que toutes les variables de questions commencent par "Q".

``` r
# Recodage de Q41 pour qu'elle match la W3
  q41_label <- attr(wave1_data$Q41, "label")
  vals <- as.integer(wave1_data$Q41)
  vals_recoded <- 8 - vals
  
  old_labels <- attr(wave1_data$Q41, "labels")
  new_labels <- setNames(8 - as.integer(old_labels), names(old_labels))
  new_labels <- new_labels[order(new_labels)]
  
  wave1_data$Q41 <- haven::labelled(vals_recoded, new_labels)
  attr(wave1_data$Q41, "label") <- q41_label
```

La question 41 ayant été codée différemment par FORS entre la W1 et la W3, elle a été recodée afin d'être harmonisée entre les deux vagues

##### Tableau 1 – Recodage de Q41

| Réponses                    | Ancien labellage | Nouveau labellage |
|-----------------------------|------------------|-------------------|
| 7 jours par semaine         | 1                | 7                 |
| 6 jours par semaine         | 2                | 6                 |
| 5 jours par semaine         | 3                | 5                 |
| 4 jours par semaine         | 4                | 4                 |
| 3 jours par semaine         | 5                | 3                 |
| 2 jours par semaine         | 6                | 2                 |
| 1 jour par semaine          | 7                | 1                 |
| Moins d’un jour par semaine | 8                | 0                 |

``` r
# Recodage de Q86 pour qu'elle match la W3
  for (var in c("Q86_1", "Q86_2", "Q86_3")) {
    var_label <- attr(wave1_data[[var]], "label")
    
    old_labels <- attr(wave1_data[[var]], "labels")
    new_labels <- setNames(as.integer(old_labels) - 1, names(old_labels))
    
    new_values <- as.integer(wave1_data[[var]]) - 1
    wave1_data[[var]] <- haven::labelled(new_values, new_labels)
    
    attr(wave1_data[[var]], "label") <- var_label
  }
```

La même chose a eu lieu avec la Q86 qu'avec la Q41. 

##### Tableau 2 – Recodage de Q86

| Réponses | Ancien labellage | Nouveau labellage |
|----------|------------------|-------------------|
| Aucun-e  | 1                | 0                 |
| 1        | 2                | 1                 |
| 2        | 3                | 2                 |
| 3        | 4                | 3                 |
| 4        | 5                | 4                 |
| 5        | 6                | 5                 |
| 6        | 7                | 6                 |
| 7        | 8                | 7                 |
| 8        | 9                | 8                 |
| 9        | 10               | 9                 |
| 10+      | 11               | 10                |


```r
if (!dir.exists(here::here("data/wave1"))) {
    dir.create(here::here("data/wave1"), recursive = TRUE)
  }
```
Création du dossier `data/wave1` où sont enregistrés les fichiers csv.
``` r
  doubles_to_convert <- grep("^Q|^(genre|date_naissance)$", names(wave1_data), value = TRUE)
  wave1_data[doubles_to_convert] <- double_to_integer(wave1_data[doubles_to_convert])
```

Certaines variables étaient des `<double>` alors qu'en réalité, elles ne contenaient que des `<integer>`. Afin de permettre à OPAL d'assigner le bon type à chaque variable et d'exécuter ses analyses sans fautes, ces colonnes ont été transformées en `<integer>`.
