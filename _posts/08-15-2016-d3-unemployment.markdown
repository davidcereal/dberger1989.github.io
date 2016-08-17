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




<div class="circle_div"></div>


<script src="https://d3js.org/d3.v3.min.js"></script>
<script>

var svgContainer = d3.select(".circle_div").append("svg")
                                   .attr("width", 200)
                                     .attr("height", 200);


 var circle = svgContainer.append("circle")
                         .attr("cx", 30)
                       .attr("cy", 30)
                      .attr("r", 20);
