# ddd

# HW 2: Networks and Maps
You are going to write a visualization capable of representing spatial network data. The data consists of a list of a power grid nodes and links for North America. You are going to write a visualization that generates a map of power nodes, shows information about their attributes, their connectivity, and the distribution through space. Then, you are going to apply what you have learned to a new dataset.

**Note:** Throughout this tutorial, the location where you should insert new code is often given, but not always. Pay close attention and think about where the next thing should be inserted. If it references a variable that you have not declared or given the right value to yet, then it is probably in the wrong place. At times, we will be modifying the way something is used by running a calculation on another piece of data, and that may mean that the original location you put something in is no longer correct when it is modified.

## Preliminary DataFiles:
* **gridkit_north_america-highvoltage-vertices.csv [1.9MB]:** These are the nodes, or vertices, of the primary elements in the power grid. Each node has an ID, a location, and attributes describing the type and capabilities of that powerstation.
* **gridkit_north_america-highvoltage-links.csv [3.4MB]:** These are the links, or edges, between the nodes in the powergrid.
* **states.json [2.4MB]:** GeoJSON shape files for the United States.

## Show the points on the map.
* Create a leaflet map that pans and zooms
* Load the vertices data file and create circles for each energy grid item

Let's start by importing leaflet. Leaflet is a fantastic javascript library for mapping data. We will need a link to the javascript library and the css files. Add the following code to the `<head></head>` section of the index file.

```html
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.4.0/dist/leaflet.css"
   integrity="sha512-puBpdR0798OZvTTbP4A8Ix/l+A4dHDD0DGqYW6RQ+9jxkRFclaxxQb/SJAWZfWAkuyeQUytO7+7N4QKrDh+drA=="
   crossorigin=""/>
<!-- Make sure you put this AFTER Leaflet's CSS -->
 <script src="https://unpkg.com/leaflet@1.4.0/dist/leaflet.js"
   integrity="sha512-QVftwZFqvtRNi0ZyCtsznlKSWOStnDORoefr1enyq5mVL4tmKB3S/EnC3rRJcxCPavG10IcrVGSmPh6Qw5lwrg=="
   crossorigin=""></script>
```

Then we will create a map object in our `<script></script`. We will need a starting location to center our map, so let's give it the American University campus. Leaflet uses the reverse convention of many other mapping tools, taking the latitude as the first argument and the longitude as the second argument. Remember, longitude is the x coordinate and latitude is the y coordinate, so leaflet uses a `[y,x]` convention. The second parameter in the setView function is our starting zoom. We are letting leaflet know that it should draw on the html element with the id `'mapid'` which in this case is our map div in the body.

```javascript
var au = new L.LatLng(38.956379, -77.106342);
var mymap = L.map('mapid').setView(au, 5);
```

Now we are going to need to include a tile layer. The tiles are the actual representations of the world; these are the base layer on which we are going to draw other shapes and visualizations. Take a look at the following code. This points to the address of a server that takes requests for a location and serves up png map images at that location. There's attribution and copyright information, a maximum zoom, and minimum zoom. The id specifies the type of imagery we want to request from the server, in this case we are getting `mapbox.light` which is a lightly shaded grayscale that works well underneath  visualization layers. The access token is the default token provided in the leaflet tutorial - if you are using your map for an external application that a lot of users are visiting you should obtain your own.

```javascript
L.tileLayer('https://api.tiles.mapbox.com/v4/{id}/{z}/{x}/{y}.png?access_token={accessToken}', {
    attribution: 'Map data &copy; <a href="http://openstreetmap.org">OpenStreetMap</a> contributors, <a href="http://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, Imagery Â© <a href="http://mapbox.com">Mapbox</a>',
    maxZoom: 10,
    minZoom: 3,
    id: 'mapbox.light',
    accessToken: 'pk.eyJ1IjoiamFnb2R3aW4iLCJhIjoiY2lnOGQxaDhiMDZzMXZkbHYzZmN4ZzdsYiJ9.Uwh_L37P-qUoeC-MBSDteA'
}).addTo(mymap);
```

Now we have a tile layer and it has been added to our div. We should have something like the following for our starting map. It supports drag to pan and mousewheel zoom.

![Starting light gray map.](images/01.png)

We will import our data soon, but first we need some place for our d3 marks to appear. We will create an svg element much like in previous assignments, but this time it needs to be as a layer in the leaflet map. This makes it much easier to move the marks as the user pans and zooms around the display. Then we will add a group `<g>` for our marks. Add the following:

