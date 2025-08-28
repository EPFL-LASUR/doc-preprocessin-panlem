# Vague 2: Consommation I

Cette page documente le preprocessing de la deuxième vague du Panel Lémanique effectué dans le fichier `preprocessing-wave2.R`.
```r
wave2_data <- haven::read_sav(
  file.path(
    folder, "EPFL Panel LÇmanique Vague 2 _FINAL_EPFL.sav"
  )
)

# Suppress participants where data has to be erased
id_suppress <- c("CH21089", "FR3456", "CH18003")
wave2_data <- wave2_data %>% 
  dplyr::filter(!IDNO %in% id_suppress)

wave2_data <- wave2_data[wave2_data$Particip_v2 == 1,]
```
Les données brutes sont lues depuis le drive LaSUR. Les participant·es ayant demandés à ce que leurs données soient supprimées sont effacé·es par sécurité, puis seules les personnes ayant rempli entièrement le questionnaire sont gardées.
```r
names(wave2_data) <- janitor::make_clean_names(names(wave2_data), case = "snake", ascii = TRUE)

# The names of four columns are slightly different in the database, so we change them here
wave2_data <- wave2_data |>
  dplyr::rename(
    participant_code = idno,
    group = groupe,
    count_miss1 = countmiss1,
    count_miss2 = countmiss2,
    numero_insee = insee,
    numero_ofs = ofs
  )
```
Les noms des colonnes sont harmonisés avec le package `janitor` et certaines colonnes sont renommées.
```r
# Columns to be included in the participants file. This is currently more liberal than the table
# definition
participants_colnames <- c(
  "participant_code", "pays", "group", "gp_age_source", "numero_insee", "numero_ofs", "weight",
  "titre_source", "cp_source", "localite_source", "titre_actuel", "cp_actuel", "localite_actuel", "code_raison_contact_1_v2",
  "code_raison_contact_2_v2", "code_raison_contact_3_v2", "particip_v2"
)

# Columns to be included in the survey_metadata (also named survey_completion) table
survey_metadata_colnames <- c(
  "count_miss1", "count_miss2", "progress", "start_date", "end_date", "temps_minute"
)

# These are additional column names that are currently simply discarded for the MVP
# TODO: figure out with Panel team what to do with these additional variables.
extra_colnames <- c(
  "particip_avant_changements",
  "flag_troll", , "suppression_suite_v2", "flag_chgmt_pays", "mobile_ordi",
  "avant_chgmt_vet", "flag_chgmt_localite_v2"
)

# Prepare the questions and question_labels output
questions <- wave2_data |>
  dplyr::select(
    -tidyselect::all_of(participants_colnames),
    -tidyselect::all_of(survey_metadata_colnames),
    -tidyselect::all_of(extra_colnames)
  )
```
Les colonnes avec les questions sont choisies.
```r
questions <- questions |>
  purrr::map(~ attr(.x, "label")) |>
  unlist() |>
  tibble::enframe(name = "question_code", value = "question_text")
```
Les labels des questions sont obtenus et triés dans un tibble
```r
questions <- questions |>
  dplyr::mutate(section_name = stringr::str_extract(question_code, "^[:alpha:]+(?=\\_)")) |>
  dplyr::mutate(section_name = dplyr::case_match(
    section_name,
    "ali" ~ "Alimentation",
    "con" ~ "Consommation",
    "end" ~ "Satisfaction",
    "ene" ~ "Energie",
    "equ" ~ "Equipement",
    "log" ~ "Logement",
    "rep" ~ "Rep",
    "temp" ~ "Temperature",
    "vot" ~ "Votation"
  ))

# Some of the questions include escape characters (\n) that need to be removed
questions <- remove_escapeseqs(questions)
```
Les noms des sections sont modifiés, et les retour à la ligne sont supprimés avec la fonction [`remove_escapeseqs`](utils.md#fonction-remove_escapeseqs).
```r
sections <- questions |>
  dplyr::select(section_name) |>
  dplyr::group_by(section_name) |>
  dplyr::slice_head(n = 1) |>
  dplyr::ungroup()
```
Le fichier contenant les sections est préparé
```r
participants <- wave2_data |>
  dplyr::select(tidyselect::all_of(participants_colnames))

participant_labels <- get_labels(participants)

participants <- participants |>
  zap_all() |>
  dplyr::mutate(numero_insee = strtoi(numero_insee, base = 10L))
```
Les fichiers contenant les informations sur les participant·es et les labels des variables concernées sont préparés.
```r
survey_completion <- wave2_data |>
  dplyr::select(
    participant_code, count_miss1, count_miss2, progress, start_date, end_date, temps_minute,
    flag_troll
  )

survey_completion_labels <- get_labels(survey_completion)

survey_completion <- survey_completion |>
  zap_all()
```
De même pour le fichier avec les métadonnées et les labels des variables concernées.
```r
responses <- wave2_data |>
  dplyr::select(
    -all_of(c(participants_colnames[-1], survey_metadata_colnames, extra_colnames))
  ) |>
  zap_all()

response_texts <- pivot_responses(
  responses,
  selection_type = "character", remove_NAs = TRUE, names_to = "question_code",
  values_to = "response_text"
) |>
  remove_escapeseqs()

response_values <- pivot_responses(
  responses,
  selection_type = "numeric", remove_NAs = TRUE, names_to = "question_code",
  values_to = "response_value"
)

responses <- response_values |>
  dplyr::bind_rows(response_texts)
```
Enfin, les réponses sont obtenues et formatées en format long avec la fonction [`pivot_responses`](utils.md#fonction-pivot_responses).
```r
readr::write_tsv(participant_labels, here::here(output_folder, "participant_labels.tsv"))
readr::write_tsv(participants, here::here(output_folder, "participants.tsv"))
readr::write_tsv(question_labels, here::here(output_folder, "question_labels.tsv"))
readr::write_tsv(questions, here::here(output_folder, "questions.tsv"))
readr::write_tsv(sections, here::here(output_folder, "sections.tsv"))
readr::write_tsv(survey_completion, here::here(output_folder, "survey_completion.tsv"))
readr::write_tsv(
  survey_completion_labels, here::here(output_folder, "survey_completion_labels.tsv")
)
readr::write_tsv(responses, here::here(output_folder, "responses.tsv"))
```
Les fichiers csv sont finalement écrits.
```r
doubles_to_convert <- setdiff(names(df), c(participants_colnames, survey_metadata_colnames, extra_colnames))
wave2_data[doubles_to_convert] <- double_to_integer(wave2_data[doubles_to_convert])
```
Les `<double>` sont transformés en `<integer>` avec la fonction [`double_to_integer`](utils.md#fonction-double_to_integer) afin de faciliter l'intégration dans OPAL.