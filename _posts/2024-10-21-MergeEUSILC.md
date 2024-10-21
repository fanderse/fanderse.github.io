---
layout: post
title: On Merging EU-SILC Data
---

**Why EU-SILC?**

Recently I pitched my draft PhD project to some Sciences Po faculty members and to my fellow PhD students. I essentially planned to compare German and French labor markets and career dynamics using linked employer-employee data. One member of the audience commented that it was unfortunate that the project would only compare two countries (small-n), because of the extensive data requirements within the countries (linked information of millions of employees and their workplaces over many years, huge-n). 

But then I thought, if I separated the project better into sub-projects, in the first step, I would not need linked data. A simple dataset of individuals tracked over time would be sufficient, given that information on their occupational position was available (irrespective of employer or workplace). One week later, I applied for access to the EU-SILC data provided by Eurostat.

The EU Statistics on Income and Living Conditions are an excellent tool for researchers to explore a variety of topics related to life course sociology, research on social stratification and mobility, and economics sociology. The data are available for up to 32 countries from 2004 to 2023. The EU-SILC offers an extensive set of variables, is very well harmonized and pre-processed data with a comprehensive documentation, allowing researchers to dive into the data with ease. For this post, I assume a basic understanding of the contents and creation of EU-SILC. For background see [here](https://ec.europa.eu/eurostat/web/microdata/european-union-statistics-on-income-and-living-conditions). 
 
**Sounds great, let’s dive into the data!**

Unfortunately, as I had to realize, the data is delivered in single .csv files. There is the household register (D-file), the household file (H-file), the individual register (R-file), and the personal data (P-file). Within each of these files, you find observations on the household or the individual level over four years. Thus, each file covers multiple countries and four years of the rotating panel. When you order the complete EU-SILC set, you will get a bunch of .csv files.

For many research projects, however, we need a merged version of these files. We may want to combine household-level and individual-level information or pool information across multiple available years. I found a package for R called [r.eusilc](https://github.com/muuankarski/r.eusilc), but it does not seem to have been maintained for eight years. I have not tried it, and instead decided to build my own program.

**Merging EU-SILC Data**

The user only needs to tell the program where the base .csv files are stored. The program then runs a loop with an added condition over all the base .csv files and merges, if requested, the register files and the “content” files of the respective level. At the end of the procedure, a simple full_join() brings the household and the individual level together. Doing it this way proved to be the most efficient in terms of storage, considering the size of the dataset. The full code is displayed below and the script can be downloaded via the github repo.

```{r}
# only required package
library(dplyr)

```

```{r}
# please specify path to files
base_path <- ""
setwd(base_path)

# would you like to merge register files?
# "y" for yes, "n" for no
# I recommend merging them!
m <- "y"

# please specify file names
file_names_data <- c(
  "",
  "",
  "",
  ""
)

# please specify file names
file_names_registers <- c(
  "",
  "",
  "",
  ""
)

```


```{r}
# create function to load and merge csv files
bind_rows_silc <- function(file_names_data, base_path) {
  # initialize an empty list to store data frames
  data_list <- list()
  j <- 1
  # loop through each file name
  for (file_name in file_names_data) {
    # build the full file path
    file_path <- file.path(base_path, file_name)
    
    # read indiividual csv
    data <- read.csv(file_path)
    
    # append to data list
    data_list[[length(data_list) + 1]] <- data
    # run gc() to free space during the loop
    gc()
    
    # combine data frames in the list into one
    combined_data <- bind_rows(data_list)
    
    # rename rows from register files to the names in the data files
    # for the merge, we use (1) personal ID, (2) country ID, and
    # (3) year as the row identifiers
    
    if (m == "y") {
      file_path <- file.path(base_path, file_names_registers[j])
      register <- read.csv(file_path)
      if (colnames(register)[1] == "RB010") {
      combined_data <- full_join(combined_data, register,
                                 by = c("PB030" = "RB030",
                                        "PB020" = "RB020",
                                        "PB010" = "RB010"))
      
    } else {
      combined_data <- full_join(combined_data, register,
                                 by = c("HB030" = "DB030",
                                        "HB020" = "DB020",
                                        "HB010" = "DB010"))
    }
                             
    }
    
    j <- j+1 
  }

  # remove duplicates (if any)
  combined_data <- distinct(combined_data)
  
  # clean up the workspace
  rm(data_list)
  gc()
  
  return(combined_data)
}

```

```{r}
# run function to create composite EU-SILC file with
# multiple annual waves and optional register data
# _p refers to "personal files"
# user may wish to use _h for household files
silc_p <- bind_rows_silc(file_names_data, base_path)
```

```{r}
# save data to workspace
save(silc_p, file = "silc_p.Rdata")
```


**Why does the program merge on three variables?**

Since EU-SILC is a cross-national longitudinal survey, each row refers to a year-country-person or year-country-household observation. Beware, person IDs are not unique across countries and years! The same person ID can appear in Portugal and Austria, and identifying unique individuals requires a composite identification of personal ID plus country ID. 

Another excellent resource for getting started with the data is this [collection](https://www.gesis.org/gml/european-microdata/eu-silc) of one of my former employers. 

**Now, let the data exploration begin!**