```javascript
var svgLayer = L.svg();
svgLayer.addTo(mymap)

var svg = d3.select("#mapid").select("svg"),
    g = svg.select("g");
    g.attr("class", "leaflet-zoom-hide");
```

Now that we have a place for our marks to live, let's load up the data file. Looking ahead at the requirements for this assignment, we can see that we are actually going to be three files. The d3 data load functions work asynchronously, meaning that we do not have a guarantee that one file will be finished loading before we start loading the next. Since our files depend on each other to create many of the visual features we will need for this assignment, we need to take a slightly different approach. We are going to use a `then()` **promise**, which will allow us to make our asynchronous data load calls but tell d3 to wait on all of them to finish before *then* proceeding with a callback function that manipulates the data or uses it to render marks.

```javascript
d3.csv('data/gridkit_north_america-highvoltage-vertices.csv')
            .then(ready);
```

In our ready function, we are going to create a circle svg mark for each of our power grid nodes. The argument for the ready function is the name of the data returned from our deferred function. Since our deferred function loaded the nodes, we call it `nodes`. We'll start off with red circles that have partial opacity so that we can see overlapping nodes.

```javascript
function ready(nodes){

    g.selectAll('circle')
        .data(nodes)
        .enter().append('circle')
            .style("opacity", .6)
            .style("fill", 'red')
            .attr('r', 2)
}
```

We still need to set and update the locations. If we were to set the location inside the enter join, they wouldn't update correctly when the user zooms. Inside the ready function, we register an event listener for the map, so that if the user zooms the location is updated. We need to convert the latitude and longitude to the correct pixel locations on the screen, so we will need a handy latlng object for each of our data items. We will define an update function that calls each time the zoom function ends on the map, then draw the circles in the updated pixel location. Then, **still inside the ready callback function**, we will call the function once at the end to place the circles initially at the correct location.

```javascript
nodes.forEach(function(d){
    d.LatLng = [+d.lat, +d.lon];
})

mymap.on('zoomend', updateNodes);

updateNodes();

function updateNodes(){
    g.selectAll('circle')
        .attr('cx', function(d){return mymap.latLngToLayerPoint(d.LatLng).x})
        .attr('cy', function(d){return mymap.latLngToLayerPoint(d.LatLng).y})
}
```

Now, we should have some nodes visible on the screen.

![Initial nodes visible.](images/02.png)

## Show the links on the map.
* Load the links data set and show the connections between nodes.
* Create a button to turn the nodes and links on or off.

Now we are going to show the links between nodes on the map. We have to load a different file into d3 in order to do that. Still inside our script file we are going to modify our promise. We now want to only call ready when all of the functions are done, so we need to `Promise.all()`. We are now deferring two load functions, and ready will wait for them both to finish before starting. `Promise.all()` will now be passing an array of objects to the callback function ready, one for each of our loaded files. We are changing the way we pass data to the ready function (the data returned from loading the links), so we add a `nodes` and `links` variables inside our ready function and assign them to the correct elements in the array.

```diff
- d3.csv('data/gridkit_north_america-highvoltage-vertices.csv')
-           .then(ready);
+ Promise.all([
+        d3.csv('data/gridkit_north_america-highvoltage-vertices.csv'),
+        d3.csv('data/gridkit_north_america-highvoltage-links.csv')])
+    .then(ready);

- function ready(nodes){
+ function ready(data){
+    var nodes = data[0];
+    var links = data[1];
```
 Inside our ready function we are going to create an svg line element for each of the links in the csv file. We will start out with light gray links connecting our nodes. Let's put this before the code that draws the circles so that the links will be underneath them.

```javascript
g.selectAll('line')
    .data(links)
        .enter().append('line')
        .style('stroke', 'gray')
        .style('opacity', .5)
```

We have a bit of an issue here! We need the locations for each of the node endpoints for our lines. If we look at the data for the links, we notice that each link has an id for the first vertex (v_id_1) and the second vertex (v_id_2) that defines its endpoints. We need a way to find those endpoints. We could also parse the last attribute of the link item, which has a linestring() object of the coordinates. However, we are going to need to know later which links are associated with each node, so we need a better solution.

