<template>
  <div style="margin:10px 20px;" class="voucher-list">
    <Card class="tabs-wrap">
      <el-tabs v-model="status" @tab-click="handleStatusClick">
        <el-tab-pane label="全部" name="全部"></el-tab-pane>
        <el-tab-pane label="待开具" name="10"></el-tab-pane>
        <el-tab-pane label="开具中" name="15"></el-tab-pane>
        <el-tab-pane label="已开具" name="20"></el-tab-pane>
        <el-tab-pane label="开具失败" name="18"></el-tab-pane>
        <el-tab-pane label="已删除" name="80"></el-tab-pane>
        <el-tab-pane label="已取消" name="90"></el-tab-pane>
      </el-tabs>
    </Card>
    <Card class="mt20">
      <Row
        :gutter="16"
        align="middle"
        type="flex"
        class="query-row"
        @keyup.enter.native="doQuery(1)"
      >
        <Col>
          <span>凭证期间：</span>
          <Date-picker
            v-model="queryForm.startDate"
            type="date"
            placeholder="开始日期"
            style="width: 147px"
            format="yyyy-MM-dd"
          ></Date-picker>
          <span>--</span>
          <Date-picker
            v-model="queryForm.endDate"
            type="date"
            placeholder="结束日期"
            style="width: 147px"
            format="yyyy-MM-dd "
          ></Date-picker>
        </Col>
        <Col>
          <span>制单日期：</span>
          <Date-picker
            v-model="queryForm.startIssueDate"
            type="date"
            placeholder="开始日期"
            style="width: 150px"
            format="yyyy-MM-dd"
          ></Date-picker>
          <span>--</span>
          <Date-picker
            v-model="queryForm.endIssueDate"
            type="date"
            placeholder="结束日期"
            style="width: 150px"
            format="yyyy-MM-dd "
          ></Date-picker>
        </Col>
        <Col>
          <span>摘要：</span>
          <Input
            placeholder="请输入"
            class="query-input"
            style="width: 180px"
            v-model="queryForm.digest"
            clearable
          ></Input>
        </Col>
        <Col :span="5">
          <div class="btn-wrap">
            <Button type="primary" class="query-button" @click="doQuery(1)"
              >查询</Button
            >
            <Button class="query-button" @click="addVoucher()">新增</Button>
            <Button class="query-button" @click="addCostVoucher()">新增(成本)</Button>
          </div>
        </Col>
      </Row>
      <Table
        border
        :columns="orderColumns"
        :data="orderList"
        style="margin-top: 20px;"
        :loading="isLoading"
      >
        <template slot="digestSlot" slot-scope="{row}">
          <div
            v-if="row.voucherList && row.voucherList.length > 1"
            class="list-in-table"
          >
            <div v-for="item in row.voucherList" class="list-in-table-item">
              {{ item.digest }}
            </div>
          </div>
          <div v-else class="list-in-table">
            {{ row.digest }}
          </div>
        </template>
        <template slot="itemListSlot" slot-scope="{row}">
          <div
            v-if="row.itemList && row.itemList.length > 1"
            class="list-in-table"
          >
            <div v-for="item in row.itemList" class="list-in-table-item">
              {{ item.codeName }}
            </div>
          </div>
          <div
            v-else-if="row.itemList && row.itemList.length === 1"
            class="list-in-table"
          >
            {{ row.itemList[0].codeName }}
          </div>
        </template>
        <template slot="mdSlot" slot-scope="{row}">
          <div v-if="row.md && row.md > 0" class="list-in-table">
            <div class="list-in-table-item">
              {{ moneyFormat(row.md) }}
            </div>
            <div class="list-in-table-item"></div>
          </div>
        </template>
        <template slot="mcSlot" slot-scope="{row}">
          <div v-if="row.mc && row.mc > 0" class="list-in-table">
            <div class="list-in-table-item"></div>
            <div class="list-in-table-item">
              {{ moneyFormat(row.mc) }}
            </div>
          </div>
        </template>
        <template slot="statusSlot" slot-scope="{row}">
          <div v-if="row.statusName === '开具失败'">
            <el-tooltip class="item" effect="dark" placement="bottom">
              <div slot="content">
                {{ row.createTime }} 失败原因： {{ row.failureReason }}
              </div>
              <div class="status">{{ row.statusName }}</div>
            </el-tooltip>
          </div>

          <div v-else class="status">{{ row.statusName }}</div>
        </template>
        <template slot="handleSlot" slot-scope="{row}">
          <div class="operates flex">
            <div @click.stop="viewDetail(row)" class="text-btn blue">查看</div>
            <div
              v-if="
                row.statusName === '待开具' || row.statusName === '开具失败'
              "
              @click.stop="viewDetail(row)"
              class="text-btn blue"
            >
              开具
            </div>
            <div
              v-if="
                row.statusName === '待开具' || row.statusName === '开具失败'
              "
              @click.stop="viewDetail(row)"
              class="text-btn"
            >
              取消
            </div>
            <div
              v-if="row.statusName === '已开具'"
              @click.stop="viewDetail(row)"
              class="text-btn red"
            >
              删除
            </div>
          </div>
        </template>
      </Table>
      <div style="margin: 20px;overflow: hidden;float: right;">
        <Page
          :total="total"
          :current.sync="queryForm.page"
          :page-size="queryForm.pageSize"
          @on-change="doQuery"
          @on-page-size-change="changePageSize"
          show-elevator
          show-sizer
          show-total
          :page-size-opts="[10, 20, 50, 100]"
        ></Page>
      </div>
    </Card>
    <newVoucher
      v-if="showAddModal"
      :show="showAddModal"
      @on-close="showAddModal = false"
      @on-success="addVoucherSuccess"
    />
    <newCostVoucher
      v-if="showCostModal"
      :show="showCostModal"
      @on-close="showCostModal = false"
      @on-success="addCostVoucherSuccess"
    />
    <el-dialog :visible.sync="dialogVisible" width="50%">
      <div slot="title" class="voucher-dialog-title">{{ dialogTitle }}</div>
      <div class="voucher-dialog-content flex">
        <Icon :color="dialogColor" size="42" :type="dialogIcon" />
        <span>{{ dialogContent }}</span>
      </div>
      <span slot="footer" class="dialog-footer">
        <el-button
          v-if="dialogIcon === 'ios-checkmark-circle'"
          @click="dialogVisible = false"
          >查看详情</el-button
        >
        <el-button
          v-if="dialogIcon === 'ios-checkmark-circle'"
          @click="dialogVisible = false"
          >知道了</el-button
        >
        <el-button
          v-if="dialogIcon !== 'ios-checkmark-circle'"
          @click="dialogVisible = false"
          >{{ dialogCancel }}</el-button
        >
        <el-button
          type="primary"
          v-if="dialogIcon !== 'ios-checkmark-circle'"
          @click="dialogVisible = false"
          >{{ dialogConfirm }}</el-button
        >
      </span>
    </el-dialog>
  </div>
