<template>
  <div class="app-container">
    <el-card>
      <div slot="header" class="clearfix">
        <span>权限路由</span>
      </div>

      <div class="filter-container">
        <el-button
          v-if="showDeletebtn"
          class="filter-item"
          type="danger"
          style="margin-right: 10px;"
          :disabled="!checkPermission(['larke-admin.auth-rule.clear'])"
          @click="handleDeleteList"
        >
          删除选中
        </el-button>

        <el-input v-model="listQuery.searchword" placeholder="请输入关键字" clearable style="width: 200px;margin-right: 10px;" class="filter-item" @keyup.enter.native="handleFilter" />

        <el-select v-model="listQuery.method" placeholder="请求方式" clearable style="width: 120px;margin-right: 10px;" class="filter-item">
          <el-option v-for="item in methodOptions" :key="item.key" :label="item.label" :value="item.key" />
        </el-select>

        <el-select v-model="listQuery.status" placeholder="状态" clearable class="filter-item" style="width: 80px;margin-right: 10px;">
          <el-option v-for="item in statusOptions" :key="item.key" :label="item.display_name" :value="item.key" />
        </el-select>

        <el-select v-model="listQuery.order" style="width: 80px;margin-right: 10px;" class="filter-item" @change="handleFilter">
          <el-option v-for="item in sortOptions" :key="item.key" :label="item.label" :value="item.key" />
        </el-select>

        <el-button v-waves class="filter-item" type="primary" icon="el-icon-search" @click="handleFilter">
          {{ $t('table.search') }}
        </el-button>

        <el-button :disabled="!checkPermission(['larke-admin.auth-rule.create'])" class="filter-item" type="primary" icon="el-icon-edit" @click="handleCreate">
          {{ $t('table.add') }}
        </el-button>

        <el-button v-permission="['larke-admin.auth-rule.index-tree']" class="filter-item" icon="tree" @click="handleTree">
          权限结构
        </el-button>
      </div>

      <el-table
        v-loading="listLoading"
        :header-cell-style="{background:'#eef1f6',color:'#606266'}"
        :data="list"
        border
        fit
        highlight-current-row
        style="width: 100%"
        @selection-change="handleSelectionChange"
      >
        <el-table-column
          type="selection"
          width="55"
          align="center"
        />

        <el-table-column width="150px" label="名称">
          <template slot-scope="scope">
            <span>{{ scope.row.title }}</span>
          </template>
        </el-table-column>

        <el-table-column min-width="100px" label="链接">
          <template slot-scope="{row}">
            <div>
              <el-tag type="info" size="mini" style="margin-bottom:3px;" @click="handleClipboard(row.slug, $event)">
                <el-tooltip placement="top">
                  <div slot="content">
                    点击复制 {{ row.slug }}
                  </div>
                  <div class="slug-item">
                    {{ row.slug }}
                  </div>
                </el-tooltip> 
              </el-tag>
            </div>

            <div>
              <el-tag :type="row.method | methodFilter" size="mini">
                {{ row.method }}
              </el-tag>

              <span style="margin-left: 5px;">{{ row.url }}</span>
            </div>
          </template>
        </el-table-column>

        <el-table-column width="75px" align="center" label="排序">
          <template slot-scope="{row, $index}">
            <div @click.stop="{{editableChangeBtn($index, 'editListorderInput')}}">
              <el-input
                v-if="editable[$index]"
                v-model="row.listorder"
                size="mini"
                class="editListorderInput"
                :disabled="!checkPermission(['larke-admin.auth-rule.listorder'])"
                @blur="editableChange($event, row, $index)"
              />
              <span v-else>{{ row.listorder }}</span>
            </div>
          </template>
        </el-table-column>

        <el-table-column width="160px" align="center" label="添加时间">
          <template slot-scope="scope">
            <span>{{ scope.row.create_time | parseTime('{y}-{m}-{d} {h}:{i}:{s}') }}</span>
          </template>
        </el-table-column>

        <el-table-column class-name="status-col" label="状态" width="80">
          <template slot-scope="scope">
            <el-switch
              v-model="scope.row.status"
              active-color="#13ce66"
              inactive-color="#ff4949"
              :active-value="1"
              :inactive-value="0"
              :disabled="!checkPermission(['larke-admin.auth-rule.enable', 'larke-admin.auth-rule.disable'])"
              @change="changeStatus($event, scope.row, scope.$index)"
            />
          </template>
        </el-table-column>

        <el-table-column align="center" label="操作" width="280">
          <template slot-scope="scope">
            <el-button :disabled="!checkPermission(['larke-admin.auth-rule.update'])" type="primary" size="mini" icon="el-icon-edit" @click="handleEdit(scope.$index, scope.row)">
              编辑
            </el-button>

            <el-button :disabled="!checkPermission(['larke-admin.auth-rule.detail'])" type="info" size="mini" icon="el-icon-info" @click="handleDetail(scope.$index, scope.row)">
              详情
            </el-button>

            <el-button v-permission="['larke-admin.auth-rule.delete']" type="danger" size="mini" icon="el-icon-delete" @click="handleDelete(scope.$index, scope.row)">
              删除
            </el-button>
          </template>
        </el-table-column>
      </el-table>

      <pagination v-show="total>0" :total="total" :page.sync="listQuery.page" :limit.sync="listQuery.limit" @pagination="getList" />
    </el-card>

    <el-dialog title="添加权限" :visible.sync="create.dialogVisible">
      <create :item="create" />
    </el-dialog>

    <el-dialog title="编辑权限" :visible.sync="edit.dialogVisible" @close="closeEdit">
      <edit :item="edit" />
    </el-dialog>

    <el-dialog title="权限详情" :visible.sync="detail.dialogVisible">
      <detail :data="detail.data" />
    </el-dialog>
  </div>
