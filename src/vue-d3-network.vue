<script>
/* eslint import/no-duplicates: "warn" */
import * as forceSimulation from 'd3-force'
import * as d3Selection from 'd3-selection'
import { event as currentEvent } from 'd3-selection'
import svgRenderer from './components/svgRenderer.vue'
import canvasRenderer from './components/canvasRenderer.vue'
import saveImage from './lib/js/saveImage.js'
import svgExport from './lib/js/svgExport.js'
import debounce from 'lodash.debounce'
import * as d3Zoom from 'd3-zoom'
const d3 = Object.assign({}, forceSimulation, d3Selection, d3Zoom)
function reset () {
  this.simulation.stop().alpha(1).nodes(this.nodes)
  if (this.forces.Link) this.simulation.force('link').links(this.links)
  this.simulation.restart()
}
const resetDebounced = debounce(reset, 150)
export default {
  name: 'd3-network',
  components: {
    canvasRenderer,
    svgRenderer
  },
  props: {
    netNodes: {
      type: Array
    },
    netLinks: {
      type: Array
    },
    options: {
      type: Object
    },
    nodeSym: {
      type: String
    },
    nodeCb: {
      type: Function
    },
    linkCb: {
      type: Function
    },
    simCb: {
      type: Function
    },
    customForces: {
      type: Object
    },
    selection: {
      type: Object,
      default: () => {
        return {
          nodes: {},
          links: {}
        }
      }
    }
  },
  data () {
    return {
      moving: false,
      canvas: false,
      nodes: [],
      links: [],
      size: {
        w: 0,
        h: 0
      },
      offset: {
        x: 0,
        y: 0
      },
      clientOffset: {
        x: 0,
        y: 0
      },
      force: 500,
      forces: {
        Center: false,
        X: 0.5,
        Y: 0.5,
        ManyBody: true,
        Link: true,
        Radial: false
      },
      noNodes: false,
      strLinks: true,
      fontSize: 10,
      dragging: false,
      linkWidth: 1,
      nodeLabels: false,
      linkLabels: false,
      nodeSize: 5,
      mouseOfst: {
        x: 0,
        y: 0
      },
      padding: {
        x: 0,
        y: 0
      },
      simulation: null,
      nodeSvg: null,
      resizeListener: true
    }
  },
  render (createElement) {
    this.ref = 'svg'
    let props = {}
    let renderer = 'svg-renderer'
    let bindProps = [
      'size',
      'nodes',
      'links',
      'selected',
      'linksSelected',
      'strLinks',
      'linkWidth',
      'nodeLabels',
      'linkLabels',
      'fontSize',
      'labelOffset',
      'offset',
      'padding',
      'nodeSize',
      'noNodes',
      'moving'
    ]

    for (let prop of bindProps) {
      props[prop] = this[prop]
    }
    props.nodeSym = this.nodeSvg

    if (this.canvas) {
      renderer = 'canvas-renderer'
      this.ref = 'canvas'
      props.canvasStyles = this.options.canvasStyles
    }

    return createElement('div', {
      attrs: { class: 'net', id: 'net' },
      on: { 'pointermove': this.move }
    }, [createElement(renderer, {
      props, ref: this.ref, on: { action: this.methodCall }
    })])
  },
  created () {
    this.updateOptions(this.$data, this.options)
    this.buildNodes(this.netNodes)
    this.buildLinks(this.netLinks)
    this.updateNodeSvg()
  },
  mounted () {
    this.onResize()
    const refNode = this.$refs[this.ref].$el
    this.zoom = d3.zoom().scaleExtent([1 / 2, 4]).on('zoom', () => {
      for (const selector of ['#l-nodes', '#l-links', '#node-labels', '#link-labels']) { 
        d3.select(selector).attr('transform', currentEvent.transform) 
      }
    })
    this.d3Node = d3.select(refNode)
    this.d3Node.call(this.zoom).on('dblclick.zoom', null)
    this.$nextTick(() => {
      this.animate()
    })
    this.resizeListener = !this.options.size
    if (this.resizeListener) window.addEventListener('resize', this.onResize)
  },
  beforeDestroy () {
    window.removeEventListener('resize', this.onResize)
  },
  computed: {
    selected () {
      return this.selection.nodes
    },
    linksSelected () {
      return this.selection.links
    },
    center () {
      return {
        x: this.size.w / 2 + (this.size.w / 200) + this.offset.x,
        y: this.size.h / 2 + (this.size.h / 200) + this.offset.y
      }
    },
    labelOffset () {
      return {
        x: (this.nodeSize / 2) + (this.fontSize / 2),
        y: this.fontSize / 2
      }
    }
  },
  watch: {
    netNodes (newValue) {
      this.buildNodes(newValue)
      resetDebounced.call(this)
    },
    netLinks (newValue) {
      this.buildLinks(newValue)
      resetDebounced.call(this)
    },
    nodeSym () {
      this.updateNodeSvg()
    },
    options (newValue, oldValue) {
      this.updateOptions(this.$data, newValue)
      if (oldValue.size && newValue.size) {
        if ((oldValue.size.w !== newValue.size.w) || (oldValue.size.h !== newValue.size.h)) {
          this.onResize()
        }
      }
      this.animate()
    },
    resizeListener (val) {
      window.removeEventListener('resize', this.onResize)
      if (val) window.addEventListener('resize', this.onResize)
    }
  },
  methods: {
    dblClick (event, node) {
      this.$emit('node-dblclick', event, node)
    },
    updateNodeSvg () {
      let svg = null
      if (this.nodeSym) {
        svg = svgExport.svgElFromString(this.nodeSym)
      }
      this.nodeSvg = svg
    },
    methodCall (action, args) {
      let method = this[action]
      if (method && typeof (method) === 'function') {
        if (args) method(...args)
        else method()
      }
    },
    onResize () {
      let size = this.options.size
      if (!size || !size.w) this.size.w = this.$el.clientWidth
      if (!size || !size.h) this.size.h = this.$el.clientHeight
      this.padding.x = 0
      this.padding.y = 0
      // serach offsets of parents
      let vm = this
      while (vm.$parent) {
        this.padding.x += vm.$el.offsetLeft || 0
        this.padding.y += vm.$el.offsetTop || 0
        vm = vm.$parent
      }
    },
    // -- Data
    updateOptions (source, options) {
      for (let op in options) {
        if (source.hasOwnProperty(op)) {
          if (typeof source[op] === 'object') {
            this.updateOptions(source[op], options[op])
          } else {
            source[op] = options[op]
          }
        }
      }
    },
    buildNodes (nodes) {
      this.nodes = nodes.map((node, index) => {
        // node formatter option
        node = this.itemCb(this.nodeCb, node)
        // index as default node id
        if (!node.id && node.id !== 0) this.$set(node, 'id', index)
        // initialize node coords
        if (!node.x) this.$set(node, 'x', 0)
        if (!node.y) this.$set(node, 'y', 0)
        // node default name, allow string 0 as name
        if (!node.name && node.name !== '0') this.$set(node, 'name', 'node ' + node.id)
        if (node.svgSym) {
          node.svgIcon = svgExport.svgElFromString(node.svgSym)
          if (!this.canvas && node.svgIcon && !node.svgObj) node.svgObj = svgExport.toObject(node.svgIcon)
        }
        return node
      })
    },

    buildLinks (links) {
      this.links = links.map((link, index) => {
        // link formatter option
        link = this.itemCb(this.linkCb, link)
        // source and target for d3
        link.source = link.sid
        link.target = link.tid
        if (!link.id) this.$set(link, 'id', 'link-' + index)
        return link
      })
    },
    itemCb (cb, item) {
      if (cb && typeof (cb) === 'function') item = cb(item)
      return item
    },
    // -- Animation
    simulate (nodes, links) {
      let forces = this.forces
      let sim = d3.forceSimulation(nodes)
        .stop()
        .alpha(0.5)

      if (forces.Center !== false) sim.force('center', d3.forceCenter(this.center.x, this.center.y))
      if (forces.X !== false) {
        sim.force('X', d3.forceX(this.center.x).strength(forces.X))
      }
      if (forces.Y !== false) {
        sim.force('Y', d3.forceY(this.center.y).strength(forces.Y))
      }
      if (forces.ManyBody !== false) {
        sim.force('charge', d3.forceManyBody().strength(-this.force))
      }
      if (forces.Link !== false) {
        const forceLink = d3.forceLink(links).id(function (d) { return d.id })
        if (forces.Radial !== false) forceLink.strength(0)
        sim.force('link', forceLink)
      }
      if (forces.Radial !== false) {
        const r = this.nodes.length * this.nodeSize / (2 * Math.PI)
        sim.force('charge', d3.forceCollide(this.nodeSize))
          .force('radial', d3.forceRadial(r, this.center.x, this.center.y))
      }
      sim = this.setCustomForces(sim)
      sim = this.itemCb(this.simCb, sim)
      return sim
    },
    setCustomForces (sim) {
      let forces = this.customForces
      if (forces) {
        for (let f in forces) {
          let d3Func = this.getD3Func('force' + f)
          if (d3Func) {
            let args = forces[f]
            sim.force('custom' + f, d3Func(...args))
          }
        }
      }
      return sim
    },
    getD3Func (name) {
      let func = d3[name]
      if (func && typeof (func) === 'function') return func
      return null
    },
    animate () {
      if (this.simulation) this.simulation.stop()
      if (this.forces.Link !== false) this.simulation = this.simulate(this.nodes, this.links)
      else this.simulation = this.simulate(this.nodes)
      this.simulation.restart()
    },
    // -- Mouse Interaction
    move (event) {
      const pos = this.clientPos(event)
      let fx = 0
      let fy = 0
      if (this.dragging !== false) {
        if (this.nodes[this.dragging]) {
          fx = this.nodes[this.dragging].fx
          fy = this.nodes[this.dragging].fy
          this.simulation.restart()
          this.simulation.alpha(0.5)
          this.nodes[this.dragging].fx = pos.x - this.mouseOfst.x
          this.nodes[this.dragging].fy = pos.y - this.mouseOfst.y
        }
      }
      if (this.calculateRadial(fx, fy) < 10) {
        this.moving = false
      } else {
        this.moving = true
      }
    },
    /**
     * calculate move radial to detect whether moving
     */
    calculateRadial (fx, fy) {
      if (this.dragging !== false) {
        return Math.sqrt(Math.pow(fx - this.nodes[this.dragging].fx, 2) + Math.pow(fy - this.nodes[this.dragging].fy, 2))
      } else {
        return 0
      }
    },
    clientPos (event) {
      let x = (event.touches) ? event.touches[0].clientX : event.clientX
      let y = (event.touches) ? event.touches[0].clientY : event.clientY
      x = x || 0
      y = y || 0
      return { x, y }
    },
    dragStart (event, nodeKey) {
      this.simulation.stop()
      this.d3Node.on('.zoom', null)
      this.dragging = (nodeKey === false) ? false : nodeKey
      this.setMouseOffset(event, this.nodes[nodeKey])
      if (this.dragging === false) {
        this.d3Node.call(this.zoom).on('dblclick.zoom', null)
        this.simulation.alpha(0.1).restart()
        this.setMouseOffset()
      }
    },
    dragEnd () {
      let node = this.nodes[this.dragging]
      if (node && !node.pinned) {
        // unfix node position
        node.fx = null
        node.fy = null
      }
      this.dragStart(undefined, false)
    },
    // -- Render helpers
    nodeClick (event, node) {
      this.$emit('node-click', event, node)
    },
    linkClick (event, link) {
      this.$emit('link-click', event, link)
    },
    setMouseOffset (event, node) {
      let x = 0
      let y = 0
      if (event && node) {
        let pos = this.clientPos(event)
        x = (pos.x) ? pos.x - node.x : node.x
        y = (pos.y) ? pos.y - node.y : node.y
      }
      this.mouseOfst = { x, y }
    },
    screenShot (name, bgColor, toSVG, svgAllCss) {
      let exportFunc
      let args = []
      if (this.canvas) {
        toSVG = false
        exportFunc = this.$refs.canvas.canvasScreenShot
        args = [bgColor]
      } else {
        exportFunc = this.$refs.svg.svgScreenShot
        args = [toSVG, bgColor, svgAllCss]
      }
      if (toSVG) name = name || 'export.svg'

      exportFunc((err, url) => {
        if (!err) {
          if (!toSVG) saveImage.save(url, name)
          else saveImage.download(url, name)
        }
        this.$emit('screen-shot', err)
      }, ...args)
    }
  }
}
</script>

<style lang="stylus">
  @import 'lib/styl/vars.styl'

  #net
    height 100%
    margin 0
    touch-action: none

    .net-svg
      // fill: white // background color to export as image

    .node
      stroke alpha($dark, 0.7)
      stroke-width 3px
      transition fill 0.5s ease
      fill $white

    .node.selected
      stroke $color2

    .node.pinned
      fill alpha($warn, 1)

    .link
      stroke alpha($dark, 0.3)

    .marker
      fill alpha($dark, 1)

    .marker.selected
      fill alpha($color2, 1)

    .node, .link .marker
      stroke-linecap round

      &:hover
        stroke $warn
        stroke-width 5px

    .link.selected
      stroke alpha($color2, 0.6)

    .curve
      fill none

    .node-label
      fill $dark
      user-select none

    .link-label
      fill $dark
      transform translate(0, -0.5em)
      text-anchor middle
</style>
