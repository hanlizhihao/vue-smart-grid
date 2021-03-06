<template>
  <div class="smart-grid" :class="{[border]: true, loading, hoverable, selectable, multiple, inline: inlineMode}">
    <div class="hidden">
      <slot></slot>
    </div>
    <div class="layer"></div>
    <table>
      <thead v-if="!hiddenColumn">
        <tr>
          <th v-if="selectable && multiple" class="checkbox-cell">
            <label class="grid-checkbox">
              <span class="checkbox-wrap" :class="{checked: allChecked}" @click="handleAllCheck"></span>
            </label>
          </th>
          <th v-for="header in headers"
            v-if="!header.hidden"
            :style="header.style"
            :class="{'img-sort': isExistsSortIcons && header.sort, sort: header.sort, [header.sortDirection]: true}"
            @click="handleSort(header)">
            {{header.label}}
            <i v-if="isExistsSortIcons && header.sort" :class="sortIcons[header.sortDirection || 'sort']"></i>
            <span class="sort-place"></span>
          </th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(row, rowIndex) in innerData"
          v-if="showAllMore || rowIndex < showRows"
          :key="seq ? row.rowData[seq] : rowIndex"
          @click="handleRowCheck(row, true)"
          :class="[row.$checked ? checked : '', row.backgroundClass]"
          :style="{width: inlineWidth, backgroundColor: row.backgroundColor}"
          @dblclick="handleDblClick(row)">
          <td v-if="selectable && multiple" class="checkbox-cell">
            <label class="grid-checkbox">
              <span class="checkbox-wrap"
                :class="{checked: row.$checked}"
                @click.stop="handleRowCheck(row)"></span>
            </label>
          </td>
          <td v-for="(cell, cellIndex) in row.cells" v-if="!cell.hidden" :style="cell.style">
            <smart-grid-cell
              :row-index="rowIndex"
              :cell-index="cellIndex"
              :row-data="row.rowData"
              :code="cell.code"
              :label="cell.label"
              :valueset="cell.valueset"
              :template-slot-fn="cell.defaultSlotFn"/>
          </td>
        </tr>
        <tr v-if="innerData.length >= showRows">
          <td :colspan="cellSize" @click="handleShowMore" class="show-more">{{showAllMore ? '↑' : '↓'}}more</td>
        </tr>
        <tr v-if="cellSize && empty">
          <td :colspan="cellSize"><slot name="empty"></slot></td>
        </tr>
      </tbody>
    </table>
    <smart-grid-pagination v-if="pageable"
      :config="config"
      :custom-template="false"
      :template-slot-fn="pagination.defaultSlotFn"
      :pagination="data"
      :event-hub="eventHub"
      :inner-event-hub="innerEventHub"
      :show-pages="showPages"
      :sizes="sizes"
      @sort-change="handleSortChange"
      @size-change="params => $emit('size-change', params)"
      @page-change="params => $emit('page-change', params)"
      @pagination-change="handlePaginationChange"
      @reload="handleReload">
    </smart-grid-pagination>
  </div>
</template>

<script>
import EventEmitter from 'event-emitter'
import '../assets/styles/main.less'
import {isObject, isEmptyObject} from '../libs/lang'
import {config} from './index'
import SmartGridCell from './SmartGridCell'
import SmartGridPagination from './SmartGridPagination'
import Vue from 'vue'

