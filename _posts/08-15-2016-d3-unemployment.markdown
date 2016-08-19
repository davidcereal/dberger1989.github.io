---
title: "Interactive Maps With D3"
subtitle:
layout: post
date: 2016-08-15 22:48
image: /assets/images/markdown.jpg
headerImage: false
tag:
- d3
- Visualization

blog: true
author: davidberger
description:    
---
# D3 is Worth It

The D3.js visualization library is an incredible tool for creative data visualizations. The level of control and dynamism possible is simply unmatched. That being said, many data scientists I know find D3 too daunting a tool to learn, as the learning curve seems pretty steep and there are easier and more ubiquitous tools out there, such as Tableau, and when I first started attending [D3 meetups](http://www.meetup.com/NYC-D3-JS/) here in NYC, I was surprised at how small a proportion of the participants were data scientists. 

While I agree that there is a steep learning curve, D3 provides so much next-level creative power that I knew I would always be bothered until I learned it. I mean just *try* reading one of [Matthew Daniel's](http://polygraph.cool) posts and then tell me you can resist using it. Since you're already reading this article, I'm going to assume you can't. The bottom line is that D3 is a premium tool, and premium tools require effort. 

In this post, I’ll speak about one of the coolest and most powerful features of D3: the ability to project and draw geographical coordinates and manipulate their attribute and style in the DOM. To illustrate the principles involved, I’ve created a choropleth map of the the U.S. which depicts the unemployment rate in each state over the span of 2005-2015:

<style>

  .axis {
  shape-rendering: crispEdges;
}

.x.axis line {
  stroke: #fff;
}
.axis text {
	font-family: sans-serif;
	font-size: 11px;
}

.x.axis .minor {
  stroke-opacity: .5;
}

.x.axis path {
  display: none;
}

.y.axis line,
.y.axis path {
  fill: none;
  stroke: #000;
}

div.tooltip {	
    position: absolute;			
    text-align: center;			
    width: 60px;					
    height: 28px;					
    padding: 2px;				
    font: 12px sans-serif;		
    background: white;	
    border: 1px;		
    border-radius: 2px;			
    pointer-events: none;			
}

</style>

<h3 style="margin-left:80px;">Unemployment Rate by State (2005-2015)</h3>
<div class="mapContainer">
<div class="d3Div" style="margin-left:-240px"></div>
</div>

<div id="slider" style="width:500px; margin-left:35px; margin-top:0px"></div>



<link rel="stylesheet" type="text/css" href="/d3.slider.css" media="screen" />
<script src="https://d3js.org/d3.v3.min.js"></script>
<script src="/d3.slider.js"></script>
<script src="http://d3js.org/topojson.v1.min.js"></script>
<script src="https://d3js.org/d3-axis.v1.min.js"></script>


<script type="text/javascript" >

d3.json("/state_unemployment.json", function(root) {

  var formatter = d3.format();
  var tickFormatter = function(d) {
    return d;
    }; 

  var slider = d3.slider().min(2005).max(2015).tickValues([2005,2006,2007,2008,2009,2010,2011,2012,2013,2014, 2015]).stepValues([2005,2006,2007,2008,2009,2010,2011,2012,2013,2014, 2015]).showRange(true).value(2008)
    .tickFormat(tickFormatter);
  
  d3.select('#slider').call(slider);
	
var div = d3.select(".mapContainer").append("div")	
    .attr("class", "tooltip")				
    .style("opacity", 0);

  var myFn = function(slider) {
    slide_value = slider.value();
    d3.selectAll('.states').style("fill", function(d) {
          var fill = d3.scale.linear()
          	.domain([3, 7.5, 12])
          	.range(["#ffffd9", "#7fcdbb", '#253494']);
           var state_name = d.id;
           return fill( root[state_name][slider.value()]);
                })
           .on("mouseover", function(d) {
    	   		d3.select(this.parentNode.appendChild(this)).transition().duration(300)
           		.style({'stroke-opacity':1,'stroke':'yellow', 'stroke-width': 2});
            div.transition()		
                .duration(200)		
                .style("opacity", .8);	
            var state_name = d.id;	
            div	.html("<strong>" + d.id + "</strong>" + "<br/>" + root[state_name][slider.value()])	
                .style("left", (d3.event.pageX) + "px")		
                .style("top", (d3.event.pageY - 48) + "px");	
            	})					
        	.on("mouseout", function(d) {
        		d3.select(this).transition().duration(300)
        		.style({'stroke-opacity':1,'stroke':'white'});
	            div.transition()		
	                .duration(500)		
	                .style("opacity", 0);
	        	});
    };



  slider.callback(myFn);

    

   d3.json("/converted_states.json", function(error, states) {
    if (error) {
      return console.error(error);
    } else {
    console.log(states);
    }


  var width = 960;
  var height = 520;
  
  var fill = d3.scale.linear()
    .domain([5, 7.5, 10])
    .range(["#ffffd9", "#7fcdbb", '#081d58']);

  var svg = d3.select(".d3Div")
          .append("svg")
          .attr("width", width)
          .attr("height", height);
  
  
  
  
  var states = topojson.feature(states, states.objects.states);
  
  var projection = d3.geo.albersUsa()
          .scale(1000);
  
  var path = d3.geo.path()
           .projection(projection);


  svg.selectAll('.states')
    .data(states.features)
    .enter()
    .append('path')  
    .attr('class', function(d) {
      return 'states' +' '+ d.id;
      })
    .attr('d', path)
    .style("stroke", "f2f2f2")
    .style("fill", function(d) {
            var state_name = d.id;
            return fill( root[state_name][slider.value()]);
      })
	.on("mouseover", function(d) {
   		d3.select(this.parentNode.appendChild(this)).transition().duration(300)
   		.style({'stroke-opacity':1,'stroke':'yellow', 'stroke-width': 2});
    div.transition()		
        .duration(200)		
        .style("opacity", .8);	
    var state_name = d.id;	
    div	.html("<strong>" + d.id + "</strong>" + "<br/>" + root[state_name][slider.value()])	
        .style("left", (d3.event.pageX) + "px")		
        .style("top", (d3.event.pageY - 48) + "px");	
    	})					
	.on("mouseout", function(d) {
		d3.select(this).transition().duration(300)
		.style({'stroke-opacity':1,'stroke':'white'});
        div.transition()		
            .duration(500)		
            .style("opacity", 0);
    	});
  
  
  
  
  var defs = svg.append("defs");
  
  
  var linearGradient = defs.append("linearGradient")
  .attr("id", "linear-gradient");
  
  linearGradient
  .attr("x1", "0%")
  .attr("y1", "0%")
  .attr("x2", "0%")
  .attr("y2", "100%");
  
  
  var colorScale = d3.scale.linear()
  .range(["#ffffd9", "#7fcdbb", '#253494']);
  
  linearGradient.selectAll("stop") 
  .data( colorScale.range() )                  
  .enter().append("stop")
  .attr("offset", function(d,i) { return i/(colorScale.range().length-1); })
  .attr("stop-color", function(d) { return d; });
  
  
  svg.append("rect")
  .attr("width", 15)
  .attr("height", 400)
  .attr("rx",0) 
  .attr("ry",0)
  .style("fill", "url(#linear-gradient)")
  .attr("transform", "translate(905, 70)")
  ;
  
  var y = d3.scale.linear()
  .domain([3, 12])
  .range([0, 400]);
  
  var yAxis = d3.svg.axis()
    .scale(y)
    .orient("left");
  
  d3.select("svg").append("g")
  .attr("class", "y axis")
  .attr("transform", "translate(900, 70)")
  .call(yAxis)
	.append("text")
	.attr("transform", "translate(60, -30)")
	.attr("y", 9)
	.attr("dy", ".71em")
	.style("text-anchor", "end")
	.text("Unemployment Rate");


    });
});

</script>

# GeoJSON and TopoJSON
GeoJSON is a format specifically designed to combine geo-coordinates with JSON format. TopoJSON is also a JSON mapping format, and it improves on geoJSON in a number of areas, one being that in topoJSON,  segmented areas that share a border (such as bordering states), don’t have a redundant path drawn, and another being that it allows for the mapping of topography. TopoJSON is especially popular because it’s file sizes are much smaller than those of geoJSON files, and thus much quicker to load. Let’s explore the contents of our topoJSON file:

```
{“objects":
	 {“states":
		 {"type": “GeometryCollection",
		 "geometries": [
				{"type": "MultiPolygon", 
				"id": “Alaska",
				 "arcs": [[[0]], [[1]], [[2]], [[3]], [[4]],
				...
				...	
				{"type": "Polygon", 
				"id": "Wyoming", "arcs": [[-228, -93, -272, -136, -212, -264]]}]}}, 					
		"type": "Topology", 
			"transform": {
				        "translate": [-178.19451843993755, 18.96390918584938], 					       
				        "scale": [0.011121861824577767, 0.005244902253759074]}, 					         
				        "arcs": [[[172, 6259], [-6, -11], [-4, 5]
				 ...

```
Here we see that our objects are states, and that each state has a `type`, an `id` with which we will use to reference it, and a series of `arc` values. The `states` object goes on like that for all 48 states, and after the last state, we see a new `type`, `Topology`, which contains the information necessary to draw each of the states.

You’ll notice that `topology` has a list of arcs just as there was in each of the state entries, but these are different. The state arcs were actually just pointers referencing the appropriate index position in the `topology` set of arcs. You may be wondering why there are negative numbers in the `state` arcs arrays, as there are in Wyoming’s, if they are references to index positions. The answer lies in topoJSON’s avoidance of redundant paths. When a pointer is negative, it means that a reference to the corresponding arc has already been made, and thus indicates not to draw it a second time, which would be a waste.

So why bring up geoJSON at all? While storing geographical data is much more efficient with topoJSON, we still need to project that data from it’s sphere-based coordinates onto a 2-dimensional plane, our DOM. To do this, we need to convert the points to geoJSON format using a specific projection style of our choosing. 

This may seem like a lot to take in. Let’s see the code necessary to make it all happen and break it down piece by piece:

```js
// Load in topoJSON data
   d3.json("converted_states.json", function(error, states) {
    if (error) {
      return console.error(error);
    } 
    }

// Define states
  var states = topoJSON.feature(states, states.objects.states);
  
  // Define projection
  var projection = d3.geo.azimuthalEqualArea()
  
  // Define path generator based on projection
  var path = d3.geo.path()
           .projection(projection);
  
  // Draw states
  // Select (create) elements with class of ‘.state’
  svg.selectAll('.states')
	// bind the data
  	.data(states.features)
    	// Append the path generator 
	.append('path')  
	// define the class of each .state element to include ‘states’ and the state name
    	.attr('class', function(d) {
      return 'states' +' '+ d.id;
     })
// For each of the data points bound from the topoJSON file, apply the 
// geo projection path generator to draw the state
.attr('d', path)
// Set the stroke color for each state
.style("stroke", "#f2f2f2") // light grey
```

First, we load the topoJSON file in. Nothing too complicated about that. Then, we define the projection we will be using to translate the topoJSON data. Let’s expand on the concept of a projection. In the visualization provided above, we took spherical (earth) based coordinates and translated them into a perfectly flat 2 dimensional drawing. That action happened because we used the albersUSA projection, which does just that and also places Alaska and Hawaii at the bottom of the map and scales down Alaska. Those are the instructions provided by the albersUSA projection. If we would have used a different projection, we would have gotten a totally different depiction of the states defined in the topoJSON. You can see d3’s built-in geo projections [here](https://github.com/d3/d3-geo-projection). We might, for example, have used the `geo.azimuthalEqualArea()` projection instead. The library defines that projection as depicting a spherical looking rendering of the planet:

<img src ="/assets/images/post_images/d3_map_post/azimuthalEqualArea.svg" style="width:400px"/>


Thus, if we apply the azimuthal equal area projection to our data set, we see our states projected onto the spherical layout, which I find pretty incredible. I don't know why, but I got immense satisfaction from playing around with the slider and hover functions and seeing that they still worked with the new projection: 

<style>
.axis {
  shape-rendering: crispEdges;
}

.x.axis line {
  stroke: #fff;
}
.axis text {
	font-family: sans-serif;
	font-size: 11px;
}

.x.axis .minor {
  stroke-opacity: .5;
}

.x.axis path {
  display: none;
}

.y.axis line,
.y.axis path {
  fill: none;
  stroke: #000;
}

div.tooltip {	
    position: absolute;			
    text-align: center;			
    width: 60px;					
    height: 28px;					
    padding: 2px;				
    font: 12px sans-serif;		
    background: white;	
    border: 1px;		
    border-radius: 2px;			
    pointer-events: none;			
}

</style>



<div class="mapContainer_1">
<div class="d3Div_1" style="margin-left:-240px"></div>
</div>

<div id="slider_1" style="width:500px; margin-left:35px; margin-top:0px"></div>




<script type="text/javascript" >

d3.json("/state_unemployment.json", function(root) {

  var formatter = d3.format();
  var tickFormatter = function(d) {
    return d;
    }; 

  var slider = d3.slider().min(2005).max(2015).tickValues([2005,2006,2007,2008,2009,2010,2011,2012,2013,2014, 2015]).stepValues([2005,2006,2007,2008,2009,2010,2011,2012,2013,2014, 2015]).showRange(true).value(2008)
    .tickFormat(tickFormatter);
  
  d3.select('#slider_1').call(slider);
	
var div = d3.select(".mapContainer_1").append("div")	
    .attr("class", "tooltip")				
    .style("opacity", 0);

  var myFn = function(slider) {
    slide_value = slider.value();
    d3.selectAll('.states').style("fill", function(d) {
          var fill = d3.scale.linear()
          	.domain([3, 7.5, 12])
          	.range(["#ffffd9", "#7fcdbb", '#253494']);
           var state_name = d.id;
           return fill( root[state_name][slider.value()]);
                })
           .on("mouseover", function(d) {
    	   		d3.select(this.parentNode.appendChild(this)).transition().duration(300)
           		.style({'stroke-opacity':1,'stroke':'yellow', 'stroke-width': 2});
            div.transition()		
                .duration(200)		
                .style("opacity", .8);	
            var state_name = d.id;	
            div	.html("<strong>" + d.id + "</strong>" + "<br/>" + root[state_name][slider.value()])	
                .style("left", (d3.event.pageX) + "px")		
                .style("top", (d3.event.pageY - 48) + "px");	
            	})					
        	.on("mouseout", function(d) {
        		d3.select(this).transition().duration(300)
        		.style({'stroke-opacity':1,'stroke':'white'});
	            div.transition()		
	                .duration(500)		
	                .style("opacity", 0);
	        	});
    };



  slider.callback(myFn);

    

   d3.json("/converted_states.json", function(error, states) {
    if (error) {
      return console.error(error);
    } else {
    console.log(states);
    }


  var width = 960;
  var height = 520;
  
  var fill = d3.scale.linear()
    .domain([5, 7.5, 10])
    .range(["#ffffd9", "#7fcdbb", '#081d58']);

  var svg = d3.select(".d3Div_1")
          .append("svg")
          .attr("width", width)
          .attr("height", height);
  
  
  
  
  var states = topojson.feature(states, states.objects.states);
  
  var projection = d3.geo.azimuthalEqualArea();
  
  var path = d3.geo.path()
           .projection(projection);
  

  svg.selectAll('.states')
    .data(states.features)
    .enter()
    .append('path')  
    .attr('class', function(d) {
      return 'states' +' '+ d.id;
      })
    .attr('d', path)
    .style("stroke", "f2f2f2")
    .style("fill", function(d) {
            var state_name = d.id;
            return fill( root[state_name][slider.value()]);
      })
	.on("mouseover", function(d) {
   		d3.select(this.parentNode.appendChild(this)).transition().duration(300)
   		.style({'stroke-opacity':1,'stroke':'yellow', 'stroke-width': 2});
    div.transition()		
        .duration(200)		
        .style("opacity", .8);	
    var state_name = d.id;	
    div	.html("<strong>" + d.id + "</strong>" + "<br/>" + root[state_name][slider.value()])	
        .style("left", (d3.event.pageX) + "px")		
        .style("top", (d3.event.pageY - 48) + "px");	
    	})					
	.on("mouseout", function(d) {
		d3.select(this).transition().duration(300)
		.style({'stroke-opacity':1,'stroke':'white'});
        div.transition()		
            .duration(500)		
            .style("opacity", 0);
    	});
  
  
  
  
  var defs = svg.append("defs");
  
  
  var linearGradient = defs.append("linearGradient")
  .attr("id", "linear-gradient");
  
  linearGradient
  .attr("x1", "0%")
  .attr("y1", "0%")
  .attr("x2", "0%")
  .attr("y2", "100%");
  
  
  var colorScale = d3.scale.linear()
  .range(["#ffffd9", "#7fcdbb", '#253494']);
  
  linearGradient.selectAll("stop") 
  .data( colorScale.range() )                  
  .enter().append("stop")
  .attr("offset", function(d,i) { return i/(colorScale.range().length-1); })
  .attr("stop-color", function(d) { return d; });
  
  
  svg.append("rect")
  .attr("width", 15)
  .attr("height", 400)
  .attr("rx",0) 
  .attr("ry",0)
  .style("fill", "url(#linear-gradient)")
  .attr("transform", "translate(905, 70)")
  ;
  
  var y = d3.scale.linear()
  .domain([3, 12])
  .range([0, 400]);
  
  var yAxis = d3.svg.axis()
    .scale(y)
    .orient("left");
  
 svg.append("g")
  .attr("class", "y axis")
  .attr("transform", "translate(900, 70)")
  .call(yAxis)
	.append("text")
	.attr("transform", "translate(60, -30)")
	.attr("y", 9)
	.attr("dy", ".71em")
	.style("text-anchor", "end")
	.text("Unemployment Rate");


    });
});

</script>


Back to the code walkthrough. Now that we’ve defined the type of projector we’re going to be using, we define a variable, `path`, to be a geo path generator that takes a path and projects it using a user-defined projection (in our case, we provide it with the albersUSA projection. 

Next, we bind the data to `.states` elements, and for each element, we pass the data bound to it to the path generator and project the mapping onto the SVG. We then give each state element a specific class name so we can apply custom fills to each one. Now that we’ve drawn the state, we can now apply any attributes, functions, and styles to it as we would any other shape, such as stroke, which I define to be light grey, and now we have a light grey border around each of the states. 


# Go Map
So there you have it! With the magic of D3 and just a few lines of code, we can project and draw any and as many locations as we can find topoJSON coordinates for, and then do as much cool stuff with them as you could with any other shape in D3, which is to say a hell of a lot. Map away.
