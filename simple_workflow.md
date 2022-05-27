---
title: "A Simple Workflow"
author: "Simon Goring, Socorro Dominguez Vidaña"
date: "2022-05-27"
output:
  html_document:
    code_folding: show
    fig_caption: yes
    keep_md: yes
    self_contained: yes
    theme: readable
    toc: yes
    toc_float: yes
    css: "text.css"
  pdf_document:
    pandoc_args: "-V geometry:vmargin=1in -V geometry:hmargin=1in"
---


```r
options(warn = -1)
suppressMessages(library(neotoma2))
suppressMessages(library(sf))
suppressMessages(library(geojsonsf))
suppressMessages(library(dplyr))
suppressMessages(library(ggplot2))
suppressMessages(library(leaflet))
```

## Site Searches

### `get_sites()`

There are several ways to find sites in `neotoma2`, but we think of `sites` as being spatial objects primarily. They have names, locations, and are found within the context of geopolitical units, but within the API and the package, the site itself does not have associated information about taxa, dataset types or ages.  It is simply the container into which we add that information.  So, when we search for sites we can search by:

  * siteid
  * sitename
  * location
  * altitude (maximum and minimum)
  * geopolitical unit

#### Site names: `sitename="%Lait%"`

We may know exactly what site we're looking for ("Lac Mouton"), or have an approximate guess for the site name (for example, we know it's something like "Lait Lake", or "Lac du Lait", but we're not sure how it was entered specifically).

We use the general format: `get_sites(sitename="XXXXX")` for searching by name.

