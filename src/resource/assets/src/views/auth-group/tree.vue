<template>
  <div class="app-container">
    <el-card>
      <div slot="header" class="clearfix">
        <span>分组结构</span>
      </div>

      <div class="filter-container">
        <el-button :disabled="!checkPermission(['larke-admin.auth-group.create'])" class="filter-item" type="primary" icon="el-icon-edit" @click="handleCreate">
          添加分组
        </el-button>

        <el-button class="filter-item" icon="tree" @click="handleIndex">
          全部分组
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
      >

        <el-table-tree-column
          :expand-all="treeExpandAll"
          file-icon="el-icon-document"
          folder-icon="el-icon-folder"
          prop="title"
          level-key="depth"
          parent-key="parentid"
          tree-key="id"
          :show-overflow-tooltip="true"
          :indent-size="25"
          label="名称"
          min-width="250"
          class-name="larke-admin-auth-group-tree"
          header-align="left"
        />

        <el-table-column width="60px" align="center" label="排序">
          <template slot-scope="scope">
            <span>{{ scope.row.listorder }}</span>
          </template>
        </el-table-column>

        <el-table-column width="100px" align="center" label="授权">
          <template slot-scope="scope">
            <el-button :disabled="!checkPermission(['larke-admin.auth-group.access'])" type="warning" size="mini" @click="handleAccess(scope.$index, scope.row)">
              授权
            </el-button>
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
              :disabled="!checkPermission(['larke-admin.auth-group.enable', 'larke-admin.auth-group.disable'])"
              @change="changeStatus($event, scope.row, scope.$index)"
            />
          </template>
        </el-table-column>

        <el-table-column align="center" label="操作" width="280">
          <template slot-scope="scope">
            <el-button :disabled="!checkPermission(['larke-admin.auth-group.update'])" type="primary" size="mini" icon="el-icon-edit" @click="handleEdit(scope.$index, scope.row)">
              编辑
            </el-button>

            <el-button :disabled="!checkPermission(['larke-admin.auth-group.detail'])" type="info" size="mini" icon="el-icon-info" @click="handleDetail(scope.$index, scope.row)">
              详情
            </el-button>

            <el-button v-permission="['larke-admin.auth-group.delete']" type="danger" size="mini" icon="el-icon-delete" @click="handleDelete(scope.$index, scope.row)">
              删除
            </el-button>
          </template>
        </el-table-column>
      </el-table>
    </el-card>

    <el-dialog title="添加分组" :visible.sync="create.dialogVisible">
      <create :item="create" />
    </el-dialog>

    <el-dialog title="编辑分组" :visible.sync="edit.dialogVisible">
      <edit :item="edit" />
    </el-dialog>

    <el-dialog title="分组详情" :visible.sync="detail.dialogVisible">
      <detail :data="detail.data" />
    </el-dialog>

    <el-dialog title="分组授权" :visible.sync="access.dialogVisible">
      <access :item="access" />
    </el-dialog>
  </div>
</template>

<script>
import md5 from 'js-md5'
import waves from '@/directive/waves' // waves directive
import permission from '@/directive/permission/index.js' // 权限判断指令
import checkPermission from '@/utils/permission' // 权限判断函数
import { parseTime } from '@/utils'
import Detail from '@/components/Larke/Detail'
import Edit from './components/Edit'
import Create from './components/Create'
import Access from './components/Access'
import {
  getGroupTreeList,
  getGroupDetail,
  deleteGroup,
  updateGroupSort,
  enableGroup,
  disableGroup
} from '@/api/authGroup'

export default {
  name: 'AuthGroupTree',
  components: { Detail, Edit, Create, Access },
  directives: { waves, permission },
  filters: {
  },
  data() {
    return {
      list: null,
      listLoading: true,
      detail: {
        dialogVisible: false,
        data: []
      },
      treeExpandAll: false,
      create: {
        dialogVisible: false
      },
      edit: {
        dialogVisible: false,
        id: ''
      },
      access: {
        dialogVisible: false,
        id: ''
      }
    }
  },
  created() {
    this.getList()
  },
  methods: {
    checkPermission,
    getList() {
      this.listLoading = true

      getGroupTreeList().then(res => {
        this.listLoading = false
        this.list = res.data.list
      })
    },
    handleAccess(index, row) {
      this.access.id = row.id
      this.access.dialogVisible = true
    },
    handleDetail(index, row) {
      getGroupDetail(row.id).then((res) => {
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
    handleCreate() {
      this.create.dialogVisible = true
    },
    handleEdit(index, row) {
      this.edit.dialogVisible = true
      this.edit.id = row.id
    },
    handleIndex() {
      this.$router.replace('/auth/group/index')
    },
    changeStatus(e, data, index) {
      if (data.status == 1) {
        enableGroup(data.id).then(() => {
          this.$message({
            message: '分组启用成功',
            type: 'success',
            duration: 2 * 1000
          })
        })
      } else {
        disableGroup(data.id).then(() => {
          this.$message({
            message: '分组禁用成功',
            type: 'success',
            duration: 2 * 1000
          })
        })
      }
    },
    handleDelete(index, row) {
      const thiz = this
      this.$confirm('确认要删除该分组吗？', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        deleteGroup(row.id).then(() => {
          this.$message({
            message: '删除分组成功',
            type: 'success',
            duration: 5 * 1000,
            onClose() {
              thiz.list.splice(index, 1)
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
</style>
