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

Ativar o serviço do Google no "ggmap". 

Para isso, não basta só o "ggmap", é necessário ter acesso a algum API de geolocalização. 
Um API muito popular e usado aqui foi o "Geocoding API" do Google (https://developers.google.com/maps/documentation/geocoding/intro). 
Basta criar uma conta no Google Cloud Platform para ter acesso à chave que pode ser usada para captar rapidamente informações de geolocalização.
Observações: você vai ter que colocar alguma forma de pagamento para ter acesso ao serviço, e ele não cobrará nada de você caso envolva no máximo 40.000 pesquisas por mês, mas.... se você ultrapassou isso: cada pesquisa irá custar US$ 0.05. Tenha cuidado!

```{r ACTIVATE GOOGLE SERVICE}

#Coloque a chave do seu API
register_google(key = "SUA_CHAVE_DE_API_AQUI")

#Verifique se a chave está ativa
ggmap::has_google_key()

```

Carregar a base de dados. 

Os dados podem ser encontrados no SISCONFI (https://siconfi.tesouro.gov.br/): siga a aba "Consultas" > "Consultar Finbra" > "Contas Anuais". Uma vez lá, marque respectivamente o "Exercício" e "Escopo" como: "2018" e "Municípios". A tabela que precisa ser escolhida é "Receitas Orçamentárias (Anexo I-C)". Pronto! Você acesso a um arquivo .csv compactado que possui as informações detalhadas de receita de todos os municípios do Brasil

```{r DATASET}

fiscal <- read.csv2("finbra_MUN_ReceitasOrcamentarias(AnexoI-C)/finbra.csv", 
                    encoding = "latin1", #normalmente, usa-se "utf-8", mas algumas vezes só dá pra ler com esse encoding
                    header = TRUE, 
                    skip = 3)

```

Ajustar a base de dados fiscais para o exercício. 

Isto inclui deixar os dados na categoria correta, mas também envolve em deixar uma referência para conseguir as coordenadas (latitude e longitude) quando usar o API de geolocalização. Sem essas coordenadas, não é possível desenhar um mapa com os pontos de interesse para a análise. 

Por isso, é importante criar uma variável com um formato adequado para o API captar bem a informação, caso contrário, é pesquisada uma coordenada qualquer que não corresponde à verdadeira. Normalmente, uma referência que funciona bem é nesse formato (vamos pegar o exemplo de São Paulo Capital): "são paulo - sp, brazil"

Observa-se que esse formato vai escolher aleatoriamente qualquer lugar da cidade. 

```{r FIX DATASET}

cities <- fiscal %>% 
  mutate(Instituição = as.character(Instituição),
         UF = str_to_lower(as.character(UF)),
         Cidade = str_remove(Instituição, "Prefeitura Municipal de "),
         address = paste(str_to_lower(Cidade), "brazil", sep = ", ")) %>% 
  filter(Coluna == "Receitas Brutas Realizadas",
         Conta %in% c("1.1.1.0.00.0.0 - Impostos", "Total Receitas")) %>% 
  spread(Conta, Valor) %>% 
  rename("Taxes" = `1.1.1.0.00.0.0 - Impostos`, 
         "Revenue" = `Total Receitas`) %>% 
  mutate(Taxes = replace_na(Taxes, 0), 
         Share = Taxes/Revenue) #Essa informação mostra a proporção de quanto que o município não recebe de recursos externos

```

Com a base de dados ajustada, consiga as coordenadas. 

```{r GEOCODING}

cities <- cities %>% 
  mutate_geocode(address, 
                 output = "latlona", 
                 source = "google")

#quando essas pesquisas são feitas, normalmente é sábio salvar elas num arquivo separado para evitar fazer tal pesquisa de novo (o limite de 40.000 lembra?!)
write.csv(cities, file = "finbra2.csv", row.names = FALSE)

```

Plotar as informações no mapa do Brasil com o auxílio das coordenadas adquiridas. Dessa forma, pode-se visualizar quais cidades potencialmente serão impactadas pela medida caso ocorra sua aprovação. 

```{r PLOT}

cities2 <- read.csv("finbra2.csv")

cities2 %>%
  filter(População < 5000 &
           Share < 0.1) %>% 
  leaflet() %>%
  addProviderTiles(providers$Stamen.Toner) %>%
  addCircleMarkers(~lon, ~lat,
                   color = "red",
                   fillOpacity =  0.3,
                   radius = 0.2)

```
![alt text](https://github.com/JimmyFlorido/PEC188-2019/blob/master/Final.png "Cidades Impactadas")


Além de construir o mapa, é importante se perguntar: caso essas cidades sejam extinguidas, ou seja, se a proposta passar, quantas cidades e eleitores em potencial serão afetadas pela medida? 

```{r THENUMBER}

cities2 %>%
  filter(População < 5000,
         Share < 0.1) %>%
  summarise(Cidades = n(),
            Eleitores = sum(População))

```

| Cidades       | Eleitores     |
| ------------- |:-------------:|
| 1,168         | 3,938,144     |

