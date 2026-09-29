# Classificação com Aprendizagem de Máquina (Não Supervisionada e Supervisionada) / Machine Learning Classification (Unsupervised and Supervised)

---

## Português

Este módulo aborda o fluxo de classificação de imagens de satélite no Google Earth Engine. O pipeline contempla o carregamento do mosaico pré-processado, uma etapa exploratória de agrupamento não supervisionado via K-Means, a geração de pontos amostrais aleatórios dentro de geometrias de interesse, a extração de atributos espectrais, o treinamento do classificador supervisionado Random Forest e a exportação do mapa final.

---

## 1. Carregamento do Mosaico Pré-processado

Carrega-se o mosaico gerado na etapa anterior a partir dos Assets do Earth Engine e aplicamos uma visualização em falsa-cor (SWIR 1, NIR e Red).

```javascript
var imageID = 'projects/USER_PROJECT/assets/mosaic-tome-acu';
var mosaic = ee.Image(imageID);

var imageVisParam = {
  bands: ['swir1_median', 'nir_median', 'red_median'],
  min: 0.03470877557992935,
  max: 0.5702108144760132
};

Map.centerObject(mosaic, 10);
Map.addLayer(mosaic, imageVisParam, 'mosaico');

```

---
## 2. Agrupamento Não Supervisionado com K-Means

Antes do treinamento supervisionado, a classificação não supervisionada auxilia na análise exploratória do mosaico, identificando agrupamentos naturais de pixels com comportamentos espectrais similares sem a necessidade prévia de classes rotuladas:

1. Amostragem não supervisionada (`mosaic.sample`): Extrai pixels de forma representativa sobre a área de estudo.
2. Treinamento do agrupador (`ee.Clusterer.wekaKMeans`): Configura o algoritmo K-Means para segmentar o espaço de atributos em $k$ agrupamentos espectrais (neste exemplo, 3 agrupamentos).
3. Agrupamento (`mosaic.sample`): Aplica o modelo treinado a todas as bandas do mosaico e exibe os agrupamentos resultantes com cores aleatórias (`randomVisualizer`).

```javascript
var samples = mosaic.sample(mosaic.geometry(), 30, null, null, 30000, 1, true, 1, true);
var cluster = ee.Clusterer.wekaKMeans({
  init: 1, nClusters: 3, fast: false, seed: 42
}).train(samples.limit(25000), mosaic.bandNames());
var clusterized = mosaic.cluster(cluster);
Map.addLayer(clusterized.randomVisualizer(), {}, 'Mosaic - KMEANS', false);
```

---

## 3. Função para Amostragem Aleatória Estratificada

Para capturar a variabilidade espectral interna de cada classe sem coletar manualmente centenas de coordenadas pontuais, definimos uma função que distribui N pontos pseudoaleatórios dentro de polígonos delimitados (`ee.FeatureCollection.randomPoints`) e associa a propriedade numérica `class` a cada ponto.

```javascript
var generatePoints = function(polygons, nPoints) {
  var points = ee.FeatureCollection.randomPoints(polygons, nPoints);
  var classValue = polygons.first().get('class');
  points = points.map(function(point) {
    return point.set('class', classValue);
  });
  return points;
};
```

---

## 4. Geração e Fusão das Amostras de Treinamento

Utilizando conjuntos de polígonos desenhados previamente na interface (por exemplo, `samples_veg`, `samples_nonVeg` e `samples_water`), chamamos a função `generatePoints` para extrair amostras balanceadas e fundimos todas as classes em uma única `ee.FeatureCollection`.

Classes de referência adotadas:

* Classe 1: Vegetação (`samples_veg`)
* Classe 2: Não Vegetação (`samples_nonVeg`)
* Classe 3: Água (`samples_water`)

```javascript
var vegetationPoints    = generatePoints(samples_veg, 100);
var notVegetationPoints = generatePoints(samples_nonVeg, 100);
var waterPoints         = generatePoints(samples_water, 50);
var samples = vegetationPoints.merge(notVegetationPoints).merge(waterPoints);
print('Amostras coletadas:', samples);
Map.addLayer(samples, {color: 'red'}, 'samples RF', false);
```

---

## 5. Extração Espectral dos Pontos Amostrais

