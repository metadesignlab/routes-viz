<template>
  <div id="app" class="flex flex-col p-4" >
    <div id="chart" class="justify-center flex my-1 border border-gray-400"> 
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
      default: () => 'business_descr_relationships.json'
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
      default: -1
    },

    height: {
      type: Number,
      default: 800
    },
    
    width: {
      type: Number,
      default: 1200
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
      stepList: [],
      profileIngredients: [],
      force: null,
      zoom: d3.zoom(),
      nodeColor: ['#FFDA95', '#FCB040', '#FF7629', '#E4140F', '#612348', '#006428',
                '#33A844', '#A0B036', '#D0E820', '#D8CECB'],
      blackOut: ['#FFDA95', '#D0E820', '#D8CECB', '#FCB040'],
      highlightFlavors: [],
      activeNode: 'Flavor',
      activeNodeColor: 'black',
      activeNodeProfile: 3,
      activeNodeSidebar: 'Click on the arrows above to explore common flavours that are paired with chili.',
      activeNodeVariants: '',
      activeNodeSauces: 0,
      allSauces: 45,
      transitionSpeed: 200,
      curves: false,
      stepper: 0,
      linkStroke: 'gray',
      radiusRange: [5, 50]
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
  },
  watch: {
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
    highlightSelective(activeIndex, decreaseOpacity) {
      // Highlight the right things
      // Check if chili or other steps
      if (activeIndex > -1) {
        let ingredients = this.profileIngredients[activeIndex]
        this.highlightNode(ingredients, false, false, decreaseOpacity)
      } else {
        let highlightLinks = this.stepper === 0 ? true : false
        this.highlightNode(activeNode, highlightLinks, true, decreaseOpacity)
      }

      this.$forceUpdate();
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
          if (node.profile in this.profileIngredients === false) {
            this.profileIngredients[node.profile] = []
          }

          this.profileIngredients[node.profile].push(node.id);
        })

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
      console.log(this.svgWidth, this.svgHeight)
      let svg = d3.select("#chart").append("svg")
              .attr("width", width)
              .attr("height", height)
              .attr("id", "container");
              //.call(this.responsivefy);

      return svg
    },

    drawChart() {
      let edges 
      if (this.curves) {
        edges = this.svg.append("g")
          .selectAll(".link")
          .data(this.links)
          .join("path")
          .attr("opacity", 0.4)
          .attr("stroke", this.linkStroke)
          .attr("fill", "transparent")
          .attr('class', function(d) {
            let source = d.source.split(' ').join('').replace(/[^\w\s]|_/g, "");
            let target = d.target.split(' ').join('').replace(/[^\w\s]|_/g, "");
            return 'link element ' + source + ' ' + target;
            })
          .attr('stroke-linecap', 'round')
          .attr('stroke-width', function (d) {
            return 1;
          });
      } else {
        edges = this.svg.append("g")
        .selectAll(".link")
        .data(this.links)
        .enter().append("line")
        .attr('class', function(d) {
          let source = d.source.split(' ').join('').replace(/[^\w\s]|_/g, "");
          let target = d.target.split(' ').join('').replace(/[^\w\s]|_/g, "");
          return 'link element ' + source + ' ' + target;
          })
        .attr('opacity', 0.4)
        .attr('stroke-linecap', 'round')
        .attr('stroke', this.linkStroke)
        .attr('stroke-width', function (d) {
            return d3.scaleLinear().domain([1, 3200]).range([1, 30])(d.value);
          });
      }
      let nodes = this.svg.append("g")
        .selectAll(".node")
        .data(this.nodes)
        .enter().append('circle')
        .attr('id', (d) => d.index)
        .attr('class', function(d) {
          let nodeClass=''
          return 'node element ' + d.id.split(' ').join('').replace(/[^\w\s]|_/g, "");
          })
        .attr('fill', (d) => d.child === true ? 'steelblue' : 'goldenrod')
        .attr('stroke', 'white')
        .attr('r', (d) => d3.scaleLinear().domain([0, 3000]).range([5, 30])(d.occurence))
        .on('mouseover', (d, e, nodes) => this.nodeMouseover(d, e, nodes))
        .on('mouseout', (d, e, nodes) => this.nodeMouseout(d, e, nodes));

      let blackOut = this.blackOut

      // Add single line text
      // this.svg.append("g")
      //   .selectAll(".text")
      //   .data(this.nodes)
      //   .enter()
      //   .append('text')
      //   .attr('class', function(d) {
      //     let nodeClass=''
      //     return 'node-text ' + d.id.split(' ').join('').replace(/[^\w\s]|_/g, "");
      //   })
      //   .text((d) => d.id)
      //   .attr('fill', (d) => this.nodeColor[d['profile']])
      //   .attr('dx', 0)
      //   .attr('alignment-baseline', 'text-top')
      //   .attr('text-anchor', 'middle')
      //   .attr('fill', 'black')
      //   .attr('font-size', 8)
      //   .attr('opacity', 0.4)
      //   .on('mouseover', (d, e, nodes) => this.nodeMouseover(d, e, nodes))
      //   .on('mouseout', (d, e, nodes) => this.nodeMouseout(d, e, nodes));

      this.initDragTickZoom()
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
      console.log(nodeClass)
      if (decreaseOpacity) {
        let opacity = 0.05;

        // Decrease opacity for all irrelevant nodes and their text
        d3.selectAll(".node.element").transition().duration(this.transitionSpeed).style("opacity", opacity);
        d3.selectAll(".node-text").transition().duration(this.transitionSpeed).style("opacity", opacity);
      }

      // Highlight relevant nodes and their text
      if (Array.isArray(nodeClass)) {
        for (let i in nodeClass){
          d3.selectAll(".node.element."+nodeClass[i]).transition().duration(this.transitionSpeed).style("opacity", 1);
          d3.selectAll(".node-text."+nodeClass[i]).transition().duration(this.transitionSpeed).style("opacity", 1)
            .attr('font-size', 12);
        }
      } else {
        d3.selectAll(".node.element."+nodeClass).transition().duration(this.transitionSpeed).style("opacity", 1);
        d3.selectAll(".node-text."+nodeClass).transition().duration(this.transitionSpeed).style("opacity", 1)
          .attr('font-size', 12);
      }

      // Applicable only to mouse interactions, and NOT to stepper
      if (highlightOthers) {
        let otherNodes = this.identifyTargets(nodeName);
        otherNodes.forEach(node => {
          d3.selectAll(".node.element."+ node).transition().duration(this.transitionSpeed).style("opacity", 1);
          d3.selectAll(".node-text."+ node).transition().duration(this.transitionSpeed).style("opacity", 1)
            .attr('font-size', 12);
        })
      }
      
      if (decreaseOpacity) {
        // Decrease opacity of irrelevant links
        d3.selectAll(".link.element").transition().duration(this.transitionSpeed).style("opacity", 0.01);
      }

      // Higlight links
      if (highlightLinks) {
        let source = nodeName.split(' ').join('').replace(/[^\w\s]|_/g, "")
        let sourceClass = ".link.element." + source
        console.log(source)
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
      d3.selectAll(".link.element").transition().duration(this.transitionSpeed).style("opacity", 0.4);
      d3.selectAll(".node-text").transition().duration(this.transitionSpeed).style("opacity", 0.4).attr('font-size', 8);
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
      this.force = d3
        .forceSimulation(this.nodes)
        .force(
          "link",
          d3.forceLink(this.links)
            .id(d => d.id)
            .distance(100)
        )
        .force("charge", d3.forceManyBody().distanceMax(40).strength(50)) //The strength of the attraction or repulsion
        .force('collision', d3.forceCollide().radius(function(d) {
          return d3.scaleLinear().domain([0, 3000]).range([5, 100])(d.occurence) + 1
        }))
        .force(
          "center",
          d3.forceCenter(this.svgWidth/2, this.svgHeight/2)
        )
        .velocityDecay(0.1);
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

.btn-orange {
  @apply transition duration-200 font-bold p-4 m-0 rounded inline-flex items-center 
        flex-grow-0 h-12 text-xl;
  background-color: #FCB040;
}

.btn-orange:hover {
  @apply text-white bg-red-700;
}

.header {
  @apply text-center justify-between;
}

/* responsive, form small screens, use 13px font size */
.sidebar-text {
  @apply w-5/6 flex-auto rounded-md p-3 grid grid-cols-1 bg-orange-100;
}

.top-header, .font-tag, .sauces, .sub-header {
  font-family: 'Gordita', 'IBM Plex Sans';
}

.btn {
  @apply p-5;
  border-radius: 50%;
}

.dot1 {
  @apply mx-2;
  height: 6px;
  width: 6px;
  background-color:#E4140F;
  border-radius: 50%;
  display: inline-block;
}

.dot2 {
  height: 12px;
  width: 12px;
  background-color: #E4140F;
  border-radius: 50%;
  display: inline-block;
}

.rectangle1 {
  height: 4px;
  width: 30px;
  opacity: 0.5;
  display: inline-block;
}

.rectangle2 {
  @apply mt-1;
  height: 8px;
  width: 30px;
  opacity: 0.5;
  display: inline-block;
}

@media only screen and (max-width: 600px) {
  body {
    background-color: lightblue;
  }
}
</style>

