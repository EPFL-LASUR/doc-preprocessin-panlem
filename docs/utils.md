# Fonction helpers

Cette page documente les fonctions du fichier `utils.R`.

## Fonction `labels_to_string`
Cette fonction reprend les labels obtenus grâce à la fonction [`get_labels`](utils.md#fonction-get_lables) et les transforme en un string. Le format visé est `1: texte_label_1, 2: texte_label_2, etc...`.

## Fonction `zap_all`
Applique toutes les fonctions du package `haven::zap_*` excepté `zap_empty`.

## Fonction `get_labels`
Cette fonction extrait les lables des variables:
```r
x <- x |>
    dplyr::select(tidyselect::where(haven::is.labelled))
```
Sélectionne les colonnes labellisées du dataframe passé en argument
```r
if (ncol(x) == 0) {
    return(tibble::tibble(
      "{names_to}" := character(),
      "{name}" = character(), "{value}" = integer()
    ))
  }
```
Retourne un tibble vide si aucune colonne labellisées n'existe
```r
label_list <- purrr::imap(
    x,
    ~ tibble::enframe(attr(.x, "labels"), name = name, value = value) |>
      dplyr::mutate(
        !!name := stringr::str_squish(
          stringr::str_replace_all(.data[[name]], "(\n|\t)", " ")
          ),
        !!names_to := .y
      ) |>
      dplyr::select(!!names_to, !!name, !!value)
  )
```
Récupère les labels, supprimes les retour à la ligne et les retours chariot, et les organise dans un tibble
```r
value_types <- purrr::map_chr(label_list, ~ typeof(.x[[value]]))
  unique_types <- unique(value_types)
  
  if (length(unique_types) > 1) {
    label_list <- purrr::map(label_list, ~ dplyr::mutate(.x, !!value := as.character(.data[[value]])))
  }
```
Transforme toutes les valeurs dans la colonne `value` en string si plusieurs types existent, afin de limiter les erreurs
```r
dplyr::bind_rows(label_list)
```
Concatène toutes les lignes des tibble en un seul tibble

## Fonction `remove_escapeseqs`
Supprime tout les retours à la ligne et les retours charriot des chaînes de caractère du dataframe passé en argument

## Fonction `pivot_responses`
Pivote le dataframe passé en argument en version longue avec `pivot_longer` et supprimes les NA. Ne sélectionne que les chaînes de caractère ou que les valeurs numériques, selon l'argument passé.

## Fonction `remove_whitespace`
Supprime les espaces du string passé en argument

## Fonction `extract_lat_lng`
Extrait les coordonnées du json-string passé en argument et les rend en format *lat, lng*. Utilisé pour reformater les coordonnées livrées par FORS.

## Fonction `is_coordinates`
Détecte si la valeur passée en argument est une coordonnée en format *lat, lng*.

## Fonction `double_to_integer`
Transforme les `<double>` du dataframe passé en argument en `<integer>`, tout en gardant les labels des valeurs. Utilisé afin de permettre à OPAL de faire tourner ses analyses.