</template>

<script>
import md5 from 'js-md5'
import waves from '@/directive/waves' // waves directive
import clipboard from '@/utils/clipboard'
import permission from '@/directive/permission/index.js' // 权限判断指令
import checkPermission from '@/utils/permission' // 权限判断函数
import { parseTime } from '@/utils'
import Pagination from '@/components/Pagination' // Secondary package based on el-pagination
import Detail from '@/components/Larke/Detail'
import Edit from './components/Edit'
import Create from './components/Create'
import {
  getRuleList,
  getRuleTreeList,
  getRuleChildrenList,
  getRuleDetail,
  createRule,
  updateRule,
  deleteRule,
  clearRule,
  updateRuleSort,
  enableRule,
  disableRule
} from '@/api/authRule'

export default {
  name: 'AuthRuleIndex',
  components: { Pagination, Detail, Edit, Create },
  directives: { waves, permission },
  filters: {
    methodFilter(method) {
      const methodMap = {
        'GET': 'success',
        'HEAD': 'info',
        'POST': 'warning',
        'PUT': 'warning',
        'DELETE': 'danger',
        'PATCH': 'warning',
        'OPTIONS': 'info'
      }
      return methodMap[method]
    }
  },
  data() {
    return {
      list: null,
      total: 0,
      listLoading: true,
      listQuery: {
        searchword: '',
        order: 'ASC',
        method: '',
        status: '',
        page: 1,
        limit: 10
      },
      methodOptions: [
        { label: 'GET', key: 'GET' },
        { label: 'HEAD', key: 'HEAD' },
        { label: 'POST', key: 'POST' },
        { label: 'PUT', key: 'PUT' },
        { label: 'DELETE', key: 'DELETE' },
        { label: 'PATCH', key: 'PATCH' },
        { label: 'OPTIONS', key: 'OPTIONS' }
      ],
      statusOptions: [
        { key: 'open', display_name: '启用' },
        { key: 'close', display_name: '禁用' }
      ],
      sortOptions: [
        { label: '正序', key: 'ASC' },
        { label: '倒叙', key: 'DESC' }
      ],
      detail: {
        dialogVisible: false,
        data: []
      },
      create: {
        dialogVisible: false
      },
      edit: {
        dialogVisible: false,
        id: ''
      },
      editable: [],
      editableItem: {},
      editableOldSort: 0,
      selectedData: [],
      showDeletebtn: false
    }
  },
  created() {
    this.getList()
  },
  methods: {
    checkPermission,
    handleClipboard(text, event) {
      clipboard(text, event)
    },
    getList() {
      this.listLoading = true
      getRuleList({
        searchword: this.listQuery.searchword,
        order: this.listQuery.order,
        method: this.listQuery.method,
        status: this.listQuery.status,
        start: (this.listQuery.page - 1) * this.listQuery.limit,
        limit: this.listQuery.limit
      }).then(response => {
        this.list = response.data.list
        this.total = response.data.total
        this.listLoading = false
      })
    },
    handleFilter() {
      this.listQuery.page = 1
      this.getList()
    },
    handleSelectionChange(data, key) {
      this.selectedData = []
      data.forEach(element => {
        this.selectedData.push(element.id)
      })

      if (this.selectedData.length > 0) {
        this.showDeletebtn = true
      } else {
        this.showDeletebtn = false
      }
    },
    handleCreate() {
      this.create.dialogVisible = true
    },
    handleEdit(index, row) {
      this.edit.dialogVisible = true
      this.edit.id = row.id
    },
    closeEdit() {
      this.edit.id = ''
    },
    editableChangeBtn(index, className) {
      this.editable = new Array(this.list.length)

      this.editable[index] = true

      this.editableItem = this.list[index]

      this.$set(this.editable, index, true)

      // 让input自动获取焦点
      this.$nextTick(function() {
        var editInputList = document.getElementsByClassName(className)
        editInputList[0].children[0].focus()
      })
    },
    editableChange(e, data, index) {
      this.editable[index] = false

      if (this.editableOldSort == data.listorder) {
        return
      }

      this.editableOldSort = data.listorder

      updateRuleSort(data.id, data.listorder).then(() => {
        this.$message({
          message: '权限排序成功',
          type: 'success',
          duration: 2 * 1000
        })
      })
    },
    handleDetail(index, row) {
      getRuleDetail(row.id).then((res) => {
        this.detail.dialogVisible = true
        const data = res.data

        this.detail.data = [
          {
            name: 'ID',
            content: data.id,
            type: 'text'
          },
          {
            name: '父级ID',
            content: data.parentid,
            type: 'text'
          },
          {
            name: '名称',
            content: data.title,
            type: 'text'
          },
          {
            name: '权限链接',
            content: data.url,
            type: 'text'
          },
          {
            name: '请求类型',
            content: data.method,
            type: 'text'
          },
          {
            name: '地址标识',
            content: data.slug,
            type: 'text'
          },
          {
            name: '描述',
            content: data.description,
            type: 'text'
          },
          {
            name: '排序',
            content: data.listorder,
            type: 'text'
          },
          {
            name: '验证权限',
            content: data.is_need_auth,
            type: 'status'
          },
          {
            name: '状态',
            content: data.status,
            type: 'boolen'
          },
          {
            name: '更新时间',
            content: data.update_time,
            type: 'time'
          },
          {
            name: '更新IP',
            content: data.update_ip,
            type: 'text'
          },
          {
            name: '添加时间',
            content: data.create_time,
            type: 'time'
          },
          {
            name: '添加IP',
            content: data.create_ip,
            type: 'text'
          }
        ]
      })
    },
    handleTree() {
      this.$router.replace('/auth/rule/tree')
    },
    changeStatus(e, data, index) {
      if (data.status == 1) {
        enableRule(data.id).then(() => {
          this.$message({
            message: '权限启用成功',
            type: 'success',
            duration: 2 * 1000
          })
        })
      } else {
        disableRule(data.id).then(() => {
          this.$message({
            message: '权限禁用成功',
            type: 'success',
            duration: 2 * 1000
          })
        })
      }
    },
    handleDelete(index, row) {
      const thiz = this
      this.$confirm('确认要删除该权限吗？', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        deleteRule(row.id).then(() => {
          this.$message({
            message: '删除权限成功',
            type: 'success',
            duration: 5 * 1000,
            onClose() {
              thiz.list.splice(index, 1)
            }
          })
        })
      }).catch(() => {

      })
    },
    handleDeleteList() {
      this.$confirm('确认要删除选中的权限吗？', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        if (this.selectedData.length < 1) {
          this.$message({
            message: '请选择要删除的权限',
            type: 'error',
            duration: 3 * 1000
          })
          return
        }

        const thiz = this
        clearRule({
          ids: this.selectedData.join(',')
        }).then(res => {
          this.$message({
            message: res.message,
            type: 'success',
            duration: 3 * 1000,
            onClose() {
              for (let i = thiz.list.length - 1; i >= 0; i--) {
                if (thiz.selectedData.includes(thiz.list[i].id)) {
                  thiz.list.splice(i, 1)
                }
              }
            }
          })
        })
      }).catch(() => {

      })
    }

  }
}
</script>

<style scoped>
.pagination-container {
  padding: 5px 2px;
}
.edit-input {
  padding-right: 100px;
}
.cancel-btn {
  position: absolute;
  right: 15px;
  top: 10px;
}
.slug-item {
  cursor: pointer;
}
</style>