</template>

<script>
import newVoucher from './new.vue'
import newCostVoucher from './newCost.vue'

export default {
  name: 'VoucherList',
  components: {
    newVoucher,
    newCostVoucher
  },
  data: function() {
    return {
      dialogVisible: false,
      dialogTitle: '提示',
      dialogContent: '',
      dialogIcon: '',
      dialogColor: '',
      dialogConfirm: '',
      dialogCancel: '',
      showAddModal: false,
      showCostModal: false,
      isLoading: false,
      orderType: 'pmdt',
      queryForm: {
        digest: null,
        startDate: null,
        endDate: null,
        startIssueDate: null,
        endIssueDate: null,
        status: null,
        type: null,
        admin: true,
        page: 1,
        pageSize: 10
      },
      total: 0,
      orderList: [],
      orderColumns: [
        {
          title: '凭证号',
          key: 'voucherNo',
          align: 'center',
          width: 100
        },
        {
          title: '凭证期间',
          key: 'yearMonth',
          align: 'center',
          width: 100
        },
        {
          title: '摘要',
          key: 'digest',
          align: 'center',
          slot: 'digestSlot',
          width: 200
        },
        {
          title: '所属科目',
          key: 'itemList',
          align: 'center',
          slot: 'itemListSlot',
          width: 200
        },
        {
          title: '制单日期',
          key: 'issueDate',
          align: 'center',
          width: 120
        },
        {
          title: '制单人',
          key: 'issueUser',
          align: 'center',
          width: 120
        },
        {
          title: '借方金额(元)',
          key: 'md',
          align: 'center',
          slot: 'mdSlot',
          width: 120
        },
        {
          title: '贷方金额(元)',
          key: 'mc',
          align: 'center',
          slot: 'mcSlot',
          width: 120
        },
        {
          title: '状态',
          key: 'statusName',
          align: 'center',
          slot: 'statusSlot',
          width: 80
        },
        {
          title: '操作',
          key: 'operate',
          align: 'center',
          slot: 'handleSlot',
          width: 200
        }
      ],
      status: '全部'
    }
  },
  created: function() {
    this.doQuery(1)
  },
  methods: {
    viewDetail: function(row) {
      this.$router.push({
        path: 'voucher-detail',
        query: {
          id: row.id
        }
      })
    },
    addVoucherSuccess: function(id, cid) {
      console.log('addVoucherSuccess', id)
      this.showAddModal = false
      this.$router.push({
        path: 'voucher-detail',
        query: {
          id: id,
          cid: cid,
          isNew: 1
        }
      })
    },
    moneyFormat(amount, options) {
      const {defaultVal, prefix} = Object.assign(
        {defaultVal: '--', prefix: ''},
        options || {}
      )
      if (amount || amount === 0) {
        return `${prefix}${amount}`.replace(
          /\d{1,3}(?=(\d{3})+(\.\d*)?$)/g,
          '$&,'
        )
      }
      return defaultVal.toFixed(2)
    },
    handleStatusClick(tab, event) {
      this.status = tab.name
      this.doQuery(1)
    },
    doQuery: function(page) {
      if (page) {
        this.queryForm.page = page
      }
      this.queryForm.status = this.status === '全部' ? null : this.status
      this.isLoading = true
      this.$ajax
        .postFnc('u8/list', this.queryForm, this.$ajax.urlJson())
        .then(res => {
          this.isLoading = false
          if (res.data.code === 0 && res.data.data) {
            var resData = res.data.data
            console.log(res.data.data)
            this.orderList = resData.list
            this.total = resData.totalCount
          } else {
            this.$Message.error(res.data.msg || '获取数据失败')
          }
        })
    },
    changePageSize: function(val) {
      this.queryForm.pageSize = val
      this.queryForm.page = 1
      this.doQuery(this.queryForm.page)
    },
    addVoucher: function(e) {
      console.log('addVoucher', e)
      this.showAddModal = true
    },
    addCostVoucher: function() {
      this.showCostModal = true
    },
    addCostVoucherSuccess: function(id) {
      this.showCostModal = false
      this.$router.push({
        path: 'voucher-detail',
        query: {
          id: id,
          isCost: 1
        }
      })
    }
  }
}
</script>