Com a coleção de pontos unificada, extraímos os valores espectrais do mosaico subjacente utilizando o redutor espacial `reduceRegions`. Em seguida, aplicamos um filtro para remover amostras que coincidam com valores nulos causados por máscaras de nuvem ou bordas da cena.

```javascript
var trainedSamples = mosaic.reduceRegions({
  collection: samples,
  reducer: ee.Reducer.first(),
  scale: 30
});
trainedSamples = trainedSamples.filter(ee.Filter.notNull(['blue_max']));
print('Amostras com atributos espectrais:', trainedSamples);

```

---

## 6. Configuração e Treinamento do Random Forest

Instanciamos o algoritmo `smileRandomForest` especificando 50 árvores de decisão. Treinamos o classificador com a tabela de amostras extraídas, indicando a coluna de classe-alvo (`classProperty`) e o vetor com todas as bandas preditoras.

```javascript
var classifier = ee.Classifier.smileRandomForest({
  numberOfTrees: 50
});

classifier = classifier.train({
  features: trainedSamples,
  classProperty: 'class',
  inputProperties: [
    'blue_max',   'blue_median',   'blue_min',
    'green_max',  'green_median',  'green_min',
    'red_max',    'red_median',    'red_min',
    'nir_max',    'nir_median',    'nir_min',
    'swir1_max',  'swir1_median',  'swir1_min',
    'swir2_max',  'swir2_median',  'swir2_min',
    'ndvi_max',   'ndvi_median',   'ndvi_min',
    'ndwi_max',   'ndwi_median',   'ndwi_min'
  ]
});

```

---

## 7. Execução da Classificação e Visualização Cartográfica

Aplicamos o modelo Random Forest treinado diretamente sobre a imagem multibanda utilizando `.classify()`. Exibimos a imagem resultante utilizando uma paleta de cores categórica.

```javascript
var classification = mosaic.classify(classifier);

Map.addLayer(classification, {
  min: 0,
  max: 3,
  palette: ['ffffff', '00aa00', 'ff0000', '0000ff'],
  format: 'png'
}, 'classification');

```

---

## 8. Exportação da Classificação como Asset

Exportamos a imagem classificada para os Assets do Earth Engine. Como o dado é discreto/categórico, a política de piramidação definida deve ser obrigatoriamente `mode` (moda estatística), preservando o valor inteiro da classe ao reduzir resoluções de zoom.

```javascript
Export.image.toAsset({
  image: classification,
  description: 'classification-tome-acu',
  assetId: 'classification-tome-acu',
  pyramidingPolicy: {'.default': 'mode'},
  region: classification.geometry(),
  scale: 30,
  maxPixels: 1e13
});

```

---

## English

This module covers both unsupervised and supervised machine learning pipelines in Google Earth Engine. It guides through loading a preprocessed composite, performing exploratory unsupervised clustering via K-Means, generating stratified random points within digitized training polygons, extracting spectral predictors, training a Random Forest classifier, and exporting the final thematic map.

---

## 1. Loading the Preprocessed Composite

We load the multi-band composite created in the previous module from Earth Engine Assets and display a false-color composite (SWIR 1, NIR, and Red) to assist in visual target recognition.

```javascript
var imageID = 'projects/USER_PROJECT/assets/mosaic-tome-acu';
var mosaic = ee.Image(imageID);

var imageVisParam = {
  bands: ['swir1_median', 'nir_median', 'red_median'],
  min: 0.03470877557992935,
  max: 0.5702108144760132
};

Map.centerObject(mosaic, 10);
Map.addLayer(mosaic, imageVisParam, 'mosaic');

```

---
## 2. Unsupervised Clustering with K-Means

Prior to supervised classification, unsupervised clustering provides exploratory insight into the dataset by identifying natural pixel groupings with similar spectral signatures without requiring predefined training labels:

1. Unsupervised Pixel Sampling (`mosaic.sample`): Extracts a representative sample of pixel feature vectors across the area.
2. Clusterer Training (`ee.Clusterer.wekaKMeans`): Fits a K-Means model to partition the feature space into $k$ clusters (3 clusters in this implementation).
3. Cluster Inference (`mosaic.cluster`): Assigns each pixel of the mosaic to its nearest cluster centroid and renders clusters using randomized visualization colors (`randomVisualizer`).