|       |        |        |         |        |       |           |                                                    |          |     |                  |         |         |        |            |               |                                                          |
|-------|--------|--------|---------|--------|-------|-----------|----------------------------------------------------|----------|-----|------------------|---------|---------|--------|------------|---------------|----------------------------------------------------------|
| l_id  | v_id_1 | v_id_2 | voltage | cables | wires | frequency | name                                               | operator | ref | length_m         | r_ohmkm | x_ohmkm | c_nfkm | i_th_max_a | from_relation | wkt_srid_4326                                            |
| 36040 | 13322  | 13394  | 230000  | 6;3;3  |       | 60        |                                                    |          |     | 8088.92196971931 |         |         |        |            | ""            | "SRID=4326;LINESTRING(-99.24352997596 19.2906424386843   , -99.1715354102171 19.2598226173141)" |
| 10570 | 5366   | 18391  |         |        |       |           |                                                    |          |     | 31590.9354760208 |         |         |        |            | ""            | "SRID=4326;LINESTRING(-88.6532774661628 33.6356934310786 , -88.3554827632603 33.650708663621)"  |
| 12952 | 16909  | 18633  |         |        |       |           |                                                    |          |     | 3978.62021133905 |         |         |        |            | ""            | "SRID=4326;LINESTRING(-88.2401665213656 37.1061274858299 , -88.282623913995 37.0930230166821)"  |
| 11669 | 21844  | 21845  | 345000  | 3      |       |           | West Medway - West Walpole 345kV transmission line |          | 389 | 46.9560505450507 |         |         |        |            | ""            | "SRID=4326;LINESTRING(-71.2814750000587 42.1297525744322 , -71.2808923527655 42.1297976771994)" |

If we look at the data file for our nodes (i.e., vertices) we can see that each node has a unique id. We need a way to keep track of them so that we can later retrieve the longitude and latitude, allowing us to draw our link lines.

|       |                   |                  |             |              |           |      |          |     |                                                     |
|-------|-------------------|------------------|-------------|--------------|-----------|------|----------|-----|-----------------------------------------------------|
| v_id  | lon               | lat              | typ         | voltage      | frequency | name | operator | ref | wkt_srid_4326                                       |
| 23470 | -90.0404267803716 | 38.7386709600085 | joint       | 345000       | 60        |      |          |     | SRID=4326;POINT(-90.0404267803716 38.7386709600085) |
| 16854 | -93.8958936452349 | 49.0643204494094 | substation  |              |           |      |          |     | SRID=4326;POINT(-93.8958936452349 49.0643204494094) |
| 25563 | -75.15960889129   | 40.7070100889546 | joint       | 230000;69000 |           |      |          |     | SRID=4326;POINT(-75.15960889129 40.7070100889546)   |
| 6044  | -78.8398726969668 | 42.7896664868582 | sub_station |              |           |      |          |     | SRID=4326;POINT(-78.8398726969668 42.7896664868582) |

We are going to head back up to our list of global variables and declare a new d3 map (not to be confused with the javascript array map function) to hold our vertices keyed by id so that we can easily retrieve them `var vertices = d3.map()`. Then, in our ready function, we are going to add a line to populate our d3 map. Once that is done, we will be able to quickly query the vertices in our data by ID.

```diff
nodes.forEach(function(d){
+    vertices.set(+d['v_id'], d)
    d.LatLng = [+d.lat, +d.lon];
})
```

After we have iterated over the nodes, we are going to iterate over the links. For each link, we are going to look up the start and end vertices in our map. We save the latitude and longitude of these points as attributes on the link itself so that we can easily adjust their position on map zoom.

```javascript
links.forEach(function(d){
    var start = vertices.get(+d['v_id_1'])
    var end = vertices.get(+d['v_id_2'])
    d.start = [+start.lat, +start.lon]
    d.end = [+end.lat, +end.lon]
})
```

Just like with the nodes, we are going to create an update function as a listener to map zoom. We will place this close to the end our ready function to keep it out of the way of the rest of our code. We could also potentially combine the `updateNodes()` and `updateLinks()` functions into a single call, but let's keep them separate for now.

```javascript
mymap.on('zoomend', updateLinks)

updateLinks()

function updateLinks(){
    g.selectAll('line')
    .attr('x1', function(d){return mymap.latLngToLayerPoint(d.start).x})
    .attr('y1', function(d){return mymap.latLngToLayerPoint(d.start).y})
    .attr('x2', function(d){return mymap.latLngToLayerPoint(d.end).x})
    .attr('y2', function(d){return mymap.latLngToLayerPoint(d.end).y})
}
```

