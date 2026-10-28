# Where else might the bumblebees be?

*A hands-on exercise in species distribution modelling*

This teaching exercise uses Swedish bumblebee observations and climate data to explore a conservation question: **where might a species occur beyond the places where it has already been recorded?**

Working in **RStudio**, students build a presence–background species distribution model (SDM), map its predictions and consider what those predictions can—and cannot—tell us about habitat suitability and conservation priorities. The exercise emphasises ecological interpretation over model complexity.

**Exercise materials:** [R Markdown handout and executable code](README.Rmd) · [Illustrated PDF handout](SDM_Exercise_Revised.pdf)

## Learning objectives

By the end of the exercise, students should be able to:

1. Map species observations and recognise limitations of opportunistically collected records.
2. Fit and interpret a simple climate-based model of **relative suitability**.
3. Evaluate model predictions and identify the additional evidence needed for conservation decisions.

## Choose a bumblebee

Each group works with one of four Swedish species. The same code runs for all four, allowing groups to compare their results.

| Code key | Swedish name | Species | Supplied records |
| --- | --- | --- | ---: |
| `mosshumla` | Mosshumla | *Bombus muscorum* | 757 |
| `berghumla` | Berghumla | *Bombus monticola* | 442 |
| `blaklockshumla` | Blåklockshumla | *Bombus soroeensis* | 5,307 |
| `ljunghumla` | Ljunghumla | *Bombus jonellus* | 1,852 |

## Getting started

1. **Download** this repository using **Code → Download ZIP**, or clone it to your computer.
2. Open `sdm-exercise.Rproj` in **RStudio**. Keep the project file and `data/` folder in their original relative locations.
3. Install the required packages if you do not already have them:

   ```r
   install.packages(c("SDMtune", "terra", "sf", "ggplot2", "plotROC"))
   ```

4. Open `README.Rmd` and **run the code chunks in order**. At the start of the exercise, select your species by changing `species_key`:

   ```r
   species_key <- "mosshumla"
   ```

   Other choices are `"berghumla"`, `"blaklockshumla"` and `"ljunghumla"`.

5. Follow the **Pause & interpret** questions as you go. The exercise saves figures under `results/<species_key>/`.

All required occurrence, climate and boundary files are expected in `data/`; the analysis does not download data while running. **Maxnet does not require Java.** If you prefer to read the exercise before running the code, use the [PDF handout](SDM_Exercise_Revised.pdf). The R Markdown source includes the full plotting code, some of which is hidden in the PDF for readability.

## What you'll do

| Stage | Activity | Main idea |
| --- | --- | --- |
| 1 | Choose a bee and load the data | Set up a reproducible workflow |
| 2 | Explore climate maps | Understand the predictors |
| 3 | Map observation records | Records are not confirmed absences elsewhere |
| 4 | Fit a Maxnet model | Compare climates at presences with available background climates |
| 5 | Map suitability and response curves | Interpret the model's environmental associations |
| 6 | Examine held-out AUC | Evaluate predictions and their limitations |
| 7 | Write a short conservation report | Translate model results into cautious ecological advice |

The model uses three **CHELSA** climate predictors—annual mean temperature (**BIO1**), annual precipitation (**BIO12**) and precipitation seasonality (**BIO15**)—over Sweden. It uses **Maxnet** through `SDMtune`, with linear and quadratic features, and a held-out subset of presence cells for evaluation.

## Outputs and short report

For your chosen species, the code produces:

- `01_observations.png` — recorded locations;
- `02_climate_suitability.png` — predicted relative climatic suitability;
- `response_bio1.png`, `response_bio12.png` and `response_bio15.png` — climate response curves;
- `03_roc.png` — a predictive evaluation plot.

These files are saved in `results/<species_key>/`.

Use your results to prepare a **short, evidence-based conservation report**. Include the Swedish and scientific names of your species, 2–3 sentences on its ecology, your suitability map, and references to the response curves and held-out AUC. Address four questions:

1. **Where** in Sweden does the model predict the most suitable climate?
2. **Which climatic conditions** are associated with higher suitability, and how cautiously should the response curves be interpreted?
3. **How convincing** are the predictions? What does held-out AUC show, and what does it miss?
4. **What would you advise a conservation manager?** Is climate information enough, or are further surveys and ecological data needed?

## Interpret with care

**This is a teaching model, not a ready-to-use conservation assessment.** The observations are opportunistic rather than a systematic presence–absence survey. The predicted score is **not** a calibrated probability of occupancy, and a high AUC does not establish that the model performs well at new sites. Sampling effort, spatial dependence, floral and nesting resources, dispersal, land use and other factors can all affect where a species actually persists.

**Central takeaway:** A climate-suitability map is a hypothesis about where a species *might* occur—not proof of where it does occur or where conservation action should automatically be directed.

## Data and acknowledgements

- **Occurrence records:** coordinate-only teaching extracts from [Artportalen](https://www.artportalen.se/). Observation dates and survey effort are not included.
- **Climate:** [CHELSA V2.1](https://www.chelsa-climate.org/datasets/chelsa_bioclim), 1981–2010 baseline, BIO1/BIO12/BIO15, approximately 2.5 arc-minutes (CHELSA lists the data under CC0).
- **Country boundary:** [Natural Earth](https://www.naturalearthdata.com/), 1:10m countries (public domain).
- **Modelling:** [SDMtune](https://consbiol-unibern.github.io/SDMtune/) with Maxnet; see also its [evaluation guidance](https://consbiol-unibern.github.io/SDMtune/articles/evaluation-strategies.html).

**Developed by Liam Kendall**, Lund University, for teaching species distribution modelling and critical conservation assessment.
