---
title: "D3 Unemployment U.S. Heatmap"
subtitle:
layout: post
date: 2016-06-26 22:48
image: /assets/images/markdown.jpg
headerImage: false
tag:
- distributed computing
- hadoop
- big data
- raspberry pi
blog: true
author: davidberger
description:    
---

<!-- Make sure you're in the directory of the project! -->


<h1 style="margin-left:180px;font-family: 'Helvetica Neue'">Annual Unemployment Rate by State</h1>

<div class="d3Div"></div>

<div id="slider" style="width:600px; margin-left:170px;"></div>
<div class="colorBar" style="width:500px; height:500px; margin-left:170px"></div>



  <!-- Add D3 and TopoJSON libraries. -->
<link rel="stylesheet" type="text/css" href="/assets/stylesheets/d3.slider.css" media="screen" />
<script src="/assets/javascripts/d3.v3.min.js"></script>
<script src="/assets/javascripts/d3.slider.js"></script>
 <script src="http://d3js.org/topojson.v1.min.js"></script>

  <!-- Add D3 Scripting here. -->
  <script>

d3.json("state_unemployment.json", function(root) {

console.log(root)
// tick formatter (since slider defaults to cama seperated thousands)
var formatter = d3.format();
var tickFormatter = function(d) {
return d;
}

// Initialize slider
var slider = d3.slider().min(2005).max(2015).tickValues([2005,2006,2007,2008,2009,2010,2011,2012,2013,2014, 2015]).stepValues([2005,2006,2007,2008,2009,2010,2011,2012,2013,2014, 2015]).showRange(true)
	.tickFormat(tickFormatter);

// Render the slider in the div
d3.select('#slider').call(slider);

var myFn = function(slider) {
	
	slide_value = slider.value()
	
	d3.selectAll('.states').style("fill", function(d) {


			    var fill = d3.scale.linear()
			    .domain([5, 7.5, 10])
			    .range(["#ffffd9", "#7fcdbb", '#253494']);

				console.log(this)
       			console.log('state name')
       			var state_name = d.id
       			return fill( root[state_name][slider.value()]);
       			
       		})
}



// Set slider callback function
slider.callback(myFn)

    

  // Use D3's JSON method to load in TopoJSON data.
  // Check out console to see what's in there!
 	d3.json("converted_states_with_tones.json", function(error, states) {
    if (error) {
      return console.error(error);
    } else {
      console.log(states);
    }

    // Add canvas.
    // Define width and height for SVG canvas.
    var width = 960;
    var height = 520;

    var fill = d3.scale.linear()
			    .domain([5, 7.5, 10])
			    .range(["#ffffd9", "#7fcdbb", '#081d58']);
    //.range(["steelblue", "brown"]);
    // Append SVG canvas to the DOM.
    var svg = d3.select(".d3Div")
                .append("svg")
                .attr("width", width)
                .attr("height", height);




    // Define states.
    // Assign the states variable to the GeoJSON feature collection for the specified topology object.
    // While TopoJSON data is stored more efficiently, we need to convert back to GeoJSON for display purposes.
    // Check out console to see your list of 51 (including DC) states!
    var states = topojson.feature(states, states.objects.states);
    console.log(states);

    // Create and append projection and paths.
    // Create a projection suited to fit the US (pre-defined in the library).
    // A projection simply describes how you want to view your specified area of the globe.  
    // Spherical coordinates are projected onto the Cartesian plane (our canvas).
    // Projections can be rotated, scaled, transformed, etc.
    // https://github.com/mbostock/d3/wiki/Geo-Projections
    var projection = d3.geo.albersUsa();
    
    // Create a path generator to draw lines around US, state borders.
    // Path generators take in a geometry/features object and create a path to be used for outline rendering.
    // Uses our previously-defined projection.
    // https://github.com/mbostock/d3/wiki/Geo-Paths
    var path = d3.geo.path()
                 .projection(projection);
    
    // Append the newly-created path generator to the map.
    svg.append("path")
       .datum(states)
       .attr("d", path);





    // Create state boundaries and coloring.
    // Create and select elements for each state.
    // The states.features data creates a specific path (boundary) for each state which is then appended.
    svg.selectAll('.states')
       .data(states.features)
       .enter()
       .append('path')  
       .attr('class', function(d) {
         return 'states' +' '+ d.id;
       })
       .attr('d', path)
       .style("stroke", "f2f2f2")
       // Add in random colors to see state borders.
       .style("fill", function(d) {


				console.log(this)
       			console.log('state name')
       			var state_name = d.id
       			return fill( root[state_name][slider.value()]);
       });






//Append a defs (for definition) element to your SVG
var defs = svg.append("defs")


//Append a linearGradient element to the defs and give it a unique id
var linearGradient = defs.append("linearGradient")
    .attr("id", "linear-gradient");

//Horizontal gradient
linearGradient
    .attr("x1", "0%")
    .attr("y1", "0%")
    .attr("x2", "0%")
    .attr("y2", "100%");

//Set the color for the start (0%)
//linearGradient.append("stop") 
    //.attr("offset", "0%")   
    //.attr("stop-color", "#ffffd9") 

//Set the color for the end (100%)
//linearGradient.append("stop") 
    //.attr("offset", "100%")   
    //.attr("stop-color", "#081d58"); //dark blue

//A color scale
var colorScale = d3.scale.linear()
    .range(["#ffffd9", "#7fcdbb", '#253494']);

//Append multiple color stops by using D3's data/enter step
linearGradient.selectAll("stop") 
    .data( colorScale.range() )                  
    .enter().append("stop")
    .attr("offset", function(d,i) { return i/(colorScale.range().length-1); })
    .attr("stop-color", function(d) { return d; });

d3.selectAll(".colorBar").append("svg")
	.attr("width", 20)
	.attr("height", 400)

//Draw the rectangle and fill with gradient
d3.selectAll(".colorBar").select("svg").append("rect")
	.attr("width", 20)
	.attr("height", 400)
	.attr("rx",0)  //rounded corners, if wanted
    .attr("ry",0)
	.style("fill", "url(#linear-gradient)")
	;

  });
 });
 
  </script>