Now we should have some links visible to go with our nodes! We have one last change to make. We want to be able to make our network visible or hidden so that it doesn't always clutter our display. In the body, before the `<hr>` element that borders the top of our map, we are going to add a button that toggles our network's visibility.

```diff
  <body>
+   <button id='netbutton'>Nodes and Links</button>
    <hr>
    <div id="mapid"></div>
```

In our global variables, we want to add a boolean to keep track of the visiblity state: `var drawNetwork = true;`. One last touch is necessary to make this work. Outside and after the `ready` function, we are going to add a listener to the button using d3. This is a little less clunky than using the html `onclick=function()` call. We check the value of `drawNetwork`, and make the network svg elements visible or hidden as a result.

```javascript
d3.select('#netbutton')
    .on('click', function (){
        if(drawNetwork){
            d3.selectAll('circle')
                .attr('visibility', 'hidden')
            d3.selectAll('line')
                .attr('visibility', 'hidden')
        }
        else{
            d3.selectAll('circle')
                .attr('visibility', 'visible')
            d3.selectAll('line')
                .attr('visibility', 'visible')
        }
        drawNetwork = !drawNetwork
})
```
![Nodes and Links Visible](images/03.png)

## Alter the point representation based on the underlying data.
* Assign node hue based on type of power grid item
* Assign node radius based on network degree

We want to color the nodes by the type of energy resource they represent. This is a categorical attribute, so we are going to need a list of the different values that the attribute can have. Looking back at our data, we can see the different types of nodes are kept in the `typ` attribute. **Notice - we are using *bracket* notation here to access the `typ` attribute.** We could look through the file and make a list of all the different values, but it's a big file! d3 can do this for us much more quickly using nest. At the beginning of our ready function, we specify `typ` as our key attribute, and pass the data object as entries for nest to organize.

```javascript
var nested_data = d3.nest()
    .key(function(d){
        return d['typ']
    })
    .entries(nodes)
```

This will give us an object organized by the different values that `typ` can have. All of the data is still present, but now it is separated into separate arrays, one for each value of the key attribute. There is also a neat trick that we can use if we want to output the contents of a variable to the console as a JSON string, which will often be much easier for us humans to read (and to copy and paste into things like web tutorials). Here is the first item in our JSON version of the nested data with the key `typ` value `joint`.

```diff
- console.log(nested_data);
+ console.log(JSON.stringify(nested_data));
```
...
```json
[
  {
    "key": "joint",
    "values": [
      {
        "v_id": "23470",
        "lon": "-90.0404267803716",
        "lat": "38.7386709600085",
        "typ": "joint",
        "voltage": "345000",
        "frequency": "60",
        "name": "",
        "operator": "",
        "ref": "",
        "wkt_srid_4326": "SRID=4326;POINT(-90.0404267803716 38.7386709600085)",
        "links": 0,
        "LatLng": [
          38.7386709600085,
          -90.0404267803716
        ]
      }
    ]
  }
]
```

Now we just need to grab all of the unique values that `nested_data.key` can have. There are certainly other ways to obtain these values, but getting comfortable with nest is extremely useful for manipulating data with d3. We use a javascript array map function (not a d3.map) to collect the unique key values from the `nested_data`.

```javascript
var types = nested_data.map(function(d){
  return d['key'];
})
```

If we send this to the console, we will get a nice list of the types:

```json
["joint","substation","sub_station","station","merge","plant"]
```

We can also simplify our code a little bit by chaining the calls to nest and map, since we are going to be using our `nested_data` variable. This makes the code a little cleaner, and this type of chained method use is common in d3 code, so it is good practice to get used to writing it yourself.

```diff
+ var types = d3.nest()
- var nested_data = d3.nest()
    .key(function(d){
        return d['typ']
    })
    .entries(nodes)
+    .map(function(d){
+            return d['key'];
+        })

- var types = nested_data.map(function(d){
-     return d['key'];
-    })
```

Once we have this list of types, we can create a color scale. d3 has multiple built-in categorical color scales. We only need about 6 distinct colors for this problem, so we will use `d3.schemeCategory10`, which will provide more colors than we need. This is slightly different than the scales we have seen in the past, which were based on `d3.scaleLinear()`. In this case, we are taking values from our list of node types as the domain and mapping them to a list of colors as the scale.

