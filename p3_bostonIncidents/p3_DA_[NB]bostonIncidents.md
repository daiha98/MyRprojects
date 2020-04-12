---
title: "p3_DA_[NB]bostonIncidents"
author: Felipe Daiha
date: April 12, 2020
output: github_document
---


remotes::install_github('rstudio/rmarkdown')


```{r Instalando Pacotes}
install.packages('dplyr') ## Manipulacao de Dados
install.packages('ggplot2') ## Visualizacao Grafica dos Dados
install.packages('RgoogleMaps') ## Visualizacao Espacial c/ Google Maps
install.packages('raster') ## Ferramentas para trabalhar c/ Shapefiles
```


```{r Carregando Pacotes}
library("dplyr", lib.loc="~/R/win-library/3.6")
library("ggplot2", lib.loc="~/R/win-library/3.6")
library("RgoogleMaps", lib.loc="~/R/win-library/3.6")
library("raster", lib.loc="~/R/win-library/3.6")
```


```{r Upload do DB}
crime = read.csv(
  "C:/Users/felip/Desktop/Cursos/Kaggle/bostonCrimes_kgl/crime.csv")
```


```{r Primeiras Linhas do DB}
head(crime)
```


```{r Summary dos Dados Pos-Processamento}
summary(crime_preproccess)
```


```{r Armazenando o Mapa de Boston no Google Maps}
coord_boston = GetMap(center = c(lat = 42.36025, lon = -71.05829), 
                    destfile = tempfile("boston_map", fileext = ".png"), 
                    zoom = 11, type = 'google-m')
```


```{r Plotando o Mapa da Cidade}
boston_map = PlotOnStaticMap(coord_boston)
```


```{r Ocorrências dos Crimes Distribuidos no Mapa}
boston_map = PlotOnStaticMap(coord_boston)
crime_occ_map = PlotOnStaticMap(boston_map, lon = crime_preproccess$Long, 
                lat = crime_preproccess$Lat, destfile = 'crime_occ_map.png',
                FUN = points, col = "red", add = T)
```


```{r Colocando o Contorno dos Bairros}
shp_nB = shapefile(
  "C:/Users/felip/Desktop/Cursos/Kaggle/bostonCrimes_kgl/Boston_Neighborhoods.shp")

  df_shp_nB = as.data.frame(shp_nB)
  
    ### Colocando na mesma projecao do Google Maps:
    crs = CRS("+proj=longlat +datum=WGS84")
    shp_nB = spTransform(shp_nB, crs)
  
    ### Transformando para 'SpatialPolygons' que e o formato que o
    ### 'PlotPolysOnStaticMap' aceita o poliogono:
    
    shp_nB = SpatialPolygons(Srl = shp_nB@polygons)
  
# Importando os poligonos para nosso mapa:
    
boston_map = PlotOnStaticMap(coord_boston)
crime_occ_map = PlotOnStaticMap(boston_map, lon = crime_preproccess$Long, 
                lat = crime_preproccess$Lat, destfile = 'crime_occ_map.png',
                FUN = points, col = "red", add = T)
PlotPolysOnStaticMap(MyMap = crime_occ_map, polys = shp_nB, add = T)
```


```{r Crimes Existentes}
levels.default(sort(crime_preproccess[["OFFENSE_CODE_GROUP"]]))
```


```{r Encoding dos Crimes}
encode_ordinal <- function(x, order = unique(x)) {
  x <- as.numeric(factor(x, levels = order, exclude = NULL))
  x
}


# Encoding Ordinal da feature 'OFFENSE_CODE_GROUP':

crime_preproccess[["OFFENSE_SORT_ENCODED"]] = 
  encode_ordinal(crime_preproccess[["OFFENSE_CODE_GROUP"]],
    order = levels.default(sort(crime_preproccess[["OFFENSE_CODE_GROUP"]])))
```


```{r Barplot - Crimes}
ggplot(data = crime_preproccess, aes(x = OFFENSE_SORT_ENCODED)) +
  geom_bar(aes(y = (..count..)), position = 'dodge', width = 0.5, fill = 'blue') +
  geom_text(stat = 'count', aes(label = ..count..), vjust = -1, size = 3) +
  xlab('Código dos Crimes') +
  ylab('Frequência Absoluta') +
  labs(title = "Tipos de Ocorrências", 
       subtitle = "Crimes cometidos na cidade de Boston-MA: 
Junho/2015 a Outubro/2019 (Fonte: data.boston.gov)") +
  scale_x_discrete(limits = c(1:67)) +
  theme_classic()
```