<style lang="less" scoped>
.query-row {
  padding: 5px 10px;
  .query-input {
    width: 135px;
  }
  .query-button {
    padding: 5px 30px;
    margin: 1px 8px;
  }
}
.mt20 {
  margin-top: 10px;
}
.list-in-table {
  .list-in-table-item {
    min-height: 40px;
    display: flex;
    align-items: center;
    justify-content: center;
    &:first-child {
      border-bottom: 1px solid #ebeef5;
    }
  }
}
.operates {
  padding: 4px 16px;
  .text-btn {
    cursor: pointer;
    margin-right: 8px;
  }
  .blue {
    color: #409eff;
  }
  .red {
    color: #f56c6c;
  }
}
</style>
<style lang="less">
.voucher-dialog-title {
  font-size: 16px;
  font-weight: bold;
}
.voucher-dialog-content {
  align-items: center;
  font-size: 16px;
  font-weight: bold;
  white-space: pre-wrap; /* 保留空白符序列，但是正常地进行换行 */
  span {
    margin-left: 10px;
  }
}
.voucher-list {
  .tabs-wrap {
    .el-tabs__header {
      margin-bottom: 0;
    }
  }
  .el-tabs__nav-wrap::after {
    background-color: #fff;
  }
  .ivu-card-body {
    padding: 8px 16px;
  }
  .btn-wrap {
    display: flex;
    justify-content: flex-end;
  }
  .ivu-table-cell {
    padding: 0;
  }
}
</style>
