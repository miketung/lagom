---
layout: default 
title: Hierarchical Clustering
categories:
- blog
---
<p class="meta">
  <a href="/">
    <i class="home fa fa-home"></i>
  </a>
</p>
<h1 class="title">{{ page.title }}</h1>
<link rel="stylesheet" type="text/css" href="{{ "/assets/css/jquery-ui-1.8.custom.css" | relative_url }}" />

<div id="Inner">
  <style>
    #Controls {font-size: 14px; }
    .slider {float: left; width: 300px; }
    .end-num {float:left; margin-left: 12px; width: 32px;  }
    .slider-row {padding: 5px 0px; }
    #Graphs div {float:left; }
  </style>
  <div id="Controls">
    <div class="slider-row" >
      <div id="Slider1" class="slider"></div><div class="end-num" id="k">100</div>
      <div style="clear:both" />
    </div>
    <div>Number of clusters</div>
    <div class="slider-row" >
      <div id="Slider2" class="slider"></div><div class="end-num" id="p">1</div>
      <div style="clear:both" />
    </div>
    <div>Noise level</div>
  </div><!-- Controls -->
  <div id="Graphs">
    <div>
    <canvas id="Graph1" width="300" height="300" ></canvas>
    <br />
    <div>Distances</div>
    </div>
    <div>
    <canvas id="Graph2" width="300" height="300" ></canvas>
    <br />
    <div>Greedy linkage</div>
    </div>
    <div>
    <canvas id="Graph21" width="300" height="300" ></canvas>
    <br />
    <div>Single linkage</div>
    </div>
    <div>
    <canvas id="Graph3" width="300" height="300" ></canvas>
    <br />
    <div>Complete linkage</div>
    </div>
    <div>
    <canvas id="Graph4" width="300" height="300" ></canvas>
    <br />
    <div>Average linkage</div>
    </div>
  </div>
<script type="text/javascript" src="{{ "/assets/js/jquery-1.4.2.min.js" | relative_url }}"></script>
<script type="text/javascript" src="{{ "/assets/js/jquery-ui-1.8.custom.min.js" | relative_url }}"></script>
<script type="text/javascript" src="{{ "/assets/js/jquery.ui.touch-punch.min.js" | relative_url }}"></script>
<script type="text/javascript" src="{{ "/assets/js/randomColor.js" | relative_url }}"></script>
<script type="text/javascript" >

var MAX_K = 20;
var N = 100;
var colours = randomColor({count:MAX_K});

jQuery(document).ready(function($){

	/* Generate Nice jQuery UI Sliders */
	$('#Slider1, #Slider2').slider({
		slide: function(event,ui){ doCluster();}
	});

	/* Set some random initial values for the sliders */
  $('#Slider1').slider('value', 25);
  $('#Slider2').slider('value', 0);
  
  $('#Graph1,#Graph2,#Graph21,#Graph3,#Graph4').mouseover(function(){ drawClusters2(JSON.parse($(this).attr('data')), this, true);}).mouseout(function(){ drawClusters2(JSON.parse($(this).attr('data')), this, false);});

	doCluster();
});

var lastK = null;
var lastPoints = null;

