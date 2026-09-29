# Construção de Mosaicos e Pré-processamento no Google Earth Engine / Mosaic Generation and Preprocessing in Google Earth Engine
## Português

Este módulo aborda o fluxo completo para carregar uma coleção de imagens de satélite, aplicar filtros temporais e espaciais, construir máscaras de nuvens baseadas em bits de qualidade, calcular índices espectrais de vegetação e água, gerar redutores estatísticos e exportar o produto final.

---
## 1. Filtragem e Seleção da Coleção

### 1.1 Região de Interesse e Janela Temporal

Carregamos o limite vetorial do município de Tomé-Açu/PA e filtramos a coleção de Reflectância de Topo de Atmosfera (TOA) do sensor Landsat 9 para o ano de 2026.

```javascript
// Create a collection filtering by ROI (Tome-Açu municipality) and date range
var tomeAcuShape = ee.FeatureCollection('USER_PATH')
                     .filter(ee.Filter.eq('NM_MUN', 'Tomé-Açu'));
var startDate    = '2026-01-01';
var endDate      = '2026-12-31';
var satellitePath = 'LANDSAT/LC09/C02/T1_TOA';

var collection = ee.ImageCollection(satellitePath)
                   .filterBounds(tomeAcuShape)
                   .filterDate(startDate, endDate);

print('Initial collection:', collection);

```

### 1.2 Filtro de Cobertura de Nuvens nos Metadados

Para evitar o carregamento desnecessário de cenas predominantemente cobertas por nebulosidade, filtramos as imagens contendo até 50% de nuvens no metadado `CLOUD_COVER`.

```javascript
collection = collection.filter(ee.Filter.lte('CLOUD_COVER', 50));
print('Images with less than 50% of cloud cover:', collection);

```

### 1.3 Seleção e Padronização dos Nomes das Bandas

Padronizamos os nomes das bandas espectrais de interesse e a banda de controle de qualidade para facilitar a referência direta nas expressões e reduções.

```javascript
var bandNames = ['B2',   'B3',    'B4',  'B5',  'B6',    'B7',    'QA_PIXEL'];
var rename    = ['blue', 'green', 'red', 'nir', 'swir1', 'swir2', 'pixel_qa'];

collection = collection.select(bandNames, rename);
print('Images with selected bands:', collection);

```

### 1.4 Visualização Inicial da Coleção

Adicionamos ao mapa a coleção bruta em falsa-cor (SWIR1, NIR, Red) para inspecionar visualmente a presença de nuvens e a sobreposição orbital sobre Tomé-Açu.

```javascript
var visParams = {
  bands: ['swir1', 'nir', 'red'],
  min: 0.049351900815963745,
  max: 0.7084327340126038,
};

Map.centerObject(tomeAcuShape, 9);
Map.addLayer(collection, visParams, 'collection');

```

---

## 2. Aplicação das Transformações em Escala

### 2.1 Remoção de Nuvens em Paralelo: Máscara de Nuvens e Sombras via Operações Bitwise

Mapeamos a função `cloudMasking` sobre todos os elementos da coleção para eliminar os artefatos de nuvem e sombra.


