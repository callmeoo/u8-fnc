<template>
  <div>
    <!-- Step 1: Select batch settlement record -->
    <el-dialog
      title="新增凭证(成本) - 选择批量结算记录"
      :visible.sync="showStep1"
      width="900px"
      :close-on-click-modal="false"
      @close="onCancel"
    >
      <div class="step-header">
        <div class="step-item active">
          <span class="step-num">1</span>
          <span>选择批量结算记录</span>
        </div>
        <div class="step-line"></div>
        <div class="step-item">
          <span class="step-num">2</span>
          <span>凭证分录预览确认</span>
        </div>
      </div>
      <div class="search-row">
        <span>结算机构：</span>
        <Input
          placeholder="请输入"
          style="width: 150px"
          v-model="searchForm.repairAgency"
          clearable
        ></Input>
        <span style="margin-left: 16px;">费用类型：</span>
        <Select v-model="searchForm.feeTypeName" style="width: 150px" placeholder="请选择" clearable>
          <Option value="服务费">服务费</Option>
          <Option value="过户费">过户费</Option>
          <Option value="整理费">整理费</Option>
        </Select>
        <span style="margin-left: 16px;">结算年月：</span>
        <Date-picker
          v-model="searchForm.accountMonth"
          type="month"
          placeholder="请选择"
          style="width: 150px"
          format="yyyy-MM"
        ></Date-picker>
        <Button type="primary" style="margin-left: 16px;" @click="searchSettlement">查询</Button>
      </div>
      <el-table
        border
        :data="settlementList"
        v-loading="settlementLoading"
        highlight-current-row
        @current-change="handleSettlementSelect"
        style="margin-top: 16px;"
      >
        <el-table-column width="55" align="center">
          <template slot-scope="scope">
            <el-radio v-model="selectedSettlementId" :label="scope.row.id">&nbsp;</el-radio>
          </template>
        </el-table-column>
        <el-table-column prop="repairAgency" label="结算机构" align="center"></el-table-column>
        <el-table-column prop="feeTypeName" label="结算费用类型" align="center"></el-table-column>
        <el-table-column prop="accountMonth" label="结算年月" align="center"></el-table-column>
        <el-table-column prop="totalAmount" label="总金额(元)" align="center">
          <template slot-scope="scope">
            {{ moneyFormat(scope.row.totalAmount) }}
          </template>
        </el-table-column>
        <el-table-column prop="accountStatusName" label="结算状态" align="center"></el-table-column>
        <el-table-column prop="digest" label="付款摘要" align="center"></el-table-column>
      </el-table>
      <span slot="footer" class="dialog-footer">
        <el-button @click="onCancel">取消</el-button>
        <el-button type="primary" :disabled="!selectedSettlementId" @click="goStep2">下一步</el-button>
      </span>
    </el-dialog>

    <!-- Step 2: Voucher entry preview -->
    <el-dialog
      title="新增凭证(成本) - 凭证分录预览确认"
      :visible.sync="showStep2"
      width="1200px"
      :close-on-click-modal="false"
      @close="onCancel"
    >
      <div class="step-header">
        <div class="step-item finished" @click="backToStep1">
          <span class="step-num">1</span>
          <span>选择批量结算记录</span>
        </div>
        <div class="step-line active"></div>
        <div class="step-item active">
          <span class="step-num">2</span>
          <span>凭证分录预览确认</span>
        </div>
      </div>

      <div class="preview-info">
        <div class="info-row">
          <span class="info-label">结算机构：</span>
          <span class="info-value">{{ selectedSettlement.repairAgency }}</span>
          <span class="info-label" style="margin-left: 40px;">费用类型：</span>
          <span class="info-value">{{ selectedSettlement.feeTypeName }}</span>
          <span class="info-label" style="margin-left: 40px;">结算年月：</span>
          <span class="info-value">{{ selectedSettlement.accountMonth }}</span>
        </div>
        <div class="info-row">
          <span class="info-label">摘要：</span>
          <span class="info-value digest-value">{{ selectedSettlement.digest }}</span>
          <span class="digest-tip">(自动带入申请单-付款摘要，不可修改)</span>
        </div>
        <div class="info-row">
          <span class="info-label">总进项税额：</span>
          <Input
            v-model="totalInputTax"
            placeholder="请输入总进项税额"
            style="width: 200px"
            @on-change="recalculateTax"
          ></Input>
          <span class="tax-tip" style="margin-left: 8px;">修改后系统自动计算每台车不含税金额</span>
        </div>
      </div>

      <el-table
        border
        :data="previewEntries"
        v-loading="previewLoading"
        style="margin-top: 16px;"
        show-summary
        :summary-method="getSummaries"
      >
        <el-table-column type="index" label="序号" width="60" align="center"></el-table-column>
        <el-table-column prop="digest" label="摘要" align="center" width="180"></el-table-column>
        <el-table-column prop="subjectCode" label="科目编码" align="center" width="120"></el-table-column>
        <el-table-column prop="subjectName" label="科目名称" align="center" width="120"></el-table-column>
        <el-table-column prop="auxiliaryFiledName" label="辅助项目名称" align="center" width="120"></el-table-column>
        <el-table-column prop="auxiliaryFiledCode" label="辅助项目编码" align="center" width="120"></el-table-column>
        <el-table-column prop="auxiliaryName" label="辅助信息名称" align="center" width="120"></el-table-column>
        <el-table-column prop="auxiliaryCode" label="辅助信息" align="center" width="120"></el-table-column>
        <el-table-column prop="md" label="借方金额(元)" align="center" width="120">
          <template slot-scope="scope">
            <span v-if="scope.row.md">{{ moneyFormat(scope.row.md) }}</span>
          </template>
        </el-table-column>
        <el-table-column prop="mc" label="贷方金额(元)" align="center" width="120">
          <template slot-scope="scope">
            <span v-if="scope.row.mc">{{ moneyFormat(scope.row.mc) }}</span>
          </template>
        </el-table-column>
      </el-table>

      <span slot="footer" class="dialog-footer">
        <el-button @click="backToStep1">上一步</el-button>
        <el-button @click="onCancel">取消</el-button>
        <el-button type="primary" :loading="submitLoading" @click="submitVoucher">确认创建</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
