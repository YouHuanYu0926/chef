<template>
  <div class="dashboard-container">
    <div class="container">
      <div class="tableBar">
        <label style="margin-right: 10px">Dish Name：</label>
        <el-input v-model="input"
                  placeholder="Please enter dish name"
                  style="width: 14%"
                  clearable
                  @clear="init"
                  @keyup.enter.native="initFun" />
        <label style="margin-right: 10px; margin-left: 20px">Sale status：</label>
        <el-select v-model="dishStatus"
                   style="width: 14%"
                   placeholder="Please choose"
                   clearable
                   @clear="init">
          <el-option v-for="item in saleStatus"
                     :key="item.value"
                     :label="item.label"
                     :value="item.value" />
        </el-select>
        <el-button class="normal-btn continue"
                   @click="init(true)">
          Search
        </el-button>

        <div class="tableLab">
          <el-button type="primary"
                     style="margin-left: 15px"
                     @click="addDishtype('add')">
            + Add dish
          </el-button>
        </div>
      </div>
      <el-table v-if="tableData.length"
                :data="tableData"
                stripe
                class="tableBox"
                @selection-change="handleSelectionChange">
        <el-table-column type="selection"
                         width="25" />
        <el-table-column prop="name"
                         label="Dish Name" />
        <el-table-column prop="image"
                         label="Image">
          <template slot-scope="{ row }">
            <el-image style="width: 80px; height: 40px; border: none; cursor: pointer"
                      :src="row.image">
              <div slot="error"
                   class="image-slot">
                <img src="./../../assets/noImg.png"
                     style="width: auto; height: 40px; border: none">
              </div>
            </el-image>
          </template>
        </el-table-column>
        <el-table-column label="Price">
          <template slot-scope="scope">
            <span style="margin-right: 10px">￥{{ (scope.row.price ).toFixed(2)*100/100 }}</span>
          </template>
        </el-table-column>
        <el-table-column label="State">
          <template v-slot="scope">
            <div class="tableColumn-status"
                 :class="{ 'stop-use': String(scope.row.status) === '0' }">
              {{ String(scope.row.status) === '0' ? 'Stop' : 'Ready' }}
            </div>
          </template>
        </el-table-column>
        <el-table-column label="Operating"
                         width="250"
                         align="center">
          <template v-slot:="scope">
            <el-button type="text"
                       size="small"
                       class="blueBug"
                       @click="addDishtype(scope.row.id)">
              Modify
            </el-button>
            <el-button type="text"
                       size="small"
                       class="delBut"
                       @click="deleteHandle('Delete One', scope.row.id)">
              Delete
            </el-button>
            <el-button type="text"
                       size="small"
                       class="non"
                       :class="{
                         blueBug: scope.row.status == '0',
                         delBut: scope.row.status != '0'
                       }"
                       @click="statusHandle(scope.row)">
              {{ scope.row.status == '0' ? 'Ready' : '' }}
            </el-button>
          </template>
        </el-table-column>
      </el-table>
      <Empty v-else
             :is-search="isSearch" />
      <el-pagination v-if="counts > 10"
                     class="pageList"
                     :page-sizes="[10, 20, 30, 40]"
                     :page-size="pageSize"
                     layout="total, sizes, prev, pager, next, jumper"
                     :total="counts"
                     @size-change="handleSizeChange"
                     @current-change="handleCurrentChange" />
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator'
import HeadLable from '@/components/HeadLable/index.vue'
import {
  getDishPage,
  editDish,
  deleteDish,
  dishStatusByStatus,
  dishCategoryList
} from '@/api/dish'
import InputAutoComplete from '@/components/InputAutoComplete/index.vue'
import Empty from '@/components/Empty/index.vue'
import { baseUrl } from '@/config.json'

