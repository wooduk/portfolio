+++
showonlyimage = false
draft = false
image = "/img/portfolio/ukturnout.png"
date = "2019-11-13T18:25:22+05:30"
title = "Command-Line Cartography for a UK Election"
weight = 0
+++

Create a map of the UK constituencies ready for election night.

<!--more-->

This tutorial adapts the steps from @mbostock’s Command-Line Cartography tutorial to show how to easily make a thematic map of UK election results using free open-source Javascript tools. This is what we will make in Part 1:

![UK 2017 Winner Map](/img/portfolio/uk_maps/uk_2017_winner.png)
*Figure 1: UK General Election 2017 results: Winners by constituency.*

The plan is to visualise the results of the last UK election by creating a map that colours each parliamentary constituency according to the party that gained the most votes. As a bonus we also visualise turnout.

## Getting the boundary geometry

The first step to creating our map is to get some polygons representing the constituency boundaries in the UK. The Ordinance Survey release the boundary-line product twice a year. I found a more convenient copy — for fetching with the command line — at mySociety.org

    curl --progress-bar http://parlvid.mysociety.org/os/bdline_gb-2019-10.zip -O bdline_gb-2019-10.zip

The archive contains lots of different administrative boundaries and is quite large (hence I added the `--progress-bar` option). We need to extract only the Westminster constituency boundaries files for this chart.
 
     unzip -o bdline_gb-2019-10.zip Data/GB/westminster_const_region.prj Data/GB/westminster_const_region.shp Data/GB/westminster_const_region.dbf Data/GB/westminster_const_region.shx
mv Data/GB/westminster_const_region.* .

As suggested by @mbostock, visiting mapshaper.org and dragging the `westminster_const_region.shp` into your browser is a good way to preview what we’ve extracted:

![UK 2017 Winner Map](/img/portfolio/uk_maps/uk_const_1.png)
*Figure 2: UK Westminster constituencies in mapshaper.org*

Looks great, but the Northern Ireland constituencies are missing. Turns out these are available separately from the Ordnance Survey Northern Ireland:

    wget http://osni-spatial-ni.opendata.arcgis.com/datasets/563dc2ec3d9943428e3fe68966d40deb_3.zip
    unzip 563dc2ec3d9943428e3fe68966d40deb_3.zip

## Get Set Up

We are going to use a series of Javascript command-line tools that Mike Bostock described in more detail in his tutorial. They can be installed using the following commands if you have not already [You’ll need node and npm installed for this next step to work]:

    npm install -g shapefile          # gives us shp2json
    npm install -g d3-geo-projection  # geoproject, geo2svg
    npm install -g topojson           # geo2topo, topo2geo
    npm install -g ndjson-cli
    npm install -g d3

Use *shp2json* to convert the Shapefiles to GeoJSON:

    shp2json westminster_const_region.shp -o gb.json

The Great Britain (GB) boundary file data is already projected onto the British National Grid (EPSG:27700). However the geometry data in the Northern Ireland (NI) boundaries are defined in EPSG:4326 (WSG 84). I tried to find a way to use d3-geo-projection to convert the NI geometry to EPSG:27700 but I have to admit it was beyond me. Instead I fell back to using ogr2ogr to transform the NI shapefile. I’m on MacOS so ogr2ogr can be installed using brew:

    brew install gdal

Then I reprojected the Shapefile from EPSG:4326 to EPSG:27700 as follows:

    ogr2ogr -f "ESRI Shapefile" ni.shp OSNI_Open_Data_Largescale_Boundaries__Parliamentary_Constituencies_2008.shp -t_srs EPSG:27700 -s_srs EPSG:4326

Then convert this Shapefile to GeoJSON as we with did with the GB boundaries previously:

    shp2json ni.shp -o ni.json

## Merging the GB and NI geometry

We have converted the binary shapefiles to more human readable GeoJSON. But we still have two separate files. It would be much more convenient to have all the boundaries together in one file.

In part 2 of his tutorial, Mike introduces `ndjson-cli`, his tool for creating and manipulating newline-delimited JSON. I figured we could use this tool make our GeoJSON files into newline-delimited JSON and then simply cat them together. I also need to pull out the constituency identifier and make it common across each file using `ndjson-map`:

    ndjson-split ‘d.features’ < gb.json \
    | ndjson-map '{id: d.properties.CODE, properties:d.properties, type:d.type, geometry:d.geometry}' > gb_id.ndjson
