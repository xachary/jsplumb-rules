<template>
  <div class="p-tree-select">
    <el-input
      v-model="filterText"
      :placeholder="filterPlaceholder"
      prefix-icon="el-icon-search p-tree-select__filter__icon"
      class="p-tree-select__filter"
      v-show="showFilter"
      clearable
    />
    <!-- :expand-on-click-node="multiple || !updating" -->
    <el-tree
      :data="parseData"
      @node-click="onSelectNode"
      @check="onSelectNode"
      @node-expand="onNodeExpand"
      node-key="id"
      :default-expanded-keys="parsedDefaultExpandedKeys"
      ref="tree"
      :highlight-current="true"
      :default-expand-all="expandAll"
      :show-checkbox="multiple"
      :filter-node-method="filterNode"
      :empty-text="emptyText">
      <div slot-scope="{ node, data }" class="p-tree-select__node">
        <span
          class="el-tree-node__label"
          v-if="!$scopedSlots['tree-node']"
          v-html="node.label.replace(html.encode(filterText), `<b>${html.encode(filterText)}</b>`)">
        </span>
        <slot name="tree-node" :node="node" :data="data" :filterText="filterText"> </slot>
      </div>
    </el-tree>
  </div>
</template>

<script>
  import mixin from '@/lib/components/mixin.js'
  //
  import html from '@/lib/prototypes/html.js'
  //
  import MixinOut from './mixin-out'

  export default {
    mixins: [mixin, MixinOut],
    name: '',
    data() {
      return {
        html,
        //
        checkedNodes: [],
        parseData: [],
        filterText: '',
        inSelectKey: '',
        inCheckKeys: [],
      }
    },
    computed: {
      // 展开节点输出
      parsedDefaultExpandedKeys() {
        // 如果指定了，以制定为住。
        // 没有指定，取expand-level指定的。
        let keys = this.expandKeys.length > 0 ? this.expandKeys : this.getNodeByLevel(this.parseData, this.expandLevel)
        if (!this.multiple) {
          if (this.inSelectKey) {
            // 单选+存在指定选中key，则展开此节点
            keys.push(this.inSelectKey)
          }
        }
        return keys
      },
    },
    watch: {
      data() {
        // 复制+处理数据源
        this.parseData = this.copyData(this.data, [], !this.updating)
      },
      filterText(val) {
        // element组件方法过滤节点
        this.$refs.tree.filter(val)
        this.updateLeafStatus(this.$refs.tree.root.childNodes)
      },
      selectKey() {
        this.inSelectKey = this.selectKey
      },
      inSelectKey() {
        if (!this.multiple) {
          // 单选节点
          this.selectNode()
        }
      },
      checkKeys() {
        this.inCheckKeys = this.checkKeys
      },
      inCheckKeys() {
        if (this.multiple) {
          // 多选节点
          this.checkNode()
        }
      },
      parseData() {
        if (this.updating) {
          // 编辑状况下
          if (this.multiple) {
            // 单选节点
            this.checkNode()
          } else {
            // 多选节点
            this.selectNode()
          }
        } else {
          // 非编辑模式下，特殊处理：
          // 1. 选中全部节点
          this.checkedNodes = this.parseData

          // 2. 按配置输出指定数据结构
          let changeData = this.createData(this.checkedNodes)
          // 3. 返回更新
          if (this.multiple) {
            this.$emit('on-change', null, changeData, this.checkedNodes)
          } else {
            this.$emit('on-change', null, changeData)
          }
          // 4. 更新model数据
          this.$emit('change', changeData)
          this.formItemTrigger()
        }
      },
    },
    created() {},
    async mounted() {
      this.inSelectKey = this.selectKey

      // 异步
      // 复制+处理数据源
      this.parseData = this.copyData(this.data, [], !this.updating)
      if (this.async) {
        // 监听控件内部树节点变化
        this.$watch(
          () => {
            return this.$refs.tree.root.childNodes
          },
          (val) => {
            this.updateLeafStatus(val)
          }
        )
      }
    },
    beforeDestroy() {},
    methods: {
      // 单选节点
      selectNode() {
        if (this.updating) {
          // 编辑状态下
          // 找到需要选择的节点
          let node = this.findByKey(this.parseData, this.inSelectKey)
          // 选中节点已经不存在
          if (!node) {
            this.inSelectKey = ''
          }
          // 输出数据
          this.onSelectNode(node)
          this.$nextTick(() => {
            // element组件方法选择节点
            this.$refs.tree.setCurrentKey(node ? this.inSelectKey : '')
          })
          // debugger
          // // 编辑状态下
          // // 找到需要选择的节点
          // let node = this.$refs.tree.getNode(this.inSelectKey)
          // // 选中节点已经不存在
          // if (!node) {
          //   this.inSelectKey = ''
          // }
          // // 输出数据
          // this.onSelectNode(node ? node.data : null)
          // this.$nextTick(() => {
          //   // element组件方法选择节点
          //   this.$refs.tree.setCurrentKey(node ? this.inSelectKey : '')
          // })
        }
      },
      // 多选节点
      checkNode() {
        if (this.updating) {
          // 编辑状态下
          // element组件方法选中节点
          this.$refs.tree.setCheckedKeys(this.inCheckKeys)
          this.$nextTick(() => {
            // 输出数据
            this.onSelectNode()
          })
        }
      },
      // 通过key找节点
      findByKey(nodes = [], key = '', node = null) {
        if (!node) {
          for (let i = 0; i < nodes.length; i++) {
            if (nodes[i].id === key) {
              return nodes[i]
            } else {
              node = this.findByKey(nodes[i].children, key, node)
            }
          }
        }
        return node
      },
      // 获取指定层级的节点
      getNodeByLevel(nodes, level, value = []) {
        if (level) {
          nodes.forEach((n) => {
            value.push(n.id)
            if (level > 1) {
              this.getNodeByLevel(n.children, level - 1, value)
            }
          })
        }
        return value
      },
      // 过滤节点逻辑
      filterNode(value, data) {
        if (!value) return true
        return data.label.indexOf(value) !== -1
      },
      // 复制数据源
      copyData(data = [], checkList = [], disabled = false, labels = [], ids = []) {
        let copy = []
        data.forEach((d) => {
          let node = Object.assign({}, d, {
            labels: labels.concat([d.label]),
            ids: ids.concat([d.id]),
          })
          // let keys = Object.keys(d)
          // let unEnumerables = Object.getOwnPropertyNames(d).reduce(function (prev, cur) {
          //   keys.indexOf(cur) === -1 && prev.push(cur)
          //   return prev
          // }, [])
          // unEnumerables.forEach((o) => {
          //   Object.defineProperty(node, o, {
          //     value: d[o],
          //     writable: true,
          //     enumerable: false,
          //   })
          // })
          node.disabled = disabled
          if (checkList.includes(node.id)) {
            Object.defineProperty(node, 'checked', {
              value: true,
              enumerable: false,
            })
          }
          node.children = this.copyData(d.children, checkList, disabled, node.labels, node.ids)
          copy.push(node)
        })
        return copy
      },
      // 选择操作+输出数据
      onSelectNode(selectNode, ...other) {
        this.$emit('node-click', selectNode, ...other)
        // 更新选责keys
        this.inSelectKey = selectNode ? selectNode.id : ''
        if (this.updating) {
          let oldNodes = this.checkedNodes
          let newNodes = []
          if (this.multiple) {
            // 多选
            newNodes = this.getCheckedNodes()
          } else {
            // 单选
            newNodes = selectNode ? [selectNode] : []
          }

          // 判断选择是否发生变化
          let same = this.compareCheckedNodes(oldNodes, newNodes)

          if (!same) {
            this.checkedNodes = newNodes
            // 更新选中keys
            this.inCheckKeys = this.checkedNodes.map((o) => o.id)
            // 按配置输出指定数据结构
            let changeData = this.createData(this.checkedNodes)
            // 返回更新
            if (this.multiple) {
              this.$emit('on-change', selectNode, changeData, this.checkedNodes)
            } else {
              this.$emit('on-change', selectNode, changeData)
            }
            // 更新model数据
            this.$emit('change', changeData)
            this.formItemTrigger()
          }
        }
        // 点击展开异步加载
        if (this.async) {
          this.onNodeExpand(...arguments)
        }
      },
      // 按配置输出指定数据结构
      createData(checkeds) {
        // 复制输入data，并赋予选择状态
        let copy = this.copyData(
          this.data,
          checkeds.map((o) => o.id)
        )
        let changeData = []
        let isId = this.modelType === 'id'
        if (this.multiple) {
          // 多选
          if (this.modelMode === 'node') {
            // 输出选中节点
            changeData = isId ? checkeds.map((o) => o.id) : checkeds
          } else if (this.modelMode === 'leaf') {
            // 输出选中节点的所有叶节点
            changeData = this.parseCheckedLeaves(checkeds, isId)
          } else if (this.modelMode === 'all') {
            // 输出选中节点的相关节点
            changeData = this.parseCheckedAll(copy, checkeds, isId)
          } else if (this.modelMode === 'tree') {
            // 输出选中节点的树结构
            changeData = this.parseCheckedTree(copy)
          } else if (this.modelMode === 'gather') {
            changeData = this.parseCheckedGather(copy, isId)
          }
        } else {
          // 单选
          if (this.modelMode === 'node') {
            // 输出选中节点
            changeData = isId ? checkeds.map((o) => o.id) : checkeds
          } else if (this.modelMode === 'leaf') {
            // 输出选中节点的所有叶节点
            changeData = this.parseSelectLeaves(checkeds, isId)
          } else if (this.modelMode === 'route') {
            // 输出选中节点的途径节点
            changeData = this.parseSelectRoute(copy, checkeds, isId)
          } else if (this.modelMode === 'all') {
            // 输出选中节点的相关节点
            changeData = this.parseSelectAll(copy, checkeds, isId)
          } else if (this.modelMode === 'tree') {
            // 输出选中节点的树结构
            changeData = this.parseSelectTree(copy)
          }
        }
        return changeData
      },
      // element组件方法获取选中节点
      getCheckedNodes() {
        return this.$refs.tree.getCheckedNodes()
      },
      // 输出选中节点的所有叶节点 - 单选
      parseSelectLeaves(nodes, isId, value = []) {
        nodes.forEach((n) => {
          if (n.children && n.children.length > 0) {
            this.parseSelectLeaves(n.children, isId, value)
          } else {
            value.push(isId ? n.id : n)
          }
        })
        return value
      },
      // 输出选中节点的所有叶节点 - 多选
      parseCheckedLeaves(nodes, isId) {
        let ns = nodes.filter((o) => o.children.length === 0)
        return isId ? ns.map((o) => o.id) : ns
      },
      // 输出途径节点 - 单选
      parseSelectRoute(data, nodes, isId, value = { parents: [] }) {
        for (let i = 0; i < data.length; i++) {
          value.parents.push(isId ? data[i].id : data[i])
          let finded = false
          for (let j = 0; j < nodes.length; j++) {
            if (data[i].id === nodes[j].id) {
              finded = true
            }
          }

          if (finded) {
            return value.parents
          } else {
            let find = this.parseSelectRoute(data[i].children instanceof Array ? data[i].children : [], nodes, isId, value)
            if (!find || find.length === 0) {
              value.parents.pop()
            } else {
              return find
            }
          }
        }
        return []
      },
      // 输出途径节点+叶节点 - 单选
      parseSelectAll(data, nodes, isId) {
        let route = this.parseSelectRoute(data, nodes, isId)
        let leaves = this.parseSelectLeaves(nodes, isId)
        let set = new Set()
        let map = new Map()
        route.forEach((o) => {
          if (isId) {
            set.add(o)
          } else {
            map.set(o.id, o)
          }
        })
        leaves.forEach((o) => {
          if (isId) {
            set.add(o)
          } else {
            map.set(o.id, o)
          }
        })

        return isId ? [...set] : [...map.values()]
      },
      // 输出途径节点+叶节点 - 多选
      parseCheckedAll(data, nodes, isId) {
        let set = new Set()
        let map = new Map()
        for (let i = 0; i < nodes.length; i++) {
          this.parseSelectRoute(data, [nodes[i]], isId).forEach((n) => {
            if (isId) {
              set.add(n)
            } else {
              map.set(n.id, n)
            }
          })
        }
        return isId ? [...set] : [...map.values()]
      },
      // 输出树节点 - 单选
      parseSelectTree(data) {
        return this.parseCheckedTree(data)
      },
      // 输出树节点 - 多选
      parseCheckedTree(data) {
        let nodes = []
        for (let i = 0; i < data.length; i++) {
          if (data[i].checked) {
            nodes.push(data[i])
          } else {
            let find = this.parseCheckedTreeInner(data[i], data[i].children)
            if (find.children.length > 0) {
              nodes.push(find)
            }
          }
        }
        return nodes
      },
      // 输出树节点 - 多选（递归）
      parseCheckedTreeInner(parent, children = []) {
        let cp = Object.assign({}, parent, { children: [] })
        children.forEach((child) => {
          if (child.checked) {
            cp.children.push(child)
          } else {
            let find = this.parseCheckedTreeInner(child, child.children)
            if (find.children.length > 0) {
              cp.children.push(find)
            }
          }
        })
        return cp
      },
      // 输出排除全选子节点 - 多选
      parseCheckedGather(data = [], isId, value = []) {
        for (let i = 0; i < data.length; i++) {
          if (data[i].checked) {
            value.push(isId ? data[i].id : data[i])
          } else {
            this.parseCheckedGather(data[i].children, isId, value)
          }
        }
        return value
      },
      // 判断选择是否发生变化
      compareCheckedNodes(oldNodes, newNodes) {
        let oldKeys = oldNodes
          .map((o) => o.id)
          .sort((l, r) => l === r)
          .join(',')
        let newKeys = newNodes
          .map((o) => o.id)
          .sort((l, r) => l === r)
          .join(',')
        return oldKeys === newKeys
      },
      // 展开节点
      async onNodeExpand(data, node, ref) {
        if (data && node && ref && this.async && data.async) {
          try {
            node.loading = true
            let nodes = await this.loader(data)
            nodes.forEach((o) => {
              this.$refs.tree.append(o, node)
            })
            this.updateLeafStatus(node.childNodes)
            this.$refs.tree.setChecked(data.id, node.checked, true)
            data.async = false
            node.loading = false
          } catch (e) {
            node.loading = false
            node.expanded = false
          }
        }
      },
      updateLeafStatus(children) {
        children.forEach((o) => {
          if (!o.data.isLeaf && o.data.async) {
            o.isLeaf = false
            o.expanded = false
          }
          this.updateLeafStatus(o.childNodes)
        })
      },
      getNode(...p) {
        return this.$refs.tree.getNode(...p)
      },
    },
    filters: {},
    inject: [],
  }
