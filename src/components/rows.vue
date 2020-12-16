<template>
  <div :id="containerId">
    <row v-for="(item, index) in tree" :value="item" :key="index" :index="index" :level="1" @remove="onRemove"></row>
  </div>
</template>

<script>
  import Vue from 'vue'
  import { jsPlumb } from 'jsplumb'
  import { v4 as uuid } from 'uuid'
  import row from './row'
  Vue.component('row', row)

  export default {
    model: {
      prop: 'value',
      event: 'change',
    },
    props: {
      value: {
        type: String,
        default: '',
      },
    },
    data() {
      return {
        tree: [],
        ready: false,
        instance: null,
        containerId: uuid(),
      }
    },
    computed: {},
    watch: {
      value: {
        immediate: true,
        handler() {
          if (this.ready) {
            this.update()
          }
        },
      },
    },
    mounted() {
      this.instance = jsPlumb.getInstance({
        Container: this.containerId,
      })
      this.instance.bind('ready', () => {
        this.ready = true
        this.update()
      })
    },
    methods: {
      update() {
        if (this.value) {
          let tree = []
          this.parseTree(this.value, tree)
          this.injectId(tree)
          this.tree = tree

          this.draw()
        } else {
          this.tree = []

          this.draw()
        }
      },
      // 注入/更新节点id
      injectId(children = [], level = 1) {
        children.forEach((o, i) => {
          o.id = o.id ? o.id : `id-${uuid()}`
          if (o.children.length > 0) {
            o.addBtnId = `id-${uuid()}`
          }
          o.insertBtnId = `id-${uuid()}`
          o.level = level
          o.index = i
          this.injectId(o.children, level + 1)
        })
      },
      // 树形数据->表达式
      parseExpression(children = [], type = '并') {
        let exps = []
        children.forEach((o) => {
          if (/[并或]/.test(o.type)) {
            exps.push(this.parseExpression(o.children, o.type))
          } else {
            exps.push(o.type)
          }
        })
        return `(${exps.join({ 并: '&&', 或: '||' }[type])})`
      },
      // 表达式->树形数据
      parseTree(expression, tree) {
        // 左、右符号位置记录栈
        let leftStack = []
        let rightStack = []
        // 记录拆分记录
        let group = []
        // 当前层级的逻辑符号
        let type = ''
        // 括号块左、右位置
        let left = 0
        let right = expression.length - 1
        // 寻找当前层级的括号快
        for (let i = 0; i < expression.length; i++) {
          if (expression[i] === '(') {
            // 左符号位置记录
            leftStack.push(i)
          } else if (expression[i] === ')') {
            // 右符号位置记录
            rightStack.push(i)
            // 寻找到括号快
            if (leftStack.length === rightStack.length) {
              // 出栈
              left = leftStack.shift()
              right = rightStack.pop()
              leftStack = []
              rightStack = []
              // 处理括号快左侧记录
              if (left >= 2) {
                if (!type) {
                  if (expression.substr(left - 2, 2) === '||') {
                    // 判断逻辑类型
                    type = '或'
                  } else if (expression.substr(left - 2, 2) === '&&') {
                    // 判断逻辑类型
                    type = '并'
                  }
                }

                // 当前括号块左侧非括号块的条件
                let rule = ''
                for (let k = left - 3; k >= 0; k--) {
                  if (expression[k] === ')') {
                    // 提取字符串（左侧有其他括号块）
                    let temp = expression
                      .substring(k + 1, left - 2)
                      .replace(/^&&/, '')
                      .replace(/^\|\|/, '')
                    if (temp) {
                      // 拆分
                      let rules = temp.split('&&')
                      if (rules.length === 0) {
                        rules = temp.split('||')
                      }
                      // 记录
                      rules.forEach((o) => {
                        group.push({
                          type: 'rule',
                          str: o,
                        })
                      })
                    }

                    break
                  } else if (k === 0) {
                    // 提取字符串（左侧没有其他括号块）
                    let temp = expression
                      .substring(k, left - 2)
                      .replace(/^&&/, '')
                      .replace(/^||/, '')
                    if (temp) {
                      // 拆分
                      let rules = temp.split('&&')
                      if (rules.length === 0) {
                        rules = temp.split('||')
                      }
                      // 记录
                      rules.forEach((o) => {
                        group.push({
                          type: 'rule',
                          str: o,
                        })
                      })
                    }
                    break
                  }
                }
              }
              // 需要递归处理的节点
              group.push({
                type: 'next',
                str: expression.substring(left + 1, right),
              })
            }
          }
        }
        // 当前括号块右侧非括号块的条件
        if (right < expression.length - 1) {
          let temp = expression.substring(right + 3, expression.length)
          // 拆分
          let rules = temp.split('&&')
          if (rules.length === 0) {
            rules = temp.split('||')
          }
          // 记录
          rules.forEach((o) => {
            group.push({
              type: 'rule',
              str: o,
            })
          })
        }
        // 辅助计算的顶级额外逻辑符号
        if (!type) {
          type = '并'
        }

        if (group.length > 0) {
          // 构建父节点
          let children = []
          tree.push({
            type,
            children,
          })
          for (let i = 0; i < group.length; i++) {
            if (group[i].type === 'next') {
              // 递归处理表达式
              this.parseTree(group[i].str, children)
            } else {
              // 已经是条件，入队
              children.push({
                type: group[i].str,
                children: [],
              })
            }
          }
        } else {
          // 当前表达式没有括号块
          // 拆分
          let children = expression.split('&&')
          let type = '并'
          if (children.length === 0) {
            children = expression.split('||')
            type = '或'
          }
          // 入队
          tree.push({
            type,
            children: children.map((o) => ({
              type: o,
              children: [],
            })),
          })
        }
      },
      // 通过key找节点
      remove(nodes = [], key = '', node = null) {
        if (!node) {
          let item = null
          let index = -1
          for (let i = 0; i < nodes.length; i++) {
            if (nodes[i].id === key) {
              index = i
              item = nodes[i]
            } else {
              node = this.remove(nodes[i].children, key, node)
            }
          }
          if (index >= 0) {
            if (nodes.length <= 2) {
              alert('最少两个节点')
            } else {
              nodes.splice(index, 1)
            }
            return item
          }
        }
        return node
      },
      // 重画线
      onRemove(item) {
        this.remove(this.tree, item.id)
        this.draw()
      },
      draw() {
        this.instance.reset()
        setTimeout(() => {
          this.tree.forEach((o) => {
            this.drawLine(o)
          })
        })
      },
      drawLine(item, level = 1) {
        let common = {
          endpoint: 'Blank',
          connector: ['Flowchart'],
        }
        if (item.addBtnId) {
          this.instance.connect(
            {
              source: item.id,
              target: item.addBtnId,
              paintStyle: { stroke: '#666666', strokeWidth: 1, dashstyle: '2 2' },
              overlays: [['Arrow', { width: 8, length: 8, location: 1 }]],
              anchor: ['Bottom', 'Top'],
            },
            common
          )
        }

        item.children.forEach((o) => {
          this.instance.connect(
            {
              source: item.id,
              target: o.id,
              paintStyle: { stroke: '#0066FF', strokeWidth: 2 },
              overlays: [['Arrow', { width: 8, length: 8, location: 1 }]],
              anchor: ['Bottom', 'Top'],
            },
            common
          )
          this.drawLine(o, level + 1)
        })
        if (level === 1) {
          this.instance.connect(
            {
              source: item.insertBtnId,
              target: item.id,
              paintStyle: { stroke: '#0066FF', strokeWidth: 2 },
              overlays: [['Arrow', { width: 8, length: 8, location: 1 }]],
              anchor: ['Bottom', 'Top'],
            },
            common
          )
        }
      },
    },
  }
</script>

<style lang="less">
  @import '../styles/row.less';
</style>