```javascript
var hue = d3.scaleOrdinal(d3.schemeCategory10)
    .domain(types);
```

Lastly, we need to update our `enter()` join for our nodes, so that the hue scale is being used rather than assigning the color red to every node.

```diff
g.selectAll('circle')
    .data(nodes)
    .enter().append('circle')
        // .style("stroke", "black")
        .style("opacity", .6)
-        .style("fill", 'red')
+        .style("fill", function(d){
+                return hue(d['typ'])
+            })
        .attr('r', 2)
```

![Nodes colored by vertex type](images/04.png)

Now we can begin to think about updating the radius of the nodes by the number of links each power grid item has. First, we need to go into our initial foreach loop when the vertices are loaded and initialize an attribute for the number of links on each node data item.

```diff
nodes.forEach(function(d){
+   d.links = 0
    vertices.set(+d['v_id'], d)
    d.LatLng = [+d.lat, +d.lon];
})
```

Heading down to the foreach loop to get the link endpoints, we keep track of the number of times that we query a node by incrementing the link count. We do this for both the start and end nodes of the link.

```diff
links.forEach(function(d){
    var start = vertices.get(+d['v_id_1'])
    var end = vertices.get(+d['v_id_2'])
+    start.links += 1
+    end.links += 1
    d.start = [+start.lat, +start.lon]
    d.end = [+end.lat, +end.lon]
})
```

Now we have the domain on which to create a new linear scaling function between the number of links an energy node has and the radius of the circle that represents it. After we create the links, we create a `d3.scaleLinear()` with a domain of the range of possible links a node can have, and a scale of the circle sizes we want to display. This needs to take place in our link loading function after we have counted the number of links using the code above. To determine the extent of the domain, we iterate over the stored values of the vertex d3 map we created before. We are using a rangeRound() range, which rounds the output values of our scale to the nearest integer. This helps keep different node sizes discriminable.

```javascript
var nodeRadius = d3.scaleLinear()
    .domain(d3.extent(nodes.map(function(d){
            return +d.links
        })))
    .rangeRound([2,15])
```

Finally, we replace our default radius with the call to the nodeRadius scale to get the ranged value based on the number of links each power grid item has.

```diff
g.selectAll('circle')
    .data(nodes)
    .enter().append('circle')
        .style("opacity", .6)
        .style("fill", function(d){
                return hue(d['typ'])
            })
-        .attr('r', 2)
+        .attr('r', function(d){
+                return nodeRadius(d.links)
+            })
```

![Nodes sized by number of links](images/05.png)

## Aggregate the points by state.
* import the turf.js library
* create a layer for polygons and add it to the map
* determine how many nodes are located within each state
* assign a color to each state polygon based on the node count
* create a button to show/hide the states

To show the states, we are going to take a different approach and use leaflet's built-in geojson parsing capability. We can still use our now familiar d3 promise scheme to asynchronously load the data in our queue and pass the loaded data to our ready function. This time, we are loading JSON data, so we use a different function: `d3.json`:

```diff
Promise.all([
        d3.csv('data/gridkit_north_america-highvoltage-vertices.csv'),
-        d3.csv('data/gridkit_north_america-highvoltage-links.csv')])
+        d3.csv('data/gridkit_north_america-highvoltage-links.csv'),
+        d3.json('data/states.json')])
    .then(ready);
```
...
```diff
function ready(data){

    var nodes = data[0];
    var links = data[1];
+   var states = data[2];
```
We are going to create another layer to hold the features, but this will not be another svg layer as before. Instead, we are going to create a more typical leaflet layer. First, we prepare by declaring our layer as a global variable `var statesLayer;`. Then, inside our ready function, we use leaflet to load the states object into the layer. We then add it to our map.

```javascript
statesLayer = L.geoJson(states)
statesLayer.addTo(mymap);
```

And that's really all it takes! The default style properties for states are blue with dark blue lines, so you should see something like this (note that the network visibility is toggled off in this image):

![US States added to map](images/06.png)

This is kind of boring, though. It would be interesting to see how many nodes each state contains in a choropleth map. We can use another library, turf.js, to perform these types of spatial calculations and many more. In our `<head></head>`, let's go ahead and add a link to turf.

```html
<script src='https://npmcdn.com/@turf/turf/turf.min.js'></script>
```