export default {
  name: 'smart-grid',
  props: {
    data: [Object, Array],
    hoverable: {
      type: Boolean,
      default: true
    },
    selectable: {
      type: Boolean,
      default: true
    },
    multiple: {
      type: Boolean,
      default: true
    },
    border: {
      type: String,
      default: 'xy'
    },
    loading: {
      type: Boolean,
      default: false
    },
    hiddenColumn: {
      type: Boolean,
      default: false
    },
    showRows: {
      type: Number,
      default: 100
    },
    inlineRows: {
      type: Number,
      default: 0
    },
    sortIcons: {
      type: Object,
      default() {
        return {}
      }
    },
    defaultDescDirection: {
      type: Boolean,
      default: true
    },
    dataConfig: {
      type: Object,
      default() {
        return {}
      }
    },
    seq: String,
    eventHub: Object,
    showPages: Number,
    sizes: Array,
    backgroundClassCode: {
      type: String,
      default: ''
    },
    backgroundColorCode: {
      type: String,
      default: ''
    }
  },
  data() {
    return {
      pageable: false,
      inlineMode: !!this.inlineRows,
      inlineWidth: this.inlineRows ? (100 / this.inlineRows + '%') : '',
      headers: [],
      pagination: {
        defaultSlotFn: null
      },
      innerData: [],
      cellSize: 0,
      empty: false,
      showAllMore: false,
      innerEventHub: EventEmitter({}),
      config: {...config, ...this.dataConfig}
    }
  },
  computed: {
    allChecked() {
      return this.innerData.length && this.innerData.every(({$checked}) => $checked)
    },
    isExistsSortIcons() {
      return !isEmptyObject(this.sortIcons)
    }
  },
  created() {
    this.parseData()
  },
  watch: {
    data() {
      this.parseData()
    }
  },
  methods: {
    parseData() {
      let innerData = this.data || []
      if (isObject(innerData)) {
        this.pageable = true
        innerData = innerData[this.config.dataNode] || []
      }
      this.innerData = innerData.map(row => {
        let innerItem = {
          rowData: row,
          backgroundColor: '',
          backgroundClass: '',
          $checked: false,
          cells: this.headers
        }
        /**
         * smartgrid支持根据数组数据的其中一个元素的字段来设置背景颜色
         * @update lizhihan1 2018.03.01
         * @type {string[]}
         */
        let keys = Object.keys(row)
        if (keys.includes(this.backgroundClassCode)) {
          innerItem.backgroundClass = row[this.backgroundClassCode]
        }
        if (keys.includes(this.backgroundColorCode)) {
          innerItem.backgroundColor = row[this.backgroundColorCode]
        }
        if (row.$checked !== undefined) {
          innerItem.$checked = row.$checked
        }
        return innerItem
      })
      this.empty = !this.innerData.length
    },
    handleAllCheck() {
      const checked = !this.allChecked
      this.innerData.forEach(item => {
        item.$checked = checked
      })
      this.$emit('all-select', checked)
    },
    handleRowCheck(row, onlyChecked = false) {
      this.handleClick(row)
      if (!this.selectable) {
        return
      }
      if (onlyChecked) {
        if (this.multiple) {
          return
        }
        this.innerData.forEach(item => {
          item.$checked = false
        })
      }
      row.$checked = !row.$checked
      this.$emit('select', row.rowData, row.$checked)
    },
    handleDblClick(row) {
      this.$emit('dblclick', row.rowData)
    },
    handleClick(row) {
      this.$emit('click', row.rowData)
    },
    addHeader(header) {
      const {label, code, valueset, hidden, sort} = header
      this.headers.push({
        code,
        label,
        valueset,
        hidden: !!hidden,
        sort,
        sortDirection: '',
        style: this.extractHeaderStyle(header),
        defaultSlotFn: header.$scopedSlots ? header.$scopedSlots.default : null
      })
      // header.$destroy()
      this.calcExpandCellSize()
    },
    addCustomPagination(pagination) {
      this.pagination = {
        defaultSlotFn: pagination.$scopedSlots ? pagination.$scopedSlots.default : null
      }
      pagination.$destroy()
    },
    setHeaderHidden(header, hidden) {
      const {code} = header
      let index = this.headers.findIndex(item => item.code === code)
      let newItemHeader = this.headers[index]
      newItemHeader.hidden = !!hidden
      Vue.set(this.headers, index, newItemHeader)
      this.calcExpandCellSize()
    },
    extractHeaderStyle(header) {
      const {width, align} = header
      const style = {}
      if (width) {
        style.width = width
      }
      if (align) {
        style.textAlign = align
      }
      return style
    },
    calcExpandCellSize() {
      let cellSize = 0
      this.headers.forEach(({hidden}) => {
        if (!hidden) {
          cellSize++
        }
      })
      if (this.selectable && this.multiple) {
        cellSize++
      }
      this.cellSize = cellSize
    },
    handleShowMore() {
      this.showAllMore = !this.showAllMore
    },
    handleSort(header) {
      if (!header.sort) {
        return
      }
      this.headers.forEach(item => {
        if (item.sort) {
          if (header.code === item.code) {
            header.sortDirection = header.sortDirection === '' ? (this.defaultDescDirection ? 'desc' : 'asc') : header.sortDirection === 'desc' ? 'asc' : 'desc'
          } else {
            item.sortDirection = ''
          }
        }
      })
      this.innerEventHub.emit('sort-change')
    },
    fillEventParams(params) {
      if (!params) {
        return
      }
      const sortHeader = this.headers.filter(({sort, sortDirection}) => sort && sortDirection)[0]
      if (sortHeader) {
        const {code, sortDirection} = sortHeader
        params.sortCode = code
        params.sortDirection = sortDirection
      }
    },
    handleSortChange(params) {
      this.fillEventParams(params)
      this.$emit('sort-change', params)
    },
    handlePaginationChange(params) {
      this.fillEventParams(params)
      this.$emit('pagination-change', params)
    },
    handleReload(params) {
      this.fillEventParams(params)
      this.$emit('reload', params)
    },
    getCheckedRows() {
      return this.innerData.filter(({$checked}) => $checked).map(({rowData}) => rowData)
    },
    resetSortStatus() {
      this.headers.forEach(item => {
        item.sortDirection = ''
      })
    }
  },
  components: {
    SmartGridCell,
    SmartGridPagination
  }
}
</script>

