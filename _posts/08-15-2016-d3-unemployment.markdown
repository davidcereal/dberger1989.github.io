---
title: "d3 Mapping"
subtitle:
layout: post
date: 2016-06-26 22:48
image: /assets/images/markdown.jpg
headerImage: false
tag:
- d3

blog: true
author: davidberger
description:    
---




<h1 style="margin-left:150px;font-family: 'Helvetica Neue'">Annual Unemployment Rate by State (2005-2015)</h1>

<div class="d3Div" style=""></div>


<div id="slider" style="width:500px; margin-left:230px; margin-top:-50px"></div>



<link rel="stylesheet" type="text/css" href="/d3.slider.css" media="screen" />
<script src="https://d3js.org/d3.v3.min.js"></script>
<script src="/d3.slider.js"></script>
 <script src="http://d3js.org/topojson.v1.min.js"></script>
 <script src="https://d3js.org/d3-axis.v1.min.js"></script>


<script>

d3.json("state_unemployment.json", function(root) {

  var formatter = d3.format();
  var tickFormatter = function(d) {
    return d;
    }; 

  var slider = d3.slider().min(2005).max(2015).tickValues([2005,2006,2007,2008,2009,2010,2011,2012,2013,2014, 2015]).stepValues([2005,2006,2007,2008,2009,2010,2011,2012,2013,2014, 2015]).showRange(true)
    .tickFormat(tickFormatter);
  
  d3.select('#slider').call(slider);

  var myFn = function(slider) {
    slide_value = slider.value();
    d3.selectAll('.states').style("fill", function(d) {
          var fill = d3.scale.linear()
          	.domain([5, 7.5, 10])
          	.range(["#ffffd9", "#7fcdbb", '#253494']);
           var state_name = d.id;
           return fill( root[state_name][slider.value()]);
                });
    };



  slider.callback(myFn);

    

   d3.json("data/converted_states.json", function(error, states) {
    if (error) {
      return console.error(error);
    } else {
    console.log(states);
    }


  var width = 960;
  var height = 425;
  
  var fill = d3.scale.linear()
    .domain([5, 7.5, 10])
    .range(["#ffffd9", "#7fcdbb", '#081d58']);

  var svg = d3.select(".d3Div")
          .append("svg")
          .attr("width", width)
          .attr("height", height);
  
  
  
  
  var states = topojson.feature(states, states.objects.states);
  
  var projection = d3.geo.albersUsa()
          .scale(820);
  
  var path = d3.geo.path()
           .projection(projection);
  
  svg.append("path")
  .datum(states)
  .attr("d", path);
  
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
  .attr("width", 20)
  .attr("height", 400)
  .attr("rx",0) 
  .attr("ry",0)
  .style("fill", "url(#linear-gradient)")
  .attr("transform", "translate(855, 65)")
  ;
  
  var y = d3.scale.linear()
  .domain([5, 10])
  .range([0, 350]);
  
  var yAxis = d3.svg.axis()
    .scale(y)
    .orient("left");
  
  d3.select("svg").append("g")
  .attr("class", "y axis")
  .attr("transform", "translate(850, 70)")
  .call(yAxis)
	.append("text")
	.attr("transform", "translate(30, -30)")
	.attr("y", 9)
	.attr("dy", ".71em")
	.style("text-anchor", "end")
	.text("Unemployment Rate");


    });
});

</script>