// Mock batch settlement records
const mockSettlementList = [
  {
    id: 'ST202604001',
    repairAgency: '北京华盛汽车服务有限公司',
    feeTypeName: '服务费',
    accountMonth: '2026-04',
    totalAmount: 35600.00,
    accountStatusName: '已结算',
    digest: '2026年4月北京华盛服务费结算',
    vehicles: [
      { licensePlate: '京A12345', amount: 12000.00 },
      { licensePlate: '京B67890', amount: 13600.00 },
      { licensePlate: '京C24680', amount: 10000.00 }
    ]
  },
  {
    id: 'ST202604002',
    repairAgency: '上海远通汽车代办中心',
    feeTypeName: '过户费',
    accountMonth: '2026-04',
    totalAmount: 9800.00,
    accountStatusName: '已结算',
    digest: '2026年4月上海远通过户费结算',
    vehicles: [
      { licensePlate: '沪A55001', amount: 3500.00 },
      { licensePlate: '沪B66002', amount: 3200.00 },
      { licensePlate: '沪C77003', amount: 3100.00 }
    ]
  },
  {
    id: 'ST202604003',
    repairAgency: '广州捷达物流有限公司',
    feeTypeName: '整理费',
    accountMonth: '2026-03',
    totalAmount: 5400.00,
    accountStatusName: '已结算',
    digest: '2026年3月广州捷达整理费结算',
    vehicles: [
      { licensePlate: '粤A11111', amount: 1800.00 },
      { licensePlate: '粤B22222', amount: 2000.00 },
      { licensePlate: '粤C33333', amount: 1600.00 }
    ]
  }
]

