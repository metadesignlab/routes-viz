<template>
  <div id="app" class="p-4" >
    <div class='inline-flex items-center bg-white w-full'>
      <div class='flex-col mx-2'>
       <div class='pr-2 font-bold text-xl'> Active: {{activeScale}}</div>
       <div class='text-tiny'> {{description[activeScale]}} </div>
      </div>

      <div class="slidecontainer mx-1">
        Show labels if mention in cases is above <b>{{sliderValue}}</b>
        <input type="range" min="100" max="2500" v-model="sliderValue" class="slider" id="myRange">
      </div>

      <button v-for="item in btns" 
        :key="`btn-`+item"
        type="submit" 
        :value="item" 
        v-on:click="activeScale = item; changeColors();"
        class='p-2 mx-3 rounded-full bg-blue-400 text-white font-bold hover:bg-blue-800'> 
        {{item}} 
      </button>
    </div>
    <div id="chart" class="justify-center flex my-1 absolute border border-gray-500"> 
    </div>
  </div>
</template>

<script>
import * as d3 from "d3";
import { json, csv } from 'd3-fetch'
//'../assets/js/mykey.js'
// DOMTokenList.prototype.indexOf = Array.prototype.indexOf;

export default {
  name: "network",
  props: {
    jsonURL: {
      type: String,
      default: () => 'routes_degree.json'
    },

    linkDistance: {
      type: Number,
      default: 30
    },
    // svg
    svgSize: {
      type: Object,
      default: () => {
        return {
          width: window.innerWidth,
          height: window.innerHeight
        };
      }
    },

    bodyStrength: {
      type: Number,
      default: -200
    },

    height: {
      type: Number,
      default: 800
    },
    
    width: {
      type: Number,
      default: 1400
    },

    sliderValue: {
      type: Number,
      default: 100
    },

    margin: {
      type: Object,
      default: function () {
        return {
          top: 5,
          right: 5,
          bottom: 5,
          left: 5
        }
      }
    } 
  },
  data() {
    return {
      bbBoxMargin: 20,
      selection: {
        links: [],
        nodes: []
      },
      pinned: [], 
      nodeList: [],
      linkList: [],
      force: null,
      zoom: d3.zoom(),
      transitionSpeed: 200,
      curves: true,
      stepper: 0,
      linkStroke: 'gray',
      btns: ['d_centrality', 'c_centrality', 'b_centrality'],
      c_centrality: [],
      b_centrality: [],
      d_centrality: [],
      radiusRange: [1, 5],
      fontsize:16, 
      activeScale: 'd_centrality',
      description: {'d_centrality': ' Degree centrality, which is defined as the number of links incident upon a node (i.e., the number of ties that a node has). The degree can be interpreted in terms of the immediate risk of a node for catching whatever is flowing through the network (such as a virus, or some information).',
        'c_centrality': 'the normalized closeness centrality (or closeness) of a node is the average length of the shortest path between the node and all other nodes in the graph. Thus the more central a node is, the closer it is to all other nodes.',
        'b_centrality': 'Betweenness centrality quantifies the number of times a node acts as a bridge along the shortest path between two other nodes. It was introduced as a measure for quantifying the control of a human on the communication between other humans in a social network'}
    };
  },
  computed: {
    nodes() {
      let nodes = this.nodeList.slice();
      let nodeIds = [];
      nodes = nodes.filter(node => {
        if (nodeIds.indexOf(node.id) === -1) {
          nodeIds.push(node.id);
          return true;
        } else {
          return false;
        }
      });
      return nodes;
    },

    links() {
      return this.linkList;
    },

    activeDomain() {
      if (this.activeScale === 'd_centrality') {
        return this.d_centrality
      } else if (this.activeScale === 'b_centrality') {
        return this.b_centrality
      } else if (this.activeScale === 'c_centrality'){
        return this.c_centrality
      }
    },

    centralityScale() {
      return d3.scaleLinear().domain(this.activeDomain).range(['white', 'steelblue'])
    } 
  },
  watch: {
    sliderValue: function() {
      this.svg.selectAll('.node-text')
        .transition().duration(this.transitionSpeed)
        .attr('opacity', d => d.occurence > this.sliderValue ? 1 : 0);
      this.$forceUpdate();
    },
    bodyStrength: function() {
      this.initData();
      this.$nextTick(function() {
        this.initDragTickZoom();
      });
    },
    linkDistance: function() {
      this.initData();
      this.$nextTick(function() {
        this.initDragTickZoom();
      });
    },
    nodes: function() {
      this.initData();
      this.$nextTick(function() {
        this.initDragTickZoom();
      });
    }
  },

  created() {
    this.initData();
  },

  mounted() {
    this.getData()
    this.initDragTickZoom();
  },

  methods: {
    changeColors(){
      let nodes = this.svg.selectAll('.node').transition().duration(this.transitionSpeed)
        .attr('fill', (d) => {console.log(d); return this.centralityScale(d[this.activeScale]);} )
    },

    update(){
      d3.select('#chart').select('svg').remove()
      this.svg = this.getSvg()
      this.drawChart()
    },

    getData () {
      if (!this.jsonURL) {
        throw new Error('Please provide either data or jsonURL')
      } else {
        this.processJSON(this.jsonURL)
      }
    },

    getString(row) {
      let label = row['country'].includes('mean') ? 'Average value for ' + row['element'] : row['country'] 
      let value = label + ': ' + (row[this.activeIndex] ? Math.round(row[this.activeIndex] * 10) / 10 : 'NIL')
      return value
    },

    processJSON (url) {
      json(url).then((data) => {
        this.linkList = data.links
        this.nodeList = data.nodes

        this.nodeList.forEach( node=> {
          this.c_centrality.push(node.c_centrality)
          this.b_centrality.push(node.b_centrality)
          this.d_centrality.push(node.d_centrality)
        })

        this.c_centrality = [0, d3.extent(this.c_centrality)[1]]
        this.b_centrality = [0, d3.extent(this.b_centrality)[1]]
        this.d_centrality = [0, d3.extent(this.d_centrality)[1]]
        this.mount = true
        this.update()
      })
    },

    getSvg () {
      const { svgWidth, svgHeight } = this
      const { margin } = this

      this.svgWidth = this.width
      this.svgHeight = this.height
      const width = this.svgWidth + margin.left + margin.right
      const height = this.svgHeight + margin.top + margin.bottom

      let svg = d3.select("#chart").append("svg")
              .attr("width", width)
              .attr("height", height)
              .attr("id", "container");
              //.call(this.responsivefy);

      return svg
    },

    drawChart() {
      let g = this.svg.append('g')

      let edges 
      if (this.curves) {
        edges = g.append("g")
          .selectAll(".link")
          .data(this.links)
          .join("path")
          .attr("opacity", 0.2)
          .attr("stroke", this.linkStroke)
          .attr("fill", "transparent")
          .attr('class', function(d) {
          
            let source = d.source.split(' ').join('').replace(/[^\w\s]|_/g, "");
            let target = d.target.split(' ').join('').replace(/[^\w\s]|_/g, "");
            return 'link element ' + source + ' ' + target;
            })
          .attr('stroke-linecap', 'round')
          .attr('stroke-width', function (d) {
            return d3.scaleLinear().domain([5, 10]).range([1, 20])(d.value);
          });
      } else {
        edges = g.append("g")
        .selectAll(".link")
        .data(this.links)
        .enter().append("line")
        .attr('class', function(d) {
          let source = d.source.split(' ').join('').replace(/[^\w\s]|_/g, "");
          let target = d.target.split(' ').join('').replace(/[^\w\s]|_/g, "");
          return 'link element ' + source + ' ' + target;
          })
        .attr('opacity', 0.2)
        .attr('stroke-linecap', 'round')
        .attr('stroke', this.linkStroke)
        .attr('stroke-width', function (d) {
            return d3.scaleLinear().domain([1, 30]).range([1, 30])(d.value);
          });
      }

      let nodes = g.append("g")
        .selectAll(".node")
        .data(this.nodes)
        .enter().append('circle')
        .attr('id', (d) => d.index)
        .attr('class', function(d) {
          let nodeClass=''
          return 'node element ' + d.id.split(' ').join('').replace(/[^\w\s]|_/g, "");
          })
        .attr('fill', (d) => this.centralityScale(d[this.activeScale]))
        .attr('stroke', (d) => d.child === true ? 'goldenrod' : 'black')
        .attr('stroke-width', 2)
        .attr('r', (d) => d3.scaleLinear().domain([1, 100]).range(this.radiusRange)(d.occurence))
        .on('mouseover', (d, e, nodes) => this.nodeMouseover(d, e, nodes))
        .on('mouseout', (d, e, nodes) => this.nodeMouseout(d, e, nodes));

      let blackOut = this.blackOut
      
      //Add single line text
      g.append("g")
        .selectAll(".text")
        .data(this.nodes)
        .enter()
        .append('text')
        .attr('class', (d) => {
          if (d.occurence > this.sliderValue) {
            return 'emph node-text ' + d.id.split(' ').join('').replace(/[^\w\s]|_/g, "");
          }

          return 'node-text ' + d.id.split(' ').join('').replace(/[^\w\s]|_/g, "");
        })
        .text((d) => d.id)
        .attr('fill', 'black')
        .attr('dx', 0)
        .attr('alignment-baseline', 'text-top')
        .attr('text-anchor', 'middle')
        .attr('font-size',this.fontsize)
        .attr('opacity', (d) => d.occurence > this.sliderValue ? 1 : 0)
        .on('mouseover', (d, e, nodes) => this.nodeMouseover(d, e, nodes))
        .on('mouseout', (d, e, nodes) => this.nodeMouseout(d, e, nodes));

      this.initDragTickZoom()

      let zoomLvl = 0.5;
      let lastK = 0;
      
      this.svg.call(d3.zoom()
      .extent([[0, 0], [this.svgWidth, this.svgHeight]])
      .scaleExtent([0.25, 80])
      .on("zoom", zoomed));

      function zoomed() {
        let e = d3.event
        
        if(e.transform.k > 2 && lastK != e.transform.k){
          lastK = e.transform.k;
          // console.log("zoomed");
          zoomLvl =Math.log2(e.transform.k);
          nodes.attr("stroke-width", 1.5/zoomLvl );
          edges.attr("stroke-width",  d => Math.sqrt(d.value)/(zoomLvl));
        }
      
        g.attr("transform", e.transform);
      }
    },

    nodeMouseover(d, e, nodes) {
      let targetNode = nodes[e]
      this.activeNode = d.id
      this.highlightNode(this.activeNode, true, true, true);
      this.$forceUpdate();
    },

    highlightNode(nodeName, highlightLinks, highlightOthers, decreaseOpacity){
      let nodeClass = []
      
      if (Array.isArray(nodeName)) {
        for (let i in nodeName){
          nodeClass.push(nodeName[i].split(' ').join('').replace(/[^\w\s]|_/g, ""))
        }
      } else {
        nodeClass = nodeName.split(' ').join('').replace(/[^\w\s]|_/g, "");
      }

      if (decreaseOpacity) {
        let opacity = 0.01;

        // Decrease opacity for all irrelevant nodes and their text
        d3.selectAll(".node.element").transition().duration(this.transitionSpeed).style("opacity", opacity);
        d3.selectAll(".node-text").transition().duration(this.transitionSpeed).style("opacity", opacity);
      }

      // Highlight relevant nodes and their text
      if (Array.isArray(nodeClass)) {
        for (let i in nodeClass){
          d3.selectAll(".node.element."+nodeClass[i]).transition().duration(this.transitionSpeed).style("opacity", 1);
          d3.selectAll(".node-text."+nodeClass[i]).transition().duration(this.transitionSpeed).style("opacity", 1)
            .attr('font-size',this.fontsize);
        }
      } else {
        d3.selectAll(".node.element."+nodeClass).transition().duration(this.transitionSpeed).style("opacity", 1);
        d3.selectAll(".node-text."+nodeClass).transition().duration(this.transitionSpeed).style("opacity", 1)
          .attr('font-size',this.fontsize);
      }

      // Applicable only to mouse interactions, and NOT to stepper
      if (highlightOthers) {
        let otherNodes = this.identifyTargets(nodeName);
        otherNodes.forEach(node => {
          d3.selectAll(".node.element."+ node).transition().duration(this.transitionSpeed).style("opacity", 1);
          d3.selectAll(".node-text."+ node).transition().duration(this.transitionSpeed).style("opacity", 1)
            .attr('font-size',this.fontsize);
        })
      }
      
      if (decreaseOpacity) {
        // Decrease opacity of irrelevant links
        d3.selectAll(".link.element").transition().duration(this.transitionSpeed).style("opacity", 0);
      }

      // Higlight links
      if (highlightLinks) {
        let source = nodeName.split(' ').join('').replace(/[^\w\s]|_/g, "")
        let sourceClass = ".link.element." + source
        d3.selectAll(sourceClass).transition().duration(this.transitionSpeed).style("opacity", 0.4);
      }
    },

    identifyTargets(nodeName) {
      let targetNodes = []
      let count = 0
      this.links.forEach(link => {
        if (link.source.id === nodeName || link.target.id === nodeName){
          count+=1;
        }
        if (link.source.id === nodeName) {
          targetNodes.push(link.target.id.split(' ').join('').replace(/[^\w\s]|_/g, ""));
        } 
        
        if (link.target.id === nodeName) {
          targetNodes.push(link.source.id.split(' ').join('').replace(/[^\w\s]|_/g, ""));
        }
      })
      return targetNodes
    },

    nodeMouseout(d, e, nodes) {
      this.noSelectedState(e);
      this.$forceUpdate();
    },

    noSelectedState() {
      d3.selectAll(".node.element").transition().duration(this.transitionSpeed).style("opacity", 1);
      d3.selectAll(".link.element").transition().duration(this.transitionSpeed).style("opacity", 0.2);
      d3.selectAll(".node-text").transition().duration(this.transitionSpeed).style('opacity', d => d.occurence > this.sliderValue ? 1 : 0);
      //d3.selectAll(".emph").transition().duration(this.transitionSpeed).style("opacity", 1).attr('font-size',this.fontsize);
    },

    responsivefy(svg) {
      // get container + svg aspect ratio
      var container = d3.select(svg.node().parentNode),
          width = parseInt(svg.style("width")),
          height = parseInt(svg.style("height")),
          aspect = width / height;

      // add viewBox and preserveAspectRatio properties,
      // and call resize so that svg resizes on inital page load
      svg.attr("viewBox", "0 0 " + width + " " + height)
          //.attr("perserveAspectRatio", "xMinYMid")
          .call(resize);

      // to register multiple listeners for same event type, 
      // you need to add namespace, i.e., 'click.foo'
      // necessary if you call invoke this function for multiple svgs
      // api docs: https://github.com/mbostock/d3/wiki/Selections#on
      d3.select(window).on("resize." + container.attr("id"), resize);

      // get width of container and resize svg to fit it
      function resize() {
          var targetWidth = parseInt(container.style("width")) > 1000 ? 1000 : parseInt(container.style("width"));
          var targetHeight = Math.round(targetWidth / aspect);
          svg.attr("width", targetWidth);
          svg.attr("height", targetHeight);
      }
    },

    initData() {
      let linkDistanceFxn = d3.scaleLinear().domain([1, 1663]).range([50, 0])
      this.force = d3
        .forceSimulation(this.nodes)
        .force( 
          "link",
          d3.forceLink(this.links)
            .id(d => d.id)
            .distance(d => {return linkDistanceFxn(d.value)})
        )
        .force("charge", d3.forceManyBody().strength(this.bodyStrength)) //The strength of the attraction or repulsion
        .force('collision', d3.forceCollide().radius(function(d) {
          return d3.scaleLinear().domain([1, 1000]).range([10, 100])(d.occurence) + 5
        }))
        .force(
          "center",
          d3.forceCenter(this.svgWidth/2, this.svgHeight/2)
        )
        .velocityDecay(0.2);
    },

    initDragTickZoom() {
      d3.selectAll(".node").call(this.drag(this.force));

      this.force.on("tick", () => {
         d3.selectAll(".node")
          .data(this.nodes)
          .attr('cx', d=> d.x)
          .attr('cy', d=> d.y);
          // .attr("cx", d => {
          //   let radius =  d3.scaleLinear().domain([1, 1700]).range([5, 20])(d.occurence)
          //   let newX = Math.max(radius, Math.min(this.width - this.bbBoxMargin - radius, d.x));
          //   return newX
          // })
          // .attr("cy", d => {
          //   let radius =  d3.scaleLinear().domain([1, 1700]).range([5, 20])(d.occurence)
          //   let newY = Math.max(radius, Math.min(this.height - this.bbBoxMargin - radius, d.y));
          //   return newY
          // });

        d3.selectAll(".node-text")
          .data(this.nodes)
          .attr('x', d=> d.x)
          .attr('y', d=> d.y);
          // .attr("x", d => {
          //   let radius =  d3.scaleLinear().domain([1, 1700]).range([5, 20])(d.occurence)
          //   let newX = Math.max(radius, Math.min(this.width - this.bbBoxMargin - radius, d.x));
          //   return newX
          // })
          // .attr("y", d => {
          //   let radius =  d3.scaleLinear().domain([1, 1700]).range([5, 20])(d.occurence)
          //   let newY = Math.max(radius, Math.min(this.height - this.bbBoxMargin - radius, d.y));
          //   return newY
          // });

        if (this.curves) {
          d3.selectAll(".link")
          .data(this.links)
          .attr('d', function (d) {
            let dx = d.target.x - d.source.x;
            let dy = d.target.y - d.source.y;
            let dr = Math.sqrt(dx * dx + dy * dy);
            return "M" + d.source.x + "," + d.source.y + "A" + dr + "," + dr + " 1 0,1 " + d.target.x + "," + d.target.y;
          });
        } else {
          d3.selectAll(".link")
          .data(this.links)
          .attr("x1", d => d.source.x)
          .attr("y1", d => d.source.y)
          .attr("x2", d => d.target.x)
          .attr("y2", d => d.target.y);
        }
      });
    },


    drag(simulation) {
      function dragstarted(d) {
        if (!d3.event.active) simulation.alphaTarget(0.2).restart();
        d3.select(this).classed('fixed', d.fixed = true);
        d.fx = d.x;
        d.fy = d.y;
      }

      function dragged(d) {
        d.fx = d3.event.x;
        d.fy = d3.event.y;
      }

      function dragended(d) {
        if (!d3.event.active) simulation.alphaTarget(0);
        d.fx = null;
        d.fy = null;
      }

      return d3
        .drag()
        .on("start", dragstarted)
        .on("drag", dragged)
        .on("end", dragended);
    },
  }
};
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Sans&display=swap');
</style>

<style scoped>
.app {
  display: inline-block;
  position: relative;
}

.element {
  transition: opacity 0.2s ease;
}
.selected {
  opacity: 0.9 !important;
}

.node,
.link {
  cursor: pointer;
}
</style>

