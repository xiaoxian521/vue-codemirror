<template>
  <div class="vue-codemirror" :class="{ merge }">
    <div ref="mergeview" v-if="merge"></div>
    <textarea ref="textarea" :name="name" :placeholder="placeholder" v-else></textarea>
  </div>
</template>

<script>
// lib
import _CodeMirror from 'codemirror'
const CodeMirror = window.CodeMirror || _CodeMirror

// pollfill
if (typeof Object.assign != 'function') {
  Object.defineProperty(Object, 'assign', {
    value(target, varArgs) {
      if (target == null) {
        throw new TypeError('Cannot convert undefined or null to object')
      }
      const to = Object(target)
      for (let index = 1; index < arguments.length; index++) {
        const nextSource = arguments[index]
        if (nextSource != null) {
          for (const nextKey in nextSource) {
            if (Object.prototype.hasOwnProperty.call(nextSource, nextKey)) {
              to[nextKey] = nextSource[nextKey]
            }
          }
        }
      }
      return to
    },
    writable: true,
    configurable: true
  })
}

// export
export default {
  name: 'codemirror',
  data() {
    return {
      content: '',
      codemirror: null,
      cminstance: null
    }
  },
  props: {
    code: String,
    value: String,
    marker: Function,
    unseenLines: Array,
    name: {
      type: String,
      default: 'codemirror'
    },
    placeholder: {
      type: String,
      default: ''
    },
    merge: {
      type: Boolean,
      default: false
    },
    options: {
      type: Object,
      default: () => ({})
    },
    events: {
      type: Array,
      default: () => ([])
    },
    globalOptions: {
      type: Object,
      default: () => ({})
    },
    globalEvents: {
      type: Array,
      default: () => ([])
    },
    dico: {
      type: Array,
      default: () => ([])
    }
  },
  watch: {
    options: {
      deep: true,
      handler(options) {
        for (const key in options) {
          this.cminstance.setOption(key, options[key])
        }
      }
    },
    merge() {
      this.$nextTick(this.switchMerge)
    },
    code(newVal) {
      this.handerCodeChange(newVal)
    },
    value(newVal) {
      this.handerCodeChange(newVal)
    },
  },
  methods: {
    initialize() {
      const cmOptions = Object.assign({
        hintOptions:
        {
          completeSingle: false,
          hint: this.hint
        }
      }, this.globalOptions, this.options)
      if (this.merge) {
        this.codemirror = CodeMirror.MergeView(this.$refs.mergeview, cmOptions)
        this.cminstance = this.codemirror.edit
      } else {
        this.codemirror = CodeMirror.fromTextArea(this.$refs.textarea, cmOptions)
        this.cminstance = this.codemirror
        this.cminstance.setValue(this.code || this.value || this.content)
      }

      this.cminstance.on('change', cm => {
        this.content = cm.getValue()
        if (this.$emit) {
          this.$emit('input', this.content, this.cminstance)
        }
      })

      this.cminstance.on("keypress", cm => {
        cm.showHint()
      })

      // 所有有效事件（驼峰命名）+ 去重
      const tmpEvents = {}
      const allEvents = [
        'scroll',
        'changes',
        'beforeChange',
        'cursorActivity',
        'keyHandled',
        'inputRead',
        'electricInput',
        'beforeSelectionChange',
        'viewportChange',
        'swapDoc',
        'gutterClick',
        'gutterContextMenu',
        'focus',
        'blur',
        'refresh',
        'optionChange',
        'scrollCursorIntoView',
        'update',
        "mousedown",
        "dblclick",
        "touchstart",
        "contextmenu",
        "keydown",
        "keypress",
        "keyup",
        "cut",
        "copy",
        "paste",
        "dragstart",
        "dragenter",
        "dragover",
        "dragleave",
        "drop"
      ]
        .concat(this.events)
        .concat(this.globalEvents)
        .filter(e => (!tmpEvents[e] && (tmpEvents[e] = true)))
        .forEach(event => {
          // 循环事件，并兼容 run-time 事件命名
          this.cminstance.on(event, (...args) => {
            // console.log('当有事件触发了', event, args)
            this.$emit(event, ...args)
            const lowerCaseEvent = event.replace(/([A-Z])/g, '-$1').toLowerCase()
            if (lowerCaseEvent !== event) {
              this.$emit(lowerCaseEvent, ...args)
            }
          })
        })

      this.$emit('ready', this.codemirror)
      this.unseenLineMarkers()

      // prevents funky dynamic rendering
      this.refresh()
    },
    refresh() {
      this.$nextTick(() => {
        this.cminstance.refresh()
      })
    },
    destroy() {
      // garbage cleanup
      const element = this.cminstance.doc.cm.getWrapperElement()
      element && element.remove && element.remove()
    },
    handerCodeChange(newVal) {
      const cm_value = this.cminstance.getValue()
      if (newVal !== cm_value) {
        const scrollInfo = this.cminstance.getScrollInfo()
        this.cminstance.setValue(newVal)
        this.content = newVal
        this.cminstance.scrollTo(scrollInfo.left, scrollInfo.top)
      }
      this.unseenLineMarkers()
    },
    unseenLineMarkers() {
      if (this.unseenLines !== undefined && this.marker !== undefined) {
        this.unseenLines.forEach(line => {
          const info = this.cminstance.lineInfo(line)
          this.cminstance.setGutterMarker(line, 'breakpoints', info.gutterMarkers ? null : this.marker())
        })
      }
    },
    switchMerge() {
      // Save current values
      const history = this.cminstance.doc.history
      const cleanGeneration = this.cminstance.doc.cleanGeneration
      this.options.value = this.cminstance.getValue()

      this.destroy()
      this.initialize()

      // Restore values
      this.cminstance.doc.history = history
      this.cminstance.doc.cleanGeneration = cleanGeneration
    },
    suggest(searchString) {
      let token = searchString
      if (searchString.startsWith(".")) token = searchString.substring(1)
      else token = searchString.toLowerCase()
      const resu = []
      const N = this.dico.length

      for (let i = 0; i < N; i++) {
        const keyword = this.dico[i].text.toLowerCase()
        let suggestion = null
        if (keyword.startsWith(token)) {
          suggestion = Object.assign({ score: N + (N - i) }, this.dico[i])
        } else if (keyword.includes(token)) {
          suggestion = Object.assign({ score: N - i }, this.dico[i])
        }
        if (suggestion) resu.push(suggestion)
      }

      if (searchString.startsWith(".")) {
        resu.forEach(s => {
          if (s.className == "column") s.score += N
          else if (s.className == "sql") s.score -= N
          return s
        })
      }

      return resu.sort((a, b) => b.score - a.score)
    },
    hint(editor) {
      const cur = editor.getCursor()
      const token = editor.getTokenAt(cur)
      const searchString = token.string
      return {
        list: this.suggest(searchString),
        from: CodeMirror.Pos(cur.line, token.start),
        to: CodeMirror.Pos(cur.line, token.end)
      }
    }
  },
  mounted() {
    this.initialize()
  },
  beforeDestroy() {
    this.destroy()
  }
}
</script>