// Fee type to U8 subject mapping
const feeTypeMapping = {
  '服务费': {
    u8FeeName: '服务商服务费',
    subjectCode: '64010102',
    auxiliaryFiledName: '项目',
    auxiliaryFiledCode: '0010',
    auxiliaryNameField: '车牌',
    auxiliaryCodeField: '车牌'
  },
  '过户费': {
    u8FeeName: '代办过户费',
    subjectCode: '64010103',
    auxiliaryFiledName: '项目',
    auxiliaryFiledCode: '0010',
    auxiliaryNameField: '车牌',
    auxiliaryCodeField: '车牌'
  },
  '整理费': {
    u8FeeName: '牌证费及邮递',
    subjectCode: '64011209',
    auxiliaryFiledName: '部门',
    auxiliaryFiledCode: '0001',
    auxiliaryNameField: 'BIP部门名称',
    auxiliaryCodeField: 'BIP部门编码'
  }
}

export default {
  name: 'newCostVoucher',
  props: {
    show: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      showStep1: this.show,
      showStep2: false,
      searchForm: {
        repairAgency: '',
        feeTypeName: '',
        accountMonth: ''
      },
      settlementList: [],
      settlementLoading: false,
      selectedSettlementId: '',
      selectedSettlement: {},
      totalInputTax: '',
      previewEntries: [],
      previewLoading: false,
      submitLoading: false
    }
  },
  created() {
    this.searchSettlement()
  },
  methods: {
    moneyFormat(amount) {
      if (amount || amount === 0) {
        return `${amount}`.replace(/\d{1,3}(?=(\d{3})+(\.\d*)?$)/g, '$&,')
      }
      return '--'
    },
    searchSettlement() {
      this.settlementLoading = true
      // Mock: filter settlement list
      setTimeout(() => {
        let list = [...mockSettlementList]
        if (this.searchForm.repairAgency) {
          list = list.filter(item => item.repairAgency.includes(this.searchForm.repairAgency))
        }
        if (this.searchForm.feeTypeName) {
          list = list.filter(item => item.feeTypeName === this.searchForm.feeTypeName)
        }
        if (this.searchForm.accountMonth) {
          const month = this.searchForm.accountMonth
          const monthStr = month.getFullYear() + '-' + String(month.getMonth() + 1).padStart(2, '0')
          list = list.filter(item => item.accountMonth === monthStr)
        }
        this.settlementList = list
        this.settlementLoading = false
      }, 300)
    },
    handleSettlementSelect(row) {
      if (row) {
        this.selectedSettlementId = row.id
        this.selectedSettlement = row
      }
    },
    goStep2() {
      if (!this.selectedSettlementId) {
        this.$Message.warning('请选择一条批量结算记录')
        return
      }
      this.showStep1 = false
      this.showStep2 = true
      this.totalInputTax = ''
      this.generatePreviewEntries()
    },
    backToStep1() {
      this.showStep2 = false
      this.showStep1 = true
    },
    generatePreviewEntries() {
      const settlement = this.selectedSettlement
      const mapping = feeTypeMapping[settlement.feeTypeName]
      if (!mapping || !settlement.vehicles) return

      const entries = []
      const totalTax = this.totalInputTax ? parseFloat(this.totalInputTax) : 0
      const vehicleCount = settlement.vehicles.length

      // Generate cost entries for each vehicle
      settlement.vehicles.forEach((vehicle, index) => {
        let excludeTaxAmount = vehicle.amount
        let taxAmount = 0

        if (totalTax > 0) {
          // Only first vehicle gets input tax deducted
          if (index === 0) {
            taxAmount = totalTax
            excludeTaxAmount = vehicle.amount - totalTax
          }
        }

        // Cost entry (debit)
        entries.push({
          digest: settlement.digest,
          subjectCode: mapping.subjectCode,
          subjectName: mapping.u8FeeName,
          auxiliaryFiledName: mapping.auxiliaryFiledName,
          auxiliaryFiledCode: mapping.auxiliaryFiledCode,
          auxiliaryName: mapping.auxiliaryNameField === '车牌' ? vehicle.licensePlate : mapping.auxiliaryNameField,
          auxiliaryCode: mapping.auxiliaryCodeField === '车牌' ? vehicle.licensePlate : mapping.auxiliaryCodeField,
          md: parseFloat(excludeTaxAmount.toFixed(2)),
          mc: null,
          type: 'cost',
          vehicleIndex: index
        })
      })

      // Input tax entry (debit) - if totalInputTax is set
      if (totalTax > 0) {
        entries.push({
          digest: settlement.digest,
          subjectCode: '2221010101',
          subjectName: '进项税额',
          auxiliaryFiledName: '',
          auxiliaryFiledCode: '',
          auxiliaryName: '',
          auxiliaryCode: '',
          md: totalTax,
          mc: null,
          type: 'tax'
        })
      }

      // Bank deposit entry (credit)
      entries.push({
        digest: settlement.digest,
        subjectCode: '10020101',
        subjectName: '银行存款',
        auxiliaryFiledName: '',
        auxiliaryFiledCode: '',
        auxiliaryName: '',
        auxiliaryCode: '',
        md: null,
        mc: settlement.totalAmount,
        type: 'bank'
      })

      this.previewEntries = entries
    },
    recalculateTax() {
      this.generatePreviewEntries()
    },
    getSummaries(param) {
      const { columns, data } = param
      const sums = []
      columns.forEach((column, index) => {
        if (index === 0) {
          sums[index] = '合计'
          return
        }
        if (column.property === 'md' || column.property === 'mc') {
          const values = data.map(item => Number(item[column.property]) || 0)
          const total = values.reduce((prev, curr) => prev + curr, 0)
          sums[index] = total > 0 ? this.moneyFormat(total.toFixed(2)) : ''
        } else {
          sums[index] = ''
        }
      })
      return sums
    },
    submitVoucher() {
      if (this.previewEntries.length === 0) {
        this.$Message.warning('没有可创建的凭证分录')
        return
      }
      this.submitLoading = true
      // Mock: simulate API call to create cost voucher
      setTimeout(() => {
        this.submitLoading = false
        const mockVoucherId = 'CV' + Date.now()
        this.$Message.success('成本凭证创建成功')
        this.$emit('on-success', mockVoucherId)
      }, 500)
    },
    onCancel() {
      this.showStep1 = false
      this.showStep2 = false
      this.$emit('on-close')
    }
  }
}
</script>

