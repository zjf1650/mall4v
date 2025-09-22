<template>
  <div class="mod-refund-refund">
    <el-form
      :inline="true"
      :model="dataForm"
      @keyup.enter="getDataList(page)"
    >
      <el-form-item label="退款单号:">
        <el-input
          v-model="dataForm.refundSn"
          placeholder="退款单号"
          clearable
        />
      </el-form-item>
      <el-form-item label="订单编号:">
        <el-input
          v-model="dataForm.orderNumber"
          placeholder="订单编号"
          clearable
        />
      </el-form-item>
      <el-form-item label="申请时间:">
        <el-date-picker
          v-model="dateRange"
          type="datetimerange"
          range-separator="至"
          value-format="YYYY-MM-DD HH:mm:ss"
          start-placeholder="开始日期"
          end-placeholder="结束日期"
        />
      </el-form-item>
      <el-form-item label="退款状态:">
        <el-select
          v-model="dataForm.status"
          clearable
          placeholder="请选择退款状态"
        >
          <el-option label="待审核" value="1" />
          <el-option label="已同意" value="2" />
          <el-option label="已拒绝" value="3" />
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-button
          type="primary"
          icon="el-icon-search"
          @click="getDataList()"
        >
          查询
        </el-button>
        <el-button @click="clearDatas()">
          清空
        </el-button>
      </el-form-item>
    </el-form>

    <div class="main">
      <div class="content">
        <div class="tit">
          <el-row style="width:100%">
            <el-col :span="4">
              <span class="item">退款单号</span>
            </el-col>
            <el-col :span="4">
              <span class="item">订单编号</span>
            </el-col>
            <el-col :span="3">
              <span class="item">申请类型</span>
            </el-col>
            <el-col :span="3">
              <span class="item">退款金额</span>
            </el-col>
            <el-col :span="3">
              <span class="item">申请时间</span>
            </el-col>
            <el-col :span="3">
              <span class="item">退款状态</span>
            </el-col>
            <el-col :span="4">
              <span class="item">操作</span>
            </el-col>
          </el-row>
        </div>

        <div
          v-for="refund in dataList"
          :key="refund.refundId"
          class="refund-item"
        >
          <el-row style="width:100%; align-items: center; padding: 15px 0;">
            <el-col :span="4">
              <span>{{ refund.refundSn }}</span>
            </el-col>
            <el-col :span="4">
              <span>{{ refund.orderNumber }}</span>
            </el-col>
            <el-col :span="3">
              <span>{{ refund.applyType === 1 ? '仅退款' : '退货退款' }}</span>
            </el-col>
            <el-col :span="3">
              <span class="amount">￥{{ refund.refundAmount }}</span>
            </el-col>
            <el-col :span="3">
              <span>{{ formatTime(refund.applyTime) }}</span>
            </el-col>
            <el-col :span="3">
              <el-tag :type="getStatusType(refund)">
                {{ getStatusText(refund) }}
              </el-tag>
            </el-col>
            <el-col :span="4">
              <el-button
                v-if="isAuth('admin:refund:info')"
                size="small"
                @click="viewDetail(refund.refundId)"
              >
                查看详情
              </el-button>
              <el-button
                v-if="refund.refundSts === 1 && isAuth('admin:refund:audit')"
                size="small"
                type="primary"
                @click="auditRefund(refund)"
              >
                审核
              </el-button>
            </el-col>
          </el-row>
        </div>
      </div>
    </div>

    <div class="page-list-con">
      <el-pagination
        :current-page="page"
        :page-sizes="[10, 20, 50, 100]"
        :page-size="limit"
        :total="total"
        layout="total, sizes, prev, pager, next, jumper"
        @size-change="sizeChangeHandle"
        @current-change="currentChangeHandle"
      />
    </div>

    <!-- 退款详情弹窗 -->
    <el-dialog
      title="退款详情"
      v-model="detailVisible"
      width="60%"
    >
      <div v-if="refundDetail">
        <el-descriptions :column="2" border>
          <el-descriptions-item label="退款单号">{{ refundDetail.refundSn }}</el-descriptions-item>
          <el-descriptions-item label="订单编号">{{ refundDetail.orderNumber }}</el-descriptions-item>
          <el-descriptions-item label="申请类型">{{ refundDetail.applyType === 1 ? '仅退款' : '退货退款' }}</el-descriptions-item>
          <el-descriptions-item label="退款金额">￥{{ refundDetail.refundAmount }}</el-descriptions-item>
          <el-descriptions-item label="申请时间">{{ formatTime(refundDetail.applyTime) }}</el-descriptions-item>
          <el-descriptions-item label="处理时间">{{ formatTime(refundDetail.handelTime) }}</el-descriptions-item>
          <el-descriptions-item label="退款状态">
            <el-tag :type="getStatusType(refundDetail)">{{ getStatusText(refundDetail) }}</el-tag>
          </el-descriptions-item>
          <el-descriptions-item label="退款原因" :span="2">{{ refundDetail.buyerMsg || '无' }}</el-descriptions-item>
          <el-descriptions-item label="商家回复" :span="2">{{ refundDetail.sellerMsg || '无' }}</el-descriptions-item>
        </el-descriptions>

        <div v-if="refundDetail.photoFiles" style="margin-top: 20px;">
          <h4>凭证图片</h4>
          <el-image
            v-for="(img, index) in refundDetail.photoFiles.split(',')"
            :key="index"
            :src="img"
            style="width: 100px; height: 100px; margin: 5px;"
            :preview-src-list="refundDetail.photoFiles.split(',')"
          />
        </div>
      </div>
    </el-dialog>

    <!-- 审核弹窗 -->
    <el-dialog
      title="审核退款"
      v-model="auditVisible"
      width="40%"
    >
      <el-form :model="auditForm" label-width="100px">
        <el-form-item label="审核结果" required>
          <el-radio-group v-model="auditForm.auditResult">
            <el-radio :label="2">同意退款</el-radio>
            <el-radio :label="3">拒绝退款</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="备注">
          <el-input
            type="textarea"
            v-model="auditForm.sellerMsg"
            placeholder="请输入审核备注"
            :rows="3"
            maxlength="200"
            show-word-limit
          />
        </el-form-item>
      </el-form>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="auditVisible = false">取 消</el-button>
          <el-button type="primary" @click="submitAudit">确 定</el-button>
        </span>
      </template>
    </el-dialog>
  </div>