@Component({
  name: 'DishType',
  components: {
    HeadLable,
    InputAutoComplete,
    Empty
  }
})
export default class extends Vue {
  private input: any = ''
  private counts: number = 0
  private page: number = 1
  private pageSize: number = 10
  private checkList: string[] = []
  private tableData: [] = []
  private dishState = ''
  private dishStatus = ''
  private isSearch: boolean = false
  private saleStatus: any = [
    {
      value: 0,
      label: 'Stop'
    },
    {
      value: 1,
      label: 'Ready'
    }
  ]

  created() {
    this.init()
  }

  initFun() {
    this.page = 1
    this.init()
  }

  private async init(isSearch?) {
    this.isSearch = isSearch
    const chefId=localStorage.getItem('chefID')
    await getDishPage({
      page: this.page,
      pageSize: this.pageSize,
      name: this.input || undefined,
      status: this.dishStatus,
      chefId:chefId
    })
      .then(res => {
        if (res.data.code === 1) {
          this.tableData = res.data && res.data.data && res.data.data.records
          this.counts = Number(res.data.data.total)
        }
      })
      .catch(err => {
        this.$message.error('Add dish type：' + err.message)
      })
  }
  private addDishtype(st: string) {


    if (st === 'add') {
      this.$router.push({ path: '/dish/add' })
    } else {
      this.$router.push({ path: '/dish/add', query: { id: st } })
    }
  }

  // 删除
  private deleteHandle(type: string, id: any) {
    this.$confirm('Ready to continue', 'Confirm', {
      confirmButtonText: 'Delete',
      cancelButtonText: 'Cancel',
      type: 'warning'
    }).then(() => {
      deleteDish(id)
        .then(res => {
          if (res && res.data && res.data.code === 1) {
            this.$message.success('Delete Success')
            this.init()
          } else {
            this.$message.error(res.data.msg)
          }
        })
        .catch(err => {
          this.$message.error('Request Erro：' + err.message)
        })
    })
  }

  //状态更改
  private statusHandle(row: any) {
    let params: any = {}
    if (typeof row === 'string') {
      params.id = this.checkList.join(',')
      params.status = row
    } else {
      params.id = row.id
      params.status = row.status ? '0' : '1'
    }
    this.dishState = params
    this.$confirm('Confirm to change status', 'Hint', {
      confirmButtonText: 'Confirm',
      cancelButtonText: 'Cancel',
      type: 'warning'
    }).then(() => {
      // 起售停售---批量起售停售接口
      dishStatusByStatus(this.dishState)
        .then(res => {
          if (res && res.data && res.data.code === 1) {
            this.$message.success('Status change success')
            this.init()
          } else {
            this.$message.error(res.data.msg)
          }
        })
        .catch(err => {
          this.$message.error('Request erro' + err.message)
        })
    })
  }

  // 全部操作
  private handleSelectionChange(val: any) {
    let checkArr: any[] = []
    val.forEach((n: any) => {
      checkArr.push(n.id)
    })
    this.checkList = checkArr
  }

  private handleSizeChange(val: any) {
    this.pageSize = val
    this.init()
  }

  private handleCurrentChange(val: any) {
    this.page = val
    this.init()
  }
}
</script>
<style lang="scss">
.el-table-column--selection .cell {
  padding-left: 10px;
}
</style>
<style lang="scss" scoped>
.dashboard {
  &-container {
    margin: 30px;
    .container {
      background: #fff;
      position: relative;
      z-index: 1;
      padding: 30px 28px;
      border-radius: 4px;
      //查询黑色按钮样式
      .normal-btn {
        background: #333333;
        color: white;
        margin-left: 20px;
      }
      .tableBar {
        margin-bottom: 20px;

        .tableLab {
          display: inline-block;
          float: right;
          span {
            cursor: pointer;
            display: inline-block;
            font-size: 14px;
            padding: 0 20px;
            color: $gray-2;
          }
        }
      }
      .tableBox {
        width: 100%;
        border: 1px solid $gray-5;
        border-bottom: 0;
      }
      .pageList {
        text-align: center;
        margin-top: 30px;
      }
    }
  }
}
</style>
