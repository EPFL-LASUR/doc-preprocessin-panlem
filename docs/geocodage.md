# Géocodage

Cette page est la documentation du fichier `Geocodage.R`. Dans les données livrées par FORS pour la vague 3, certaines adresses étaient sous forme de coordonnées géographiques (latitude, longitude), tandis que d'autres étaient sous forme d'adresse postale (e.g. 10 Downing Street, London). Afin d'harmoniser les données dans la vague 3 et avec les autres vagues, ces adresses postales ont été géocodées en coordonnées à l'aide du package `tidygeocoder`.

## Fonction `geocoding`
- Description: cette fonction transforme une adresse postale en coordonnées géographique
- Input: un string contenant une adresse
- Output: un string avec les coordonnées géographiques en format *lat,lng*.
- Détail:
```r
if (is.na(addr) || addr == "" || addr == "-998.998,-998.998" || is_coordinates(addr)) {
    return(addr)
```
Vérifie que le string n'est pas vide, un NA, une valeure équivalente à un NA et est bien une adresse à l'aide de la fonction [`is_coordinates`](utils.md#fonction-is_coordinates).
```r
address_df <- data.frame(address = addr, stringsAsFactors = FALSE)
```
Transforme l'adresse en dataframe, vu que c'est le format attendu par la fonction `geocode` du package `tidygeocoder`.
```r

        res <- geocode(.tbl = address_df, address = address, method = "osm",
                       lat = latitude, long = longitude, limit = 1, verbose = FALSE)
        Sys.sleep(1)
        res
      },
      error = function(e) {
        message("Error during geocoding: ", e$message)
        return(NULL)
      }
```
La fonction `geocode` est appliquée sur l'adresse. Une pause d'une seconde est forcée entre chaque requête pour respecter les limites de demande d'OpenStreetMap. Un message d'erreur est généré si nécessaire.
```r
if (is.data.frame(result) && !is.na(result$latitude)) {
      return(paste(result$latitude, result$longitude, sep = ","))
    } else {
      return(NA_character_)
    }
```
Si le résultat est valide, il est formaté sous le format *lat,lng* est retourné. Autrement, un NA est retourné.

## Fonction `main`
```r
raw_data <- "base_ponderee_panel_vague_3_mobilite_clean.sav"
geodata <- "Recodage-adresse.csv"

wave3_data <- haven::read_sav(file.path(folder, raw_data))
existing_coords <- readr::read_tsv(file.path(folder, geodata), show_col_types = FALSE)
```
Les données brutes de la vague 3 sont chargées, ainsi certaines données ayant déjà été géocodées au préalable et enregistrées dans le fichier `Recodage-adresse.csv`.
```r
cols_to_process <- c("W1_q14", "Q14_regroup", "Q43", "Q48", "W1_q48",
                       "Q48_regroup", "Q50", "W1_q50", "Q50_regroup",
                       "Q51", "W1_q51", "Q51_regroup")
  
geolocalised_columns <- wave3_data %>%
  select(IDNO, all_of(cols_to_process)) %>%
  mutate(across(-IDNO, as.character))
```
Les colonnes à géocoder sont sélectionnées et transformées en string.
```r
for (col in cols_to_process) {
  geolocalised_columns[[col]] <- sapply(geolocalised_columns[[col]], extract_lat_lng)
  }
```
Le formatage des coordonnées géographiques livrées dans la vague 3 par FORS étant spécial, ces données sont reformatées de la forme *lat, lng* à l'aide de [extract_lat_lng](utils.md#fonction-extract_lat_lng).
```r
for (col in cols_to_process) {
  output_file <- file.path(output_folder, paste0("geocoded_", col, ".rds"))
  
  if (file.exists(output_file)) {
    message("Skipping already processed column: ", col)
    geolocalised_columns[[col]] <- readRDS(output_file)
  } else {
    message("Geocoding column: ", col)
    geolocalised_columns[[col]] <- map_chr(geolocalised_columns[[col]], geocoding)
    saveRDS(geolocalised_columns[[col]], output_file)
  }
}
```
Les données sont ensuite géocodées. Comme le code était long à tourner, il a été lancé en plusieurs fois. C'est pourquoi, dès qu'une colonne était géocodée, elle est enregistrées en tant que .rds. Avant de lancer le géocodage, on vérifie qu'un fichier .rds n'existe pas déjà, afin de ne pas refaire tourner le code inutilement.
```r
existing_coords <- existing_coords %>%
    mutate(lat_lng = paste(Q14_R_lat, Q14_R_lng, sep = ","))

  geolocalised_columns <- geolocalised_columns %>%
    left_join(existing_coords %>% select(IDNO, lat_lng), by = "IDNO")
  

  cols <- colnames(geolocalised_columns)
  cols <- cols[cols != "lat_lng"]

  new_order <- c(cols[1], "lat_lng", cols[-1])
  geolocalised_columns <- geolocalised_columns[, new_order]

  readr::write_csv(geolocalised_columns, file.path(output_folder, "geolocalised_data.csv"))
```
Les coordonnées contenue dans le fichier `Recodage-adresse.csv` sont reformatées avant d'être ajoutées au fichier avec toutes les coordonnées nouvellement obtenues. Enfin, l'ensemble des coordonnées est sauvegardée dans un fichier csv.