Turf has a couple of features that would be useful here. For example, if we look at the [Turf Api](http://turfjs.org/docs/), we can see a function called `booleanContains()` that takes a two geometries (e.g., point, polygon, line) and determines if the second geometry is completely inside the first geometry. Our energy nodes are points and our states are polygons, so it is pretty easy to imagine a nested for loop that would count the number of points located within each polygon and store the results. We don't have to do it that way, though, as turf also has another function, `collect()` that handles this exact situation. It takes a feature collection of points and a feature collection of polygons and aggregates the points into the polygons for us. Our feature collection of polygons is already in place, as that is what we have loaded from the states file. We just need to create a feature collection of the nodes, which turf can also handle for us!

Inside our ready function, let's create a new array that we will use for this called `var nodeFeatures = []`. Inside our nodes iterator at the beginning of our ready function, we are going to make a small addition to create point spatial features from our nodes. Then, once we have all of these created, we can use turf to transform these individual point features into a feature collection. Notice there is a small gotcha in the convention that turf uses for coordinates: turf expects longitude first, then latitude, like our familiar `[x,y]` convention. This is different from leaflet, as we discussed earlier in this tutorial, so that is something to be wary of if you are using them together and your points are not appearing in the right place!

```diff
nodes.forEach(function(d){
    d.links = 0
    vertices.set(+d['v_id'], d)
    d.LatLng = [+d.lat, +d.lon];
+    nodeFeatures.push(turf.point([+d.lon, +d.lat], d))
})

+   nodeFeatures = turf.featureCollection(nodeFeatures)
```

We are going to going to use turf to `collect()` the points into the polygons for the states. We need to specify an attribute that turf will pull from each node and add to a new attribute array in the polygons. In this case, we will make a list of all the power grid nodes that are contained within each state in a new attribute called `ID`. We call the new collected polygons `chorostates` and use them for our leaflet layer in place of our original states collection.

```diff
+   var chorostates = turf.collect(states,nodeFeatures, 'v_id', 'ID')
+   statesLayer = L.geoJson(chorostates)
-   statesLayer = L.geoJson(states)
statesLayer.addTo(mymap);
```

Visually, the rendered result is the same. We need to provide a way to scale the color of the states based on the number of links. We know how to do this using d3, but let's take a look at the convention that leaflet uses for its shape layers. When we create our layer, we can also specify a style function that leaflet will use to draw the states. If we create our function using the style function below, we will get orange states with dashed white lines as the borders. When we add the states to the state layer, we just need to let leaflet know which function we are using to generate the style object.

```javascript
function statestyle(feature) {
    return {
        weight: 2,
        opacity: 1,
        color: 'white',
        dashArray: '3',
        fillOpacity: 0.7,
        fillColor:  'orange'
    };
}
```
```diff
-   statesLayer = L.geoJson(chorostates)
+   statesLayer = L.geoJson(chorostates, {
+                style: statestyle
+            })
```

![States with updates style file](images/07.png)

There are some definite similarities here with the way d3 assigns style properties to marks. We could imagine creating a color scale for d3 that returns a color from a range of values based on the number of links from our domain of link counts. Let's try and keep doing this using the leaflet convention, which lines up closely with the [Leaflet Interactive Choropleth Map tutorial](http://leafletjs.com/examples/choropleth/) in case you are interested in extending the functions we cover in this homework. We will want to create a function that gives us a color based on the number of links each state contains. Then, we update the value in our style function so that it calls the `getColor()` function by passing the length of the `ID` array we collected for each state. This length is the number of the unique v_id values that turf identified as being within the boundaries of each state. We pick our colors based on the great collection available at [Color Brewer](http://colorbrewer2.org/).

```javascript
function getColor(d) {
    return d > 1000 ? '#800026' :
            d > 500  ? '#BD0026' :
            d > 200  ? '#E31A1C' :
            d > 100  ? '#FC4E2A' :
            d > 50   ? '#FD8D3C' :
            d > 20   ? '#FEB24C' :
            d > 10   ? '#FED976' :
                        '#FFEDA0';
}
```
```diff
function statestyle(feature) {
    return {
        weight: 2,
        opacity: 1,
        color: 'white',
        dashArray: '3',
        fillOpacity: 0.7,
-       fillColor:  'orange'
+        fillColor:  getColor(feature.properties.ID.length)
    };
}
```

And there we have it! Our states should now be rendering using our nice color scale based on the number of nodes in each state. The last thing we need is a way to turn the states on and off, in case we want to more easily inspect the network structure. Let's add a new button to the top of our html body, called `<button id='statebutton'>States</button>`. We will also create a new global variable, `var drawstates = true`. Finally, in our ready function, we register a callback function as an event listener on the statebutton. This works a little bit differently than our network visibility callback, as we don't have to individually turn the svg elements visible or hidden. We can turn entire layers on and off in leaflet by just adding them to or removing them from the map.

```javascript
d3.select('#statebutton')
    .on('click', function (){
        if(drawstates){
            mymap.removeLayer(statesLayer)
        }
        else{
            statesLayer.addTo(mymap)
        }
    drawstates = !drawstates
})
```

![Chloropleth map](images/08.png)

## Aggregate the nodes by hexagon.
* create a bounding box for all nodes.
* using turf, create hexbins within that bounding box
* using turf collect(), determine how many nodes are located within each hex
* assign a color to each hex based on the node count
* create a button to show/hide the hexbins

One of the problems with our current choropleth is that the states are all very different sizes and shapes. It is also difficult to tell where in each state the concentration of nodes are located. We could turn on both the network layer and the state layer at the same time, but there is a little bit of interference between the channels. We're going to use a uniformly sized shape, a hexagon, to cover the map and aggregate our values in a different way.

Again, turf is going to do most of the heavy lifting for us. Given a bounding box, turf can create a regularly spaced grid of points or shapes on the map. We can create that bounding box so that it only includes the area of our nodes data, or the feature collection of the node points that we created earlier. We want our nodes to be about 200 kilometers in size. We can vary the size of the hexes to give our binning higher or lower resolution, but 200km works fine for now. We then add our new hexgrid to a newly created hexlayer, which we add to our map so that we can verify that it actually covers the area.

```javascript
var bbox = turf.bbox(nodeFeatures)
var cellSize = 200;
var units = 'kilometers';
var options = {};
options.units = units;

var hexgrid = turf.hexGrid(bbox, cellSize, options);
var hexLayer = L.geoJson(hexgrid).addTo(mymap)
```

![Hex layer across the bounding box for our node points](images/09.png)

It looks like our hex layer is working! Now we turn our attention to aggregating the points by hex feature. These newly created hex features are in a feature collection, exactly as our states were when we loaded them. All we need to do now is use `turf.collect()` to once again aggregate the nodefeatures within the hex features. This also lets us filter out all of the hexes that do not contain any nodes. We know that we are going to use the new featurecollection that results for our rendering, so after we create it we are going to substitute it for our original hexgrid in the created hexlayer.

**Warning** For some reason, the accuracy of the `turf.hexGrid()` feature collections in turf has been a little off lately, particularly in the use of `turf.Collect()` calls. If you run into trouble, and collect just does not seem to work with turf and the hexgrid, you may substitute `turf.squareGrid()`, which works in a very similar way but does not appear to have the same issues.

```diff
var hexgrid = turf.hexGrid(bbox, cellSize, units);
+   var hexbins = turf.collect(hexgrid,nodeFeatures, 'v_id', 'ID')

+   hexbins.features = hexbins.features.filter(function(d){
+           return d['properties']['ID'].length > 0
+       })
-   var hexLayer = L.geoJson(hexgrid).addTo(mymap)
+   var hexLayer = L.geoJson(hexbins).addTo(mymap)
```

![Only showing hexes that contain at least one node.](images/10.png)

Now we create the style function that will specify the hex color using a combination of leaflet and d3. First, we create a `hexStyle()` function that returns a style object based on a feature. We are going to default the fill color to orange for now. Then, we update our layer creation so that it uses our style function, just as we did when we created our states.

```javascript
function hexstyle(feature) {
    return {
        opacity: 0,
        fillOpacity: 0.7,
        fillColor: 'orange'
    };
}
```
```diff
- var hexLayer = L.geoJson(hexbins).addTo(mymap)
+ var hexLayer = L.geoJson(hexbins, {
+        style: hexstyle,
+    }).addTo(mymap)
```

Orange looks okay, but be want a dynamically assigned value using d3. Luckily, d3 has built-in sequential color scales that have been determined through research to be extremely effective. We are going to use the magma sequential scale, which looks like this:

![Magma color scheme](images/magma.png)

Our map is very light, so we are going to use the dark colors at the low end of the color scheme to represent high values in our data so that they stand out more. The light colors at the high end of the color scheme are much close in value to the light shades of the map, so they will not pop out as much. The color scheme takes a value from `[0,1]` and returns a color that far along the scale. We are going to use a `d3.scaleLinear()` to generate the values from `[0,1]` that the scheme will use, except that we flip the range to `[1,0]` so that the color scheme is reversed to match our very light gray map. Our domain is created from the minimum and maximum number of nodes within each hex feature, or the extent. Then, we create another function, `getHexColor()`, to use the `d3.interpolateMagma()` function with our scaled and reversed value.

```javascript
var magma = d3.scaleLinear()
    .domain(d3.extent(hexbins.features.map(function(d){
        return d['properties']['ID'].length
    })))
    .range([1,0])

function getHexColor(d) {
    return d3.interpolateMagma(magma(d))
}
```

We are almost there! We need to update our `hexStyle()` function so that the object it returns uses our new magma scale.

```diff
function hexstyle(feature) {
+   var hc = getHexColor(feature.properties.ID.length)
    return {
        opacity: 0,
        fillOpacity: 0.7,
-       fillColor: 'orange'
+       fillColor: hc
    };
}
```

![Hexes colored by contained nodes](images/11.png)

Let's wrap everything up by adding our last button for the hexlayer. In our body, we create one final `<button id='hexbutton'>Hexbins</button>` in our list of buttons. Then, in our ready function, we register a callback function as an event listener on the button. Just like with the states, we remove the layer from the map if we don't want to see it, and the add it again to the map if we want to show it.

```javascript
d3.select('#hexbutton')
    .on('click', function(d){
            if(drawHex){
                mymap.removeLayer(hexLayer)
            }
            else{
                hexLayer.addTo(mymap)
            }
        drawHex = !drawHex
    })
```

We can now turn all three layers on and off independently in our final product. Which is great, because otherwise we would be stuck with this - yuck!

![Too many layers](images/12.png)

# Apply these steps to a new data set
Included in this repository is a second collection of data sets for the European power grid. This data is of the same type as the previous North American data set (nodes, links, and boundaries), but **unlike the last assignment**, you will load the new data set into the map view at the same time. You will need to plan accordingly - this means that you can reuse many of the same ideas, but you will not be able to make simple substitutions to make this work. If you are running into trouble, I always recommend checking geojson data out at [geojson.io](http://geojson.io); this may give you some clues why a dataset or boundary is not behaving the way that you expect.

For full credit on this assignment, you will need to apply the steps in this tutorial to the European data set. This tutorial will get you most of the way there, but there is some additional work required. Accounting for two distinct data sets is not so simple, and requires some transformation - you cannot just swap the file loaded into the `d3.csv()` or `d3.json()` functions (though that is a definite first step). For full credit on this assignment, you will need to do the following:

1. [20%] Promises. Load the European data sets in addition to the North American data sets. This means modifying the Promise structure and the ready method in order to schedule loading all of the files.
1. [20%] Nodes. For each node in the European grid, assign the scaling of the circle so that the radius represents the extent of the **European grid only**. North American will keep its own scale, Europe will get its own unique scale. You may use the same color scheme for nodes only if the European data contains the same types of nodes.
1. [20%] Links. For each link in both data sets, modify the thickness of the line so that it represents the voltage. This should be based on the extent of the range of voltage values within each country. If no voltage information is available for a line, it should be drawn at the minimum width (nonzero) and be red in color.
1. [20%] Boundaries. Create new state boundary and hexgrid bins for the European dataset. The colors for each should be based on the European dataset only, but can use the same color schemes. For the hexgrid bins, the bounding box should only include the points in the European dataset (this is trickier than it sounds).
1. [10%] Buttons. Add new buttons to turn on the European grid, country boundaries, and bins on and off. Add two extra buttons with the following names and behavior:

    - Uniform. When this button is selected, both the North American and European representations will scale to worldwide min and max values.
    - Independent. When this button is selected, both the North American and European representations will scale to min and max values based only on the extent of the data within each continent (this is the default behavior you have been assigned in the preceding items).
1. [10%] HTML Table. To the right of the div containing the map, construct an HTML table of the grid nodes only. This means adding a DIV to the body and using d3 to populate it with HTML elements. It does not have to be scrollable or sortable, but should contain the following items for each node:

    - v_id
    - lon
    - lat
    - typ
    - voltage
1. [+5%] Pan and Zoom. When the user clicks on any item in the table row, the map should pan and zoom to that location.
