# Where else might the bumblebees be? An exercise in species distribution modelling.

In this exercise, you will turn observation records and climate maps into a **species distribution model (SDM)**. You will make two presentation-ready maps: **where the bee has been recorded** and **where the climate model predicts relatively suitable conditions**. Then you will decide what those maps can, and cannot, tell a conservation manager.

You can use the same code for any of four Swedish bumblebees. The modelling method is **Maxnet**, an R implementation of maximum entropy modelling. It compares the climates at recorded locations with climates available across the study area; it does **not** need confirmed absences.

| Choose this key | Swedish name | Scientific name | Records supplied |
| --- | --- | --- | ---: |
| `"mosshumla"` | Mosshumla | *Bombus muscorum* | 757 |
| `"berghumla"` | Berghumla | *Bombus monticola* | 442 |
| `"blaklockshumla"` | Blåklockshumla | *Bombus soroeensis* | 5,307 |
| `"ljunghumla"` | Ljunghumla | *Bombus jonellus* | 1,852 |

**Mission:** Pick a species, make its maps, and find one *surprise* in the result. When groups finish, compare maps: do different bees seem to favour different parts of Sweden?

## Before you start

1. Download the prepared repository ZIP (or clone the prepared repository) and open `sdm-exercise.Rproj` in RStudio. The `data` folder should be beside the project file. Keep the working directory at the project root: `getwd()` should end in `sdm-exercise`.
2. Install the R packages once, ideally **before class**. The exercise reads all observation, climate and map data from the project folder; it does not download data when you run it. Maxnet does not need Java.
3. Run the code blocks **in order**. Change only `species_key` to switch bees. Files for each species go into a separate folder under `results`. You can also open `README.Rmd` in RStudio and run its chunks or knit it as an illustrated HTML handout (install `rmarkdown` first if RStudio asks).

```r
packages <- c("SDMtune", "terra", "sf", "ggplot2","plotROC")
missing <- packages[!vapply(packages, requireNamespace, logical(1),
                            quietly = TRUE)]
if (length(missing)) {
  stop("Install these R packages before class: ", paste(missing, collapse = ", "))
}

library(SDMtune)
library(terra)
library(sf)
library(ggplot2)
library(plotROC)

set.seed(42)  # Repeatable thinning and background sampling
```