<style lang="less">
.smart-grid {
  &.selectable:not(.multiple) {
    td {
      cursor: pointer;
    }
    tr.checked td {
      background-color: #eff7ff;
    }
  }
  &.hoverable tbody tr:hover td {
    transition: all .3s;
  }
  &.inline {
    tbody {
      tr {
        float: left;
        padding: 10px;
        td {
          display: block;
        }
      }
    }
  }
  &.none {
    td, th {
      border: none!important;
      border: none!important;
    }
  }
  &.x {
    td, th {
      border-left: none!important;
      border-right: none!important;
    }
  }
  &.y {
    td, th {
      border-top: none!important;
      border-bottom: none!important;
    }
  }
  table {
    border-spacing: 0;
    width: 100%;
  }
  thead + tbody tr:first-child td {
    border-top: 0;
  }
  th, td {
    padding: 8px 12px;
    &.checkbox-cell {
      width: 30px;
      text-align: center;
    }
    &:not(:first-child) {
      border-left: 0;
    }
    text-align: left;
  }
  th {
    background-color: #fbfaf7;
    border: 1px solid #e4e4dc;
    color: #333;
    font-size: 14px;
    .sort-place {
      display: none;
    }
    &.sort {
      cursor: pointer;
      transition: all .3s;
      &.asc, &.desc {
        background-color: #e7f6fd;
      }
      &:not(.img-sort) {
        &.asc .sort-place {
         background: url(../assets/images/asc.png) no-repeat center center;
        }
        &.desc .sort-place {
          background: url(../assets/images/desc.png) no-repeat center center;
        }
        .sort-place {
          display: inline-block;
          position: relative;
          top: 2px;
          width: 14px;
          height: 14px;
          background: url(../assets/images/sort.png) no-repeat center center;
          transition: all .3s;
        }
      }
    }
  }
  tr {
    &:last-child td {
      border-bottom: 1px solid #f2f1ec;
    }
  }
  td {
    color: #666;
    font-size: 12px;
    border: 1px solid #f2f1ec;
    border-bottom: 0;
    &>.empty-cell {
      display: inline-block;
      height: 14px;
      width: 14px;
    }
    &.show-more {
      text-align: center;
      cursor: pointer;
    }
  }
  .layer {
    position: absolute;
    top: 0;
    left: 0;
    bottom: 0;
    right: 0;
    background: url(../assets/images/loading.gif) no-repeat center center rgba(192, 192, 192, .5);
    z-index: -1;
    opacity: 0;
    text-align: center;
  }
  &.loading {
    position: relative;
    .layer {
      transition: all .3s;
      z-index: 1;
      opacity: 1;
    }
  }
  .hidden {
    display: none;
    visibility: hidden;
  }

  .grid-checkbox {
    display: inline-block;
    height: 14px;
    width: 14px;
    line-height: 14px;
    border-radius: 4px;
    border: 1px solid #e4e4dc;
    overflow: hidden;
    .checkbox-wrap {
      display: inline-block;
      position: relative;
      background-color: #fff;
      height: 100%;
      width: 100%;
      &::before, &::after {
        content: " ";
        display: inline-block;
        opacity: 0;
      }
      &::before {
        position: absolute;
        width: 5px;
        height: 10px;
        top: -1px;
        right: 2.5px;
        transform: rotate(45deg) scale(.7);
        border-right: 2px solid #fff;
        border-bottom: 2px solid #fff;
      }
      &::after {
        background-color: #0c8fd3;
        height: 100%;
        width: 100%;
        cursor: pointer;
        border-radius: 2px;
      }
      &.checked {
        &::before, &::after {
          transition: all .15s;
          opacity: 1;
        }
      }
    }
  }
}
</style>