ndjson-split 'd.features' < ni.json \
    | ndjson-map '{id: d.properties.PC_ID, properties:d.properties, type:d.type, geometry:d.geometry}' > ni_id.ndjson
cat gb_id.ndjson ni_id.ndjson > uk.ndjson

I tried working from this point forward with ndjson and streaming the geometry into `geoprojection` but this was not successful. I found it necessary to convert the concatenated boundaries back to a single JSON object using the `ndjson-reduce` command before continuing with projections:

    ndjson-reduce 'p.features.push(d), p' '{type: "FeatureCollection", features: []}' < uk.ndjson > uk.json

Great we have all the boundaries in one JSON file. However, this file is about 120M. Clearly these geometry definitions are more detailed than we need for a web based visualisation. To reduce the file size we can define a bounding box — in pixels — for our chart and then simplify the polygons appropriately for this box without noticeable loss of detail.

Use the `geoproject` tool with the `.fitsize()` method of `d3.geoIdentity()` to map the points to a bounding box of our choosing [I also took the opportunity to flip the polygon definitions vertically as we will eventually render as svg]:

    geoproject 'd3.geoTransform({point: function(x, y) { this.stream.point(x, -y); }})' < uk.json \
    | geoproject 'd3.geoIdentity().fitSize([960, 960], d)'\
    > uk-960.json

Now we can reduce the size of our GeoJSON file by switching to TopoJSON and then simplifying, quantising, and compressing the polygon definitions. Please read Part 3 of Mike Bostock’s tutorial for a more detailed explanation of these steps, which I copy below.

I used `geo2topo` to convert to TopoJSON and then use the `toposimplify` and `topoquantize` tools to reduce file size from 120M to 385K:

    geo2topo const=uk-960.json \
    | toposimplify -p 1 -f \
    | topoquantize 1e5 \
    > uk-simpl-topo.json

Now let’s have a quick look at our output. To produce an svg at the command line we can use the `geo2svg` tools. First we do a conversion back to GeoJSON using `topo2geo`:

    topo2geo const=- < uk-simpl-topo.json \
    | geo2svg -p 1 -w 960 -h 960 > uk.svg

## Adding Election Results

So we have prepared our constituency boundaries. Now we want to shade these polygons according to some election results.
I found a csv containing election results back to 1918 on the UK parliament website:

    wget http://researchbriefings.files.parliament.uk/documents/CBP-8647/1918-2017election_results.csv -O 1918_2017election_results.csv

I used the command-line tool `gawk` to filter this csv to get just the 2017 results and fill in nulls with zeros. The tricky part was that some of the constituency names contained commas which needed to be ignored.

    gawk -F ',' -v OFS=',' '{for (i=1;i<=NF;i++) { if ($i=="") $i="0" }; print}' < 1918_2017election_results.csv | gawk 'NR==1 {print}; { if ($18 == 2017) print}' FPAT='([^,]+)|("[^"]+")' > 2017_results.csv

Mike shows how to join data like this with the geometry by first converting to newline-delimited json. Following his example I used csv2json and his ndjson-cli tools to first convert the election results:

    csv2json < 2017_results.csv | tr -d '\n' | ndjson-split > 2017_results.ndjson

Similarly we prepare our boundary geometry as ndjson:

    topo2geo const=- < uk-simpl-topo.json | \
    ndjson-split 'd.features' > uk-simpl.ndjson

And join the two together:

    ndjson-join 'd.id' 'd.constituency_id' uk-simpl.ndjson 2017_results.ndjson \
    | ndjson-map 'd[0].results = d[1], d[0]' \
    > uk_2017_results.ndjson

Mike’s example is to colour the polygons of the map according to the value of population density in that area. The most similar quantity I have in the election results — i.e. a continuous quantity — is turnout. So in order to be able to follow his example I decided to make a Chloropleth using this turnout information before moving on to looking at winners.