> **Teaching tip:** The prepared files `data/climate_sweden.tif` and `data/sweden.geojson` are **included** in this project. Students do not run a data download. To rebuild these files later, delete them and run `Rscript scripts/prepare_data.R` on a computer with internet access, then commit both resulting files. The script reads three [CHELSA V2.1](https://www.chelsa-climate.org/datasets/chelsa_bioclim) bioclimatic layers for 1981–2010 and a [Natural Earth](https://www.naturalearthdata.com/) boundary, then crops and resamples them to about **2.5 arc-minutes** for a manageable classroom exercise.

## 1. Choose your bee

The keys use plain ASCII so they are easy to type; the filenames still have their Swedish spelling. This is the **only** line you need to change for a different species.

```r
species_key <- "mosshumla"  # Or "berghumla", "blaklockshumla", "ljunghumla"

species <- list(
  mosshumla = list(file = "mosshumla.csv",
                   common = "Mosshumla", latin = "Bombus muscorum"),
  berghumla = list(file = "berghumla.csv",
                   common = "Berghumla", latin = "Bombus monticola"),
  blaklockshumla = list(file = "blåklockshumla.csv",
                        common = "Blåklockshumla", latin = "Bombus soroeensis"),
  ljunghumla = list(file = "ljunghumla.csv",
                    common = "Ljunghumla", latin = "Bombus jonellus")
)

stopifnot(species_key %in% names(species))
bee <- species[[species_key]]
out_dir <- file.path("results", species_key)
dir.create(out_dir, recursive = TRUE, showWarnings = FALSE)
bee$latin
```

## 2. Give the model a climate map

We use just **three** CHELSA variables so that the model and its response curves are easier to explore. They describe annual mean temperature (**BIO1**), annual precipitation (**BIO12**), and precipitation seasonality (**BIO15**). The model can learn curved relationships using linear and quadratic features.

Sweden is our shared **study area** for all four species. We mask out neighbouring countries so the background points and the final map cover the same area.

```r
inputs <- c("data/sweden.geojson", "data/climate_sweden.tif")
if (!all(file.exists(inputs))) {
  stop("Missing prepared files: ", paste(inputs[!file.exists(inputs)],
                                         collapse = ", "),
       ". Ask your teacher for the complete project download.")
}
sweden <- terra::vect("data/sweden.geojson")
climate <- terra::rast("data/climate_sweden.tif")
stopifnot(terra::nlyr(climate) == 3)
names(climate) <- c("bio1", "bio12", "bio15")

cols <- grDevices::colorRampPalette(
  c("#143D55", "#287D91", "#A5CCC0", "#F4D392", "#D76346")
)(100)
names(climate)  # Check the names before plotting response curves

terra::plot(climate[[1]], col = cols, main = "Annual mean temperature")
terra::lines(sweden, col = "#294851")
terra::plot(climate[[2]], col = cols, main = "Total annual precipitation")
terra::lines(sweden, col = "#294851")
terra::plot(climate[[3]], col = cols, main = "Precipitation seasonality")
terra::lines(sweden, col = "#294851")
```

**Look at the climate map:** What parts of Sweden are most different? What else, apart from climate, could make those regions differ for bumblebees?

## 3. Map where people found the bee

The supplied CSVs contain Swedish **RT90 coordinates (EPSG:3021)**. We transform them to longitude and latitude to match WorldClim. These are *observations*, not a systematic survey or proof that the species is absent elsewhere.

```r
records <- read.csv(file.path("data", bee$file), sep = ";",
                    fileEncoding = "UTF-8-BOM")
stopifnot(all(c("Ost", "Nord") %in% names(records)))

observations <- sf::st_as_sf(records, coords = c("Ost", "Nord"),
                             crs = 3021)
observations <- sf::st_transform(observations, 4326)

records_map <- ggplot() +
  geom_sf(data = sf::st_as_sf(sweden), fill = "#EAF1ED",
          colour = "#91A99F", linewidth = 0.25) +
  geom_sf(data = observations, colour = "#C84E3B",
          alpha = 0.45, size = 0.55) +
  coord_sf(expand = FALSE, datum = NA) +
  labs(title = paste(bee$common, "|", bee$latin),
       subtitle = paste(format(nrow(records), big.mark = ","),
                        "supplied observation records"),
       caption = "Dots show reported locations, not surveyed absences") +
  theme_void(base_size = 12) +
  theme(plot.title = element_text(face = "bold", size = 17,
                                  colour = "#17394A"),
        plot.subtitle = element_text(colour = "#536A71"),
        plot.caption = element_text(colour = "#536A71"),
        plot.background = element_rect(fill = "white", colour = NA))

print(records_map)
ggsave(file.path(out_dir, "01_observations.png"), plot = records_map,
       width = 7, height = 8, dpi = 250, bg = "white")
```

**Pause:** Do the dots resemble a biological distribution, a map of where people go, or both? Name a place you would like to survey next.

## 4. Train the model

Maxnet needs **presence points** (records) and **background points** (climates available across Sweden). Background points are *not known absences*. To avoid counting the same climate grid cell repeatedly, keep at most one observation per cell.
Records outside the prepared climate grid (for example, on very small islands) will not enter the model, although they remain visible on the observation map.

```r
xy <- as.data.frame(sf::st_coordinates(observations))
names(xy) <- c("x", "y")
presence <- as.data.frame(SDMtune::thinData(xy, env = climate))

background <- terra::spatSample(climate[[1]], size = 2000,
                                method = "random", na.rm = TRUE,
                                xy = TRUE, values = FALSE)

cat("Original records:", nrow(records), "\n")
cat("Cells retained for modelling:", nrow(presence), "\n")

samples <- SDMtune::prepareSWD(species = bee$latin,
                                p = presence, a = background,
                                env = climate)

# Hold back 20% of presence cells for a brief predictive check later.
parts <- SDMtune::trainValTest(samples, test = 0.2,
                               only_presence = TRUE, seed = 42)

model <- SDMtune::train(method = "Maxnet", data = parts[[1]],
                        fc = "lq", reg = 1)
model
```

The model asks: **which combinations of these three climates occur more often at reported bee locations than across Sweden as a whole?**

## 5. Reveal the suitability map

Now apply the fitted relationships to **every climate grid cell in Sweden**. The palette runs from cool blue (lower relative suitability) to warm coral (higher). This map is a modelled *climatic suitability score*, **not** a census, a confirmed range boundary, or a calibrated probability that a bee occupies a cell.

```r
suitability <- predict(model, data = climate, type = "logistic")

palette <- c("#143D55", "#287D91", "#A5CCC0", "#F4D392", "#D76346")
map_cells <- as.data.frame(suitability, xy = TRUE, na.rm = TRUE)
names(map_cells)[3] <- "score"

suitability_map <- ggplot(map_cells, aes(x = x, y = y, fill = score)) +
  geom_raster() +
  geom_sf(data = sf::st_as_sf(sweden), inherit.aes = FALSE,
          fill = NA, colour = "#294851", linewidth = 0.25) +
  scale_fill_gradientn(colours = palette, limits = c(0, 1),
                       name = "Relative\nsuitability") +
  coord_sf(expand = FALSE, datum = NA) +
  labs(title = paste("Where could", bee$common, "find suitable climate?"),
       subtitle = paste(bee$latin, "| Three climate variables | Sweden"),
       caption = "Relative climate suitability • CHELSA 1981–2010") +
  theme_void(base_size = 12) +
  theme(plot.title = element_text(face = "bold", size = 16,
                                  colour = "#17394A"),
        plot.subtitle = element_text(colour = "#536A71"),
        plot.caption = element_text(colour = "#536A71"),
        legend.position = "bottom",
        legend.key.width = grid::unit(1.5, "cm"),
        plot.background = element_rect(fill = "white", colour = NA))

print(suitability_map)
ggsave(file.path(out_dir, "02_climate_suitability.png"),
       plot = suitability_map, width = 8, height = 8,
       dpi = 250, bg = "white")
```

**Compare the two maps:** Where does the model predict suitable climate without many records? What might explain that gap? Is there an area with records but relatively low modelled suitability?

### What climate relationships did it learn?

Response curves vary one climate variable while keeping the other two at their mean values. The small marks on the x-axis show climates at records and background points. Pay most attention to parts of a curve supported by observations; three climate variables can still be correlated.

```r
climate_labels <- c("Annual mean temperature",
                    "Annual precipitation",
                    "Precipitation seasonality")

for (i in seq_along(climate_labels)) {
  curve <- SDMtune::plotResponse(
    model, var = names(climate)[i], type = "logistic",
    marginal = TRUE, rug = TRUE
  ) +
    labs(title = climate_labels[i],
         subtitle = paste("Model response for", bee$latin),
         y = "Relative suitability score") +
    theme_minimal(base_size = 12)

  print(curve)
  ggsave(file.path(out_dir, paste0("response_bio",
                                   c(1, 12, 15)[i], ".png")),
         plot = curve, width = 7, height = 4.5,
         dpi = 220, bg = "white")
}
```

**Think:** Which variable appears to change the predicted score most? Can a response curve by itself show that temperature or rainfall *causes* the distribution?

## Aside: How convincing is the prediction?

We withheld 20% of presence cells before fitting. AUC asks how often the model ranks a withheld presence above a background location. Compare **training AUC** and **held-out AUC** rather than interpreting a training score alone.

```r
train_auc <- SDMtune::auc(model)
test_auc <- SDMtune::auc(model, test = parts[[2]])
cat(sprintf("Training AUC: %.3f | Held-out AUC: %.3f\n",
            train_auc, test_auc))

roc_plot <- SDMtune::plotROC(model, test = parts[[2]]) +
  labs(title = paste("A predictive check for", bee$latin),
       subtitle = "Training and held-out presence versus background") +
  theme_minimal(base_size = 12)

print(roc_plot)
ggsave(file.path(out_dir, "03_roc.png"), plot = roc_plot,
       width = 7, height = 5, dpi = 220, bg = "white")
```

A high score can still be misleading. Nearby observations may share almost the same climate, even when randomly assigned to opposite sets. AUC also depends on the **background area**: distinguishing a bee's records from the whole of Sweden may be easier than predicting a new locality nearby. Neither AUC nor a striking map tells us whether flowers, nesting habitat, land use, disease, dispersal, or recorder effort limit the species. For serious use, you would examine record quality and dates, consider spatial validation, and test additional ecological predictors. [SDMtune's evaluation guide](https://consbiol-unibern.github.io/SDMtune/articles/evaluation-strategies.html)

## Your final challenge

Work with another group modelling a different species. Show each other the two maps and answer:

1. **Discovery:** What is the most interesting predicted area for your bee, and why?
2. **Evidence:** Do the observed records support that pattern? What do the response curves add?
3. **Decision:** If you could survey only one new area, where would you go? How would a survey help distinguish model error from missing reports?
4. **Limit:** Could these climate-only maps establish whether climate change or land-use change is more important? What new data would you need?

**Takeaway:** An SDM can turn scattered records into an explicit, testable map of where environmental conditions look similar to those at known sites. Its best use is often to generate better questions and guide the next survey.

### Data and tools

- Occurrences: supplied coordinate-only extracts from [Artportalen](https://www.artportalen.se/). Observation dates and survey effort are not included in these teaching files.
- Climate: [CHELSA V2.1](https://www.chelsa-climate.org/datasets/chelsa_bioclim), 1981–2010 baseline; BIO1, BIO12 and BIO15, aggregated to roughly 2.5 arc-minutes. CHELSA lists the data under CC0.
- Boundary: [Natural Earth](https://www.naturalearthdata.com/), 1:10m countries; public domain.
- Modelling: [SDMtune](https://consbiol-unibern.github.io/SDMtune/), Maxnet implementation.
