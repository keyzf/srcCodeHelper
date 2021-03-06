<template>
  <div class="detail-path bl-card"
       ref="root">
  </div>
</template>
<script type="text/javascript">
const DIRECTION = {
  CLOCKWISE: 0,
  ANTI_CLOCKWISE: 1
};
import * as d3 from "d3";
export default {
  data() {
    return {
      graphData: {},
      depData: null,
      svgWidth: null,
      svgHeight: null,
      forceStrength: {
        long: -30,
        indirect: -120,
        direct: -120
      },
      type: null,
      depTypeColorMap: d3
        .scaleOrdinal(["#fbb4ae", "#b3cde3", "#ccebc5", "#decbe4", "#fed9a6"])
        .domain([
          "ImportSpecifier",
          "ImportDefaultSpecifier",
          "ImportNamespaceSpecifier",
          "ExportSpecifier",
          "ExportAllSpecifier"
        ])
    };
  },
  mounted() {
    this.$bus.$on("highlight-dep", dep => {
      this.type = dep.type;
      this.$axios
        .get("files/getDetailBadDepInfoByDepId", {
          lenThreshold: this.lenThreshold,
          type: dep.type,
          depId: dep.id,
          // libName:'d3'
          libName:'vue'
        })
        .then(({ data }) => {
          this.depData = data;
          this.dataAdapter();
          this.draw();
        });
    });
    this.svgWidth = Math.floor(this.$refs.root.clientWidth);
    this.svgHeight = Math.floor(this.$refs.root.clientHeight);
    this.svg = d3
      .select(this.$refs.root)
      .append("svg")
      .attr("width", this.svgWidth)
      .attr("height", this.svgHeight);
  },
  props: ["lenThreshold"],
  methods: {
    // 判断顺逆时针需要考虑坐标系转换
    getDirection(nodes) {
      let len = nodes.length,
        posNum = 0,
        negNum = 0,
        maxY = -1;
      nodes.forEach((val, idx) => {
        let prevIdx = idx - 1,
          nextIdx = (idx + 1) % len;
        prevIdx = prevIdx < 0 ? (prevIdx + len) % len : prevIdx;
        let prevNode = nodes[prevIdx],
          nextNode = nodes[nextIdx];
        if (
          (val.x - prevNode.x) * (val.y - nextNode.y) -
            (prevNode.y - val.y) * (nextNode.x - val.x) >
          0
        )
          posNum++;
        else negNum++;
      });
      return posNum > negNum ? DIRECTION.ANTI_CLOCKWISE : DIRECTION.CLOCKWISE;
    },
    genRelPath(path) {
      let match = path.match(/\/Users\/wendahuang\/Desktop\/vue\/src\/(.*)/);
      return match ? match[1] : path;
    },
    draw() {
      d3.select(this.$refs.root)
        .selectAll("svg *")
        .remove();
      let vm = this;
      // 添加箭头形状
      let arrowMarker = this.svg
        .append("defs")
        .append("marker")
        .attr("id", "detail-path-arrow")
        .attr("viewBox", "0 -5 10 10")
        .attr("refX", 15)
        .attr("markerWidth", 3)
        .attr("markerHeight", 3)
        .attr("orient", "auto")
        .append("path")
        .attr("d", "M0,-5L10,0L0,5");

      var simulation = d3
        .forceSimulation()
        .force(
          "link",
          d3.forceLink().id(function(d) {
            return d.id;
          })
        )
        .force(
          "charge",
          d3
            .forceManyBody()
            .distanceMin(60)
            .strength(this.forceStrength[this.type])
        )
        // .force("charge", d3.forceCollide())
        // .force("charge", d3.forceRadial())
        .force("center", d3.forceCenter(this.svgWidth / 2, this.svgHeight / 2));

      // 画线
      const links = this.svg
        .append("g")
        .attr("class", "links")
        .selectAll("path")
        .data(this.graphData.links)
        .enter()
        .append("g");
      // 线路径
      this.linksPath = links
        .append("path")
        .attr("class", "link")
        .attr("id", d => `${d.source}|${d.target}`)
        .style("stroke", d => this.depTypeColorMap(d.type))
        .attr("marker-end", function(d) {
          return "url(#detail-path-arrow)";
        });
      // 线文字信息
      /*      this.linksText = links.append("text")
              .attr('dy',10)
              .append("textPath")
              .attr('href', d => `#${d.source}|${d.target}`)
              .attr('startOffset', 10)
              .text(d => "text")*/

      // 画点
      this.nodes = this.svg
        .append("g")
        .attr("class", "nodes")
        .selectAll("circle")
        .data(this.graphData.nodes)
        .enter()
        .append("circle")
        .attr("r", 5)
        .attr("fill", d => {
          return "white";
        })
        .attr("stroke", "black")
        .on("click", d => {
          this.$bus.$emit("draw-codechart", d.id);
          this.$bus.$emit("draw-wordcloud", this.genRelPath(d.id));
        })
        .call(
          d3
            .drag()
            .on("start", dragstarted)
            .on("drag", dragged)
            .on("end", dragended)
        );

      // 标题
      this.nodes.append("title").text(function(d) {
        return d.id;
      });
      // 结点文字信息
      this.text = this.svg
        .append("g")
        .selectAll("text")
        .data(this.graphData.nodes)
        .enter()
        .append("text")
        .attr("x", 8)
        .attr("y", ".31em")
        // .attr("text-anchor","middle")
        .text(function(d) {
          return vm.formatText(d.id);
        });
      // 边文字信息
      this.svg
        .append("g")
        .selectAll(".link-text")
        .data();
      simulation
        .nodes(this.graphData.nodes)
        .on("tick", ticked)
        .on("end", d => {
          /*          console.log(vm.nodes.nodes(), vm.nodes, vm.graphData.nodes)
          console.log(d3.select())
          vm.nodes.each((d) => {
            console.log(d)
          })
          console.log(vm.getDirection(vm.graphData.nodes))*/
        });

      simulation.force("link").links(this.graphData.links);

      function ticked() {
        let direction = DIRECTION.CLOCKWISE;
        if (vm.type === "indirect" || vm.type === "direct") {
          direction = vm.getDirection(vm.graphData.nodes);
        }
        arrowMarker.attr(
          "refY",
          direction === DIRECTION.CLOCKWISE ? -1.5 : 1.5
        );
        vm.linksPath.attr("d", getDirectArcPath(direction));
        vm.nodes.attr("transform", transform);
        vm.text.attr("transform", transform);
      }

      function dragstarted(d) {
        if (!d3.event.active) simulation.alphaTarget(0.3).restart();
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

      function getDirectArcPath(direction) {
        return function linkArc(d) {
          var dx = d.target.x - d.source.x,
            dy = d.target.y - d.source.y,
            dr = Math.sqrt(dx * dx + dy * dy);
          let rotateDir = direction === DIRECTION.CLOCKWISE ? 1 : 0;
          // let rotateDir=1
          return (
            "M" +
            d.source.x +
            "," +
            d.source.y +
            "A" +
            dr +
            "," +
            dr +
            ` 0 0,${rotateDir} ` +
            d.target.x +
            "," +
            d.target.y
          );
        };
      }

      function transform(d) {
        return "translate(" + d.x + "," + d.y + ")";
      }
    },
    formatText(text) {
      let idx = text.lastIndexOf("/");
      return text.slice(idx + 1).match(/(.*)(\.js)$/)[1];
    },
    dataAdapter() {
      this.graphData.nodes = this.depData.nodes.map((d) => ({ id: d }))
      this.graphData.links = this.depData.links.slice() 
 
    }
  }
};
</script>
<style type="text/css" lang="scss">
.detail-path {
  .link {
    fill: none;
    stroke: #666;
    stroke-width: 3px;
  }
}
</style>