The code below uses ndjson-map to assign each feature (i.e. constituency) in the GeoJson with a property called `fill`. When we use `geo2svg` to produce our image it will pick up on this property and use it to colour the polygons appropriately. The value of fill needs to be a string containing the hex code of the colour that we want each polygon to be (e.g. `#0087dc`).

    ndjson-map -r d3 'z = d3.scaleSequential(d3.interpolateViridis).domain([0.5, .8]), d.properties.fill = z(d.results["turnout "]), d' < uk_2017_results.ndjson \
    | geo2svg -n --stroke none -p 1 -w 960 -h 960 > uk_2017_turnout.svg

It might be worth trying to break this one-liner down a bit to understand what is going on. We use d3 to translate the turnout information (a number between 0 and 1 representing the proportion of the electorate who actually voted in that constituency) into a hex code. The ndjson-map tool lets us step through each constituency and apply some kind of operation. The argument we pass to ndjson-map is three lines of Javascript.

The second line:

   d.properties.fill = z(d.results["turnout "])

says that for each feature create a property called fill whose value is the output of a function z . That function z takes the value of turnout — which we already attached to the feature in our previous join — and returns a hexcode representing a colour. This z function is defined in the first line of Javascript:

    z = d3.scaleSequential(d3.interpolateViridis).domain([0.5,0.8])

It uses d3’s colour schemes to transform numbers representing to turnout to an appropriate colour from the Viridis colour scheme. I inspected the values of turnout present in the election data in order to set bounds of domain so that as much of the colour scale could be used as possible.

We can open that uk_2017_turnour.svg file in a browser:

![UK 2017 Turnout Map](/img/portfolio/uk_maps/uk_2017_turnout.png)
*Figure 3: A green and pleasant land: turnout in the 2017 UK General Election.*

The legend was created separately in an Observable notebook and copy-pasted into the svg file.

## Adapting the approach to plot winners

Really my goal for this tutorial was to create a map that shows the winners in each constituency. To do this we rework the last step to use `ndjson-map` to look for the party with the highest vote share then setting the geometry’s fill property to a colour representing the party.

The Javascript I pass to `ndjson-map` feels a little more clunky here as I define an object that holds the lookup for the colour for each party.

    ndjson-map -r d3 's={"con_share":"#0087dc", "lib_share":"#FDBB30", "lab_share":"#d50000", "natSW_share":"#3F8428", "pld":"#3F8428", "oth_share":"#aaaaaa"}, u = ({ con_share, lab_share, lib_share, oth_share, natSW_share }) => ({ con_share, lab_share, lib_share, oth_share, natSW_share }), d.properties.fill = s[Object.keys(u(d.results)).reduce((a, b) => d.results[a] > d.results[b] ? a : b)], d' < uk_2017_results.ndjson \
    | geo2svg -n --stroke none -p 1 -w 960 -h 960 > uk_2017_winner.svg

Again, breaking that javascript down a bit, the first line:

    s={"con_share":"#0087dc", "lib_share":"#FDBB30", "lab_share":"#d50000", "natSW_share":"#3F8428", "pld":"#3F8428", "oth_share":"#aaaaaa"}

simply defines a lookup from party to colour. The second line:

    u = ({ con_share, lab_share, lib_share, oth_share, natSW_share }) => ({ con_share, lab_share, lib_share, oth_share, natSW_share })

Is a destructuring assignment which we use to extract the voteshare numbers from the election results object. Then the final line:

    d.properties.fill = s[Object.keys(u(d.results)).reduce((a, b) => d.results[a] > d.results[b] ? a : b)]

Finds the key of the party with the largest voteshare and uses this in the lookup object `s` to write the desired colour into the `fill` property.

Taking a look at the result:

![UK 2017 Winner Map](/img/portfolio/uk_maps/uk_2017_winner.png)
*Figure 4: 2017 General Election UK results constituencies coloured by winner.*

So there we have it. Again a legend was created separately and pasted in to the svg text.
Another day we can debate the merits of showing this data on a geographical map rather than, say, a cartogram that places equal area on the constituencies.

In the second tutorial in this series I look at how the data munging part of this tutorial might be more easily achieved using a Python/GeoPandas stack.

Thanks to @mbostock for all his tools and writings which are inspirational.