<style lang="less" scoped>
.step-header {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 24px;
  padding: 0 60px;
  .step-item {
    display: flex;
    align-items: center;
    color: #999;
    cursor: default;
    .step-num {
      width: 28px;
      height: 28px;
      border-radius: 50%;
      background: #e0e0e0;
      color: #fff;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 14px;
      margin-right: 8px;
    }
    &.active {
      color: #409eff;
      .step-num {
        background: #409eff;
      }
    }
    &.finished {
      color: #67c23a;
      cursor: pointer;
      .step-num {
        background: #67c23a;
      }
    }
  }
  .step-line {
    width: 120px;
    height: 2px;
    background: #e0e0e0;
    margin: 0 16px;
    &.active {
      background: #409eff;
    }
  }
}
.search-row {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  padding: 8px 0;
  span {
    white-space: nowrap;
  }
}
.preview-info {
  background: #f5f7fa;
  padding: 16px 20px;
  border-radius: 4px;
  .info-row {
    display: flex;
    align-items: center;
    margin-bottom: 12px;
    &:last-child {
      margin-bottom: 0;
    }
    .info-label {
      color: #606266;
      font-weight: 500;
      white-space: nowrap;
    }
    .info-value {
      color: #303133;
      margin-right: 8px;
    }
    .digest-value {
      font-weight: bold;
      color: #409eff;
    }
    .digest-tip {
      color: #999;
      font-size: 12px;
    }
    .tax-tip {
      color: #e6a23c;
      font-size: 12px;
    }
  }
}
</style>