</template>

<script>
import { isAuth } from '@/utils'

export default {
  data() {
    return {
      dataForm: {
        refundSn: '',
        orderNumber: '',
        status: ''
      },
      dateRange: [],
      dataList: [],
      page: 1,
      limit: 10,
      total: 0,
      detailVisible: false,
      refundDetail: null,
      auditVisible: false,
      auditForm: {
        refundId: null,
        auditResult: 2,
        sellerMsg: ''
      }
    }
  },
  activated() {
    this.getDataList()
  },
  methods: {
    isAuth,
    // 获取数据列表
    getDataList() {
      const params = {
        current: this.page,
        size: this.limit,
        ...this.dataForm
      }

      if (this.dateRange && this.dateRange.length === 2) {
        params.startTime = this.dateRange[0]
        params.endTime = this.dateRange[1]
      }

      this.$http({
        url: this.$http.adornUrl('/admin/refund/list'),
        method: 'get',
        params: this.$http.adornParams(params)
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.dataList = data.data.records
          this.total = data.data.total
        } else {
          this.dataList = []
          this.total = 0
        }
      })
    },
    // 每页数
    sizeChangeHandle(val) {
      this.limit = val
      this.page = 1
      this.getDataList()
    },
    // 当前页
    currentChangeHandle(val) {
      this.page = val
      this.getDataList()
    },
    // 清空搜索条件
    clearDatas() {
      this.dataForm = {
        refundSn: '',
        orderNumber: '',
        status: ''
      }
      this.dateRange = []
      this.getDataList()
    },
    // 查看详情
    viewDetail(refundId) {
      this.$http({
        url: this.$http.adornUrl(`/admin/refund/detail/${refundId}`),
        method: 'get'
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.refundDetail = data.data
          this.detailVisible = true
        } else {
          this.$message.error(data.msg || '获取详情失败')
        }
      })
    },
    // 审核退款
    auditRefund(refund) {
      this.auditForm = {
        refundId: refund.refundId,
        auditResult: 2,
        sellerMsg: ''
      }
      this.auditVisible = true
    },
    // 提交审核
    submitAudit() {
      if (!this.auditForm.auditResult) {
        this.$message.error('请选择审核结果')
        return
      }

      this.$http({
        url: this.$http.adornUrl('/admin/refund/audit'),
        method: 'put',
        data: this.$http.adornData(this.auditForm)
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message.success('审核成功')
          this.auditVisible = false
          this.getDataList()
        } else {
          this.$message.error(data.msg || '审核失败')
        }
      })
    },
    // 格式化时间
    formatTime(time) {
      if (!time) return ''
      return new Date(time).toLocaleString()
    },
    // 获取状态文本
    getStatusText(refund) {
      const statusMap = {
        1: '待审核',
        2: '已同意',
        3: '已拒绝'
      }

      let statusText = statusMap[refund.refundSts] || '未知状态'

      if (refund.refundSts === 2) {
        const moneyStatusMap = {
          0: '退款处理中',
          1: '退款成功',
          '-1': '退款失败'
        }
        statusText = moneyStatusMap[refund.returnMoneySts] || statusText
      }

      return statusText
    },
    // 获取状态类型
    getStatusType(refund) {
      if (refund.refundSts === 1) return 'warning'
      if (refund.refundSts === 2) {
        if (refund.returnMoneySts === 1) return 'success'
        if (refund.returnMoneySts === -1) return 'danger'
        return 'primary'
      }
      if (refund.refundSts === 3) return 'danger'
      return 'info'
    }
  }
}
</script>

<style>
.mod-refund-refund .tit {
  background: #f5f7fa;
  padding: 10px;
  border: 1px solid #ebeef5;
  font-weight: bold;
}

.mod-refund-refund .refund-item {
  border: 1px solid #ebeef5;
  border-top: none;
}

.mod-refund-refund .refund-item:hover {
  background-color: #f5f7fa;
}

.mod-refund-refund .amount {
  color: #f56c6c;
  font-weight: bold;
}

.mod-refund-refund .page-list-con {
  text-align: center;
  margin: 20px 0;
}
</style>