PostgreSQL (and the API) uses the percent sign as a wildcard.  So `"%Lait%"` would pick up ["Lac du Lait"](https://data.neotomadb.org/4180) for us (and would pick up "Lake Lait" and "This Old **Lait**y Hei-dee-ho Bog" if they existed).  Note that the search query is also case insensitive, so you could simply write `"%lait%"`.


```r
spo_sites <- neotoma2::get_sites(sitename = "%Lait%")
spo_sites
```

```
##  siteid    sitename      lat    long altitude
##    3220 Lac du Lait 45.31417 6.81528     2190
```

```r
plotLeaflet(spo_sites)
```

```{=html}
<div id="htmlwidget-30ff48b4b4f100b307f7" style="width:672px;height:480px;" class="leaflet html-widget"></div>
<script type="application/json" data-for="htmlwidget-30ff48b4b4f100b307f7">{"x":{"options":{"crs":{"crsClass":"L.CRS.EPSG3857","code":null,"proj4def":null,"projectedBounds":null,"options":{}}},"calls":[{"method":"addProviderTiles","args":["Stamen.TerrainBackground",null,null,{"errorTileUrl":"","noWrap":false,"detectRetina":false}]},{"method":"addTiles","args":["https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",null,null,{"minZoom":0,"maxZoom":18,"tileSize":256,"subdomains":"abc","errorTileUrl":"","tms":false,"noWrap":false,"zoomOffset":0,"zoomReverse":false,"opacity":1,"zIndex":1,"detectRetina":false,"attribution":"&copy; <a href=\"https://openstreetmap.org\">OpenStreetMap<\/a> contributors, <a href=\"https://creativecommons.org/licenses/by-sa/2.0/\">CC-BY-SA<\/a>"}]},{"method":"addCircleMarkers","args":[45.31417,6.81528,10,null,null,{"interactive":true,"draggable":false,"keyboard":true,"title":"","alt":"","zIndexOffset":0,"opacity":1,"riseOnHover":true,"riseOffset":250,"stroke":true,"color":"#03F","weight":5,"opacity.1":0.5,"fill":true,"fillColor":"#03F","fillOpacity":0.2},{"showCoverageOnHover":true,"zoomToBoundsOnClick":true,"spiderfyOnMaxZoom":true,"removeOutsideVisibleBounds":true,"spiderLegPolylineOptions":{"weight":1.5,"color":"#222","opacity":0.5},"freezeAtZoom":false},null,"<b>Lac du Lait<\/b><br><b>Description:<\/b> Lake almost filled. Physiography: depression in mountain. Surrouding vegetation: pasture with rare uncinata.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3220>Explorer Link<\/a>",null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},null]}],"limits":{"lat":[45.31417,45.31417],"lng":[6.81528,6.81528]}},"evals":[],"jsHooks":[]}</script>
```

#### Location: `loc=c()`

The `neotoma` package used a bounding box for locations, structured as a vector of latitude and longitude values: `c(xmin, ymin, xmax, ymax)`.  The `neotoma2` R package supports both this simple bounding box, but also more complex spatial objects, using the [`sf` package](https://r-spatial.github.io/sf/). Using the `sf` package allows us to more easily work with raster and polygon data in R, and to select sites from more complex spatial objects.  The `loc` parameter works with the simple vector, [WKT](https://arthur-e.github.io/Wicket/sandbox-gmaps3.html), [geoJSON](http://geojson.io/#map=2/20.0/0.0) objects and native `sf` objects in R.  **Note however** that the `neotoma2` package is a wrapper for a simple API call using a URL ([api.neotomadb.org](https://api.neotomadb.org)), and URL strings can only be 1028 characters long, so the API cannot accept very long/complex spatial objects.

Looking for sites using a location:


```r
cz <- list(geoJSON = '{"type": "Polygon",
        "coordinates": [[
            [12.40, 50.14],
            [14.10, 48.64],
            [16.95, 48.66],
            [18.91, 49.61],
            [15.24, 50.99],
            [12.40, 50.14]]]}',
        WKT = 'POLYGON ((12.4 50.14, 
                         14.1 48.64, 
                         16.95 48.66, 
                         18.91 49.61,
                         15.24 50.99,
                         12.4 50.14))',
        bbox = c(12.4, 48.64, 18.91, 50.99))

cz$sf <- geojsonsf::geojson_sf(cz$geoJSON)

cz_sites <- neotoma2::get_sites(loc = cz[[1]], all_data = TRUE)
```

You can always simply `plot()` the `sites` objects, but you will lose some of the geographic context.  The `plotLeaflet()` function returns a `leaflet()` map, and allows you to further customize it, or add additional spatial data (like our original bounding polygon):


```r
neotoma2::plotLeaflet(cz_sites) %>% 
  leaflet::addPolygons(map = ., 
                       data = cz$sf, 
                       color = "green")
```

```{=html}
<div id="htmlwidget-05bf5cb8b5e035422e5b" style="width:672px;height:480px;" class="leaflet html-widget"></div>
<script type="application/json" data-for="htmlwidget-05bf5cb8b5e035422e5b">{"x":{"options":{"crs":{"crsClass":"L.CRS.EPSG3857","code":null,"proj4def":null,"projectedBounds":null,"options":{}}},"calls":[{"method":"addProviderTiles","args":["Stamen.TerrainBackground",null,null,{"errorTileUrl":"","noWrap":false,"detectRetina":false}]},{"method":"addTiles","args":["https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",null,null,{"minZoom":0,"maxZoom":18,"tileSize":256,"subdomains":"abc","errorTileUrl":"","tms":false,"noWrap":false,"zoomOffset":0,"zoomReverse":false,"opacity":1,"zIndex":1,"detectRetina":false,"attribution":"&copy; <a href=\"https://openstreetmap.org\">OpenStreetMap<\/a> contributors, <a href=\"https://creativecommons.org/licenses/by-sa/2.0/\">CC-BY-SA<\/a>"}]},{"method":"addCircleMarkers","args":[[49.726332,49.04174,49.76853,48.776394,49.75811,48.985354,48.77444,48.86363,49.232262,48.955424,48.942402,49.143988,50.601566,49.324306,49.69159,49.001672,49.02481,48.988224,49.68134,49.22716,49.250916,48.943358,50.604486,48.981118,48.977644],[15.970602,15.19097,15.372874,16.4207,15.35538,14.708586,14.927156,14.79583,14.621568,14.804076,14.807094,14.703396,14.609992,15.532934,15.45905,14.777506,14.76891,16.392582,15.47796,15.372418,14.087762,17.068908,16.213004,16.663514,17.199456],10,null,null,{"interactive":true,"draggable":false,"keyboard":true,"title":"","alt":"","zIndexOffset":0,"opacity":1,"riseOnHover":true,"riseOffset":250,"stroke":true,"color":"#03F","weight":5,"opacity.1":0.5,"fill":true,"fillColor":"#03F","fillOpacity":0.2},{"showCoverageOnHover":true,"zoomToBoundsOnClick":true,"spiderfyOnMaxZoom":true,"removeOutsideVisibleBounds":true,"spiderLegPolylineOptions":{"weight":1.5,"color":"#222","opacity":0.5},"freezeAtZoom":false},null,["<b>Kameničky<\/b><br><b>Description:<\/b> Drained sloping spring mire. Physiography: Kameničská kotlina Basin at its N margin. Surrounding vegetation: Alder carr.<br><a href=http://apps.neotomadb.org/explorer/?siteids=1399>Explorer Link<\/a>","<b>Bláto<\/b><br><b>Description:<\/b> After deep drainage quite mineral peat. Physiography: broad flat closure of a brook valley. Surrounding vegetation: present: secondary spruce plantations.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3021>Explorer Link<\/a>","<b>Chraňbož<\/b><br><b>Description:<\/b> Small mire. Surrounding vegetation: fagion, Luzulo-Fagion.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3052>Explorer Link<\/a>","<b>Dvůr Anšov<\/b><br><b>Description:<\/b> Fen. Physiography: dyje river alluvium. Surrounding vegetation: carpinion.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3090>Explorer Link<\/a>","<b>Hroznotín<\/b><br><b>Description:<\/b> Small mire. Surrounding vegetation: fagion, Luzulo-Fagion.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3152>Explorer Link<\/a>","<b>Spolí<\/b><br><b>Description:<\/b> NA<br><a href=http://apps.neotomadb.org/explorer/?siteids=3168>Explorer Link<\/a>","<b>Velanská cesta<\/b><br><b>Description:<\/b> NA<br><a href=http://apps.neotomadb.org/explorer/?siteids=3169>Explorer Link<\/a>","<b>Červené blato<\/b><br><b>Description:<\/b> NA<br><a href=http://apps.neotomadb.org/explorer/?siteids=3170>Explorer Link<\/a>","<b>Borkovická blata<\/b><br><b>Description:<\/b> Peat sediment complex undergoing peat excavation.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3171>Explorer Link<\/a>","<b>Branná<\/b><br><b>Description:<\/b> NA<br><a href=http://apps.neotomadb.org/explorer/?siteids=3172>Explorer Link<\/a>","<b>Barbora<\/b><br><b>Description:<\/b> NA<br><a href=http://apps.neotomadb.org/explorer/?siteids=3173>Explorer Link<\/a>","<b>Švarcenberk<\/b><br><b>Description:<\/b> Lakes area. Physiography: Basin in plain. Surrounding vegetation: Cultivated fields and forest.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3174>Explorer Link<\/a>","<b>Jestřebské blato<\/b><br><b>Description:<\/b> Vast swamps. Surrounding vegetation: pino-Quercetum.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3175>Explorer Link<\/a>","<b>Loučky<\/b><br><b>Description:<\/b> Sloping spring fen. Physiography: a closure of broad flat brook valley. Surrounding vegetation: pine and spruce plantations.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3254>Explorer Link<\/a>","<b>Malčín<\/b><br><b>Description:<\/b> Broad alluvium of a stream. Surrounding vegetation: quercion robori-petraeae.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3269>Explorer Link<\/a>","<b>Mokré louky (South)<\/b><br><b>Description:<\/b> Cultural meadow. Surrounding vegetation: quercion robori-petraeae.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3289>Explorer Link<\/a>","<b>Mokré louky (North)<\/b><br><b>Description:<\/b> NA<br><a href=http://apps.neotomadb.org/explorer/?siteids=3290>Explorer Link<\/a>","<b>Olbramovice<\/b><br><b>Description:<\/b> Spring peat bog. Surrounding vegetation: alnenion glutinoso-incanae.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3327>Explorer Link<\/a>","<b>Palašiny<\/b><br><b>Description:<\/b> Broad alluvium of a stream. Surrounding vegetation: quercion robori-petraeae.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3341>Explorer Link<\/a>","<b>Řásná<\/b><br><b>Description:<\/b> Artificial lake with neighbouring fen. Physiography: valley mire. Surrounding vegetation: spruce plantation.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3371>Explorer Link<\/a>","<b>Řežabinec<\/b><br><b>Description:<\/b> Artificial fish pond with marginal fen. Physiography: slightly undulated landscape. Surrounding vegetation: meadows, fields, alder carr.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3377>Explorer Link<\/a>","<b>Svatobořice-Mistřín<\/b><br><b>Description:<\/b> Peat bog. Surrounding vegetation: quercetea robori-petraeae.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3452>Explorer Link<\/a>","<b>Vernéřovice<\/b><br><b>Description:<\/b> Flat valley. Physiography: valley fen, slightly sloping. Surrounding vegetation: cultivated fields, meadow, spruce.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3492>Explorer Link<\/a>","<b>Velké Němčice<\/b><br><b>Description:<\/b> Svratka river bank. Surrounding vegetation: carpinion.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3497>Explorer Link<\/a>","<b>Vracov<\/b><br><b>Description:<\/b> Artificial lake after peat exploitation. Physiography: flat valley. Surrounding vegetation: pine plantations and fields.<br><a href=http://apps.neotomadb.org/explorer/?siteids=3502>Explorer Link<\/a>"],null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},null]},{"method":"addPolygons","args":[[[[{"lng":[12.4,14.1,16.95,18.91,15.24,12.4],"lat":[50.14,48.64,48.66,49.61,50.99,50.14]}]]],null,null,{"interactive":true,"className":"","stroke":true,"color":"green","weight":5,"opacity":0.5,"fill":true,"fillColor":"green","fillOpacity":0.2,"smoothFactor":1,"noClip":false},null,null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},null]}],"limits":{"lat":[48.64,50.99],"lng":[12.4,18.91]}},"evals":[],"jsHooks":[]}</script>
```

#### Site Helpers

If we look at the [UML diagram](https://en.wikipedia.org/wiki/Unified_Modeling_Language) for the objects in the `neotoma2` R package we can see that there are a set of functions that can operate on `sites`.  As we add to `sites` objects, using `get_datasets()` or `get_downloads()`, we are able to use more of these helper functions. As it is, we can take advantage of sunctions like `summary()` to get a more complete sense of the types of data we have as part of this set of sites.  The following code gives the summary table. We do some R magic here to change the way the data is displayed (turning it into a `datatable()` object), but the main piece is the `summary()` call.


```r
neotoma2::summary(cz_sites) %>%
  DT::datatable(data = ., rownames = FALSE, 
                options = list(scrollX = "100%"))
```

```{=html}
<div id="htmlwidget-d068d00465706c16b15f" style="width:100%;height:auto;" class="datatables html-widget"></div>
<script type="application/json" data-for="htmlwidget-d068d00465706c16b15f">{"x":{"filter":"none","vertical":false,"data":[[1399,3021,3021,3052,3052,3090,3152,3168,3169,3170,3170,3170,3171,3171,3171,3172,3173,3174,3174,3174,3175,3254,3269,3289,3290,3327,3341,3341,3371,3377,3452,3492,3497,3502,3502],["Kameničky","Bláto","Bláto","Chraňbož","Chraňbož","Dvůr Anšov","Hroznotín","Spolí","Velanská cesta","Červené blato","Červené blato","Červené blato","Borkovická blata","Borkovická blata","Borkovická blata","Branná","Barbora","Švarcenberk","Švarcenberk","Švarcenberk","Jestřebské blato","Loučky","Malčín","Mokré louky (South)","Mokré louky (North)","Olbramovice","Palašiny","Palašiny","Řásná","Řežabinec","Svatobořice-Mistřín","Vernéřovice","Velké Němčice","Vracov","Vracov"],["KAMEN","BLATO1","BLATO2","CHB1","CHB2","DVURANSO","HROZNOTI","JC-13-A","JC-2-A","JC-3-A","JC-3-AA","JC-3-B","JC-5-C","JC-5-D","JC-5-A","JC-6-A","JC-6-B","JC-7-B","SVARCENB","SVARCEN3","JESTREB","LOUCKY","MALCIN","MLOUKY","MLOUKY-B","OLBRAM","PALAS_A","PALASINY","RASNA","REZABIN","SVATOBOR","VERNER","VNEMCICE","VRACOV","VRACOV1"],[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],[3,2,2,2,2,3,2,2,1,2,2,1,2,3,2,1,2,2,2,2,2,2,1,3,1,2,1,2,2,4,3,2,3,3,2],["pollen,testate amoebae,geochronologic","geochronologic,pollen","pollen,testate amoebae","pollen,testate amoebae","testate amoebae,pollen","testate amoebae,pollen,geochronologic","pollen,testate amoebae","testate amoebae,pollen","pollen","pollen,testate amoebae","testate amoebae,pollen","pollen","testate amoebae,pollen","geochronologic,testate amoebae,pollen","geochronologic,pollen","pollen","testate amoebae,pollen","pollen,testate amoebae","geochronologic,pollen","geochronologic,pollen","pollen,testate amoebae","pollen,geochronologic","pollen","testate amoebae,geochronologic,pollen","pollen","geochronologic,pollen","pollen","geochronologic,pollen","pollen,geochronologic","diatom,testate amoebae,geochronologic,pollen","pollen,geochronologic,testate amoebae","pollen,geochronologic","geochronologic,testate amoebae,pollen","pollen,geochronologic,plant macrofossil","geochronologic,pollen"]],"container":"<table class=\"display\">\n  <thead>\n    <tr>\n      <th>siteid<\/th>\n      <th>sitename<\/th>\n      <th>collectionunit<\/th>\n      <th>chronolgies<\/th>\n      <th>datasets<\/th>\n      <th>types<\/th>\n    <\/tr>\n  <\/thead>\n<\/table>","options":{"scrollX":"100%","columnDefs":[{"className":"dt-right","targets":[0,3,4]}],"order":[],"autoWidth":false,"orderClasses":false}},"evals":[],"jsHooks":[]}</script>
```

We can see here that there are no chronologies associated with the `site` objects. This is because, at present, we have not pulled in the `dataset` information we need.  All we know from `get_sites()` is what kind of datasets we have.

### Searching for datasets:

We know that collection units and datasets are contained within sites.  Similarly, a `sites` object contains `collectionunits` which contain `datasets`. From the table above we can see that some of the sites we've looked at contain pollen records. That said, we only have the `sites`, it's just that (for convenience) the `sites` API returns some information about datasets so to make it easier to navigate the records.

With a `sites` object we can directly call `get_datasets()`, to pull in more metadata about the datasets.  At any time we can use `datasets()` to get more information about any datasets that a `sites` object may contain.  Compare the output of `datasets(cz_sites)` to the output of a similar call using the following:


```r
cz_datasets <- neotoma2::get_datasets(cz_sites, all_data = TRUE)
datasets(cz_datasets) %>% 
  as.data.frame() %>% 
  DT::datatable(data = ., 
                options = list(scrollX = "100%"))
```

```{=html}
<div id="htmlwidget-4cfc6c0ddd9a71f71bda" style="width:100%;height:auto;" class="datatables html-widget"></div>
<script type="application/json" data-for="htmlwidget-4cfc6c0ddd9a71f71bda">{"x":{"filter":"none","vertical":false,"data":[["1","2","3","4","5","6","7","8","9","10","11","12","13","14","15","16","17","18","19","20","21","22","23","24","25","26","27","28","29","30","31","32","33","34","35","36","37","38","39","40","41","42","43","44","45","46","47","48","49","50","51","52","53","54","55","56","57","58","59","60","61","62","63","64","65","66","67","68","69","70","71","72","73"],[1435,8232,10544,4025,8956,10576,4100,10577,4121,10578,4122,4128,4129,10583,10585,4131,4222,9071,4246,4269,9091,10590,4270,9120,4319,4371,9146,9153,10467,10594,4378,4469,9216,10597,9243,4517,4522,10598,9246,3935,8914,3936,10572,3981,10573,3982,10574,4123,10579,4124,10580,4125,4126,10581,10582,4127,9011,24255,24256,4130,10584,24084,24083,24086,24085,4336,4337,9130,9249,4527,41482,24096,24097],["European Pollen Database","European Pollen Database",null,"European Pollen Database","European Pollen Database",null,"European Pollen Database",null,"European Pollen Database",null,"European Pollen Database","European Pollen Database","European Pollen Database",null,null,"European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database",null,"European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database",null,null,"European Pollen Database","European Pollen Database","European Pollen Database",null,"European Pollen Database","European Pollen Database","European Pollen Database",null,"European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database",null,"European Pollen Database",null,"European Pollen Database",null,"European Pollen Database",null,"European Pollen Database",null,"European Pollen Database","European Pollen Database",null,null,"European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database",null,"European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database","European Pollen Database"],["pollen","geochronologic","testate amoebae","pollen","geochronologic","testate amoebae","pollen","testate amoebae","pollen","testate amoebae","pollen","pollen","pollen","testate amoebae","testate amoebae","pollen","pollen","geochronologic","pollen","pollen","geochronologic","testate amoebae","pollen","geochronologic","pollen","pollen","geochronologic","geochronologic","diatom","testate amoebae","pollen","pollen","geochronologic","testate amoebae","geochronologic","pollen","pollen","testate amoebae","geochronologic","pollen","geochronologic","pollen","testate amoebae","pollen","testate amoebae","pollen","testate amoebae","pollen","testate amoebae","pollen","testate amoebae","pollen","pollen","testate amoebae","testate amoebae","pollen","geochronologic","geochronologic","pollen","pollen","testate amoebae","pollen","geochronologic","pollen","geochronologic","pollen","pollen","geochronologic","geochronologic","pollen","plant macrofossil","geochronologic","pollen"],[13680,null,null,11954,null,null,null,null,null,null,null,7091,9019,null,null,null,14178,null,null,11192,null,null,null,null,4407,11537,null,null,null,null,12323,9191,null,null,null,13656,null,null,null,13835,null,null,null,null,null,null,null,17284,null,null,null,null,null,null,null,16014,null,null,13503,null,null,15319,null,13791,null,null,11712,null,null,17361,null,null,14740],[-107,null,null,718,null,null,null,null,null,null,null,0,0,null,null,null,-524,null,null,155,null,null,null,null,460,-36,null,null,null,null,-68,-46,null,null,null,-49,null,null,null,-56,null,null,null,null,null,null,null,-348,null,null,null,null,null,null,null,9400,null,null,-22,null,null,5203,null,1364,null,null,-486,null,null,66,null,null,85],["Data contributed by Rybnícková Eliska.",null,null,"Data contributed by Svobodová Helena.",null,null,"Data contributed by Jankovská Vlasta.",null,"Data contributed by Jankovská Vlasta.",null,"Data contributed by Jankovská Vlasta.","Data contributed by Jankovská Vlasta.","Data contributed by Jankovská Vlasta.",null,null,"Data contributed by Jankovská Vlasta.","Data contributed by Rybníčková Eliška.",null,"Data contributed by Svobodová Helena.","Data contributed by Svobodová Helena.",null,null,"Data contributed by Jankovská Vlasta.",null,"Data contributed by Svobodová Helena.","Data contributed by Rybníčková Eliška.",null,null,null,null,"Data contributed by Rybnícková Eliska.","Data contributed by Svobodová Helena.",null,null,null,"Data contributed by Rybnícková Eliska.","Data contributed by Svobodová Helena.",null,null,"Data contributed by Rybníčková Eliška.",null,"Data contributed by Rybnícková Eliska.",null,"Data contributed by Jankovská Vlasta.",null,"Data contributed by Jankovská Vlasta.",null,"Data contributed by Jankovská Vlasta.",null,"Data contributed by Jankovská Vlasta.",null,"Data contributed by Jankovská Vlasta.","Data contributed by Jankovská Vlasta.",null,null,"Data contributed by Jankovská Vlasta.",null,null,"Data contributed by Jankovská Vlasta.","Data contributed by Jankovská Vlasta.",null,"Data contributed by PALYCZ via Kunes Petr.",null,"Data contributed by PALYCZ via Kunes Petr.",null,"Data contributed by Svobodová Helena.","Data contributed by Jankovská Vlasta.",null,null,"Data contributed by Svobodová Helena.",null,null,"Data contributed by Rybnícková Eliska."]],"container":"<table class=\"display\">\n  <thead>\n    <tr>\n      <th> <\/th>\n      <th>datasetid<\/th>\n      <th>database<\/th>\n      <th>datasettype<\/th>\n      <th>age_range_old<\/th>\n      <th>age_range_young<\/th>\n      <th>notes<\/th>\n    <\/tr>\n  <\/thead>\n<\/table>","options":{"scrollX":"100%","columnDefs":[{"className":"dt-right","targets":[1,4,5]},{"orderable":false,"targets":0}],"order":[],"autoWidth":false,"orderClasses":false}},"evals":[],"jsHooks":[]}</script>
```

If we choose to pull in information about only a single dataset type, or if there is additional filtering we want to do before we download the data, we can use the `filter()` function.  For example, if we only want pollen records, we can filter:


```r
cz_pollen <- cz_datasets %>% 
  neotoma2::filter(datasettype == "pollen")

neotoma2::summary(cz_pollen) %>% DT::datatable(data = ., 
                options = list(scrollX = "100%"))
```

```{=html}
<div id="htmlwidget-139a3cd2b4913a719a8a" style="width:100%;height:auto;" class="datatables html-widget"></div>
<script type="application/json" data-for="htmlwidget-139a3cd2b4913a719a8a">{"x":{"filter":"none","vertical":false,"data":[["1","2","3","4","5","6","7","8","9","10","11","12","13","14","15","16","17","18","19","20","21","22","23","24","25","26","27","28","29","30","31","32","33","34","35"],[1399,3090,3152,3168,3169,3172,3173,3175,3254,3269,3289,3290,3327,3371,3377,3452,3492,3497,3021,3021,3052,3052,3170,3170,3170,3171,3171,3171,3174,3174,3174,3341,3341,3502,3502],["Kameničky","Dvůr Anšov","Hroznotín","Spolí","Velanská cesta","Branná","Barbora","Jestřebské blato","Loučky","Malčín","Mokré louky (South)","Mokré louky (North)","Olbramovice","Řásná","Řežabinec","Svatobořice-Mistřín","Vernéřovice","Velké Němčice","Bláto","Bláto","Chraňbož","Chraňbož","Červené blato","Červené blato","Červené blato","Borkovická blata","Borkovická blata","Borkovická blata","Švarcenberk","Švarcenberk","Švarcenberk","Palašiny","Palašiny","Vracov","Vracov"],["KAMEN","DVURANSO","HROZNOTI","JC-13-A","JC-2-A","JC-6-A","JC-6-B","JESTREB","LOUCKY","MALCIN","MLOUKY","MLOUKY-B","OLBRAM","RASNA","REZABIN","SVATOBOR","VERNER","VNEMCICE","BLATO1","BLATO2","CHB1","CHB2","JC-3-A","JC-3-AA","JC-3-B","JC-5-C","JC-5-D","JC-5-A","JC-7-B","SVARCENB","SVARCEN3","PALAS_A","PALASINY","VRACOV","VRACOV1"],[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],[1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],["pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen"]],"container":"<table class=\"display\">\n  <thead>\n    <tr>\n      <th> <\/th>\n      <th>siteid<\/th>\n      <th>sitename<\/th>\n      <th>collectionunit<\/th>\n      <th>chronolgies<\/th>\n      <th>datasets<\/th>\n      <th>types<\/th>\n    <\/tr>\n  <\/thead>\n<\/table>","options":{"scrollX":"100%","columnDefs":[{"className":"dt-right","targets":[1,4,5]},{"orderable":false,"targets":0}],"order":[],"autoWidth":false,"orderClasses":false}},"evals":[],"jsHooks":[]}</script>
```

We can see now that the data table looks different, and there are fewer total sites.

Note that R is sensitive to the order in which packages are loaded.  Using `neotoma2::` tells R explicitly that you want to use the `neotoma2` package to fun the `filter()` operation.  `filter()` exists in other packages as well, such as `dplyr`, so if you see an error that looks like:

```bash
Error in UseMethod("filter") : 
  no applicable method for 'filter' applied to an object of class "sites"
```

it's likely that the wrong package is trying to run `filter()`.

### Pulling in the `sample()` data.

Because sample data adds a lot of overhead (for the Czech pollen data, the object that includes the dataset with samples is 20 times larger than the `dataset` alone), we try to call `get_downloads()` after we've done our preliminary filtering. After `get_datasets()` you have enough information to filter based on location, time bounds and dataset type.  When we move to `get_download()` we can do more fine-tuned filtering at the analysis unit or taxon level.

The following call can take some time, but we've frozen the object as an RDS data file. You can run this command on your own, and let it run for a bit, or you can just load the object in.


```r
## This line is commented out because we've already run it for you.
## cz_dl <- cz_pollen %>% get_downloads(all_data = TRUE)
cz_dl <- readRDS('data/czDownload.RDS')
```

Once we've downloaded, we now have information for each site about all the associated collection units, the datasets, and, for each dataset, all the samples associated with the datasets.  To extract all the samples we can call:


```r
allSamp <- samples(cz_dl)
```

When we've done this, we get a `data.frame` that is 130889 rows long and 37 columns wide.  The reason the table is so wide is that we are returning data in a **long** format.  Each row contains all the information you should need to properly interpret it:


```
##  [1] "age"             "agetype"         "ageolder"        "ageyounger"     
##  [5] "chronologyid"    "chronologyname"  "units"           "value"          
##  [9] "context"         "element"         "taxonid"         "symmetry"       
## [13] "taxongroup"      "elementtype"     "variablename"    "ecologicalgroup"
## [17] "analysisunitid"  "sampleanalyst"   "sampleid"        "depth"          
## [21] "thickness"       "samplename"      "datasetid"       "siteid"         
## [25] "sitename"        "lat"             "long"            "area"           
## [29] "sitenotes"       "description"     "elev"            "collunitid"     
## [33] "database"        "datasettype"     "age_range_old"   "age_range_young"
## [37] "datasetnotes"
```

For some dataset types, or analyses some of these columns may not be needed, however, for other dataset types they may be critically important.  To allow the `neotoma2` package to be as useful as possible for the community we've included as many as we can.

If you want to know what taxa we have in the record you can use the helper function `taxa()` on the sites object:


```r
neotomatx <- neotoma2::taxa(cz_dl) %>% 
  unique()

DT::datatable(data = head(neotomatx, n = 20), rownames = FALSE, 
                options = list(scrollX = "100%"))
```

```{=html}
<div id="htmlwidget-990899b0b9358b1bc07e" style="width:100%;height:auto;" class="datatables html-widget"></div>
<script type="application/json" data-for="htmlwidget-990899b0b9358b1bc07e">{"x":{"filter":"none","vertical":false,"data":[["grains/tablet","ml","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP"],[null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null],["concentration","volume","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen"],[930,276,219,858,1087,1416,4973,5233,5621,25,391,736,967,3439,3696,4373,3438,4123,160,271],[null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null],["Laboratory analyses","Laboratory analyses","Vascular plants","Vascular plants","Vascular plants","Vascular plants","Vascular plants","Vascular plants","Vascular plants","Vascular plants","Vascular plants","Vascular plants","Vascular plants","Vascular plants","Vascular plants","Vascular plants","Vascular plants","Vascular plants","Vascular plants","Vascular plants"],["concentration","volume","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen","pollen"],["Lycopodium tablets","Sample quantity","Plantago major","Ligustrum","Rumex obtusifolius-type","Galium-type","Viola palustris-type","Peucedanum-type","Chaerophyllum hirsutum-type","Artemisia","Amaranthaceae","Rumex acetosa-type","Secale","Aster-type","Lysimachia vulgaris-type","Heracleum-type","Anthemis-type","Scrophularia-type","Cichorioideae","Salix"],["LABO","LABO","UPHE","TRSH","UPHE","UPHE","AQVP","UPHE","UPHE","UPHE","UPHE","UPHE","UPHE","UPHE","UPHE","UPHE","UPHE","UPHE","UPHE","TRSH"]],"container":"<table class=\"display\">\n  <thead>\n    <tr>\n      <th>units<\/th>\n      <th>context<\/th>\n      <th>element<\/th>\n      <th>taxonid<\/th>\n      <th>symmetry<\/th>\n      <th>taxongroup<\/th>\n      <th>elementtype<\/th>\n      <th>variablename<\/th>\n      <th>ecologicalgroup<\/th>\n    <\/tr>\n  <\/thead>\n<\/table>","options":{"scrollX":"100%","columnDefs":[{"className":"dt-right","targets":3}],"order":[],"autoWidth":false,"orderClasses":false}},"evals":[],"jsHooks":[]}</script>
```

You'll see that the taxonids here can be linked to the taxonid column in the samples.  This allows us to build translation tables if we choose to.  You may also note that the `taxonname` is in the field `variablename`.  This is because of the way that individual samples are reported in Neotoma.

#### Simple Translation

Lets say we want all *Plantago* taxa to be grouped together into one pseudo-taxon.  There are several ways of doing this, either directly, or by creating an external "translation" table (which we did in the prior `neotoma` package).

Here we'll do a simple transformation.  We're using `dplyr` type coding here to `mutate()` the column `variablename` so that any time we detect (`str_detect()`) a `variablename` that starts with `Plantago` (the `.*` represents a wildcard for any character [`.`], zero or more times [`*`]) we `replace()` it with the character string `"Plantago"`.  Note that this changes *Plantago* in the `allSamp` object, but if we were to call `samples()` again, the taxonomy would return to its original form.


```r
allSamp <- allSamp %>% 
  mutate(variablename = replace(variablename, 
                                stringr::str_detect(variablename, "Plantago.*"), 
                                "Plantago"))
```

We can use a similar pattern to work from a table.  For example, a table of pairs (what we want changed, and the name we want it replaced with) can be generated:

| original | replacement |
| -------- | ----------- |
| Abies.*  | Abies |
| Vaccinium.* | Ericaceae |
| Typha.* | Aquatic |

We can get the list of original names directly from the `taxa()` call, or we can create one from `allSamp` in a way that gives us a sense of the number of times those taxa appear, either across sites, or samples:


```countbysitessamples
taxaSites <- allSamp %>%
  group_by(variablename, siteid) %>% 
  group_by(variablename) %>% 
  summarise(sites = length(unique(siteid)))

taxaSamples <- allSamp %>%
  group_by(variablename) %>% 
  summarise(samples = n())

taxaplots <- taxaSites %>% inner_join(taxaSamples, by = "variablename")

ggplot(data = taxaplots, aes(x = sites, y = samples)) +
  geom_point() +
  stat_smooth(method = 'glm', 
              method.args = list(family = 'poisson')) +
  xlab("Number of Sites") +
  ylab("Number of Samples") +
  theme_bw()
```

This is mostly for illustration, but we can see, as a sanity check, that the relationship is as we'd expect.

You can then export either one of these tables and add a column with the counts, you could also add extra contextual information, such as the `ecologicalgroup` or `taxongroup` to help you out. Once you've cleaned up the translation table you can load it in, and then apply the transformation:


```r
translation <- readr::read_csv("data/taxontable.csv")
```

```
## Rows: 1195 Columns: 4
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: ","
## chr (2): variablename, replacement
## dbl (2): sites, samples
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.
```

```r
DT::datatable(translation, rownames = FALSE, 
                options = list(scrollX = "100%"))
```

```{=html}
<div id="htmlwidget-320a099547cb6a2ceb75" style="width:100%;height:auto;" class="datatables html-widget"></div>
<script type="application/json" data-for="htmlwidget-320a099547cb6a2ceb75">{"x":{"filter":"none","vertical":false,"data":[["?Microthyrium (type 8, HdV)","Abies","Abies alba","Acacia","Acer","Achillea","Achillea-type","Aconitum","Aconitum-type","Acrotritia ardua (type 106, HdV)","Actaea spicata","Actinopeltis undiff.","Adenostyles alliariae","Adiantum capillus-veneris","Adonis aestivalis-type","Adonis aestivalis/A. flammea","Adonis annua-type","Aesculus","Aethusa cynapium","Agrimonia","Agrostemma githago","Ailanthus","Alchemilla","Alchemilla-type","Algae","Algae (type 209, HdV)","Algae? (type 225, HdV)","Alisma","Alisma plantago-aquatica","Allium","Allium-type","Alnus","Alnus glutinosa","Alnus glutinosa-type","Alnus glutinosa/A. incana","Alnus undiff.","Alnus viridis","Alnus viridis-type","Alona rustica (type 72, HdV)","Amaranthaceae","Ambrosia","Ambrosia-type","Amphitrema","Amphitrema sp.","Amphitrema undiff.","Amphitrema wrightianum","Anagallis","Anagallis arvensis","Anagallis arvensis-type","Anagallis tenella-type","Anagallis-type","Anchusa/Pulmonaria","Andromeda","Andromeda polifolia","Androsace elongata-type","Androsace-type","Anemone","Anemone hepatica","Anemone nemorosa group","Anemone nemorosa-type","Anemone sect. Hepatica-type","Anemone sect. Pulsatilla","Anemone-type","Anthemis arvensis-type","Anthemis-type","Anthericum","Anthericum-type","Anthoceros","Anthoceros laevis","Anthoceros laevis-type","Anthoceros punctatus-type","Anthostomella cf. A. fuegiana (type 4, HdV)","Anthostomella fuegiana","Anthriscus caucalis","Anthriscus sylvestris","Anthriscus sylvestris-type","Anthriscus-type","Anthyllis","Aphanes","Apiaceae","Apiaceae undiff.","Apium","Apium inundatum-type","Apium-type","Araneae","Arcella","Arcella cf. A. megastoma","Arcella discoides","Arcella sp.","Archerella flavum","Archerella flavum (type 31A, HdV)","Arctium","Arctostaphylos uva-ursi","Arctostaphylos uva-ursi-type","Arctous alpina","Arenaria","Arenaria serpyllifolia-type","Armeria","Armeria maritima (type A)","Armeria maritima (type B)","Armeria maritima-type","Arnica montana","Artemisia","Arthrodesmus","Aruncus","Ascaris","Ascaris lumbricoides","Ascomycota","Ascospora","Aspiromitus punctatus","Asplenium","Asplenium-type","Assulina","Assulina muscorum","Assulina seminulum","Assulina sp.","Aster-type","Asteraceae","Asteroideae","Asteroideae undiff.","Asterosporium","Astragalus alpinus-type","Astragalus danicus-type","Astragalus exscapus","Astragalus-type","Astrantia major","Astrantia-type","Athyrium","Athyrium filix-femina","Athyrium-type","Avena-type","Avena/Triticum-type","Bambusa-type","Barbarea-type","Betula","Betula nana","Betula nana-type","Betula pubescens","Betula pubescens-type","Bidens","Bidens-type","Bistorta","Bistorta officinalis","Bistorta officinalis-type","Blechnum","Boraginaceae","Botrychium","Botryococcus","Botryococcus braunii","Botryococcus neglectus","Botryococcus pila","Botryococcus sp.","Brachysporium obovatum/B. bloxami/Bactrodesmium betulicola (type 359, HdV)","Brachysporium pendulisporum (type 360, HdV)","Brassica","Brassicaceae","Brassicaceae undiff.","Brassicaceae-type","Bruckenthalia-type","Bryales","Bryonia alba/Helianthemum","Bryophyta","Bryophyta (type 354, HdV)","Bupleurum","Bupleurum falcatum-type","Bupleurum-type","Butomus","Butomus umbellatus","Butomus-type","Byssothecium circinans","Byssothecium circinans (type 16A, HdV)","Byssothecium circinans (type 16B, HdV)","Byssothecium circinans (type 16C, HdV)","Calla","Callitriche","Calluna","Calluna vulgaris","Caltha","Caltha palustris-type","Caltha-type","Calystegia","Campanula","Campanula rapunculoides","Campanula-type","Campanula/Phyteuma","Campanulaceae","Cannabis sativa","Cannabis-type","Cardamine","Cardamine pratensis-type","Cardamine-type","Carduus","Carex","Carpinus","Carpinus betulus","Carpinus-type","Carum carvi","Carya","Caryophyllaceae","Caryophyllaceae undiff.","Caryospora sp. (type 1001, HdV)","Castanea","Castanea sativa","Castanea-type","Cedrus","Cedrus-type","Centaurea","Centaurea alpina-type","Centaurea cyanus","Centaurea cyanus-type","Centaurea jacea","Centaurea jacea-type","Centaurea jacea/C. stoebe","Centaurea montana-type","Centaurea nigra-type","Centaurea scabiosa","Centaurea scabiosa-type","Centaurea sp.","Centaurea stoebe","Centaurea stoebe-type","Centaurea undiff.","Centropyxis","Centropyxis aculeata","Cerastium","Cerastium arvense-type","Cerastium cerastoides-type","Cerastium fontanum-type","Cerastium-type","Ceratophyllum","Cercophora sp. (type 112, HdV)","Cerinthe","Cerinthe minor","cf. Actinopeltis (type 8C, HdV)","cf. Androsace","cf. Diphasiastrum-type","cf. Endophragmia (type 572, HdV)","cf. Entophlyctis lobata","cf. Helianthemum","cf. Lamiaceae","cf. Lobelia","cf. Papaver","cf. Pedicularis","cf. Penium","cf. Persiciospora (type 124, HdV)","cf. Riccia","cf. Richonia variospora (type 140, HdV)","cf. Spadicoides bina (type 98, HdV)","cf. Trichocladium opacum (type 10, HdV)","cf. Trichoglossum hirsutum (type 77B, HdV)","Chaerophyllum","Chaerophyllum hirsutum","Chaerophyllum hirsutum-type","Chaetomium sp. (type 7A, HdV)","Chaetomium undiff.","Chamaenerion","Chamaenerion angustifolium","Chamaenerion angustifolium-type","Charcoal","Chelidonium","Chelidonium majus","Chironomidae","Chlamydomonadaceae","Chrysophyceae","Chrysosplenium","Chrysosplenium-type","Cicatricosporites australiensis","Cichorioideae","Cicuta virosa","Circaea","Cirsium","Cirsium-type","Cirsium/Carduus","Cladium mariscus","Cladocera","Clasterosporium caricinum (type 126, HdV)","Clematis","Closterium","Closterium undiff.","Coelastrum reticulatum","Comarum","Comarum palustre","Comarum-type","Coniferae-type","Coniochaeta cf. C. ligniaria","Coniochaeta ligniaria (type 172, HdV)","Coniochaeta xylariispora","Consolida regalis","Consolida-type","Convallaria","Convolvulus","Convolvulus arvensis","Convolvulus-type","Copepoda","Cornus","Cornus mas","Cornus sanguinea","Cornus suecica","Coronilla undiff.","Corylus","Corylus avellana","Crataegus","Crataegus-type","Crupina","Cryptomeria","Cupressaceae","Cuscuta","Cuscuta epithymum","Cuscuta europaea-type","Cyperaceae","Cystopteris","Daphne","Daucus","Daucus carota","Daucus-type","Delitschia","Descurainia sophia","Desmidiaceae","Desmidiales","Dianthus","Dianthus-type","Difflugia","Digitalis purpurea-type","Dinoflagellata","Dinoflagellata undiff.","Diphasiastrum alpinum","Diphasiastrum alpinum-type","Diphasiastrum complanatum","Diporotheca","Dipsacoideae","Dipsacus","Drosera","Drosera intermedia","Drosera rotundifolia","Drosera rotundifolia-type","Dryas octopetala","Dryas-type","Dryopteris","Dryopteris carthusiana-type","Dryopteris dilatata-type","Dryopteris filix-mas-type","Dryopteris-type","Echinops","Echium","Echium vulgare","Echium-type","Elatine","Elymus-type","Empetrum","Empetrum nigrum-type","Empetrum-type","Encalypta-type","Engelhardia-type","Entophlyctis lobata","Entorrhiza sp. (type HdV-527)","Ephedra","Ephedra cf. E. distachya","Ephedra cf. E. foeminea","Ephedra cf. E. fragilis","Ephedra distachya","Ephedra distachya-type","Ephedra foeminea","Ephedra foeminea-type","Ephedra fragilis","Ephedra fragilis-type","Epilobium","Epilobium-type","Epipactis","Equisetaceae","Equisetum","Eranthis hyemalis-type","Erica ciliaris-type","Ericaceae","Ericaceae undiff.","Ericales","Ericales undiff.","Eriophorum","Erodium","Eryngium","Eryngium-type","Euglypha","Eunotia sp.","Euonymus","Euonymus europaeus","Euphorbia","Euphorbia-type","Euphorbiaceae","Euphrasia","Euphrasia-type","Fabaceae","Fabaceae undiff.","Faboideae","Fagopyrum","Fagopyrum esculentum","Fagopyrum-type","Fagus","Fagus sylvatica","Falcaria vulgaris","Falcaria-type","Fallopia convolvulus","Fallopia convolvulus-type","Fallopia convolvulus/F. dumetorum","Filinia longiseta","Filipendula","Filipendula ulmaria/F. vulgaris","Frangula","Frangula alnus","Fraxinus","Fraxinus excelsior","Fraxinus ornus","Fumaria-type","Fungi","Fungi (type 11, HdV)","Fungi (type 123, HdV)","Fungi (type 17, HdV)","Fungi (type 173, HdV)","Fungi (type 18, HdV)","Fungi (type 19, HdV)","Fungi (type 20, HdV)","Fungi (type 200, HdV)","Fungi (type 23, HdV)","Fungi (type 24, HdV)","Fungi (type 3A, HdV)","Fungi (type 404, HdV)","Fungi (type 408, HdV)","Fungi (type 47, HdV)","Fungi (type 53, HdV)","Fungi (type 54B, HdV)","Fungi (type 571, HdV)","Fungi (type 64, HdV)","Fungi (type 65, HdV)","Fungi (type 729, HdV)","Fungi (type 73, HdV)","Fungi (type 778. HdV)","Fungi (type 83, HdV)","Fungi (type 8A, HdV)","Fungi (type 8D, HdV)","Fungi (type 8E, HdV)","Fungi (type 90, HdV)","Fungi (type 96B, HdV)","Fungi undiff.","Fungi-type","Gaeumannomyces","Gaeumannomyces cf. G. caricis","Gaeumannomyces undiff.","Galeopsis","Galeopsis-type","Galeopsis/Ballota-type","Galium","Galium-type","Gelasinospora","Gelasinospora (type 1, HdV)","Gelasinospora (type 1A, HdV)","Gelasinospora (type 1B, HdV)","Gelasinospora reticulispora","Gelasinospora undiff.","Genista","Genista-type","Gentiana","Gentiana cruciata-type","Gentiana pneumonanthe-type","Gentiana undiff.","Gentiana-type","Gentianaceae","Gentianaceae undiff.","Gentianella","Gentianella campestris-type","Geoglossum sphagnophilum","Geoglossum sphagnophilum (type 77A, HdV)","Geoglossum sphagnophilum/Trichoglossum hirsutum (type 77A/77B, HdV)","Geraniaceae","Geranium","Geranium-type","Geum","Geum-type","Glaucium","Glaucium corniculatum","Glomus","Glomus cf. G. fasciculatum","Glyceria-type","Glyptotendipes pallens group (type 509, HdV)","Gnaphalium-type","Gratiola officinalis","Gymnocarpium dryopteris","Gypsophila","Gypsophila repens","Gypsophila repens-type","Gypsophila-type","Gyratrix hermaphroditus","Habrotrocha","Habrotrocha angusticollis","Habrotrocha angusticollis (type 37, HdV)","Hedera","Hedera helix","Hedysarum-type","Helianthemum","Helianthemum nummularium-type","Helianthemum oelandicum subsp. alpestris-type","Helicoma","Helicoön pluriseptatum","Helicosporium/Helicoön pluriseptatum (type 30, HdV)","Heliotropium europaeum","Helleborus","Heracleum","Heracleum sphondylium","Heracleum-type","Herniaria","Herniaria-type","Hippophaë","Hippophaë rhamnoides","Hippuris vulgaris","Hordeum","Hordeum-type","Hottonia palustris","Humulus","Humulus lupulus","Humulus/Cannabis","Humulus/Cannabis-type","Huperzia selago","Hyalosphenia papilio","Hyalosphenia subflava","Hydrocotyle","Hydrocotyle vulgaris","Hydrocotyle-type","Hydrodictyon","Hymenophyllum","Hyoscyamus","Hypericum","Hypericum perforatum-type","Hypericum perforatum/H. androsaemum-type","Hystrix","Ilex","Illecebrum verticillatum","Impatiens","Impatiens noli-tangere","Impatiens parviflora","Indeterminable","Indeterminable undiff.","Iris","Iris pseudacorus","Iris pseudacorus-type","Iris-type","Isoëtes","Isoëtes lacustris","Jasione","Jasione montana","Jasione-type","Juglandaceae","Juglandaceae-type","Juglans","Juncus sp. (type 2.2, MI)","Juniperus","Juniperus communis","Juniperus-type","Knautia","Knautia arvensis","Knautia arvensis-type","Koenigia alpina","Lamiaceae","Lamiaceae (tricolpate)","Lamium album-type","Lamium-type","Larix","Larix decidua","Larix-type","Larix/Pseudotsuga","Laserpitium","Laserpitium latifolium-type","Lasiosphaeria cf. L. caudata","Lasiosphaeria sp.","Lasiosphaeria sp. (type 63C, HdV)","Lasiosphaeria-type","Lathyrus-type","Lemna","Lemnoideae","Lepidoptera","Ligustrum","Liliaceae","Liliaceae-type","Linum","Linum austriacum-type","Linum catharticum","Linum catharticum-type","Linum flavum","Linum usitatissimum-type","Liquidambar","Listera ovata","Listera-type","Lithospermum officinale","Lobelia","Lonicera","Lonicera periclymenum","Lonicera xylosteum","Loranthus","Loranthus europaeus","Lotus","Lotus cf. L. corniculatus","Lotus pedunculatus","Lotus-type","Ludwigia palustris","Lycopodiaceae","Lycopodiaceae cf. Diphasiastrum complanatum","Lycopodiaceae undiff.","Lycopodiella inundata","Lycopodium","Lycopodium annotinum","Lycopodium annotinum-type","Lycopodium clavatum","Lycopodium clavatum-type","Lycopodium spike","Lycopodium tablets","Lycopodium undiff.","Lycopus","Lycopus-type","Lysimachia","Lysimachia cf. L. vulgaris","Lysimachia nemorum","Lysimachia nemorum-type","Lysimachia thyrsiflora","Lysimachia vulgaris","Lysimachia vulgaris-type","Lysimachia-type","Lythrum","Lythrum portula-type","Lythrum salicaria","Lythrum-type","Macrobiotus","Macrobiotus cf. M. echinogenitus","Magnolia-type","Maianthemum bifolium","Malus","Malus undiff.","Malus-type","Malva-type","Malvaceae","Marchantiophyta","Marrubium","Matricaria-type","Medicago falcata-type","Melampyrum","Melampyrum-type","Meliola","Mentha","Mentha-type","Menyanthes","Menyanthes trifoliata","Menyanthes trifoliata-type","Mercurialis","Mercurialis perennis-type","Micranthes stellaris","Micranthes stellaris-type","Microdalyellia armigera","Microrrhinum minus","Microthyriaceae","Microthyrium","Microthyrium microscopicum","Microthyrium undiff.","Microthyrium-type","Minuartia-type","Monactinus simplex","Monactinus simplex var. echinulatum","Monactinus simplex var. simplex","Monocotyledoneae","Montia","Mougeotia","Mougeotia cf. M. gracillima (type 61, HdV)","Mougeotia undiff.","Mougeotia-type","Muscari","Myosotis","Myosotis arvensis-type","Myrica-type","Myricaria germanica","Myriophyllum","Myriophyllum alterniflorum","Myriophyllum cf. M. spicatum","Myriophyllum cf. M. verticillatum","Myriophyllum heterophyllum","Myriophyllum spicatum","Myriophyllum spicatum-type","Myriophyllum undiff.","Myriophyllum verticillatum","Myriophyllum verticillatum-type","Nebela","Nebela parvula","Nebela undiff.","Neurospora","Normapolles","Nuphar","Nymphaea","Nymphaea alba","Nymphaea alba-type","Nymphaea cf. N. candida","Nymphaea undiff.","Nymphaeaceae","Nymphoides peltata-type","Nyssa","Odontites","Odontites-type","Oenanthe","Oenanthe-type","Oenothera","Olea","Oleaceae","Onagraceae","Onobrychis","Ononis","Ononis-type","Onopordum","Ophioglossum","Ophioglossum vulgatum","Orchidaceae","Oribatida (type 396, HdV)","Orlaya grandiflora","Ornithogalum umbellatum-type","Ornithogalum-type","Ostrya-type","Oxalis","Oxyria-type","Papaver","Papaver rhoeas-type","Papaver somniferum","Papaveraceae","Parapediastrum biradiatum","Parnassia","Parnassia palustris","Parnassia-type","Pediastrum","Pediastrum angulosum","Pediastrum angulosum var. angulosum","Pediastrum angulosum var. asperum","Pediastrum braunii","Pediastrum duplex","Pediastrum duplex var. duplex","Pediastrum duplex var. rugulosum","Pediastrum muticum var. scutum","Pediastrum orientale","Pediastrum undiff.","Pedicularis","Pedicularis palustris-type","Penium","Persicaria","Persicaria amphibia","Persicaria amphibia-type","Persicaria cf. P. lapathifolia","Persicaria lapathifolia","Persicaria maculosa","Persicaria maculosa-type","Petasites","Petasites hybridus-type","Petasites-type","Peucedanum","Peucedanum palustre-type","Peucedanum-type","Phegopteris","Phegopteris connectilis","Phragmites","Phragmites australis","Phragmites-type","Phyteuma","Phyteuma-type","Picea","Picea abies","Pilularia","Pimpinella anisum","Pimpinella major","Pimpinella major-type","Pimpinella major/P. saxifraga","Pinguicula","Pinnularia","Pinus","Pinus (Tertiary)","Pinus cembra","Pinus cembra-type","Pinus sp.","Pinus subg. Pinus","Pinus subg. Strobus-type","Pinus sylvestris","Pinus sylvestris-type","Pisum sativum","Plantaginaceae","Plantaginaceae undiff.","Plantago","Plantago alpina","Plantago alpina-type","Plantago atrata-type","Plantago coronopus","Plantago lanceolata","Plantago lanceolata-type","Plantago major","Plantago major-type","Plantago major/P. media","Plantago major/P. media-type","Plantago maritima-type","Plantago media","Plantago media-type","Plantago sp.","Platanus","Platycarya","Platyhelminthes (type 353A, HdV)","Platyhelminthes (type 353B, HdV)","Pleospora sp. (type 3B, HdV)","Pleospora undiff.","Pleurospermum","Pleurospermum austriacum","Pleurospermum austriacum-type","Poaceae","Poaceae (Cerealia-type excluding Secale)","Poaceae (Cerealia-type)","Poaceae (Cerealia)","Poaceae (Cerealia) excluding Secale","Poaceae (Cerealia) undiff.","Podocarpus-type","Podospora sp./Zopfiella sp. (type 466, HdV)","Podospora-type (type 368, HdV)","Polemonium","Polemonium caeruleum","Polygala","Polygonaceae","Polygonaceae undiff.","Polygonatum","Polygonum","Polygonum aviculare","Polygonum aviculare-type","Polygonum sp.","Polypodiaceae","Polypodiophyta (monolete, psilate)","Polypodiophyta (monolete, verrucate)","Polypodium","Polypodium vulgare","Populus","Porifera","Potamogeton","Potamogeton natans-type","Potamogeton-type","Potentilla","Potentilla-type","Potentilla/Comarum","Potentilla/Comarum-type","Potentilla/Fragaria","Poterium sanguisorba","Poterium sanguisorba subsp. sanguisorba","Primula","Primula clusiana-type","Primula farinosa-type","Primula veris-type","Primulaceae","Prunella-type","Prunus","Prunus sp.","Prunus-type","Pseudopediastrum boryanum","Pseudopediastrum boryanum var. boryanum","Pseudopediastrum boryanum var. boryanum sensu lato","Pseudopediastrum boryanum var. cornutum","Pseudopediastrum boryanum var. longicorne","Pseudopediastrum brevicorne","Pseudopediastrum integrum","Pseudopediastrum kawraiskyi","Pteridium","Pteridium aquilinum","Pteridophyta","Pteridophyta (monolete, verrucate)","Pteridophyta (monolete) undiff.","Pterocarya","Pterocarya fraxinifolia","Pulmonaria","Pulmonaria-type","Pyrola","Pyxidicula","Quercus","Quercus coccifera","Quercus pubescens","Ranunculaceae","Ranunculaceae undiff.","Ranunculus","Ranunculus acris-type","Ranunculus acris/R. flammula/R. sceleratus group","Ranunculus aquatilis-type","Ranunculus arvensis","Ranunculus arvensis-type","Ranunculus flammula-type","Ranunculus sect. Batrachium","Ranunculus sect. Batrachium-type","Ranunculus-type","Reseda","Reseda lutea-type","Rhabdocoela","Rhabdocoela (type 353, HdV)","Rhamnaceae","Rhamnus","Rhinanthus","Rhinanthus-type","Rhinanthus/Veronica","Rhizopoda","Rhizopoda undiff.","Rhododendron","Rhododendron subsect. Ledum","Rhododendron tomentosum","Rhus-type","Ribes","Ribes alpinum","Ribes uva-crispa","Riccia","Rivularia","Rosa","Rosa-type","Rosaceae","Rosaceae undiff.","Rotifera undiff.","Rotifera/Tardigrada","Rubiaceae","Rubus","Rubus chamaemorus","Rumex","Rumex acetosa","Rumex acetosa-type","Rumex acetosa/R. acetosella","Rumex acetosella","Rumex acetosella-type","Rumex aquaticus-type","Rumex cf. R. alpinus","Rumex maritimus-type","Rumex obtusifolius","Rumex obtusifolius-type","Rumex subg. Rumex","Rumex undiff.","Rumex-type","Rumex/Oxyria-type","Sagina","Sagina procumbens-type","Sagina-type","Sagittaria","Sagittaria-type","Salix","Salix herbacea-type","Salvia","Sambucus","Sambucus cf. S. ebulus","Sambucus cf. S. nigra","Sambucus cf. S. racemosa","Sambucus ebulus","Sambucus nigra","Sambucus nigra-type","Sambucus nigra/S. racemosa","Sambucus racemosa","Samolus valerandi","Sample quantity","Sanguisorba minor-type","Sanguisorba officinalis","Sanicula europaea","Sanicula-type","Saussurea-type","Saxifraga","Saxifraga aizoides-type","Saxifraga granulata","Saxifraga granulata-type","Saxifraga hirculus-type","Saxifraga oppositifolia","Saxifraga oppositifolia-type","Saxifraga undiff.","Saxifragaceae","Saxifragaceae undiff.","Scabiosa","Scabiosa columbaria subsp. pratensis-type","Scabiosa columbaria-type","Scandix pecten-veneris/Caucalis platycarpos","Scenedesmus","Scenedesmus undiff.","Scheuchzeria","Scheuchzeria palustris","Sciadopitys","Sciadopitys-type","Scilla-type","Scleranthus","Scleranthus annuus","Scleranthus annuus-type","Scleranthus cf. S. annuus","Scleranthus cf. S. perennis","Scleranthus perennis","Scleranthus perennis-type","Scleranthus-type","Scrophularia","Scrophularia-type","Scrophulariaceae","Scutellaria","Secale","Secale cereale","Secale-type","Securigera varia","Sedum","Sedum-type","Selaginella","Selaginella selaginoides","Selaginellaceae","Senecio","Senecio-type","Senecio/Aster","Sequoia","Serratula","Serratula-type","Seseli-type","Sigmopollis","Silene","Silene dioica-type","Silene flos-cuculi","Silene latifolia","Silene viscaria-type","Silene vulgaris","Silene vulgaris-type","Silene-type","Silene-type undiff.","Sileneae","Silenoideae-type","Sinapis-type","Sium latifolium-type","Solanaceae","Solanum","Solanum cf. S. nigrum","Solanum dulcamara","Solanum nigrum","Solanum nigrum-type","Soldanella","Sorbus","Sorbus aria-type","Sorbus aucuparia","Sorbus group","Sorbus torminalis","Sorbus-type","Sordaria-type (type 55A, HdV)","Sordariaceae","Sordariaceae/Sordaria (type 55B, HdV)","Sparganium","Sparganium erectum","Sparganium-type","Spergula","Spergula-type","Spergularia-type","Sphagnum","Spirogyra","Spirogyra cf. S. scrobiculata","Spirogyra-type","Sporormiella","Sporormiella (type 113, HdV)","Stachys","Stachys-type","Staurastrum","Staurastrum undiff.","Stauridium tetras","Stellaria","Stellaria holostea","Stellaria-type","Stratiotes aloides","Succisa","Succisa pratensis","Succisa-type","Succisella","Swertia perennis","Symphytum","Symphytum cf. S. officinale","Symphytum-type","Symplocos","Tardigrada","Tardigrada (type 902, HdV)","Tardigrada undiff.","Taxus","Taxus baccata","Tetraëdron","Tetraëdron minimum","Tetraploa scheueri","Teucrium","Thalictrum","Thalictrum-type","Thecaphora","Thelypteris","Thelypteris palustris","Thesium","Thesium-type","Tilia","Tilia cordata","Tilia platyphyllos","Tilia undiff.","Tilletia sphagni","Tilletia sphagni (type 27, HdV)","Tofieldia","Transeauina","Transeauina (type 214, HdV)","Transeauina undiff.","Trapa","Trapa natans","Trichocladium opacum (type 10, HdV)","Trichuris trichiura","Trientalis","Trientalis europaea","Trifolium","Trifolium pratense","Trifolium pratense-type","Trifolium repens-type","Trifolium-type","Triglochin","Triticum","Triticum-type","Trochiscia undiff.","Trollius","Trollius europaeus","Trollius-type","Tsuga","Tsuga diversifolia-type","Tsuga-type","Turgenia latifolia","Typha","Typha angustifolia","Typha angustifolia-type","Typha angustifolia/Sparganium","Typha angustifolia/Sparganium-type","Typha latifolia","Typha latifolia-type","Typhaceae","Ulex-type","Ulmus","Ulmus/Zelkova","Umbilicus rupestris-type","Unknown","Unknown (Cretaceous)","Unknown (monolete, psilate)","Unknown (monolete)","Unknown (monolete) undiff.","Unknown (pre-Quaternary)","Unknown (Tertiary)","Unknown (trilete)","Unknown (trilete) undiff.","Unknown (type 160, HdV)","Unknown (type 181, HdV)","Unknown (type 224, HdV)","Unknown (type 33, HdV)","Unknown (type 366, HdV)","Unknown (type 38, HdV)","Unknown (type 41, HdV)","Unknown (type 708, HdV)","Unknown (type 74, HdV)","Unknown (type 86, HdV)","Unknown (type 91, HdV)","Urtica","Ustulina deusta","Ustulina deusta (type 44, HdV)","Utricularia","Vaccinioideae","Vaccinium","Vaccinium oxycoccos","Vaccinium-type","Valeriana","Valeriana cf. V. dioica","Valeriana cf. V. officinalis","Valeriana dioica","Valeriana dioica-type","Valeriana officinalis","Valeriana officinalis-type","Valerianella","Varia","Veratrum","Veratrum album","Veratrum-type","Verbascum","Vermes","Veronica","Veronica beccabunga","Veronica beccabunga-type","Veronica-type","Viburnum","Viburnum cf. V. opulus","Viburnum lantana","Viburnum opulus","Viburnum opulus-type","Viburnum undiff.","Vicia","Vicia cracca-type","Vicia-type","Vicia/Lathyrus","Vicia/Lathyrus-type","Viola","Viola canina-type","Viola palustris","Viola palustris-type","Viola tricolor","Viscum","Vitis","Vitis vinifera","Xanthium","Xanthium-type","Xylariaceae","Xylomyces chlamydosporis/X. aquaticus (type 201, HdV)","Zea mays","Zygnema","Zygnema-type","Zygnemataceae","Zygnemataceae undiff."],[1,72,23,1,78,1,15,11,1,1,2,2,1,2,1,1,1,2,1,1,10,2,5,4,7,1,1,13,1,3,4,79,1,5,1,1,8,1,1,85,5,5,5,1,1,5,2,1,2,1,1,1,1,1,2,1,1,1,1,15,1,5,19,1,13,3,1,5,5,1,1,1,1,1,1,4,1,2,1,70,15,1,1,1,1,14,1,2,2,20,1,1,1,1,1,6,1,4,2,2,1,1,87,1,1,3,1,4,1,13,1,1,6,7,3,2,9,13,65,2,1,1,3,1,6,1,4,2,6,1,23,1,1,17,83,8,1,1,2,12,3,3,23,25,2,12,44,22,11,3,3,3,1,1,1,65,1,6,1,40,1,10,2,5,1,5,6,1,1,3,1,1,3,1,2,27,57,4,1,45,7,45,1,7,1,5,7,11,2,13,19,12,1,65,26,1,1,4,28,1,1,4,1,3,1,1,11,1,62,7,11,31,1,1,2,22,7,1,1,2,7,1,1,3,6,6,1,5,6,1,2,1,1,1,1,2,3,1,1,1,1,2,1,1,1,2,1,1,1,1,2,6,1,1,6,12,1,9,3,1,2,1,1,20,4,1,81,2,9,12,41,4,2,1,1,1,2,1,5,18,2,4,1,2,1,2,1,6,1,7,9,1,8,8,10,17,1,1,68,26,1,2,1,1,2,8,1,2,84,1,3,1,1,11,1,1,3,1,3,4,1,1,4,1,1,1,1,8,3,3,15,1,6,1,3,1,2,1,2,4,20,1,18,2,2,7,1,3,1,6,1,2,8,1,6,1,3,1,10,9,1,3,4,10,42,3,1,1,74,1,1,21,7,3,1,1,4,2,1,8,1,12,1,13,1,1,2,2,51,1,3,23,1,4,75,11,1,1,3,3,1,1,81,1,27,38,68,14,1,1,6,1,1,2,1,3,2,2,2,2,1,4,1,1,1,1,1,2,1,1,1,2,1,3,3,1,3,4,1,3,1,4,6,1,1,2,1,14,31,6,1,1,1,1,1,1,7,10,1,5,1,3,6,1,2,4,2,2,1,1,32,1,11,10,2,1,2,1,5,1,11,1,8,5,1,4,4,6,1,14,1,24,26,1,42,1,1,3,8,1,1,1,6,2,6,1,2,7,10,2,1,7,1,7,5,32,6,7,1,5,1,1,1,1,1,1,27,3,1,2,2,1,10,1,1,1,1,2,1,2,3,1,1,5,8,1,2,1,57,1,56,1,1,21,2,3,1,40,1,1,21,30,2,1,1,1,1,1,1,1,1,11,4,1,1,5,25,2,3,2,12,1,1,3,1,1,2,1,2,13,2,3,8,2,1,1,4,32,1,12,1,2,4,4,56,5,44,3,8,5,1,9,11,30,1,1,1,1,7,19,9,19,3,9,1,1,1,1,1,2,1,1,1,6,3,1,1,1,54,1,1,7,24,7,33,2,8,2,2,5,6,1,7,32,4,2,1,4,4,2,2,8,4,12,1,2,1,1,3,1,1,1,2,17,1,1,3,24,1,2,16,6,5,1,1,1,1,15,14,2,1,1,1,5,1,2,2,3,4,1,1,2,1,1,4,1,1,1,13,2,2,1,1,1,1,1,9,2,6,2,1,10,1,4,16,1,9,5,3,2,1,10,1,8,2,1,3,19,1,1,1,8,2,1,1,13,27,3,1,13,1,1,12,2,1,2,2,5,6,1,74,23,1,1,1,12,1,3,1,77,1,6,3,1,3,1,3,2,4,3,1,3,6,3,1,2,68,18,20,13,48,8,3,17,3,2,1,1,1,2,1,1,2,13,1,86,2,16,14,1,20,1,1,1,9,5,6,3,1,2,1,56,8,3,42,1,1,15,30,44,1,36,1,12,9,27,9,11,1,12,2,4,1,1,4,2,1,6,1,13,13,5,5,8,9,3,11,6,22,46,3,2,5,3,1,1,6,1,1,85,2,1,57,4,1,20,1,1,4,1,1,13,4,31,1,1,3,1,1,11,9,2,1,4,5,1,3,4,1,16,1,1,3,1,6,1,65,3,1,1,56,12,1,36,10,41,1,20,9,1,1,1,3,4,2,4,2,1,3,1,2,4,1,86,2,1,13,1,6,4,5,18,10,1,12,1,6,1,50,1,1,1,3,2,1,3,1,1,4,2,13,1,9,1,1,1,11,2,6,6,2,1,1,4,3,2,1,1,1,1,5,3,3,14,1,35,32,16,1,13,2,3,19,1,1,12,1,1,1,1,1,1,3,8,8,1,1,1,8,23,1,28,12,1,1,1,1,1,9,1,6,1,24,1,2,1,2,8,3,1,1,12,2,8,2,1,1,77,7,1,3,2,1,5,4,6,1,2,2,3,2,3,15,8,1,1,2,21,1,2,1,2,1,1,1,2,7,9,1,3,75,3,4,1,7,4,1,79,9,10,4,12,1,1,1,1,3,1,3,2,2,1,2,6,6,22,11,22,3,27,27,1,5,11,1,4,1,1,1,1,8,1,33,10,41,13,1,1,85,1,1,24,1,2,14,5,2,5,1,7,1,1,1,2,1,2,2,1,1,2,2,71,5,1,17,11,22,5,37,30,1,3,1,8,20,8,2,57,5,1,2,9,1,9,1,2,4,10,1,2,5,1,1,1,1,24,1,1,13,1,5,5,1,47,9,2,3,1,1,3,5,3,5,3,1],[15,1822,836,1,1157,1,216,76,3,1,3,7,1,2,3,5,5,3,1,1,29,10,23,56,18,1,1,51,8,46,6,3219,20,172,94,130,190,5,1,1977,6,26,119,2,1,18,3,2,7,1,1,1,1,3,11,1,2,1,1,108,1,13,89,16,331,6,4,17,12,1,1,5,1,1,8,24,1,4,1,1767,331,1,1,30,1,79,2,6,5,190,26,1,5,1,1,8,1,11,3,3,1,2,3419,4,1,13,2,53,7,38,1,1,30,189,51,31,152,134,1119,43,1,1,8,2,15,1,8,36,20,1,325,2,2,186,3727,177,42,29,80,83,3,10,121,139,2,37,231,475,148,7,18,21,2,1,30,1163,22,153,5,614,27,203,4,11,2,8,37,1,2,20,3,1,38,3,11,356,895,25,1,365,14,239,3,19,3,10,95,180,46,163,151,44,7,1148,620,130,1,37,288,30,1,18,1,15,2,1,35,3,432,35,57,136,14,1,2,74,14,2,3,7,13,2,1,12,18,27,7,29,130,12,3,1,12,2,1,18,47,1,19,1,10,10,2,1,2,15,1,49,98,1,5,45,7,1,42,28,2,867,6,2,4,2,1,88,24,1,1869,11,13,52,416,49,7,5,37,3,3,2,50,337,20,36,8,103,9,56,2,32,1,22,46,1,64,9,72,50,1,1,2273,1107,1,7,1,3,11,64,10,31,3751,2,8,1,1,320,1,1,48,1,3,25,2,1,88,9,30,7,2,68,4,5,20,5,16,4,15,1,67,2,2,8,744,2,40,3,2,13,2,5,89,50,1,19,59,3,21,7,7,1,35,30,1,6,25,37,150,9,1,1,1686,1,2,202,44,207,118,1,5,31,1,66,1,28,5,18,5,3,2,3,350,5,5,46,3,12,2328,438,1,3,3,3,9,22,2359,18,188,327,1293,484,1,2,116,3,2,11,1,47,4,4,30,5,52,80,1,4,1,34,5,2,52,1,7,11,1,79,46,21,51,141,30,60,8,55,131,12,2,2,4,218,572,19,13,10,2,5,1,2,19,39,1,8,9,3,15,1,2,10,16,21,3,2,79,9,23,31,7,1,13,19,21,2,82,1,47,16,1,26,40,66,8,251,34,49,128,2,308,3,1,17,196,20,9,1,40,6,25,1,2,20,43,3,3,74,5,16,80,497,40,19,2,12,1,5,1,1,3,6,143,50,23,9,15,1,17,5,12,26,51,3,1,3,11,1,2,8,11,2,13,6,194,1,862,4,12,33,5,6,2,422,17,2,85,99,3,10,1,2,1,63,2,2,8,14,17,12,1,27,85,4,6,2,22,2,2,6,3,1,12,2,2,29,2,3,18,2,6,1,18,162,3,95,2,3,6,8,370,23,178,11,679,525,1,27,66,152,1,3,1,1,21,142,62,87,5,48,20,2,2,3,4,2,1,3,2,14,8,1,20,3,445,2,2,39,253,32,264,3,14,3,15,14,140,1,78,246,18,32,7,7,29,32,47,41,6,188,4,2,1,1,3,2,6,1,2,87,19,2,44,225,1,2,126,112,8,1,1,1,1,110,135,62,5,61,3,86,1,15,4,66,16,1,1,5,1,1,25,1,1,3,28,3,2,2,1,1,5,1,13,15,14,3,4,26,1,11,37,1,58,36,69,2,4,165,1,211,4,37,41,30,1,2,2,11,5,1,1,38,69,52,2,185,16,25,207,2,1,22,11,219,11,4,2610,1005,2,1,11,136,4,4,2,3548,1,85,9,15,210,27,99,80,21,12,27,7,26,22,1,2,1352,203,203,119,452,50,12,330,6,3,1,1,23,93,2,42,2,22,2,3892,77,148,129,26,239,3,6,1,15,10,7,5,2,2,1,402,57,26,1022,35,2,430,151,473,1,460,3,163,132,496,105,85,6,14,5,11,1,2,12,3,1,19,1,52,208,58,211,150,253,6,297,36,85,517,81,8,255,12,2,1,9,1,2,3128,24,14,1409,129,24,445,16,3,4,16,9,155,17,417,2,1,71,10,1,39,51,5,6,26,31,1,12,30,3,57,2,1,3,7,24,3,787,45,6,1,1248,37,7,610,86,874,29,252,172,3,1,1,6,12,2,35,3,1,4,1,2,5,1,2878,20,1,31,1,40,10,10,121,63,4,30,1,368,5,183,1,1,1,6,2,5,7,3,2,11,2,42,3,12,2,1,1,225,7,32,30,18,3,1,10,6,3,2,2,2,1,11,8,7,40,3,467,408,136,1,43,2,5,73,1,1,38,10,4,4,1,1,2,7,18,65,1,1,1,19,126,1,275,82,2,2,2,1,1,57,3,14,1,81,2,7,2,2,80,8,26,2,111,19,181,8,1,1,2145,81,3,5,35,10,12,10,38,1,14,10,6,4,3,40,9,2,1,2,43,7,4,1,13,1,3,2,10,137,138,2,16,1446,4,18,3,60,11,1,2445,144,106,50,92,28,1,1,1,8,3,28,14,6,1,3,13,20,85,29,178,12,291,225,5,16,27,1,21,1,7,1,2,42,2,358,50,378,254,1,1,2862,3,3,741,6,178,628,150,25,153,5,60,5,4,4,19,6,2,17,8,2,2,9,1140,94,23,65,95,304,37,362,170,2,16,2,24,96,28,6,2165,9,1,18,18,1,42,8,4,23,27,1,4,29,4,2,10,1,115,11,3,27,1,10,10,1,164,13,3,10,26,4,17,7,20,25,22,40],["Other","Abies","Abies","Acacia","Acer","Achillea","Achillea","Aconitum","Aconitum","Other","Actaea","Other","Other","Other","Adonis","Adonis","Adonis","Aesculus","Other","Agrimonia","Agrostemma githago","Ailanthus","Alchemilla","Alchemilla","Algae","Algae","Algae","Alisma","Alisma","Allium","Allium","Alnus","Alnus","Alnus","Alnus","Alnus","Alnus","Alnus","Other","Amaranthaceae","Ambrosia","Ambrosia","Amphitrema","Amphitrema","Amphitrema","Amphitrema","Anagallis","Anagallis","Anagallis","Anagallis","Anagallis","Other","Other","Other","Other","Other","Anemone","Anemone","Anemone","Anemone","Anemone","Anemone","Anemone","Anthemis","Anthemis","Anthericum","Anthericum","Anthoceros","Anthoceros","Anthoceros","Anthoceros","Fungi","Fungi","Anthriscus caucalis","Anthriscus sylvestris","Anthriscus sylvestris-type","Anthriscus-type","Anthyllis","Aphanes","Apiaceae","Apiaceae undiff.","Apium","Apium inundatum-type","Apium-type","Araneae","Arcella","Arcella cf. A. megastoma","Arcella discoides","Arcella sp.","Fungi","Fungi","Arctium","Arctostaphylos uva-ursi","Arctostaphylos uva-ursi-type","Arctous alpina","Arenaria","Arenaria serpyllifolia-type","Armeria","Armeria","Armeria","Armeria","Arnica montana","Artemisia","Arthrodesmus","Aruncus","Ascaris","Ascaris lumbricoides","Ascomycota","Ascospora","Aspiromitus punctatus","Asplenium","Asplenium-type","Assulina","Assulina muscorum","Assulina seminulum","Assulina sp.","Aster-type","Asteraceae","Asteroideae","Asteroideae undiff.","Asterosporium","Astragalus alpinus-type","Astragalus danicus-type","Astragalus exscapus","Astragalus-type","Astrantia major","Astrantia-type","Athyrium","Athyrium filix-femina","Athyrium-type","Avena-type","Avena/Triticum-type","Bambusa-type","Barbarea-type","Betula","Betula","Betula","Betula","Betula","Bidens","Bidens-type","Bistorta","Bistorta officinalis","Bistorta officinalis-type","Blechnum","Boraginaceae","Botrychium","Botryococcus","Botryococcus braunii","Botryococcus neglectus","Botryococcus pila","Botryococcus sp.","Brachysporium obovatum/B. bloxami/Bactrodesmium betulicola (type 359, HdV)","Brachysporium pendulisporum (type 360, HdV)","Brassica","Brassicaceae","Brassicaceae undiff.","Brassicaceae-type","Bruckenthalia-type","Bryales","Bryonia alba/Helianthemum","Bryophyta","Bryophyta (type 354, HdV)","Bupleurum","Bupleurum falcatum-type","Bupleurum-type","Butomus","Butomus umbellatus","Butomus-type","Byssothecium circinans","Byssothecium circinans (type 16A, HdV)","Byssothecium circinans (type 16B, HdV)","Byssothecium circinans (type 16C, HdV)","Calla","Callitriche","Calluna","Calluna vulgaris","Caltha","Caltha palustris-type","Caltha-type","Calystegia","Campanula","Campanula rapunculoides","Campanula-type","Campanula/Phyteuma","Campanulaceae","Cannabis sativa","Cannabis-type","Cardamine","Cardamine pratensis-type","Cardamine-type","Carduus","Carex","Carpinus","Carpinus","Carpinus","Carum carvi","Carya","Caryophyllaceae","Caryophyllaceae undiff.","Caryospora sp. (type 1001, HdV)","Castanea","Castanea sativa","Castanea-type","Cedrus","Cedrus-type","Centaurea","Centaurea","Centaurea","Centaurea","Centaurea","Centaurea","Centaurea","Centaurea","Centaurea","Centaurea","Centaurea","Centaurea","Centaurea","Centaurea","Centaurea","Centropyxis","Centropyxis aculeata","Cerastium","Cerastium arvense-type","Cerastium cerastoides-type","Cerastium fontanum-type","Cerastium-type","Ceratophyllum","Cercophora sp. (type 112, HdV)","Cerinthe","Cerinthe minor","cf. Actinopeltis (type 8C, HdV)","cf. Androsace","cf. Diphasiastrum-type","cf. Endophragmia (type 572, HdV)","cf. Entophlyctis lobata","cf. Helianthemum","cf. Lamiaceae","cf. Lobelia","cf. Papaver","cf. Pedicularis","cf. Penium","cf. Persiciospora (type 124, HdV)","cf. Riccia","cf. Richonia variospora (type 140, HdV)","cf. Spadicoides bina (type 98, HdV)","cf. Trichocladium opacum (type 10, HdV)","cf. Trichoglossum hirsutum (type 77B, HdV)","Chaerophyllum","Chaerophyllum hirsutum","Chaerophyllum hirsutum-type","Chaetomium sp. (type 7A, HdV)","Chaetomium undiff.","Chamaenerion","Chamaenerion angustifolium","Chamaenerion angustifolium-type","Charcoal","Chelidonium","Chelidonium majus","Chironomidae","Chlamydomonadaceae","Chrysophyceae","Chrysosplenium","Chrysosplenium-type","Cicatricosporites australiensis","Cichorioideae","Cicuta virosa","Circaea","Cirsium","Cirsium-type","Cirsium/Carduus","Cladium mariscus","Cladocera","Clasterosporium caricinum (type 126, HdV)","Clematis","Closterium","Closterium undiff.","Coelastrum reticulatum","Comarum","Comarum palustre","Comarum-type","Coniferae-type","Coniochaeta cf. C. ligniaria","Coniochaeta ligniaria (type 172, HdV)","Coniochaeta xylariispora","Consolida regalis","Consolida-type","Convallaria","Convolvulus","Convolvulus arvensis","Convolvulus-type","Copepoda","Cornus","Cornus","Cornus","Cornus","Coronilla undiff.","Corylus","Corylus avellana","Crataegus","Crataegus-type","Crupina","Cryptomeria","Cupressaceae","Cuscuta","Cuscuta epithymum","Cuscuta europaea-type","Cyperaceae","Cystopteris","Daphne","Daucus","Daucus carota","Daucus-type","Delitschia","Descurainia sophia","Desmidiaceae","Desmidiales","Dianthus","Dianthus-type","Difflugia","Digitalis purpurea-type","Dinoflagellata","Dinoflagellata undiff.","Diphasiastrum alpinum","Diphasiastrum alpinum-type","Diphasiastrum complanatum","Diporotheca","Dipsacoideae","Dipsacus","Drosera","Drosera intermedia","Drosera rotundifolia","Drosera rotundifolia-type","Dryas octopetala","Dryas-type","Dryopteris","Dryopteris carthusiana-type","Dryopteris dilatata-type","Dryopteris filix-mas-type","Dryopteris-type","Echinops","Echium","Echium vulgare","Echium-type","Elatine","Elymus-type","Empetrum","Empetrum nigrum-type","Empetrum-type","Encalypta-type","Engelhardia-type","Entophlyctis lobata","Entorrhiza sp. (type HdV-527)","Ephedra","Ephedra cf. E. distachya","Ephedra cf. E. foeminea","Ephedra cf. E. fragilis","Ephedra distachya","Ephedra distachya-type","Ephedra foeminea","Ephedra foeminea-type","Ephedra fragilis","Ephedra fragilis-type","Epilobium","Epilobium-type","Epipactis","Equisetaceae","Equisetum","Eranthis hyemalis-type","Erica ciliaris-type","Ericaceae","Ericaceae undiff.","Ericales","Ericales undiff.","Eriophorum","Erodium","Eryngium","Eryngium-type","Euglypha","Eunotia sp.","Euonymus","Euonymus europaeus","Euphorbia","Euphorbia-type","Euphorbiaceae","Euphrasia","Euphrasia-type","Fabaceae","Fabaceae undiff.","Faboideae","Fagopyrum","Fagopyrum esculentum","Fagopyrum-type","Fagus","Fagus sylvatica","Falcaria vulgaris","Falcaria-type","Fallopia convolvulus","Fallopia convolvulus-type","Fallopia convolvulus/F. dumetorum","Filinia longiseta","Filipendula","Filipendula ulmaria/F. vulgaris","Frangula","Frangula alnus","Fraxinus","Fraxinus excelsior","Fraxinus ornus","Fumaria-type","Fungi","Fungi (type 11, HdV)","Fungi (type 123, HdV)","Fungi (type 17, HdV)","Fungi","Fungi (type 18, HdV)","Fungi (type 19, HdV)","Fungi (type 20, HdV)","Fungi (type 200, HdV)","Fungi (type 23, HdV)","Fungi (type 24, HdV)","Fungi (type 3A, HdV)","Fungi","Fungi (type 408, HdV)","Fungi","Fungi (type 53, HdV)","Fungi (type 54B, HdV)","Fungi (type 571, HdV)","Fungi (type 64, HdV)","Fungi","Fungi (type 729, HdV)","Fungi (type 73, HdV)","Fungi","Fungi (type 83, HdV)","Fungi (type 8A, HdV)","Fungi (type 8D, HdV)","Fungi (type 8E, HdV)","Fungi (type 90, HdV)","Fungi (type 96B, HdV)","Fungi undiff.","Fungi-type","Gaeumannomyces","Gaeumannomyces cf. G. caricis","Gaeumannomyces undiff.","Galeopsis","Galeopsis-type","Galeopsis/Ballota-type","Galium","Galium-type","Gelasinospora","Gelasinospora (type 1, HdV)","Gelasinospora (type 1A, HdV)","Gelasinospora (type 1B, HdV)","Gelasinospora reticulispora","Gelasinospora undiff.","Genista","Genista-type","Gentiana","Gentiana cruciata-type","Gentiana pneumonanthe-type","Gentiana undiff.","Gentiana-type","Gentianaceae","Gentianaceae undiff.","Gentianella","Gentianella campestris-type","Geoglossum sphagnophilum","Geoglossum sphagnophilum (type 77A, HdV)","Geoglossum sphagnophilum/Trichoglossum hirsutum (type 77A/77B, HdV)","Geraniaceae","Geranium","Geranium-type","Geum","Geum-type","Glaucium","Glaucium corniculatum","Glomus","Glomus cf. G. fasciculatum","Glyceria-type","Glyptotendipes pallens group (type 509, HdV)","Gnaphalium-type","Gratiola officinalis","Gymnocarpium dryopteris","Gypsophila","Gypsophila repens","Gypsophila repens-type","Gypsophila-type","Gyratrix hermaphroditus","Habrotrocha","Habrotrocha angusticollis","Habrotrocha angusticollis (type 37, HdV)","Hedera","Hedera helix","Hedysarum-type","Helianthemum","Helianthemum nummularium-type","Helianthemum oelandicum subsp. alpestris-type","Helicoma","Helicoön pluriseptatum","Helicosporium/Helicoön pluriseptatum (type 30, HdV)","Heliotropium europaeum","Helleborus","Heracleum","Heracleum sphondylium","Heracleum-type","Herniaria","Herniaria-type","Hippophaë","Hippophaë rhamnoides","Hippuris vulgaris","Hordeum","Hordeum-type","Hottonia palustris","Humulus","Humulus lupulus","Humulus/Cannabis","Humulus/Cannabis-type","Huperzia selago","Hyalosphenia papilio","Hyalosphenia subflava","Hydrocotyle","Hydrocotyle vulgaris","Hydrocotyle-type","Hydrodictyon","Hymenophyllum","Hyoscyamus","Hypericum","Hypericum perforatum-type","Hypericum perforatum/H. androsaemum-type","Hystrix","Ilex","Illecebrum verticillatum","Impatiens","Impatiens noli-tangere","Impatiens parviflora","Indeterminable","Indeterminable undiff.","Iris","Iris pseudacorus","Iris pseudacorus-type","Iris-type","Isoëtes","Isoëtes lacustris","Jasione","Jasione montana","Jasione-type","Juglandaceae","Juglandaceae-type","Juglans","Juncus sp. (type 2.2, MI)","Juniperus","Juniperus communis","Juniperus-type","Knautia","Knautia arvensis","Knautia arvensis-type","Koenigia alpina","Lamiaceae","Lamiaceae (tricolpate)","Lamium album-type","Lamium-type","Larix","Larix decidua","Larix-type","Larix/Pseudotsuga","Laserpitium","Laserpitium latifolium-type","Lasiosphaeria cf. L. caudata","Lasiosphaeria sp.","Lasiosphaeria sp. (type 63C, HdV)","Lasiosphaeria-type","Lathyrus-type","Lemna","Lemnoideae","Lepidoptera","Ligustrum","Liliaceae","Liliaceae-type","Linum","Linum austriacum-type","Linum catharticum","Linum catharticum-type","Linum flavum","Linum usitatissimum-type","Liquidambar","Listera ovata","Listera-type","Lithospermum officinale","Lobelia","Lonicera","Lonicera periclymenum","Lonicera xylosteum","Loranthus","Loranthus europaeus","Lotus","Lotus cf. L. corniculatus","Lotus pedunculatus","Lotus-type","Ludwigia palustris","Lycopodiaceae","Lycopodiaceae cf. Diphasiastrum complanatum","Lycopodiaceae undiff.","Lycopodiella inundata","Lycopodium","Lycopodium annotinum","Lycopodium annotinum-type","Lycopodium clavatum","Lycopodium clavatum-type","Lycopodium spike","Lycopodium tablets","Lycopodium undiff.","Lycopus","Lycopus-type","Lysimachia","Lysimachia cf. L. vulgaris","Lysimachia nemorum","Lysimachia nemorum-type","Lysimachia thyrsiflora","Lysimachia vulgaris","Lysimachia vulgaris-type","Lysimachia-type","Lythrum","Lythrum portula-type","Lythrum salicaria","Lythrum-type","Macrobiotus","Macrobiotus cf. M. echinogenitus","Magnolia-type","Maianthemum bifolium","Malus","Malus undiff.","Malus-type","Malva-type","Malvaceae","Marchantiophyta","Marrubium","Matricaria-type","Medicago falcata-type","Melampyrum","Melampyrum-type","Meliola","Mentha","Mentha-type","Menyanthes","Menyanthes trifoliata","Menyanthes trifoliata-type","Mercurialis","Mercurialis perennis-type","Micranthes stellaris","Micranthes stellaris-type","Microdalyellia armigera","Microrrhinum minus","Microthyriaceae","Microthyrium","Microthyrium microscopicum","Microthyrium undiff.","Microthyrium-type","Minuartia-type","Monactinus simplex","Monactinus simplex var. echinulatum","Monactinus simplex var. simplex","Monocotyledoneae","Montia","Mougeotia","Mougeotia cf. M. gracillima (type 61, HdV)","Mougeotia undiff.","Mougeotia-type","Muscari","Myosotis","Myosotis arvensis-type","Myrica-type","Myricaria germanica","Myriophyllum","Myriophyllum alterniflorum","Myriophyllum cf. M. spicatum","Myriophyllum cf. M. verticillatum","Myriophyllum heterophyllum","Myriophyllum spicatum","Myriophyllum spicatum-type","Myriophyllum undiff.","Myriophyllum verticillatum","Myriophyllum verticillatum-type","Nebela","Nebela parvula","Nebela undiff.","Neurospora","Normapolles","Nuphar","Nymphaea","Nymphaea alba","Nymphaea alba-type","Nymphaea cf. N. candida","Nymphaea undiff.","Nymphaeaceae","Nymphoides peltata-type","Nyssa","Odontites","Odontites-type","Oenanthe","Oenanthe-type","Oenothera","Olea","Oleaceae","Onagraceae","Onobrychis","Ononis","Ononis-type","Onopordum","Ophioglossum","Ophioglossum vulgatum","Orchidaceae","Oribatida (type 396, HdV)","Orlaya grandiflora","Ornithogalum umbellatum-type","Ornithogalum-type","Ostrya-type","Oxalis","Oxyria-type","Papaver","Papaver rhoeas-type","Papaver somniferum","Papaveraceae","Parapediastrum biradiatum","Parnassia","Parnassia palustris","Parnassia-type","Pediastrum","Pediastrum angulosum","Pediastrum angulosum var. angulosum","Pediastrum angulosum var. asperum","Pediastrum braunii","Pediastrum duplex","Pediastrum duplex var. duplex","Pediastrum duplex var. rugulosum","Pediastrum muticum var. scutum","Pediastrum orientale","Pediastrum undiff.","Pedicularis","Pedicularis palustris-type","Penium","Persicaria","Persicaria amphibia","Persicaria amphibia-type","Persicaria cf. P. lapathifolia","Persicaria lapathifolia","Persicaria maculosa","Persicaria maculosa-type","Petasites","Petasites hybridus-type","Petasites-type","Peucedanum","Peucedanum palustre-type","Peucedanum-type","Phegopteris","Phegopteris connectilis","Phragmites","Phragmites australis","Phragmites-type","Phyteuma","Phyteuma-type","Picea","Picea abies","Pilularia","Pimpinella anisum","Pimpinella major","Pimpinella major-type","Pimpinella major/P. saxifraga","Pinguicula","Pinnularia","Pinus","Pinus (Tertiary)","Pinus cembra","Pinus cembra-type","Pinus sp.","Pinus subg. Pinus","Pinus subg. Strobus-type","Pinus sylvestris","Pinus sylvestris-type","Pisum sativum","Plantaginaceae","Plantaginaceae undiff.","Plantago","Plantago alpina","Plantago alpina-type","Plantago atrata-type","Plantago coronopus","Plantago lanceolata","Plantago lanceolata-type","Plantago major","Plantago major-type","Plantago major/P. media","Plantago major/P. media-type","Plantago maritima-type","Plantago media","Plantago media-type","Plantago sp.","Platanus","Platycarya","Platyhelminthes (type 353A, HdV)","Platyhelminthes (type 353B, HdV)","Pleospora sp. (type 3B, HdV)","Pleospora undiff.","Pleurospermum","Pleurospermum austriacum","Pleurospermum austriacum-type","Poaceae","Poaceae (Cerealia-type excluding Secale)","Poaceae (Cerealia-type)","Poaceae (Cerealia)","Poaceae (Cerealia) excluding Secale","Poaceae (Cerealia) undiff.","Podocarpus-type","Podospora sp./Zopfiella sp. (type 466, HdV)","Podospora-type (type 368, HdV)","Polemonium","Polemonium caeruleum","Polygala","Polygonaceae","Polygonaceae undiff.","Polygonatum","Polygonum","Polygonum aviculare","Polygonum aviculare-type","Polygonum sp.","Polypodiaceae","Polypodiophyta (monolete, psilate)","Polypodiophyta (monolete, verrucate)","Polypodium","Polypodium vulgare","Populus","Porifera","Potamogeton","Potamogeton natans-type","Potamogeton-type","Potentilla","Potentilla-type","Potentilla/Comarum","Potentilla/Comarum-type","Potentilla/Fragaria","Poterium sanguisorba","Poterium sanguisorba subsp. sanguisorba","Primula","Primula clusiana-type","Primula farinosa-type","Primula veris-type","Primulaceae","Prunella-type","Prunus","Prunus sp.","Prunus-type","Pseudopediastrum boryanum","Pseudopediastrum boryanum var. boryanum","Pseudopediastrum boryanum var. boryanum sensu lato","Pseudopediastrum boryanum var. cornutum","Pseudopediastrum boryanum var. longicorne","Pseudopediastrum brevicorne","Pseudopediastrum integrum","Pseudopediastrum kawraiskyi","Pteridium","Pteridium aquilinum","Pteridophyta","Pteridophyta (monolete, verrucate)","Pteridophyta (monolete) undiff.","Pterocarya","Pterocarya fraxinifolia","Pulmonaria","Pulmonaria-type","Pyrola","Pyxidicula","Quercus","Quercus coccifera","Quercus pubescens","Ranunculaceae","Ranunculaceae","Ranunculaceae","Ranunculaceae","Ranunculaceae","Ranunculaceae","Ranunculaceae","Ranunculaceae","Ranunculaceae","Ranunculaceae","Ranunculaceae","Ranunculaceae","Reseda","Reseda lutea-type","Rhabdocoela","Rhabdocoela (type 353, HdV)","Rhamnaceae","Rhamnus","Rhinanthus","Rhinanthus-type","Rhinanthus/Veronica","Rhizopoda","Rhizopoda undiff.","Rhododendron","Rhododendron subsect. Ledum","Rhododendron tomentosum","Rhus-type","Ribes","Ribes","Ribes","Riccia","Rivularia","Rosa","Rosa-type","Rosaceae","Rosaceae undiff.","Rotifera undiff.","Rotifera/Tardigrada","Rubiaceae","Rubus","Rubus","Rumex","Rumex","Rumex","Rumex","Rumex","Rumex","Rumex","Rumex","Rumex","Rumex","Rumex","Rumex","Rumex","Rumex","Rumex","Sagina","Sagina procumbens-type","Sagina-type","Sagittaria","Sagittaria-type","Salix","Salix herbacea-type","Salvia","Sambucus","Sambucus","Sambucus","Sambucus","Sambucus","Sambucus","Sambucus","Sambucus","Sambucus","Samolus valerandi","Sample quantity","Sanguisorba minor-type","Sanguisorba officinalis","Sanicula europaea","Sanicula-type","Saussurea-type","Saxifraga","Saxifraga aizoides-type","Saxifraga granulata","Saxifraga granulata-type","Saxifraga hirculus-type","Saxifraga oppositifolia","Saxifraga oppositifolia-type","Saxifraga undiff.","Saxifragaceae","Saxifragaceae undiff.","Scabiosa","Scabiosa columbaria subsp. pratensis-type","Scabiosa columbaria-type","Scandix pecten-veneris/Caucalis platycarpos","Scenedesmus","Scenedesmus undiff.","Scheuchzeria","Scheuchzeria palustris","Sciadopitys","Sciadopitys-type","Scilla-type","Scleranthus","Scleranthus annuus","Scleranthus annuus-type","Scleranthus cf. S. annuus","Scleranthus cf. S. perennis","Scleranthus perennis","Scleranthus perennis-type","Scleranthus-type","Scrophularia","Scrophularia-type","Scrophulariaceae","Scutellaria","Secale","Secale cereale","Secale-type","Securigera varia","Sedum","Sedum-type","Selaginella","Selaginella selaginoides","Selaginellaceae","Senecio","Senecio-type","Senecio/Aster","Sequoia","Serratula","Serratula-type","Seseli-type","Sigmopollis","Silene","Silene dioica-type","Silene flos-cuculi","Silene latifolia","Silene viscaria-type","Silene vulgaris","Silene vulgaris-type","Silene-type","Silene-type undiff.","Sileneae","Silenoideae-type","Sinapis-type","Sium latifolium-type","Solanaceae","Solanum","Solanum cf. S. nigrum","Solanum dulcamara","Solanum nigrum","Solanum nigrum-type","Soldanella","Sorbus","Sorbus aria-type","Sorbus aucuparia","Sorbus group","Sorbus torminalis","Sorbus-type","Sordaria-type (type 55A, HdV)","Sordariaceae","Sordariaceae/Sordaria (type 55B, HdV)","Sparganium","Sparganium erectum","Sparganium-type","Spergula","Spergula-type","Spergularia-type","Sphagnum","Spirogyra","Spirogyra cf. S. scrobiculata","Spirogyra-type","Sporormiella","Sporormiella (type 113, HdV)","Stachys","Stachys-type","Staurastrum","Staurastrum undiff.","Stauridium tetras","Stellaria","Stellaria holostea","Stellaria-type","Stratiotes aloides","Succisa","Succisa pratensis","Succisa-type","Succisella","Swertia perennis","Symphytum","Symphytum cf. S. officinale","Symphytum-type","Symplocos","Tardigrada","Tardigrada (type 902, HdV)","Tardigrada undiff.","Taxus","Taxus baccata","Tetraëdron","Tetraëdron minimum","Tetraploa scheueri","Teucrium","Thalictrum","Thalictrum-type","Thecaphora","Thelypteris","Thelypteris palustris","Thesium","Thesium-type","Tilia","Tilia cordata","Tilia platyphyllos","Tilia undiff.","Tilletia sphagni","Tilletia sphagni (type 27, HdV)","Tofieldia","Transeauina","Transeauina (type 214, HdV)","Transeauina undiff.","Trapa","Trapa natans","Trichocladium opacum (type 10, HdV)","Trichuris trichiura","Trientalis","Trientalis europaea","Trifolium","Trifolium pratense","Trifolium pratense-type","Trifolium repens-type","Trifolium-type","Triglochin","Triticum","Triticum-type","Trochiscia undiff.","Trollius","Trollius europaeus","Trollius-type","Tsuga","Tsuga diversifolia-type","Tsuga-type","Turgenia latifolia","Typha","Typha angustifolia","Typha angustifolia-type","Typha angustifolia/Sparganium","Typha angustifolia/Sparganium-type","Typha latifolia","Typha latifolia-type","Typhaceae","Ulex-type","Ulmus","Ulmus/Zelkova","Umbilicus rupestris-type","Unknown","Unknown (Cretaceous)","Unknown (monolete, psilate)","Unknown (monolete)","Unknown (monolete) undiff.","Unknown (pre-Quaternary)","Unknown (Tertiary)","Unknown (trilete)","Unknown (trilete) undiff.","Unknown (type 160, HdV)","Unknown (type 181, HdV)","Unknown (type 224, HdV)","Unknown (type 33, HdV)","Unknown (type 366, HdV)","Unknown (type 38, HdV)","Unknown (type 41, HdV)","Unknown (type 708, HdV)","Unknown (type 74, HdV)","Unknown (type 86, HdV)","Unknown (type 91, HdV)","Urtica","Ustulina deusta","Ustulina deusta (type 44, HdV)","Utricularia","Vaccinioideae","Vaccinium","Vaccinium oxycoccos","Vaccinium-type","Valeriana","Valeriana cf. V. dioica","Valeriana cf. V. officinalis","Valeriana dioica","Valeriana dioica-type","Valeriana officinalis","Valeriana officinalis-type","Valerianella","Varia","Veratrum","Veratrum album","Veratrum-type","Verbascum","Vermes","Veronica","Veronica beccabunga","Veronica beccabunga-type","Veronica-type","Viburnum","Viburnum cf. V. opulus","Viburnum lantana","Viburnum opulus","Viburnum opulus-type","Viburnum undiff.","Vicia","Vicia cracca-type","Vicia-type","Vicia/Lathyrus","Vicia/Lathyrus-type","Viola","Viola canina-type","Viola palustris","Viola palustris-type","Viola tricolor","Viscum","Vitis","Vitis vinifera","Xanthium","Xanthium-type","Xylariaceae","Xylomyces chlamydosporis/X. aquaticus (type 201, HdV)","Zea mays","Zygnema","Zygnema-type","Zygnemataceae","Zygnemataceae undiff."]],"container":"<table class=\"display\">\n  <thead>\n    <tr>\n      <th>variablename<\/th>\n      <th>sites<\/th>\n      <th>samples<\/th>\n      <th>replacement<\/th>\n    <\/tr>\n  <\/thead>\n<\/table>","options":{"scrollX":"100%","columnDefs":[{"className":"dt-right","targets":[1,2]}],"order":[],"autoWidth":false,"orderClasses":false}},"evals":[],"jsHooks":[]}</script>
```

So you can see we've changed some of the taxon names in the taxon table (don't look too far, I just did this as an example).  To replace the names in the `samples()` output, we'll join the two tables using an `inner_join()` (meaning the `variablename` must appear in both tables for the result to be included), and then we're going to select only those elements of the sample tables that are relevant to our later analysis:


```r
allSamp <- samples(cz_dl)

allSamp <- allSamp %>%
  inner_join(translation, by = c("variablename" = "variablename")) %>% 
  select(!c("variablename", "sites", "samples")) %>% 
  group_by(siteid, sitename, replacement,
           sampleid, units, age,
           agetype, depth, datasetid,
           long, lat) %>%
  summarise(value = sum(value))
```

```
## `summarise()` has grouped output by 'siteid', 'sitename', 'replacement',
## 'sampleid', 'units', 'age', 'agetype', 'depth', 'datasetid', 'long'. You can
## override using the `.groups` argument.
```

```r
DT::datatable(head(allSamp, n = 50), rownames = FALSE,
                options = list(scrollX = "100%"))
```

```{=html}
<div id="htmlwidget-3dcc47e31c475e8d8803" style="width:100%;height:auto;" class="datatables html-widget"></div>
<script type="application/json" data-for="htmlwidget-3dcc47e31c475e8d8803">{"x":{"filter":"none","vertical":false,"data":[[1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399,1399],["Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky","Kameničky"],["Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies","Abies"],[240212,240213,240214,240215,240216,240217,240218,240219,240220,240221,240222,240223,240224,240225,240226,240227,240228,240229,240230,240231,240232,240233,240234,240235,240236,240237,240238,240239,240240,240241,240242,240243,240244,240245,240246,240247,240248,240249,240250,240251,240252,240253,240254,240255,240256,240257,240258,240259,240260,342283],["NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP","NISP"],[385,611,629,647,664,682,700,718,736,753,771,789,812,841,870,899,927,956,985,1014,1042,1071,1100,1129,1159,1192,1224,1256,1289,1321,1354,1386,1419,1451,1484,1516,1548,1581,1613,1646,1678,1710,1743,1775,1808,1840,1873,1905,1925,48],["Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP","Calibrated radiocarbon years BP"],[5,10,15,20,25,30,35,40,45,50,55,60,65,70,75,80,85,90,95,100,105,110,115,120,125,130,135,140,145,150,155,160,165,170,175,180,185,190,195,200,205,210,215,220,225,230,235,240,243,0],[24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,24238,1435],[15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602,15.970602],[49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332,49.726332],[8,8,19,43,39,32,26,20,31,34,19,14,9,8,11,8,9,13,34,14,13,7,12,20,9,4,7,6,10,23,24,50,19,37,23,40,13,10,4,10,12,11,11,7,10,16,11,33,46,39]],"container":"<table class=\"display\">\n  <thead>\n    <tr>\n      <th>siteid<\/th>\n      <th>sitename<\/th>\n      <th>replacement<\/th>\n      <th>sampleid<\/th>\n      <th>units<\/th>\n      <th>age<\/th>\n      <th>agetype<\/th>\n      <th>depth<\/th>\n      <th>datasetid<\/th>\n      <th>long<\/th>\n      <th>lat<\/th>\n      <th>value<\/th>\n    <\/tr>\n  <\/thead>\n<\/table>","options":{"scrollX":"100%","columnDefs":[{"className":"dt-right","targets":[0,3,5,7,8,9,10,11]}],"order":[],"autoWidth":false,"orderClasses":false}},"evals":[],"jsHooks":[]}</script>
```

## Simple Analytics

### Stratigraphic Plotting

We can use packages like `rioja` to do stratigraphic plotting for a single record, but first we need to do some different data management.


```r
# Get a particular site:
onesite <- samples(cz_dl[[1]]) %>% 
  inner_join(translation, by = c("variablename" = "variablename")) %>% 
  select(!c("variablename", "sites", "samples")) %>% 
  group_by(siteid, sitename, replacement,
           sampleid, units, age,
           agetype, depth, datasetid,
           long, lat) %>%
  summarise(value = sum(value))
```

```
## `summarise()` has grouped output by 'siteid', 'sitename', 'replacement',
## 'sampleid', 'units', 'age', 'agetype', 'depth', 'datasetid', 'long'. You can
## override using the `.groups` argument.
```

```r
onesite <- onesite %>%
  filter(units == "NISP") %>%
  group_by(age) %>%
  mutate(pollencount = sum(value, na.rm = TRUE)) %>%
  group_by(replacement) %>% 
  mutate(prop = value / pollencount)

topcounts <- onesite %>%
  group_by(replacement) %>%
  summarise(n = n()) %>%
  arrange(desc(n)) %>%
  head(n = 10)

widetable <- onesite %>%
  filter(replacement %in% topcounts$replacement) %>%
  select(age, replacement, prop) %>% 
  mutate(prop = as.numeric(prop))

counts <- tidyr::pivot_wider(widetable,
                             id_cols = age,
                             names_from = replacement,
                             values_from = prop,
                             values_fill = 0)

rioja::strat.plot(counts[,-1], yvar = counts$age,
                  title = cz_dl[[1]]$sitename)
```

![](simple_workflow_files/figure-html/stratiplot-1.png)<!-- -->

### Change in Time Across Sites

We now have site information across the Czech Republic, with samples, and with taxon names. I'm interested in looking at the distributions of taxa across time, their presence/absence. I'm going to pick the top 20 taxa (based on the number of times they appear in the records) and look at their distributions in time:


```r
taxabyage <- allSamp %>% 
  group_by(replacement, "age" = round(age, -2)) %>% 
  summarise(n = n())
```

```
## `summarise()` has grouped output by 'replacement'. You can override using the
## `.groups` argument.
```

```r
samplesbyage <- allSamp %>% 
  group_by("age" = round(age, -2)) %>% 
  summarise(samples = length(unique(sampleid)))

taxabyage <- taxabyage %>%
  inner_join(samplesbyage, by = "age") %>% 
  mutate(proportion = n / samples)

toptaxa <- taxabyage %>%
  group_by(replacement) %>%
  summarise(n = n()) %>%
  arrange(desc(n)) %>%
  head(n = 10)

groupbyage <- taxabyage %>%
  filter(replacement %in% toptaxa$replacement)

ggplot(groupbyage, aes(x = age, y = proportion)) +
  geom_point() +
  geom_smooth(method = 'gam', 
              method.args = list(family = 'binomial')) +
  facet_wrap(~replacement) +
  coord_cartesian(xlim = c(0, 20000), ylim = c(0, 1))
```

```
## `geom_smooth()` using formula 'y ~ s(x, bs = "cs")'
```

![](simple_workflow_files/figure-html/summarizeByTime-1.png)<!-- -->

We've now shown how taxa have changed (across all sites) over time.

### Distributions in Climate (July max temperature) from Rasters


```r
modern <- allSamp %>% filter(age < 50)

spatial <- sf::st_as_sf(modern, 
                        coords = c("long", "lat"),
                        crs = "+proj=longlat +datum=WGS84")
spatial
```

```
## Simple feature collection with 1553 features and 10 fields
## Geometry type: POINT
## Dimension:     XY
## Bounding box:  xmin: 13.21399 ymin: 48.77639 xmax: 18.63083 ymax: 50.76618
## CRS:           +proj=longlat +datum=WGS84
## # A tibble: 1,553 × 11
## # Groups:   siteid, sitename, replacement, sampleid, units, age, agetype,
## #   depth, datasetid [1,553]
##    siteid sitename  replacement     sampleid units   age agetype depth datasetid
##  *  <int> <chr>     <chr>              <int> <chr> <int> <chr>   <dbl>     <int>
##  1   1399 Kameničky Abies             342283 NISP     48 Calibr…     0      1435
##  2   1399 Kameničky Achillea          342283 NISP     48 Calibr…     0      1435
##  3   1399 Kameničky Alnus             342283 NISP     48 Calibr…     0      1435
##  4   1399 Kameničky Amaranthaceae     342283 NISP     48 Calibr…     0      1435
##  5   1399 Kameničky Apiaceae          342283 NISP     48 Calibr…     0      1435
##  6   1399 Kameničky Artemisia         342283 NISP     48 Calibr…     0      1435
##  7   1399 Kameničky Barbarea-type     342283 NISP     48 Calibr…     0      1435
##  8   1399 Kameničky Betula            342283 NISP     48 Calibr…     0      1435
##  9   1399 Kameničky Bistorta offic…   342283 NISP     48 Calibr…     0      1435
## 10   1399 Kameničky Cardamine prat…   342283 NISP     48 Calibr…     0      1435
## # … with 1,543 more rows, and 2 more variables: value <int>,
## #   geometry <POINT [°]>
```


```r
worldTmax <- raster::getData('worldclim', var = 'tmax', res = 10)
worldTmax
```

```
## class      : RasterStack 
## dimensions : 900, 2160, 1944000, 12  (nrow, ncol, ncell, nlayers)
## resolution : 0.1666667, 0.1666667  (x, y)
## extent     : -180, 180, -60, 90  (xmin, xmax, ymin, ymax)
## crs        : +proj=longlat +datum=WGS84 
## names      : tmax1, tmax2, tmax3, tmax4, tmax5, tmax6, tmax7, tmax8, tmax9, tmax10, tmax11, tmax12
```


```r
modern$tmax7 <- raster::extract(worldTmax, spatial)[,7]
head(modern)
```

```
## # A tibble: 6 × 13
## # Groups:   siteid, sitename, replacement, sampleid, units, age, agetype,
## #   depth, datasetid, long [6]
##   siteid sitename replacement sampleid units   age agetype depth datasetid  long
##    <int> <chr>    <chr>          <int> <chr> <int> <chr>   <dbl>     <int> <dbl>
## 1   1399 Kamenič… Abies         342283 NISP     48 Calibr…     0      1435  16.0
## 2   1399 Kamenič… Achillea      342283 NISP     48 Calibr…     0      1435  16.0
## 3   1399 Kamenič… Alnus         342283 NISP     48 Calibr…     0      1435  16.0
## 4   1399 Kamenič… Amaranthac…   342283 NISP     48 Calibr…     0      1435  16.0
## 5   1399 Kamenič… Apiaceae      342283 NISP     48 Calibr…     0      1435  16.0
## 6   1399 Kamenič… Artemisia     342283 NISP     48 Calibr…     0      1435  16.0
## # … with 3 more variables: lat <dbl>, value <int>, tmax7 <dbl>
```

#### Choosing Taxa


```r
maxsamp <- modern %>% 
  group_by(siteid, sitename) %>% 
  dplyr::distinct(tmax7)
head(maxsamp)
```

```
## # A tibble: 6 × 3
## # Groups:   siteid, sitename [6]
##   siteid sitename   tmax7
##    <int> <chr>      <dbl>
## 1   1399 Kameničky    210
## 2   3052 Chraňbož     222
## 3   3090 Dvůr Anšov   255
## 4   3172 Branná       234
## 5   3173 Barbora      234
## 6   3254 Loučky       215
```

Top 10

```r
topten <- allSamp %>% 
  dplyr::group_by(replacement) %>% 
  dplyr::summarise(n = dplyr::n()) %>% 
  dplyr::arrange(desc(n))
topten
```

```
## # A tibble: 1,078 × 2
##    replacement     n
##    <chr>       <int>
##  1 Poaceae      3892
##  2 Betula       3837
##  3 Cyperaceae   3751
##  4 Alnus        3625
##  5 Pinus        3518
##  6 Artemisia    3419
##  7 Quercus      3128
##  8 Salix        2878
##  9 Ulmus        2862
## 10 Abies        2658
## # … with 1,068 more rows
```


```r
pollen_subsamp <- modern %>% 
  dplyr::filter(replacement %in% topten$replacement[1:16])
head(pollen_subsamp)
```

```
## # A tibble: 6 × 13
## # Groups:   siteid, sitename, replacement, sampleid, units, age, agetype,
## #   depth, datasetid, long [6]
##   siteid sitename replacement sampleid units   age agetype depth datasetid  long
##    <int> <chr>    <chr>          <int> <chr> <int> <chr>   <dbl>     <int> <dbl>
## 1   1399 Kamenič… Abies         342283 NISP     48 Calibr…     0      1435  16.0
## 2   1399 Kamenič… Alnus         342283 NISP     48 Calibr…     0      1435  16.0
## 3   1399 Kamenič… Artemisia     342283 NISP     48 Calibr…     0      1435  16.0
## 4   1399 Kamenič… Betula        342283 NISP     48 Calibr…     0      1435  16.0
## 5   1399 Kamenič… Cyperaceae    342283 NISP     48 Calibr…     0      1435  16.0
## 6   1399 Kamenič… Filipendula   342283 NISP     48 Calibr…     0      1435  16.0
## # … with 3 more variables: lat <dbl>, value <int>, tmax7 <dbl>
```

Plot your results!


```r
ggplot() +
  geom_density(data = pollen_subsamp,
               aes(x = round(tmax7 / 10, 0)), col = 2) +
  facet_wrap(~replacement) +
  geom_density(data = maxsamp, aes(x = tmax7 / 10)) +
  xlab("Maximum July Temperature") +
  ylab("Kernel Density")
```

![](simple_workflow_files/figure-html/ggplot-1.png)<!-- -->