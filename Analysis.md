---
title: "PEC"
author: "Lira"
date: "07/11/2019"
output: html_document
---

Os pacotes necessários

```{r PACKAGES}

library(tidyverse)
library(ggmap)
library(leaflet)

```

Ativar o serviço do Google no GGMAP. 

Para isso, não basta só o "ggmap", é necessário ter acesso a algum API de geolocalização. 
Um API muito popular e usado aqui foi o "Geocoding API" do Google (https://developers.google.com/maps/documentation/geocoding/intro). 
Basta criar uma conta no Google Cloud Platform para ter acesso à chave que pode ser usada para captar rapidamente informações de geolocalização.
Observações: você vai ter que colocar alguma forma de pagamento para ter acesso ao serviço, e ele não cobrará nada de você caso envolva no máximo 40.000 pesquisas por mês, mas.... se você ultrapassou isso: cada pesquisa irá custar US$ 0.05. Tenha cuidado!

```{r ACTIVATE GOOGLE SERVICE}

#Coloque a chave do seu API
register_google(key = "YOUR_API_KEY")

#Verifique se a chave funciona
ggmap::has_google_key()

```

Baixar a base de dados fiscais e ajustar ela para o exercício

```{r DATASET2}

fiscal <- read.csv2("finbra.csv", 
                    encoding = "utf-8", 
                    header = TRUE, 
                    skip = 3)

fiscal2 <- fiscal %>% 
  filter(Coluna == "Receitas Brutas Realizadas",
         Conta %in% c("1.7.0.0.00.0.0 - Transferências Correntes", "Total Receitas")) %>% 
  spread(Conta, Valor) %>% 
  mutate(Share = `1.7.0.0.00.0.0 - Transferências Correntes`/`Total Receitas`)

```

Baixar a base de dados da população e consertar ela

```{r DATASET}

cities <- read.csv2("Cities.csv", encoding = "utf-8")

cities <- cities %>% 
  mutate(Município = as.character(Município))

cities2 <- stringr::str_split_fixed(cities$Município, "\\(", 2) %>% 
  as.data.frame()

cities2 <- cities2 %>% 
  rename("Município" = V1,
         "UF" = V2) %>% 
  mutate(Município = trimws(Município, which = "right"),
         UF = as.character(UF),
         UF = str_remove(UF, "\\)"))

cities2 <- cities2 %>% 
  add_column(População = cities$População_residente_estimada_.Pessoas._2019) %>% 
  mutate(lugar = paste(Município, UF, sep = " - "),
         lugar = str_to_lower(paste(lugar, ", brazil")))

```

Conseguir as coordenadas

```{r GEOCODING, eval=FALSE, include=FALSE}

cities2 <- cities2 %>% 
  mutate_geocode(lugar, 
                 output = "latlona", 
                 source = "google")

write.csv(cities2, file = "Cities2.csv", row.names = FALSE)

```

Plotar as informações no mapa

```{r PLOT1}

cities3 <- read.csv("Cities2.csv")

cities3 %>% 
  filter(População < 5000) %>% 
  leaflet() %>% 
  addTiles() %>% 
  addMarkers(~lon, ~lat,
             clusterOptions = markerClusterOptions())


```

Buscar uma plotagem mais bonita

```{r PLOT2}

cities3 %>%
  filter(População < 5000) %>%
  leaflet() %>%
  addProviderTiles(providers$Stamen.Toner) %>%
  addCircleMarkers(~lon, ~lat,
                   color = "red",
                   fillOpacity =  0.3,
                   radius = 0.2)

# cities3 %>% 
#   filter(População < 5000) %>% 
#   leaflet() %>% 
#   addProviderTiles(providers$MtbMap) %>%
#   addProviderTiles(providers$Stamen.TonerLines,
#                    options = providerTileOptions(opacity = 0.35)) %>%
#   addProviderTiles(providers$Stamen.TonerLabels) %>% 
#   addCircleMarkers(~lon, ~lat, 
#                    color = "red", 
#                    fillOpacity =  0.3,
#                    radius = 0.2)


```