```{r Pieplot - Envolvimento c/ Arma de Fogo}
n = sum(crime_preproccess$SHOOTING == 'N')
y = sum(crime_preproccess$SHOOTING == 'Y')

shoot_perc = c(n, y)

piepercent = paste(round((100 * shoot_perc)/(sum(shoot_perc)), 2), 
                   "%", sep="")


# Fazendo um grafico do tipo pizza para verificar se as ocorrencias cometidas
# tiveram envolvimento de tiro ou nao:

pie(shoot_perc, labels = piepercent, col = c('darkgrey', 'white'),
    main = 'Porcentagem de Crimes\nEnvolvimento com Arma de Fogo\nBoston-MA Crimes', border = 'black')
legend("bottomright", c('No', 'Yes'), cex = 0.9, fill = c('darkgrey', 'white'))
text(0, 1, "Junho/2015 a Outubro/2019 (Fonte: data.boston.gov)", col = "black")
```


```{r PreProcess - Time Series}
# Criando a coluna 'DATE' para trabalhar com time series:

  ## Copiando dados de 'OCCURRED_ON_DATE':

  crime_preproccess$DATE = crime_preproccess$OCCURRED_ON_DATE

  ## Transformando para class 'character':

  crime_preproccess$DATE = as.character(crime_preproccess$DATE)

  ## Transformando para class 'POSIXlt':
  
  crime_preproccess$DATE = strptime(crime_preproccess$DATE, 
                                    format = "%Y-%m-%d %H:%M:%S")
  
  ## Alterando para class 'date':
  
  crime_preproccess$DATE = format(crime_preproccess$DATE, "%Y-%m-%d")
  crime_preproccess$DATE = as.Date.character(crime_preproccess$DATE)
```


```{r Time Series - Ocorrencias por Mes/Ano}
ggplot(data = crime_preproccess, aes(x = MONTH)) +
  geom_line(stat = "count", colour = 'darkblue', size = 0.5) +
  facet_grid(YEAR ~.) +
  geom_text(stat = 'count', aes(label = ..count..), 
            vjust = -1, size = 3) +
  scale_x_continuous(labels = c(1:12), 
                     breaks = c(1:12)) +
  scale_y_continuous(limits = c(4000, 10000),
                     breaks = c(seq(4000, 10000, by = 2000))) +
  xlab('Mês') +
  ylab('N° Total de Crimes') +
  labs(title = 'Número de Crimes por Mes-Ano', 
       subtitle = "Boston-MA: Junho/2015 a Setembro/2019 (Fonte: data.boston.gov)") +
  theme_minimal()
      ### Como a pesquisa termina no início de Outubro/2019, nao foi posta no grafico por não apresentar dados relativos do mes todo.
```


```{r Time Series - Ocorrencias por Hora/Ano}
ggplot(data = crime_preproccess, aes(x = HOUR)) +
    geom_line(stat = "count", colour = 'darkgrey', size = 1) +
    facet_grid(YEAR ~.) +
    geom_text(stat = 'count', aes(label = ..count..), 
              vjust = -1, size = 3) +
    scale_x_continuous(labels = c(0:23), 
                       breaks = c(0:23)) +
    scale_y_continuous(limits = c(0, 7500),
                       breaks = c(seq(0, 7500, by = 2500))) +
    xlab('Hora') +
    ylab('N° Total de Crimes') +
    labs(title = 'Número de Crimes por Hora-Ano', 
         subtitle = "Boston-MA: Junho/2015 a Setembro/2019 
         (Fonte: data.boston.gov)") +
    theme_minimal()
```


```{r Heat Map - PreProccess}
# Utilizando a funcao 'fortify' para transformar o shapefile em 
# dataframe e pegar as coordenadas dos poligonos:
  
shp_nB.fort = shp_nB
shp_nB.fort = fortify(shp_nB.fort)
```


```{r [1] Heat Map - Crimes em Geral}
ggplot(shp_nB.fort, aes(x = long, y = lat, group = group)) +
    geom_polygon(colour = 'black', fill = 'white') +
    stat_density2d(data = crime_preproccess, aes(x = Long, y = Lat, 
                                                 fill = ..level..), 
                   alpha = 0.5, inherit.aes = FALSE, geom = "polygon") +
    scale_fill_distiller(palette = "Spectral") +
    theme_minimal()
```


```{r [2] Heat Map - Crimes por Ano}
ggplot(shp_nB.fort, aes(x = long, y = lat, group = group)) +
    geom_polygon(colour = 'black', fill = 'white') +
    stat_density2d(data = crime_preproccess, aes(x = Long, y = Lat, 
                                                 fill = ..level..), 
                   alpha = 0.5, inherit.aes = FALSE, geom = "polygon") +
    facet_wrap(~YEAR) + ## Code Added
    scale_fill_distiller(palette = "Spectral") +
    theme_minimal()
```