```javascript
var samples = mosaic.sample(mosaic.geometry(), 30, null, null, 30000, 1, true, 1, true);
var cluster = ee.Clusterer.wekaKMeans({
  init: 1, nClusters: 3, fast: false, seed: 42
}).train(samples.limit(25000), mosaic.bandNames());
var clusterized = mosaic.cluster(cluster);
Map.addLayer(clusterized.randomVisualizer(), {}, 'Mosaic - KMEANS', false);
```


## 3. Stratified Random Sampling Function

To capture the intra-class spectral variance across land covers without manually collecting single point coordinates, we declare a helper function that generates pseudo-random points within geometry boundaries (`ee.FeatureCollection.randomPoints`) and writes the integer `class` property to each point feature.

```javascript
var generatePoints = function(polygons, nPoints) {
  var points = ee.FeatureCollection.randomPoints(polygons, nPoints);
  var classValue = polygons.first().get('class');
  points = points.map(function(point) {
    return point.set('class', classValue);
  });
  
  return points;
};

```

---

## 4. Sampling Generation and Merging

Using digitized training polygons imported in the editor interface (`samples_veg`, `samples_nonVeg`, and `samples_water`), we execute `generatePoints` and merge individual class subsets into a unified `ee.FeatureCollection`.

Target classes:

* Class 1: Vegetation (`samples_veg`)
* Class 2: Non-vegetation (`samples_nonVeg`)
* Class 3: Water (`samples_water`)

```javascript
var vegetationPoints    = generatePoints(samples_veg, 100);
var notVegetationPoints = generatePoints(samples_nonVeg, 100);
var waterPoints         = generatePoints(samples_water, 50);

var samples = vegetationPoints.merge(notVegetationPoints).merge(waterPoints);
print('Merged samples:', samples);
Map.addLayer(samples, {color: 'red'}, 'samples RF', false);

```

---

### 5. Extracting Spectral Predictors

We extract the underlying pixel spectral values for all training locations using `reduceRegions`. An integrity check (`ee.Filter.notNull`) drops points located on missing values caused by cloud masking or scene edges.

```javascript
var trainedSamples = mosaic.reduceRegions({
  collection: samples,
  reducer: ee.Reducer.first(),
  scale: 30
});

trainedSamples = trainedSamples.filter(ee.Filter.notNull(['blue_max']));
print('Extracted training features:', trainedSamples);

```

---

## 6. Random Forest Configuration and Model Training

We instantiate `ee.Classifier.smileRandomForest` configured with 50 decision trees. We fit the model using the sampled features, designating `class` as the prediction target and listing all statistical and spectral index bands as independent input variables.

```javascript
var classifier = ee.Classifier.smileRandomForest({
  numberOfTrees: 50
});

classifier = classifier.train({
  features: trainedSamples,
  classProperty: 'class',
  inputProperties: [
    'blue_max',   'blue_median',   'blue_min',
    'green_max',  'green_median',  'green_min',
    'red_max',    'red_median',    'red_min',
    'nir_max',    'nir_median',    'nir_min',
    'swir1_max',  'swir1_median',  'swir1_min',
    'swir2_max',  'swir2_median',  'swir2_min',
    'ndvi_max',   'ndvi_median',   'ndvi_min',
    'ndwi_max',   'ndwi_median',   'ndwi_min'
  ]
});

```

---

## 7. Executing Model Inference and Visualization

We apply the fitted Random Forest model across the entire input mosaic using `.classify()`. The categorical predictions are displayed on the map using a discrete color palette.

```javascript
var classification = mosaic.classify(classifier);
Map.addLayer(classification, {
  min: 0,
  max: 3,
  palette: ['ffffff', '00aa00', 'ff0000', '0000ff'],
  format: 'png'
}, 'classification');

```

---

## 8. Exporting Thematic Image to Asset

We export the classified image to Earth Engine Assets. Because thematic classifications represent discrete categorical data, the pyramiding policy must be set to `mode` to prevent decimal aggregation across zoom scales.

```javascript
Export.image.toAsset({
  image: classification,
  description: 'classification-tome-acu',
  assetId: 'classification-tome-acu',
  pyramidingPolicy: {'.default': 'mode'},
  region: classification.geometry(),
  scale: 30,
  maxPixels: 1e13
});

```