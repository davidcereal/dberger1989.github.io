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

<iframe src="http://bl.ocks.org/dberger1989/raw/9196827dee69b39b669fb7e1904b812d/" marginwidth="0" marginheight="0" style="height:500px;width=560px">
</iframe>


<script src="https://d3js.org/d3.v3.min.js"></script>
<script type="text/javascript">
//
var svgContainer = d3.select("body").append("svg")
                                   .attr("width", 200)
                                     .attr("height", 200);

 //Draw the Circle
 var circle = svgContainer.append("circle")
                         .attr("cx", 30)
                       .attr("cy", 30)
                      .attr("r", 20);


</script>
