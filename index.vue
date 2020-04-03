<template>
  <div class="d3-tree-container" :class="{'empty-data-classifty':!emptyFlag}">
    <div id="product_tree" />
    <div v-if="!emptyFlag" class="empty-card-s">
      <el-row>
        <el-col :span="24"><div class="grid-content-flex">
          <img src="@/assets/layout/empy1.png" alt="">
          <span>暂无数据</span>
        </div></el-col>
      </el-row>
    </div>
  </div>
</template>

<script>
// import { mockData } from './GetCompanyEquityShareMap.js'
import { getCompanyEquityShareMap } from '@/api/ei-web/business'
var container
var zoom
var depthInfo
var _data
export default {
  name: 'CompanyGuquan',
  data() {
    return {
      directions: ['upward', 'downward'],
      treeData: {},
      d3: null,
      rootData: null,
      forUpward: true,
      rootRectWidth: 0, // 根节点rect的宽度
      rootName: '', // 根节点名称
      projObject: this.$route.query.projObject || '53363',
      emptyFlag: true
    }
  },
  async mounted() {
    var child = document.getElementById('product_tree')
    child.innerHTML = ''
    const projObject = this.projObject
    const resultData = await this.getTreeMap(projObject)
    const result = resultData.Result
    console.log('result:', resultData)
    if (!resultData.Result) {
      this.emptyFlag = false
    }
    this.init(result)
  },
  methods: {
    getTreeMap(projObject) {
      return new Promise((resolve, reject) => {
        getCompanyEquityShareMap(projObject).then(res => {
          if (res.status === '0') {
            const data = JSON.parse(res.data.companyEquityShareMap)
            resolve(data)
            this.treeData = data.Result
          }
        })
      })
    },
    getData(data) {
      // 测试内容
      const _children = data.children
      const _name = data.name
      this.rootData = {
        'downward': {
          'direction': 'downward',
          'name': 'origin',
          'children': []
        },
        'upward': {
          'direction': 'upward',
          'name': 'origin',
          'children': _children
        }
      }
      this.rootName = _name
    },
    getTreeConfig() {
      var treeConfig = {
        'margin': {
          'top': 10,
          'right': 5,
          'bottom': 0,
          'left': 30
        }
      }

      const clientWidth = document.documentElement.clientWidth - 197 // 减掉左边侧边菜单
      const clientHeight = document.documentElement.clientHeight - 102 // 上面头部

      treeConfig.chartWidth = (clientWidth - treeConfig.margin.right - treeConfig.margin.left)
      treeConfig.chartHeight = (clientHeight - treeConfig.margin.top - treeConfig.margin.bottom)
      treeConfig.centralHeight = treeConfig.chartHeight / 2
      treeConfig.centralWidth = treeConfig.chartWidth / 2
      treeConfig.linkLength = 120
      treeConfig.duration = 500 // 动画时间
      return treeConfig
    },
    graphTree(config) {
      var _self = this
      var d3 = this.d3
      var linkLength = config.linkLength
      var duration = config.duration
      var hasChildNodeArr = []
      var id = 0
      let forUpward = this.forUpward
      const rootRectWidth = this.rootRectWidth

      var diagonal = d3.svg.diagonal()
        .source(function(d) {
          return {
            'x': d.source.x,
            'y': d.source.name === 'origin' ? (forUpward ? d.source.y - 20 : d.source.y + 20) : (forUpward ? d.source.y - 60 : d.source.y + 60)
          }
        })
        .target(function(d) {
          return {
            'x': d.target.x,
            'y': d.target.y
          }
        })
        .projection(function(d) {
          return [d.x, d.y]
        })

      // d3.behavior.zoom() 加载图像时设置初始化缩放倍数跟移动大小后，再次用鼠标滚轮进行缩放时
      var zoom = d3.behavior.zoom()
        .scaleExtent([0.3, 2])
        .on('zoom', redraw)

      var svg = d3.select('#product_tree')
        .append('svg')
        .attr('width', config.chartWidth + config.margin.right + config.margin.left)
        .attr('height', config.chartHeight + config.margin.top + config.margin.bottom)
        .attr('xmlns', 'http://www.w3.org/2000/svg')
        .on('mousedown', disableRightClick)
        .call(zoom)
        .on('dblclick.zoom', null)
      var treeG = svg.append('g')
        .attr('class', 'gbox')
        .attr('transform', 'translate(' + config.margin.left + ',' + config.margin.top + ')')

      /*
        * 绘制箭头
        * @param  {string} markerUnits [设置为strokeWidth箭头会随着线的粗细发生变化]
        * @param {string} viewBox 坐标系的区域
        * @param {number} markerWidth,markerHeight 标识的大小
        * @param {string} orient 绘制方向，可设定为：auto（自动确认方向）和 角度值
        * @param {number} stroke-width 箭头宽度
        * @param {string} d 箭头的路径
        * @param {string} fill 箭头颜色
        * @param {string} id resolved0表示公司 resolved1表示个人
        * 直接用一个marker达不到两种颜色都展示的效果
        */

      // 箭头(下半部分)
      var markerDown = svg.append('marker')
        .attr('id', 'resolvedDown')
        .attr('markerUnits', 'strokeWidth') // 设置为strokeWidth箭头会随着线的粗细发生变化
        .attr('markerUnits', 'userSpaceOnUse')
        .attr('viewBox', '0 -5 10 10') // 坐标系的区域
        .attr('markerWidth', 12) // 标识的大小
        .attr('markerHeight', 12)
        .attr('orient', '90') // 绘制方向，可设定为：auto（自动确认方向）和 角度值
        .attr('refX', 0) // 箭头坐标
        .attr('refY', 15)
        .attr('stroke-width', 2) // 箭头宽度
        .append('path')
        .attr('d', 'M0,-5L10,0L0,5') // 箭头的路径
        .attr('fill', '#000') // 箭头颜色
        // 箭头(上半部分)
      var markerUp = svg.append('marker')
        .attr('id', 'resolvedUp')
        .attr('markerUnits', 'strokeWidth') // 设置为strokeWidth箭头会随着线的粗细发生变化
        .attr('markerUnits', 'userSpaceOnUse')
        .attr('viewBox', '0 -5 10 10') // 坐标系的区域
        .attr('markerWidth', 4) // 标识的大小
        .attr('markerHeight', 4)
        .attr('refX', -3) // 箭头坐标
        // .attr('refY', 15)
        .attr('orient', '90') // 绘制方向，可设定为：auto（自动确认方向）和 角度值
        .attr('stroke-width', 2) // 箭头宽度
        .append('path')
        .attr('d', 'M0,-5L10,0L0,5') // 箭头的路径
        .attr('fill', '#b40005') // 箭头颜色

      // Initialize the tree nodes and update chart.
      for (var d in this.directions) {
        var direction = this.directions[d]
        var data = _self.treeData[direction]
        data.x0 = config.centralWidth
        data.y0 = config.centralHeight
        data.children.forEach(collapse)
        update(data, data, treeG)
      }
      function update(source, originalData, g) {
        var direction = originalData['direction']
        forUpward = direction === 'upward'
        const nodeClass = direction + 'Node'
        const linkClass = direction + 'Link'
        var downwardSign = (forUpward) ? -1 : 1
        var nodeColor = (forUpward) ? '#37592b' : '#8b4513'

        var isExpand = false
        var statusUp = true
        var statusDown = true
        var nodeSpace = 130
        var tree = d3.layout.tree().sort(sortByDate).nodeSize([nodeSpace, 0])
        var nodes = tree.nodes(originalData)
        var links = tree.links(nodes)
        var offsetX = -config.centralWidth
        nodes.forEach(function(d) {
          d.y = downwardSign * (d.depth * linkLength) + config.centralHeight
          d.x = d.x - offsetX
          if (d.name == 'origin') {
            d.x = config.centralWidth
            d.y += downwardSign * 0 // 上下两树图根节点之间的距离
            console.log('origin:', d)
          }
          console.log('d:', d)
        })

        // Update the node.
        var node = g.selectAll('g.' + nodeClass)
          .data(nodes, function(d) {
            return d.id || (d.id = ++id)
          })
        var nodeEnter = node.enter().append('g')
          .attr('class', nodeClass)
          .attr('transform', function(d) {
            return 'translate(' + source.x0 + ',' + source.y0 + ')'
          })
          .style('cursor', function(d) {
            return (d.name == 'origin') ? '' : (d.children || d._children) ? 'pointer' : ''
          })
        // .on('click', click);

        nodeEnter.append('svg:rect')
          .attr('x', function(d) {
            return (d.name === 'origin') ? -(rootRectWidth / 2) : -60
          })
          .attr('y', function(d) {
            // return (d.name === 'origin') ? -20 : forUpward ? -52 : 12
            return (d.name === 'origin') ? -20 : forUpward ? -62 : 12
          })
          .attr('width', function(d) {
            return (d.name === 'origin') ? rootRectWidth : 120
          })
          .attr('height', function(d) {
            return (d.name === 'origin') ? 38 : 62
          })
          .attr('rx', 0)
          .style('stroke', function(d) {
            return (d.name === 'origin') ? '#128bed' : '#CCC'
          })
          .style('fill', '#fff')

        nodeEnter.append('circle')
          .attr('r', 1e-6)
        nodeEnter.append('text')
          .attr('class', 'text-style-name')
          .attr('x', function(d) {
            return (d.name === 'origin') ? '0' : '0'
          })
          .attr('dy', function(d) {
            return (d.name === 'origin') ? '.35em' : forUpward ? '-40' : '24'
          })
          .attr('text-anchor', function(d) {
            return (d.name === 'origin') ? 'middle' : 'middle'
          })
          .attr('fill', '#000')
          .text((d) => {
            if (d.name === 'origin') {
              return _self.rootName
            }
            if (d.repeated) {
              return '[Recurring] ' + d.name
            }
            return (d.name.length > 10) ? d.name.substr(0, 10) : d.name
          })
          .style({
            'fill-opacity': 1e-6,
            'fill': function(d) {
              if (d.name === 'origin') {
                return '#fff'
              }
            },
            'font-size': function(d) {
              return (d.name === 'origin') ? 14 : 11
            },
            'cursor': 'pointer'
          })
          .on('click', ChangeModal)

        nodeEnter.append('text')
          .attr('class', 'text-style-name')
          .attr('x', '0')
          .attr('dy', function(d) {
            return (d.name === 'origin') ? '.35em' : forUpward ? '-24' : '35'
          })
          .attr('text-anchor', function() {
            return (d.name === 'origin') ? 'middle' : 'middle'
          })
          .text(function(d) {
            const str = d.name.substr(10, d.name.length)
            const name = str.length > 10 ? d.name.substring(10, 19) + '...' : str
            return name
          })
          .style({
            'fill': '#337ab7',
            'font-size': function(d) {
              return (d.name === 'origin') ? 14 : 11
            },
            'cursor': 'pointer'
          })

        nodeEnter.append('text')
          .attr('x', '0')
          .attr('dy', function(d) {
            return (d.name === 'origin') ? '.35em' : forUpward ? '-10' : '48'
          })
          .attr('text-anchor', 'middle')
          .attr('class', 'linkname')
          .style('fill', '#666666')
          .style('font-size', 10)
          .text((d) => {
            // 大于16
            const str = (d.name === 'origin') ? '' : d.SubConAmt ? '认缴金额:' + d.SubConAmt : ''
            const strReturn = str.length > 16 ? str.substring(0, 16) + '...' : str
            return strReturn
          })
        nodeEnter.append('text')
          .attr('x', '5')
          .attr('dy', function(d) {
            return (d.name === 'origin') ? '.35em' : forUpward ? '10' : '10'
          })
          .attr('text-anchor', 'start')
          .attr('class', 'linkname')
          .style('fill', '#128bed')
          .style('font-size', 10)
          .text((d) => {
            return (d.name === 'origin') ? '' : d.FundedRatio
          })

        // Transition nodes to their new position.原有节点更新到新位置
        var nodeUpdate = node.transition()
          .duration(duration)
          .attr('transform', function(d) {
            return 'translate(' + d.x + ',' + d.y + ')'
          })

        // +- 圆 控制
        nodeUpdate.select('circle')
          .attr('r', function(d) {
            return (d.name == 'origin') ? 0 : (hasChildNodeArr.indexOf(d) == -1) ? 0 : 6
          })
          .attr('cy', function(d) {
            return (d.name == 'origin') ? -20 : (forUpward) ? -68 : 59
          })
          .style('fill', function(d) {
            return hasChildNodeArr.indexOf(d) != -1 ? '#fff' : ''
            // if (d._children || d.children) { return "#fff"; } else { return "rgba(0,0,0,0)"; }
          })
          .style('stroke', function(d) {
            return hasChildNodeArr.indexOf(d) != -1 ? '#128bed' : ''
            // if (d._children || d.children) { return "#8b4513"; } else { return "rgba(0,0,0,0)"; }
          })
          .style('fill-opacity', function(d) {
            if (d.children) {
              return 0.35
            }
          })
        // Setting summary node style as class as mass style setting is
        // not compatible to circles.
          .style('stroke-width', function(d) {
            if (d.repeated) {
              return 5
            }
          })
          // 代表是否展开的+-号
        nodeEnter.append('svg:text')
          .attr('class', 'isExpand')
          .attr('x', '0')
          .attr('dy', function(d) {
            return forUpward ? -64 : 62
          })
          .attr('text-anchor', 'middle')
          .style('fill', '#128bed')
          .style('font-size', 14)
          .text(function(d) {
            if (d.name === 'origin') {
              return ''
            }
            return hasChildNodeArr.indexOf(d) !== -1 ? '+' : ''
          })
          .on('click', click)

        nodeUpdate.select('text').style('fill-opacity', 1)

        var nodeExit = node.exit().transition()
          .duration(duration)
          .attr('transform', function(d) {
            return 'translate(' + source.x + ',' + source.y + ')'
          })
          .remove()
        nodeExit.select('circle')
          .attr('r', 1e-6)
          .attr('stroke', function(d) {
            return '#128bed'
          })
        nodeExit.select('text')
          .style('fill-opacity', 1e-6)
        // 数据连接，根据目标节点的id绑定数据
        var link = g.selectAll('path.' + linkClass)
          .data(links, function(d) {
            return d.target.id
          })
        // 增加新连接
        link.enter().insert('path', 'g')
          .attr('class', linkClass)
          .attr('stroke', function(d) {
            return '#7a7a7a'
          })
          .attr('fill', 'none')
          .attr('stroke-width', '1px')
          .attr('opacity', 0.5)
          .attr('d', function(d) {
            var o = {
              x: source.x0,
              y: source.y0
            }
            return diagonal({
              source: o,
              target: o
            })
          })
          .attr('marker-end', function(d) {
            return forUpward ? 'url(#resolvedUp)' : 'url(#resolvedDown)'
          }) // 根据箭头标记的id号标记箭头;
          .attr('id', function(d, i) {
            return 'mypath' + i
          })
        // 原有连接更新位置
        link.transition()
          .duration(duration)
          .attr('d', diagonal)
        // 折叠的链接，收缩到源节点处
        link.exit().transition()
          .duration(duration)
          .attr('d', function(d) {
            var o = {
              x: source.x,
              y: source.y
            }
            return diagonal({
              source: o,
              target: o
            })
          })
          .remove()
        // 把旧位置存下来，用以过渡
        nodes.forEach(function(d) {
          d.x0 = d.x
          d.y0 = d.y
        })
        function ChangeModal() {
          _self.Modal = true
        }
        // 切换折叠与否
        function click(d) {
          if (forUpward) {} else {
            if (d._children) {
              console.log('对外投资--ok')
            } else {
              console.log('对外投资--no')
            }
          }
          isExpand = !isExpand
          if (d.name == 'origin') {
            return
          }
          if (d.children) {
            d._children = d.children
            d.children = null
            d3.select(this).text('+')
          } else {
            d.children = d._children
            d._children = null
            // expand all if it's the first node
            if (d.name == 'origin') {
              d.children.forEach(expand)
            }
            d3.select(this).text('-')
          }
          update(d, originalData, g)
        }
      }

      function expand(d) {
        if (d._children) {
          d.children = d._children
          d.children.forEach(expand)
          d._children = null
        }
      }

      function collapse(d) {
        if (d.children && d.children.length != 0) {
          d._children = d.children
          d._children.forEach(collapse)
          d.children = null
          hasChildNodeArr.push(d)
        }
      }

      function redraw() {
        treeG.attr('transform', 'translate(' + d3.event.translate + ')' +
            ' scale(' + d3.event.scale + ')')
      }

      function disableRightClick() {
        // stop zoom
        if (d3.event.button == 2) {
          console.log('No right click allowed')
          d3.event.stopImmediatePropagation()
        }
      }

      function sortByDate(a, b) {
        var aNum = a.name.substr(a.name.lastIndexOf('(') + 1, 4)
        var bNum = b.name.substr(b.name.lastIndexOf('(') + 1, 4)
        return d3.ascending(aNum, bNum) ||
            d3.ascending(a.name, b.name) ||
            d3.ascending(a.id, b.id)
      }
    },
    drawing() {
      const _this = this
      let downwardLength = 0
      let upwardLength = 0

      const TreeChart = (d3Object) => {
        _this.d3 = d3Object
      }

      TreeChart.prototype.drawChart = () => {
        // First get tree data for both directions.
        _this.directions.forEach((direction) => {
          _this.treeData[direction] = _this.rootData[direction]
        })
        // rootName = '上海冰鉴信息科技有限公司';
        this.rootRectWidth = _this.rootName.length * 15
        // 获得upward第一级节点的个数
        upwardLength = _this.rootData.upward.children.length
        // 获得downward第一级节点的个数
        downwardLength = _this.rootData.downward.children.length
        _this.graphTree(_this.getTreeConfig())
      }

      var d3GenerationChart = new TreeChart(d3)
      d3GenerationChart.drawChart()
    },
    init(data) {
      this.getData(data)
      this.drawing()
    }
  }
}
</script>

<style lang="scss">
.empty-data-classifty{
  display: flex;
  justify-content: center;
  align-items: center;
}
.d3-tree-container{
  background-color: #fff;
  min-height: calc(100vh - 84px);
  .text-style-name{
    fill: #333!important;
  }
  .empty-card-s{
    .grid-content-flex{
      display: flex;
      justify-content: center;
      align-items: center;
      flex-direction: column;
      img{
        width: 193px;
        height: 188px;
      }
    }
  }
}
</style>
