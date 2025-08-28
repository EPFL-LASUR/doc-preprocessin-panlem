# Vague loisirs été

Cette page documente le preprocessing de la vague loisirs été effectué dans le fichier `preprocess_loisirs_ete.R`.

## Fonction `get_section`
Le but est le même que dans la [vague 1](wave1.md#fonction-get_section). Néanmoins, cette vague possédait des sous-sections, la structure de la fonction est différente
```r
if (num >= 34 && num < 43) {
    decimal <- as.numeric(sub("^Q[0-9]+\\.(\\d+).*", "\\1", question_code))
    if (!is.na(decimal)) {
      return(switch(as.character(decimal),
                    "1" = "4.1 Aller chez des ami·es, la famille",
                    "2" = "4.2 Restaurants, bars, …",
                    "3" = "4.3 Activités culturelles",
                    "4" = "4.4 Activité de divertissement et de consommation",
                    "5" = "4.5 Activités sportives et de détente en extérieur",
                    "6" = "4.6 Sports sur terrain ou en salle",
                    "4. Les loisirs"
      ))
    }
    return("4. Les loisirs")
  }
```
Les sous-sections 4.1 à 4.6 sont attribuées avec le code ci-dessus, tandis que le reste des sections est attribué avec le code ci-dessous:
```r

  if (num >= 1 && num <= 4) return("1. L’emploi du temps estival")
  if (num >= 5 && num <= 21) return("2. Les courts séjours")
  if (num >= 22 && num <= 32) return("3. Les excursions")
  if (num >= 33 && num <= 42 || num == 131) return("4. Les loisirs")
  if (num >= 43 && num <= 45) return("5. Données personnelles")
  
  
  return(NA)
```

## Fonction `documentation`
Similaire à la [vague 1](wave1.md#fonction-documentation)

## Fonction `write_files_questions`
Similaire à la [vague 1](wave1.md#fonction-write_files_questions).

## Fonction `write_label_file`
Similaire à la [vague 1](wave1.md#fonciton-write_label_file)

## Fonction `write_file_section`
Similaire à la [vague 1](wave1.md#fonction-write_file_section)

## Fonction `write_files_participants`
Similaire à la [vague 3](wave3.md#fonction-write_files_participants). Seules les questions `Q43` (date de naissance) et `Q107_V1` (genre) ont été rajoutée.

## Fonction `write_files_survey_completion`
Similaire à la [vague 1](wave1.md#fonction-write_files_survey_completion)

## Fonction `write_file_answers`
Similaire à la [vague 1](wave1.md#fonction-write_file_answers)

## Fonction `main`
Similaire à la [vague 3](wave3.md#fonction-main). Les différences sont notées ci-dessous:
```r
data_ete <- data_ete %>%
  dplyr::rename_with(
    ~ gsub("^_", "", gsub("(.*)(latitude|longitude)(.*)", "\\1\\3_\\2", .)),
    .cols = matches("latitude|longitude")
  )
```
Si les variables contiennent des géodonnées, la mention *latitude* ou *longitude* est déplacée à la fin du nom de la variable.

```r
data_ete <- data_ete %>%
    dplyr::rename_with(
      ~ gsub("^END_(Q[0-9.]+(?:_.*)?)$", "\\1_END", .)
    )
```
De même pour les variables appartennant à la dernière section et le suffixe *END*.
```r
# Renomme les questions 34 à 42
  data_ete <- data_ete %>%
    dplyr::rename_with(~ sub("^x[1-6]_", "", .))
```
Les questions 34 à 42 étaient nommées `x1_Q34_1_1`, `x4_Q38_4_3`, etc. Le préfixe `x*_` a donc été supprimé afin de faire commencer le nom de la variable par "Q".