Fonte: [USGS Landsat 9 Collection 2 Tier 1 TOA Reflectance: Bands](https://developers.google.com/earth-engine/datasets/catalog/LANDSAT_LC09_C02_T1_TOA#bands)

- Bit 0: Fill
    - 0: Image data
    - 1: Fill data
- **Bit 1: Dilated Cloud**
    - **0: Cloud is not dilated or no cloud**
    - **1: Cloud dilation**
- Bit 2: Cirrus
    - 0: Cirrus Confidence: no confidence level set or Low Confidence
    - 1: High confidence cirrus
- **Bit 3: Cloud**
    - **0: Cloud confidence is not high**
    - **1: High confidence cloud**
- **Bit 4: Cloud Shadow**
    - **0: Cloud Shadow Confidence is not high**
    - **1: High confidence cloud shadow**
- Bit 5: Snow
    - 0: Snow/Ice Confidence is not high
    - 1: High confidence snow cover
- Bit 6: Clear
    - 0: Cloud or Dilated Cloud bits are set
    - 1: Cloud and Dilated Cloud bits are not set
- Bit 7: Water
    - 0: Land or cloud
    - 1: Water
- Bits 8-9: Cloud Confidence
    - 0: No confidence level set
    - 1: Low confidence
    - 2: Medium confidence
    - 3: High confidence
- Bits 10-11: Cloud Shadow Confidence
    - 0: No confidence level set
    - 1: Low confidence
    - 2: Reserved
    - 3: High confidence
- Bits 12-13: Snow / Ice Confidence
    - 0: No confidence level set
    - 1: Low confidence
    - 2: Reserved
    - 3: High confidence
- Bits 14-15: Cirrus Confidence
    - 0: No confidence level set
    - 1: Low confidence
    - 2: Reserved
    - 3: High confidence

A banda de qualidade do Landsat (`QA_PIXEL`, renomeada no fluxo como `pixel_qa`) armazena informações sobre a integridade dos pixels em níveis de bits. Identificamos pixels de nuvens dilatadas (bit 1), nuvens (bit 3) e sombras de nuvens (bit 4) utilizando operações binárias com deslocamento de bits (`<<`) e a operação lógica `bitwiseAnd`. Em seguida, invertemos os bits com `.not()` para preservar apenas as áreas limpas.

```javascript
var cloudMasking = function (image) {
  var dilatedCloud = 1 << 1;
  var cloud        = 1 << 3;
  var shadown      = 1 << 4;
  
  var qaBand = image.select('pixel_qa');
  
  var mask = qaBand.bitwiseAnd(dilatedCloud)
    .or(qaBand.bitwiseAnd(cloud))
    .or(qaBand.bitwiseAnd(shadown));
    
  return image.updateMask(mask.not());
};

var collectionWithoutClouds = collection.map(cloudMasking);

Map.addLayer(collectionWithoutClouds, visParams, 'collection w/o clouds');
print('Collection without clouds:', collectionWithoutClouds);

```

### 2.2 Cálculo e Inclusão dos Índices Espectrais: NDVI e NDWI

Com a coleção limpa, calculamos os índices de Vegetação por Diferença Normalizada (NDVI) e o de Água por Diferença Normalizada NDWI cena por cena encadeando o método `.map()`.

```javascript
var NDVI = function (image) {
  var exp = '( b("nir") - b("red") ) / ( b("nir") + b("red") )';
  var ndvi = image.expression(exp).rename("ndvi");
  return image.addBands(ndvi);
};

var NDWI = function (image) {
  var exp = 'float(b("nir") - b("swir1"))/(b("nir") + b("swir1"))';
  var ndwi = image.expression(exp).rename("ndwi");
  return image.addBands(ndwi);
};

var collectionWithIndexes = collectionWithoutClouds
                            .map(NDVI)
                            .map(NDWI);

var visNdvi = {
  bands: ['ndvi'],
  min: 0,
  max: 1,
  palette: 'ff0000,ffff00,00aa00',
  format: 'png'
};

Map.addLayer(collectionWithIndexes, visNdvi, 'collection with indexes (ndvi)', false);

```

---

## 3. Redutores Temporais e Síntese de Mosaico

### 3.1 Cálculo dos Redutores (Mediana, Mínimo e Máximo)

Colapsamos o eixo temporal da coleção aplicando redutores estatísticos (`ee.Reducer`). Cada redutor gera uma imagem na qual o nome de cada banda recebe o sufixo correspondente (`_median`, `_min`, `_max`).

```javascript
var median = collectionWithIndexes.reduce(ee.Reducer.median());
var min    = collectionWithIndexes.reduce(ee.Reducer.min());
var max    = collectionWithIndexes.reduce(ee.Reducer.max());

Map.addLayer(median, {}, 'collectionWithoutClouds Median (all bands)');
Map.addLayer(min.select('ndvi_min'), {}, 'collectionWithoutClouds Min (NDVI)');
Map.addLayer(max.select('ndvi_max'), {}, 'collectionWithoutClouds Max (NDVI)');

```

### 3.2 Fusão das Bandas e Visualização do Mosaico Final

Combinamos as bandas resultantes em uma única imagem multibanda com `.addBands()` e configuramos a exibição em falsa-cor e gradiente de vegetação.

```javascript
// Merges the median, minimum and maximum mosaics
var mosaic = median.addBands(min).addBands(max);
Map.addLayer(mosaic, {}, 'mosaic with median, min and max', false);

var visNdviMedian = {
  bands: ['ndvi_median'],
  min: 0,
  max: 1,
  palette: 'ff0000,ffff00,00aa00',
  format: 'png'
};

var visFalseColor = {
  bands: ['swir1_median', 'nir_median', 'red_median'],
  min: 0.03446105122566223,
  max: 0.5457578301429749
};

Map.addLayer(mosaic, visFalseColor, 'False color', false);
Map.addLayer(mosaic, visNdviMedian, 'NDVI median mosaic');

print('final mosaic:', mosaic);

```

---

## 4. Exportação do Mosaico como Asset

Para evitar reprocessar a coleção inteira em etapas posteriores de aprendizagem de máquina, exportamos o mosaico gerado para os Assets da conta no Earth Engine.

```javascript
Export.image.toAsset({
  image: mosaic, 
  description: 'mosaic-tome-acu', 
  assetId: 'mosaic-tome-acu', 
  pyramidingPolicy: {'.default': 'mean'}, 
  region: tomeAcuShape, 
  scale: 30, 
  maxPixels: 1e13
});

```


---
# [ENGLISH] Mosaic Generation and Preprocessing in Google Earth Engine

This module covers the complete workflow for loading a satellite image collection, applying temporal and spatial filters, building bitwise-based cloud and shadow masks, calculating spectral indices (NDVI and NDWI), generating statistical temporal reducers, and exporting the final mosaic product.

---

## 1. Collection Filtering and Selection

### 1.1 Region of Interest (ROI) and Temporal Window

We load the administrative boundaries for the municipality of Tomé-Açu/PA and filter the Landsat 9 Top of Atmosphere (TOA) Reflectance collection for the year 2026.

```javascript
// Create a collection filtering by ROI (Tome-Açu municipality) and date range
var tomeAcuShape = ee.FeatureCollection('USER_PATH')
                     .filter(ee.Filter.eq('NM_MUN', 'Tomé-Açu'));
var startDate    = '2026-01-01';
var endDate      = '2026-12-31';
var satellitePath = 'LANDSAT/LC09/C02/T1_TOA';

var collection = ee.ImageCollection(satellitePath)
                   .filterBounds(tomeAcuShape)
                   .filterDate(startDate, endDate);

print('Initial collection:', collection);

```

### 1.2 Metadata-based Cloud Cover Filtering

To prevent loading scenes dominated by cloud cover, we filter images containing up to 50% cloud cover using the `CLOUD_COVER` metadata property.

```javascript
collection = collection.filter(ee.Filter.lte('CLOUD_COVER', 50));
print('Images with less than 50% of cloud cover:', collection);

```

### 1.3 Band Selection and Renaming

We standardize the names of the spectral bands and the pixel quality band to ensure clean syntax across expressions, functions, and reducers.

```javascript
var bandNames = ['B2',   'B3',    'B4',  'B5',  'B6',    'B7',    'QA_PIXEL'];
var rename    = ['blue', 'green', 'red', 'nir', 'swir1', 'swir2', 'pixel_qa'];

collection = collection.select(bandNames, rename);
print('Images with selected bands:', collection);

```

### 1.4 Initial Collection Visualization

We add the raw collection to the map using a false-color composite (SWIR1, NIR, Red) to visually inspect cloud distribution and orbital coverage over Tomé-Açu.

```javascript
var visParams = {
  bands: ['swir1', 'nir', 'red'],
  min: 0.049351900815963745,
  max: 0.7084327340126038,
};

Map.centerObject(tomeAcuShape, 9);
Map.addLayer(collection, visParams, 'collection');

```

---

## 2. Applying Scaled Transformations

### 2.1 Parallel Cloud Masking: Bitwise Operations for Clouds and Shadows

We map the `cloudMasking` function across all images in the collection to mask out cloud and shadow artifacts in parallel.

Source: [USGS Landsat 9 Collection 2 Tier 1 TOA Reflectance: Bands](https://developers.google.com/earth-engine/datasets/catalog/LANDSAT_LC09_C02_T1_TOA?utm_source=gemini#bands)

* Bit 0: Fill
* 0: Image data
* 1: Fill data


* **Bit 1: Dilated Cloud**
* **0: Cloud is not dilated or no cloud**
* **1: Cloud dilation**


* Bit 2: Cirrus
* 0: Cirrus Confidence: no confidence level set or Low Confidence
* 1: High confidence cirrus


* **Bit 3: Cloud**
* **0: Cloud confidence is not high**
* **1: High confidence cloud**


* **Bit 4: Cloud Shadow**
* **0: Cloud Shadow Confidence is not high**
* **1: High confidence cloud shadow**


* Bit 5: Snow
* 0: Snow/Ice Confidence is not high
* 1: High confidence snow cover


* Bit 6: Clear
* 0: Cloud or Dilated Cloud bits are set
* 1: Cloud and Dilated Cloud bits are not set


* Bit 7: Water
* 0: Land or cloud
* 1: Water


* Bits 8-9: Cloud Confidence
* 0: No confidence level set
* 1: Low confidence
* 2: Medium confidence
* 3: High confidence


* Bits 10-11: Cloud Shadow Confidence
* 0: No confidence level set
* 1: Low confidence
* 2: Reserved
* 3: High confidence


* Bits 12-13: Snow / Ice Confidence
* 0: No confidence level set
* 1: Low confidence
* 2: Reserved
* 3: High confidence


* Bits 14-15: Cirrus Confidence
* 0: No confidence level set
* 1: Low confidence
* 2: Reserved
* 3: High confidence



The Landsat quality assessment band (`QA_PIXEL`, renamed in our workflow as `pixel_qa`) stores pixel condition flags as individual bits. We identify dilated clouds (bit 1), clouds (bit 3), and cloud shadows (bit 4) using bit shifts (`<<`) and the logical operation `bitwiseAnd`. Using `.not()` on the combined mask preserves only clear observation pixels.

```javascript
var cloudMasking = function (image) {
  var dilatedCloud = 1 << 1;
  var cloud        = 1 << 3;
  var shadown      = 1 << 4;
  
  var qaBand = image.select('pixel_qa');
  
  var mask = qaBand.bitwiseAnd(dilatedCloud)
    .or(qaBand.bitwiseAnd(cloud))
    .or(qaBand.bitwiseAnd(shadown));
    
  return image.updateMask(mask.not());
};

var collectionWithoutClouds = collection.map(cloudMasking);

Map.addLayer(collectionWithoutClouds, visParams, 'collection w/o clouds');
print('Collection without clouds:', collectionWithoutClouds);

```

### 2.2 Calculating and Appending Spectral Indices: NDVI and NDWI

On the cloud-free collection, we compute the Normalized Difference Vegetation Index (NDVI) and the Normalized Difference Water Index (NDWI) on a scene-by-scene basis by chaining the `.map()` method.

```javascript
var NDVI = function (image) {
  var exp = '( b("nir") - b("red") ) / ( b("nir") + b("red") )';
  var ndvi = image.expression(exp).rename("ndvi");
  return image.addBands(ndvi);
};

var NDWI = function (image) {
  var exp = 'float(b("nir") - b("swir1"))/(b("nir") + b("swir1"))';
  var ndwi = image.expression(exp).rename("ndwi");
  return image.addBands(ndwi);
};

var collectionWithIndexes = collectionWithoutClouds
                            .map(NDVI)
                            .map(NDWI);

var visNdvi = {
  bands: ['ndvi'],
  min: 0,
  max: 1,
  palette: 'ff0000,ffff00,00aa00',
  format: 'png'
};

Map.addLayer(collectionWithIndexes, visNdvi, 'collection with indexes (ndvi)', false);

```

---

## 3. Temporal Reducers and Mosaic Synthesis

### 3.1 Statistical Reducers (Median, Minimum, and Maximum)

We collapse the temporal dimension of the collection using statistical reducers (`ee.Reducer`). Each reducer produces an image where every original band name receives an operational suffix (`_median`, `_min`, `_max`).

```javascript
var median = collectionWithIndexes.reduce(ee.Reducer.median());
var min    = collectionWithIndexes.reduce(ee.Reducer.min());
var max    = collectionWithIndexes.reduce(ee.Reducer.max());

Map.addLayer(median, {}, 'collectionWithoutClouds Median (all bands)');
Map.addLayer(min.select('ndvi_min'), {}, 'collectionWithoutClouds Min (NDVI)');
Map.addLayer(max.select('ndvi_max'), {}, 'collectionWithoutClouds Max (NDVI)');

```

### 3.2 Band Merging and Final Mosaic Visualization

We combine the reduced bands into a single multiband composite using `.addBands()` and set up visualization layers for false-color analysis and vegetation gradients.

```javascript
// Merges the median, minimum and maximum mosaics
var mosaic = median.addBands(min).addBands(max);
Map.addLayer(mosaic, {}, 'mosaic with median, min and max', false);

var visNdviMedian = {
  bands: ['ndvi_median'],
  min: 0,
  max: 1,
  palette: 'ff0000,ffff00,00aa00',
  format: 'png'
};

var visFalseColor = {
  bands: ['swir1_median', 'nir_median', 'red_median'],
  min: 0.03446105122566223,
  max: 0.5457578301429749
};

Map.addLayer(mosaic, visFalseColor, 'False color', false);
Map.addLayer(mosaic, visNdviMedian, 'NDVI median mosaic');

print('final mosaic:', mosaic);

```

---

## 4. Exporting the Mosaic as an Asset

To prevent redundant reprocessing of the entire image stack during downstream machine learning classification steps, we export the final composite directly to Earth Engine Assets.

```javascript
Export.image.toAsset({
  image: mosaic, 
  description: 'mosaic-tome-acu', 
  assetId: 'mosaic-tome-acu', 
  pyramidingPolicy: {'.default': 'mean'}, 
  region: tomeAcuShape, 
  scale: 30, 
  maxPixels: 1e13
});

```