</script>

<style lang="less">
  .p-tree-select {
    display: flex;
    flex-direction: column;
    flex-grow: 1;
    height: 500px;
    .p-tree-select__filter {
      flex-shrink: 0;
      border-bottom: 1px solid #efefef99;
      .el-input__inner {
        height: 32px;
        line-height: 32px;
        // background: rgba(247, 248, 249, 1);
        border: none;
      }
      .p-tree-select__filter__icon {
        line-height: 32px;
      }
    }
    .el-tree {
      flex-grow: 1;
      height: 0;
      overflow: auto;
      & > .el-tree-node {
        &:first-child {
          margin-top: 12px;
        }
      }
      .el-tree-node__content {
        display: flex;
        align-items: center;
        height: auto;
      }
      .icon {
        font-family: element-icons !important;
        font-weight: 400;
        display: inline-block;
        vertical-align: baseline;
        line-height: 1;

        -webkit-font-smoothing: antialiased;
        font-style: normal;
        font-variant: normal;
        speak: none;
        text-transform: none;
      }
      .el-tree__empty-block {
        .icon();
        text-align: left;
        height: 72px;
        min-height: auto;
        padding: 23px;
        background: #fff3f1;
        border: 1px solid #feeae7;
        border-radius: 2px;
        &:before {
          content: '\e79d';
          font-size: 14px;
          color: #ff8e70;
          position: relative;
          display: inline-block;
          margin-right: 6px;
        }
        .el-tree__empty-text {
          position: relative;
          left: auto;
          left: auto;
          transform: none;
        }
      }
      .el-tree__empty-text {
        font-size: 14px;
        position: static;
        top: auto;
        left: auto;
      }
      .p-tree-select__node {
        text-align: left;
        cursor: pointer;
        display: flex;
        align-items: center;
        width: 100%;
        & > * {
          display: flex;
          align-items: center;
        }
        .el-tree-node__label {
        }
        b {
          color: rgb(var(--color-success));
        }
        label {
          cursor: pointer;
        }
      }
    }
  }
</style>
