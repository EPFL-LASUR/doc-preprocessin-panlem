# Vagues loisirs hiver

Cette page documente le preprocessing de la vague loisirs hiver effectué dans le fichier `preprocess_loisirs_hiver.R`.

## Fonction `get_section`
Similaire à la [vague été](ete.md#fonction-get_section)

## Fonction `documentation`
Similaire à la [vague été](ete.md#fonction-documentation)

## Fonction `write_label_file`
Similaire à la [vague été](ete.md#fonction-write_label_file)

## Fonction `write_file_section`
Similaire à la [vague été](ete.md#fonction-write_file_section)

## Fonction `write_files_questions`
Similaire à la [vague été](ete.md#fonction-write_files_questions)

## Fonction `write_files_survey_completion`
Similaire à la [vague été](ete.md#fonction-write_files_survey_completion)

## Fonction `write_file_answers`
Similaire à la [vague été](ete.md#fonction-write_file_answers)

## Fonction `main`
Similaire à la [vague été](ete.md#fonction-main). Seules quelques questions ont dû être renommée à la main pour être au format `Q*`:
```r
# Rajoute le "Q" devant le num de la question s'il manque
  cols_a_renommer <- c("34.4", "35.4", "36.4", "41.4", "34.5", "35.5")
  names(data_hiver)[names(data_hiver) %in% cols_a_renommer] <- paste0("Q", cols_a_renommer)
  
  data_hiver <- data_hiver %>%
    dplyr::rename_with(~ paste0("Q", .), 
                       .cols = dplyr::starts_with("37.4_1_"))
```