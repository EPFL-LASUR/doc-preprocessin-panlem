# Vague acceptabilité sociale de mesures d'aménagement du territoire

Cette page documente le preprocessing de la vague sur l'aménagement du territoire effectué dans le fichier `preprocess_pdcn.R`. L'abbréviation PDCn référe au Plan Directeur Cantonal.

## Fonction `get_section`
Similaire à la [vague été](ete.md#fonction-get_section)

## Fonction `documentation`
Similaire à la [vague été](ete.md#fonction-documentation)

## Fonction `write_file_section`
Similaire à la [vague été](ete.md#fonction-write_file_section)

## Fonction `write_files_questions`
Similaire à la [vague été](ete.md#fonction-write_files_questions)

## Fonction `write_label_file`
Similaire à la [vague été](ete.md#fonction-write_label_file)

## Fonction `write_files_survey_completion`
Similaire à la [vague 3](wave3.md#fonction-write_files_survey_completion)

## Fonction `write_files_survey_completion`
Similaire à la [vague 3](wave3.md#fonction-write_files_survey_completion). Les colonnes avec les poids pour la pondération ont été rajoutés:
```r
ponderation_colnames <- names(data)[startsWith(names(data), "wgt")]
```

## Fonction `write_file_answers`
Similaire à la [vague 3](wave3.md#fonction-write_file_answers)

## Fonction `main`
Similaire à la [vague 3](wave3.md#fonction-main). Le drive du LaSUR possédait encore un fichier rassemblant des poids pour la pondération socio-économique. Ceux-ci ont dont été rajoutés aux données brutes:
```r
ponderation <- read.csv(file.path(folder, raw_ponderation))
  
  col_ponderation <- c("IDNO", "wgt_cant", "wgt_agg", "wgt_cant_trim", "wgt_agg_trim")
  weights <- ponderation %>% dplyr::select(dplyr::all_of(col_ponderation))
  
  data_pdcn <- data_pdcn %>% dplyr::left_join(weights, by = "IDNO")
```