var doCluster = function(){
  var k = Math.floor($('#Slider1').slider('value')*MAX_K/100)+1; // range 1-MAX_K
  var p = (Math.round($('#Slider2').slider('value')*20/100)) / 20; // range 0-1
  $('#k').text(k); 
  $('#p').text(p); 

  // generate a random cluster assignment for 100 points
  var points = [];
  if(k==lastK && lastPoints)
    points = lastPoints;
  else {
    for(var i=0;i<N;i++) points[i] = Math.floor(Math.random() * k);
    lastPoints=points;
    lastK=k;
  }
  var distM = [];
  for(var i=0;i<N;i++){
    distM.push([]);
    for(var j=0;j<N;j++){
      var x = points[i]==points[j] ? 0 : 1; 
      if (1-p > Math.random())
        distM[i][j] = x;
      else
        distM[i][j] = Math.random();
      //distM[i][j] = (1-p) * x + p * Math.random();
    }
  }

  // graph distance matrix
  var canvas = document.getElementById("Graph1");
  clearCanvas(canvas);
  var ctx = canvas.getContext("2d");
  var idx = [];
  for(var i=0;i<N;i++) idx[i] = i;
  var sorted = idx.sort(function(a,b){
    if(points[a] < points[b]) return -1;
    else if (points[a] > points[b]) return 1;
    else return 0;
  });
  for(var i=0;i<N;i++){
    for(var j=0;j<N;j++){
      var x = Math.floor(distM[sorted[i]][sorted[j]] * 255);
      ctx.fillStyle = 'rgb('+x+','+x+','+x+')';
      ctx.fillRect(i*3,j*3, i*3+3, j*3+3);
    }
  }

  // compute greedy-linkage with 0.5 threshold
  var clusters = initClusters(points);
  var changed = false;
  do {
    //console.log(JSON.stringify(clusters));
    changed = false;
    var minDist = 1;
    outer:
    for(var i=0;i<clusters.length;i++){
      for(var j=i+1;j<clusters.length;j++){
        var c1 = clusters[i], c2 = clusters[j];
        compareCluster:
        for(var u=0;u<c1.length;u++){
          for(var v=0;v<c2.length;v++){
            if(distM[c1[u]][c2[v]] < 0.5) {
              // merge i,j clusters
              clusters[i] = clusters[i].concat(clusters[j])
              clusters.splice(j, 1);
              changed = true;
              break outer;
            }
          }
        }
        if(changed) break;
      }
      if(changed) break;
    }
  }while(changed);
  drawClusters(points, clusters, "Graph2");

  // single-linkage
  var clusters = initClusters(points);
  var changed = false;
  do {
    //console.log(JSON.stringify(clusters));
    changed = false;
    var minDist = 1;
    var pair = null;
    for(var i=0;i<clusters.length;i++){
      for(var j=i+1;j<clusters.length;j++){
        var c1 = clusters[i], c2 = clusters[j];
        compareCluster:
        var min = 1;
        for(var u=0;u<c1.length;u++){
          for(var v=0;v<c2.length;v++){
            if(distM[c1[u]][c2[v]] < min)
              min = distM[c1[u]][c2[v]];
          }
        }
        if (min < minDist){
          minDist = min;
          pair = [i,j];
        }
      }
    }
    if(minDist < 0.5){
      var i = pair[0], j = pair[1]; 
      clusters[i] = clusters[i].concat(clusters[j])
      clusters.splice(j, 1);
    } else break;
  }while(true);
  drawClusters(points, clusters, "Graph21");

  // complete-linkage
  var clusters = initClusters(points);
  var changed = false;
  do {
    //console.log(JSON.stringify(clusters));
    changed = false;
    var minDist = 1;
    var pair = null;
    for(var i=0;i<clusters.length;i++){
      for(var j=i+1;j<clusters.length;j++){
        var c1 = clusters[i], c2 = clusters[j];
        compareCluster:
        var max = 0;
        for(var u=0;u<c1.length;u++){
          for(var v=0;v<c2.length;v++){
            if(distM[c1[u]][c2[v]] > max)
              max = distM[c1[u]][c2[v]];
          }
        }
        if (max < minDist){
          minDist = max;
          pair = [i,j];
        }
      }
    }
    if(minDist < 0.5){
      var i = pair[0], j = pair[1]; 
      clusters[i] = clusters[i].concat(clusters[j])
      clusters.splice(j, 1);
    } else break;
  }while(true);
  drawClusters(points, clusters, "Graph3");

  // average-linkage
  var clusters = initClusters(points);
  var changed = false;
  do {
    //console.log(JSON.stringify(clusters));
    changed = false;
    var minDist = 1;
    var pair = null;
    for(var i=0;i<clusters.length;i++){
      for(var j=i+1;j<clusters.length;j++){
        var c1 = clusters[i], c2 = clusters[j];
        compareCluster:
        var sum = 0;
        for(var u=0;u<c1.length;u++){
          for(var v=0;v<c2.length;v++){
            sum += distM[c1[u]][c2[v]]; 
          }
        }
        var avg = sum / (c1.length * c2.length);
        if (avg < minDist){
          minDist = avg;
          pair = [i,j];
        }
      }
    }
    if(minDist < 0.5){
      var i = pair[0], j = pair[1]; 
      clusters[i] = clusters[i].concat(clusters[j])
      clusters.splice(j, 1);
    } else break;
  }while(true);
  drawClusters(points, clusters, "Graph4");
}

var initClusters = function(points){ // each point is a cluster
  var clusters = [];
  for(var i=0;i<points.length;i++){
    clusters.push([i]);
  }
  return clusters;
}

var clearCanvas = function(element){
  $(element).attr('width',$(element).attr('width'));
}

var drawClusters = function(points, clusters, id, showtext){
  var fl = clusters.reduce(function(a, b) {
  return a.concat(b);
  }, []); 
  var canvas = document.getElementById(id);
  // stash flattened in the dom
  var flattened = [];
  for(var i=0;i<fl.length; i++){
    flattened[i] = points[fl[i]];
  }
  canvas.setAttribute('data', JSON.stringify(flattened));
  drawClusters2(flattened, canvas, showtext);
}
var drawClusters2 = function(flattened, canvas, showtext){
  clearCanvas(canvas);
  var ctx = canvas.getContext("2d");
  ctx.font = "bold 16px 'Open Sans', Arial, sans-serif";
  var idx=0;
  for(var i=0;i<10;i++){
    for(var j=0;j<10;j++){
      ctx.fillStyle = colours[flattened[idx]];
      ctx.fillRect(j*30, i*30, j*30 + 30, i*30+30);
      if(showtext){
        ctx.fillStyle = 'white';
        ctx.fillText(flattened[idx], j*30+8, i*30+20);
      }
      idx++;
    }
  }
}
</script>
</div>
