# Vague 3: Mobilité II

## Fonctions `get_section`

Les deux fonctions `get_section_W1` et `get_section_W3` fonctionnent comment la fonction [`get_section`](wave1.md#fonction-get_section) de la vague 1 et permettent de récupérer les sections du questionnaire dans laquelle se trouve la question donnant en argument.

## Fonction `clean_embedded_adjectives`

-   Description: le but de cette fonction est de donner aux questions 97 à 100 un format comme à la W1, donc trois colonnes pour chaque question, une par adjectif sélectionné. Les adjectifs sont ensuite labelled de 1 à 26. Dans la structure de FORS, les adjectifs étaient contenus dans des colonnes nommées *EmbeddedDataCF_36* à *EmbeddedDataCF_61*. Les adjectifs n'étaient en prime pas dans le même ordre pour chaque participant. Il s'agissait donc de construire un dictionnaire des adjectifs et de reconstruire ensuite le choix des particpant·es dynamiquement, selon l'ordre des adjectifs dans leur ligne.
-   Input: les données brutes de la vague 3
-   Output: les données de la vague 3 avec les questions 97 à 100 recodées
-   Détail:

``` r
questions <- paste0("Q", 97:100)
  embedded_cols <- paste0("EmbeddedData", 36:61, "_CF")
  
  # Extracts dynamically all the adjectives present
  adjectives <- data[embedded_cols] |>
    unlist(use.names = FALSE) |>
    as.character() |>
    trimws() |>
    unique()
  
  adjective_list <- adjectives[!is.na(adjectives) & adjectives != ""]
```

Extrait la liste des adjectifs

``` r
  for (q in questions){
    response_cols <- paste0(q, "_", c(1:13, 21:33))
    
    temp <- purrr::pmap_dfr(
      tibble::tibble(
        responses = split(data[response_cols], seq_len(nrow(data))),
        adjectives = split(data[embedded_cols], seq_len(nrow(data)))
      ),
      function(responses, adjectives){
        responses <- as.numeric(responses)
        adjectives <- as.character(adjectives)
        
        selected_positions <- which(responses == 1)
        selected_adjectives <- adjectives[selected_positions]
        standardized <- match(selected_adjectives, adjective_list)
        
        tibble::tibble(
          !!paste0(q, "_adj1") := standardized[1],
          !!paste0(q, "_adj2") := standardized[2],
          !!paste0(q, "_adj3") := standardized[3]
        )
      }
    )
```

Vérifie quels adjectifs ont été sélectionnés par chaque participant·e et les répartit en trois colonnes

``` r
temp <- temp |>
      dplyr::mutate(
        dplyr::across(
          everything(),
          ~ haven::labelled(.x, labels)
        )
      )
    
    results[[q]] <- temp
  }
  final_result <- dplyr::bind_cols(results)
```

Les labels sont ajoutés aux adjectifs

``` r
questions_labels <- c(
    "97" = "Pouvez-vous citer trois adjectifs qui vous paraissent les plus adaptés pour qualifier la voiture ? - Adjectif ",
    "98" = "Pouvez-vous citer trois adjectifs qui vous paraissent les plus adaptés pour qualifier le train (y compris Léman Express) ? - Adjectif ",
    "99" = "Pouvez-vous citer trois adjectifs qui vous paraissent les plus adaptés pour qualifier les autres transports publics (métro, tram, bus) ? - Adjectif ",
    "100" = "Pouvez-vous citer trois adjectifs qui vous paraissent les plus adaptés pour qualifier le vélo (conventionnel ou électrique) ? - Adjectif "
  )
```

Les labels des questions sont crées

``` r
texte_map <- list(
    "Q97_TEXTE" = "Autre_TEXTE_voiture",
    "Q98_TEXTE" = "Autre_TEXTE_train",
    "Q99_TEXTE" = "Autre_TEXTE_autresTPs",
    "Q100_TEXTE" = "Autre_TEXTE_velo"
  )
  
  for (q_num in 97:100) {
    q <- paste0("Q", q_num)
    new_colname <- paste0(q, "_TEXTE")
    source_col <- texte_map[[new_colname]]
    
    if (source_col %in% names(data)) {
      text_vector <- as.character(data[[source_col]])
      text_vector[text_vector == "" | text_vector == "-99"] <- NA
      
      text_col <- haven::labelled(
        text_vector,
        label = paste0(
          gsub(" - Adjectif $", "", questions_labels[as.character(q_num)]),
          " - Autre adjectif (texte libre)"
        )
      )
      
      # Trouver position d'insertion après Qxx_adj3
      insertion_point <- match(paste0(q, "_adj3"), names(final_result)) + 1
      
      if (insertion_point > ncol(final_result)) {
        final_result <- dplyr::bind_cols(
          final_result,
          tibble::tibble(!!new_colname := text_col)
        )
      } else {
        final_result <- dplyr::bind_cols(
          final_result[ , 1:(insertion_point - 1)],
          tibble::tibble(!!new_colname := text_col),
          final_result[ , insertion_point:ncol(final_result)]
        )
      }
    } else {
      warning("Colonne manquante dans les données : ", source_col)
    }
  }
```

Les participant·es avaient également la possibilité de rajouter des adjectifs en cliquant sur "Autre". Ces réponses sous forme de texte sont récupérées, labeled et ajoutées dans le tibble au bon endroit.

``` r
to_remove_q <- unlist(lapply(97:100, function(q) paste0("Q", q, "_", c(1:13, 21:33))))
  to_remove_embedded <- paste0("EmbeddedData", 36:61, "_CF")
  
  adjectif_suffixes <- c("voiture", "train", "autresTPs", "velo")
  pattern <- paste0("^[^Q].*_(", paste(adjectif_suffixes, collapse = "|"), ")$")
  to_remove_adjectif <- grep(pattern, names(data), value = TRUE)
  
  to_remove <- c(to_remove_q, to_remove_embedded, to_remove_adjectif)
  
  insert_pos <- min(which(names(data) %in% to_remove_q))
  
  data_cleaned <- data[ , !(names(data) %in% to_remove)]
  before <- data_cleaned[ , 1:(insert_pos - 1), drop = FALSE]
  after  <- data_cleaned[ , insert_pos:ncol(data_cleaned), drop = FALSE]
  
  data_final <- dplyr::bind_cols(before, final_result, after)
```

Les colonnes obsolétes sont supprimées et les colonnes nouvellement crées insérées à leur place.

``` r
for (q in 97:100) {
    for (adj_num in 1:3) {
      colname <- paste0("Q", q, "_adj", adj_num)
      if (colname %in% names(data_final)) {
        attr(data_final[[colname]], "label") <- paste0(
          questions_labels[as.character(q)],
          adj_num
        )
      }
    }
  }
```

Les colonnes sont labellisées.

## Fonction `write_files_questions`

Reprend la structure de la même fonction dans la [vague 1](wave1.md#fonction-write_files_questions). Néanmoins, certaines questions de la vague 1 étaient reprises sous la forme `W1_q*`. La section de ces questions dans le questionnaire de la vague 1 est obtenue avec la fonction [`get_section_W1][wave3.md#fonction-get_section], tandis que pour les questions de la vague 3, la section est obtenue avec`get_section_W3\`.

Comme certaines questions sont les mêmes entre les vagues 1 et 3, une nouvelle colonne a été créée, indiquant si une question de la vague 3 reprenait une question de la vague 1. Cette colonne est créée avec:

``` r
if (length(matches) > 0) {
          original_match <- names(data)[tolower(names(data)) == matches[1]]
          lien_avec_W1 <- c(lien_avec_W1, original_match)
        } else {
          lien_avec_W1 <- c(lien_avec_W1, NA_character_)
        }
      } else {
        # Si ce n’est pas une Q de W3, alors pas de lien
        lien_avec_W1 <- c(lien_avec_W1, NA_character_)
      }
```

Pour certaines questions, la section ainsi que le lien avec la question de la W1 a été attribué manuellement:

``` r
num <- suppressWarnings(as.integer(stringr::str_extract(i, "(?i)(?<=q)\\d+")))

      if (stringr::str_detect(i, "Q320|Q285|Q291")) {
        section_name <- c(section_name, "W3- Travail/Formation")
      } else if (stringr::str_detect(i, "Q218")) {
        section_name <- c(section_name, "W3- Activités")
      } else if (stringr::str_detect(i, "^Q303(_\\d+)?$")) {
        section_name <- c(section_name, "W3- Séjours")
      } else if (stringr::str_detect(i, "Q129_R")) {
        section_name <- c(section_name, "W3- Satisfaction")
      } else if (stringr::str_detect(i, "W1")) {
        section_name <- c(section_name, get_section_W1(num))
      } else {
        section_name <- c(section_name, get_section_W3(num))
      }
      
      if (stringr::str_detect(i, "^Q325(_\\d+)?$")) {
        lien_avec_W1 <- c(lien_avec_W1, "W1_q117")
      } else if (stringr::str_detect(i, "^Q326(_\\d+)?$")) {
        lien_avec_W1 <- c(lien_avec_W1, "W1_q118")
      } else if (stringr::str_detect(i, "^Q327(_\\d+)?$")) {
        lien_avec_W1 <- c(lien_avec_W1, "W1_q119")
      } else if (stringr::str_detect(i, "^Q\\d+")) {
        prefix <- paste0("w1_q", num)  # lower
        pattern <- paste0("^", prefix, "(_|$)")
```

## Fonction `write_files_embeddeddata`

Les adjectifs des questions 97 à 100 n'étaient pas les seules informations stockées dans des *EmbeddedData*. Cette fonction liste donc les autres *EmbeddedData* ainsi que leur label.

## Fonction `write_file_section`

Similaire à la [vague 1](wave1.md#fonction-write_file_section).

## Fonction `documentation`

Similaire à la [vague 1](wave1.md#fonction-documentation)

## Fonction `write_label_file`

Similaire à la [vague 1](wave1.md#fonction-write_label_file)

## Fonction `write_files_survey_completion`

Similaire à la [vague 1](wave1.md#fonction-write_files_survey_completion)

## Fonction `write_files_participants`

Similaire à la [vague 1](wave1.md#fonction-write_files_participants). Néanmois, certaines variables ont été rajoutées:

-   Les variables *FlagAvantLancement* et *FlagPrblmStatut*

-   Les variables *Titre_source* et *Titre_actuel*, représentant le titre avec lequel les participant·es aimeraient être adressé·es (M, Mme, ...)

-   Les variables *code_raison_contact*, indiquant la raison pour une potentielle prise de contact avec FORS.

## Fonction `main`
Principalement similaire à la [vague 1](wave1.md#fonction-main). Les spécificités sont listées ci-dessous:
```r
geodata <- "geolocalised_data.csv"
raw_q129 <- "EPFL-PanelLémanique-Vague3-mobilité_Q129 anonymisée_EPFL-NOPASSWORD.sav"
```
Certaines géodonnées dans les données brutes de FORS étaient sous forme d'adresses, d'autres en tant que coordonnées. Les géodonnées sous forme d'adresses ont donc été géocodées dans un fichier à part et sont importées ici.

De plus, FORS a envoyé des données manquantes dans un fichier séparé, elles sont donc chargées ici (`raw_Q129`).
```r
wave3_data <- wave3_data[, -which(names(wave3_data) == "W1_q112")]
attr(wave3_data$Q112_regroup, "label") <- "Quelle profession exercez-vous actuellement?"
  attr(wave3_data$Q114_regroup, "label") <- "Quelle profession exercez-vous actuellement?"
  attr(wave3_data$Q41_regroup, "label") <- "En moyenne, combien de jours par semaine travaillez-vous ? - regroupé"
```
La question `W1_q112` étant vide, elle est supprimée. Les labels des questions 112, 114 et 41 sont rajoutés manuellement
```r
coords_csv <- coords_csv |>
    dplyr::rename(Q14 = lat_lng)
  
  names(coords_csv)[2:ncol(coords_csv)] <- paste0(names(coords_csv)[2:ncol(coords_csv)], "_geocoded")
  coords_csv$Q14_geocoded[coords_csv$Q14_geocoded == "NA,NA"] <- NA
  
  # Labels the variables from coords_csv
  for (col in names(coords_csv)) {
    if (endsWith(col, "_geocoded")) {
      original_col <- sub("_geocoded$", "", col)
      
      if (original_col %in% names(wave3_data)) {
        label <- attr(wave3_data[[original_col]], "label")
        if (!is.null(label)) {
          attr(coords_csv[[col]], "label") <- label
        }
      }
    }
  }
  
  wave3_data <- dplyr::left_join(wave3_data, coords_csv, by = "IDNO")
```
Les géodonnées sont labellisées et rajoutées au reste des données
```r
cols_to_process <- c("Q14", "W1_q14", "Q14_regroup", "Q43", "Q48", "W1_q48", "Q48_regroup", "Q50", "W1_q50", "Q50_regroup", "Q51", "W1_q51", "Q51_regroup")
  for (col in cols_to_process) {
    lbl <- attr(wave3_data[[col]], "label")
    wave3_data[[col]] <- sapply(wave3_data[[col]], extract_lat_lng)
    attr(wave3_data[[col]], "label") <- lbl
  }
```
Elles sont ensuite reformatées à l'aide de `extract_lat_lng`.
```r
cols_to_replace <- setdiff(names(Q129), c("Q129_R", "IDNO"))
  for (colname in cols_to_replace) {
    Q129[[colname]][is.na(Q129[[colname]])] <- 0
  }
  
  Q129 <- dplyr::select(Q129, IDNO, Q129_R, Opt_Out_de_tout_le_projet, FlagAutreRepQ129_R) %>%
    dplyr::rename(Suppression_Q129_W3C5 = Opt_Out_de_tout_le_projet)
  
  wave3_data <- dplyr::left_join(wave3_data, Q129, by = "IDNO")
```
Les colonnes importante du fichier `raw_q129` sont rajoutées aux données de base.