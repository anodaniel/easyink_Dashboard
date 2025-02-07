<!--
 * @Description: 客户活跃度
 * @Author: xulinbin
 * @LastEditors: xulinbin
-->
<template>
  <div class="activeness-page">
    <!-- 选择员工/部门弹窗 -->
    <SelectUser
      :visible.sync="dialogVisibleSelectUser"
      title="选择员工/部门"
      :is-only-leaf="false"
      :selected-user-list="userAndDepartmentList"
      @success="selectedUserOrDepartment"
    />
    <RightContainer>
      <template v-slot:search>
        <el-form
          ref="queryForm"
          :inline="true"
          :model="query"
          label-width="80px"
          class="top-search"
          size="small"
        >
          <el-form-item prop="name">
            <div class="tag-input" @click="dialogVisibleSelectUser = true">
              <span v-if="!userAndDepartmentList.length" class="tag-place">请选择员工/部门</span>
              <template v-else>
                <el-tag
                  v-for="(unit, unique) in userAndDepartmentList"
                  :key="unique"
                  class="theme-text-color user-tag iaic"
                >
                  <TagUserShow :name="unit.name" :show-icon="!unit.userId" />
                </el-tag>
              </template>
            </div>
          </el-form-item>
          <el-form-item label="添加时间">
            <el-date-picker
              v-model="addDateRange"
              value-format="yyyy-MM-dd"
              type="daterange"
              range-separator="-"
              start-placeholder="开始日期"
              end-placeholder="结束日期"
              :clearable="false"
            />
          </el-form-item>
          <el-form-item label="发送时间">
            <el-date-picker
              v-model="sendDateRange"
              value-format="yyyy-MM-dd"
              type="daterange"
              range-separator="-"
              start-placeholder="开始日期"
              end-placeholder="结束日期"
              :picker-options="pickerOptions"
              :clearable="false"
            />
          </el-form-item>
          <el-form-item>
            <el-button
              type="primary"
              @click="onSearch"
            >查询</el-button>
            <el-button
              class="btn-reset"
              @click="resetForm"
            >重置</el-button>
          </el-form-item>
        </el-form>
      </template>
      <template v-slot:data>
        <div class="tendency-chart">
          <div class="chart-title">趋势图</div>
          <div class="chart-info">统计某个时间段内员工添加的客户，在某个时间段内发送的消息情况</div>
          <div class="echars-show">
            <Graphics
              :echars-options-data="echarsOptionsData"
            />
          </div>
        </div>
        <div class="data-details">
          <div class="details-title">数据详情</div>
          <div class="forms-handle-btn">
            <el-radio-group v-model="activeName" class="radio-group-div" size="medium">
              <el-radio-button :label="DATA_DIMENSION['date']">日期维度</el-radio-button>
              <el-radio-button :label="DATA_DIMENSION['staff']">员工维度</el-radio-button>
              <el-radio-button :label="DATA_DIMENSION['client']">客户维度</el-radio-button>
            </el-radio-group>
            <!-- <el-button
              v-hasPermi="['statistic:customerContact:export']"
              class="btn-reset"
              @click="exportForms"
            >导出报表</el-button> -->
          </div>
          <el-table
            v-loading="loading"
            :data="list"
          >
            <template slot="empty">
              <empty-default-icon :length="list.length" />
            </template>
            <!-- 日期维度时不显示 -->
            <el-table-column
              v-if="activeName !== DATA_DIMENSION['date']"
              key="staffOrClient"
              prop=""
              :label="activeName === DATA_DIMENSION['staff'] ? '员工' : '客户'"
              min-width="180"
            >
              <template slot-scope="{ row }">
                <UserItem :data="row" :is-staff="activeName === DATA_DIMENSION['staff']" />
              </template>
            </el-table-column>
            <!-- 日期维度时才显示 -->
            <el-table-column
              v-if="activeName === DATA_DIMENSION['date']"
              key="time"
              prop="time"
              label="日期"
              min-width="180"
            />
            <!-- 客户维度时才显示 -->
            <el-table-column
              v-if="activeName === DATA_DIMENSION['client']"
              key="userInfo"
              prop=""
              label="所属员工"
              min-width="180"
            >
              <template slot-scope="{ row }">
                <div>
                  <div>{{ row.userName }}</div>
                  <div>{{ row.departmentName }}</div>
                </div>
              </template>
            </el-table-column>
            <el-table-column
              key="userSendMessageCnt"
              prop="userSendMessageCnt"
              label="员工发送消息数"
              min-width="180"
            />
            <el-table-column
              key="customerSendMessageCnt"
              prop="customerSendMessageCnt"
              label="客户发送消息数"
              min-width="180"
            />
            <!-- 客户维度时不显示 -->
            <el-table-column
              v-if="activeName !== DATA_DIMENSION['client']"
              key="chatCustomerCnt"
              prop="chatCustomerCnt"
              label="会话客户数"
              min-width="180"
            />
            <!-- 客户维度时不显示 -->
            <el-table-column
              v-if="activeName !== DATA_DIMENSION['client']"
              key="customerAverageMessageCnt"
              prop="customerAverageMessageCnt"
              label="客户平均消息数"
              min-width="180"
            />
            <!-- 日期维度时不显示 -->
            <el-table-column
              v-if="activeName !== DATA_DIMENSION['date']"
              key="handle"
              label="操作"
              min-width="180"
            >
              <template slot-scope="{ row }">
                <!-- 员工维度 -->
                <el-button
                  v-if="activeName === DATA_DIMENSION['staff']"
                  v-hasPermi="['wecom:code:query']"
                  type="text"
                  @click="openClientDialog(row)"
                >详情</el-button>
                <!-- 客户维度 -->
                <!-- 跳转到客户资料详情页 -->
                <el-button
                  v-if="activeName === DATA_DIMENSION['client']"
                  v-hasPermi="['wecom:code:query']"
                  type="text"
                  @click="skipToUserDetails(row)"
                >客户详情</el-button>
              </template>
            </el-table-column>
          </el-table>
          <pagination
            :total="total * 1"
            :page.sync="query.pageNum"
            :limit.sync="query.pageSize"
            @pagination="getList()"
          />
        </div>
      </template>
    </RightContainer>
    <ClientDetailsDialog
      ref="clientDetailsDialogRef"
      :visible.sync="detailsDialogVisible"
      :show-details-client="showDetailsClient"
      :parent-query="query"
    />
  </div>
</template>
<script>
import RightContainer from '@/components/RightContainer';
import { PAGE_LIMIT, DATA_DIMENSION } from '@/utils/constant';
import ClientDetailsDialog from './clientDetailsDialog.vue';
import Graphics from './graphics.vue';
import UserItem from './userItem.vue';
import EmptyDefaultIcon from '@/components/EmptyDefaultIcon';
import TagUserShow from '@/components/TagUserShow';
import SelectUser from '@/components/SelectUser/index.vue';
import { groupByScopeType } from '@/utils/common';
import { YESTERDAY_TIME, FIXED_DAYS_AGO_TIME, ONE_MOUNTH_AGO, ONE_MOUNTH_LATER } from '@/utils/common';
import {
  getCustomerActivityOfDateChart,
  getCustomerActivityOfDate,
  getCustomerActivityOfUser,
  getCustomerActivityOfCustomer,
  exportCustomerActivityOfDate,
  exportCustomerActivityOfUser,
  exportCustomerActivityOfCustomer
} from '@/api/statistics';
import { goRouteWithQuery } from '@/utils/index';
import { getQueryObject } from '@/utils/index';
export default {
  name: '',
  components: { RightContainer, ClientDetailsDialog, Graphics, UserItem, EmptyDefaultIcon, TagUserShow, SelectUser },
  data() {
    return {
      // 选择添加人弹窗显隐
      dialogVisibleSelectUser: false,
      // 搜索框选择的员工/部门
      userAndDepartmentList: [],
      // 发送时间 日期选择器第一次选中的时间
      pickerMinDate: '',
      // 发送时间 日期范围选择限定
      pickerOptions: {
        onPick: obj => {
          this.pickerMinDate = new Date(obj.minDate).getTime();
        },
        disabledDate: v => {
          if (this.pickerMinDate) {
            const minTime = new Date(ONE_MOUNTH_AGO(this.pickerMinDate)).getTime();
            const maxTime = new Date(ONE_MOUNTH_LATER(this.pickerMinDate)).getTime();
            return v.getTime() > maxTime || v.getTime() < minTime || v.getTime() < new Date(FIXED_DAYS_AGO_TIME).getTime() || v.getTime() > new Date(YESTERDAY_TIME).getTime();
          } else {
            return v.getTime() < new Date(FIXED_DAYS_AGO_TIME).getTime() || v.getTime() > new Date(YESTERDAY_TIME).getTime();
          }
        }
      },
      // 数据详情维度选择
      activeName: DATA_DIMENSION['date'],
      // 添加时间日期范围
      addDateRange: [YESTERDAY_TIME, YESTERDAY_TIME],
      // 发送时间日期范围
      sendDateRange: [YESTERDAY_TIME, YESTERDAY_TIME],
      // 查询参数
      query: {
        departmentIds: [], // 部门id列表
        userIds: [], // 用户id列表
        pageNum: 1,
        pageSize: PAGE_LIMIT,
        beginTime: undefined, // 添加时间的开始时间
        endTime: undefined, // 添加时间的结束时间
        sendStartTime: undefined, // 发送时间的开始时间
        sendEndTime: undefined // 发送时间的结束时间
      },
      // 总条数
      total: 0,
      // 客户概览表格数据
      list: [],
      // 表格loading
      loading: false,
      DATA_DIMENSION,
      // 活跃度详情弹窗显隐
      detailsDialogVisible: false,
      // 展示员工活跃度详情的员工信息
      showDetailsClient: undefined,
      // echars图表需要的options数据
      echarsOptionsData: {
        // echars X轴数据
        xAxisData: [],
        // 客户发送消息数
        clientSendMessageList: [],
        // 员工发送消息数
        userSendMessageList: []
      }
    };
  },
  watch: {
    // 切换数据详情的维度时
    activeName(val) {
      this.getList(true);
    }
  },
  created() {
    const queryObject = getQueryObject();
    if (queryObject.detailsActiveName_) {
      this.activeName = Number(queryObject.detailsActiveName_);
    } else {
      this.getList(true);
    }
    this.getChartData();
  },
  methods: {
    /**
     * @description 选择员工/部门的回调
     */
    selectedUserOrDepartment(list) {
      this.userAndDepartmentList = list;
    },
    /**
     * @description 获取搜索的传参
     */
    getSearchPayload() {
      if (this.addDateRange) {
        this.query.beginTime = this.addDateRange[0];
        this.query.endTime = this.addDateRange[1];
      } else {
        this.query.beginTime = '';
        this.query.endTime = '';
      }
      if (this.sendDateRange) {
        this.query.sendStartTime = this.sendDateRange[0];
        this.query.sendEndTime = this.sendDateRange[1];
      } else {
        this.query.sendStartTime = '';
        this.query.sendEndTime = '';
      }
      if (this.userAndDepartmentList && this.userAndDepartmentList.length > 0) {
        const allListObj = groupByScopeType(this.userAndDepartmentList);
        this.query.departmentIds = allListObj.useDepartmentList.map(item => item.id);
        this.query.userIds = allListObj.useEmployeesList.map(item => item.userId);
      } else {
        this.query.departmentIds = [];
        this.query.userIds = [];
      }
      return this.query;
    },
    /**
     * @description: 获取数据详情各维度数据
     * @param {*} initPage  是否重置为第一页
     * @return {*}
     */
    getList(initPage) {
      initPage ? this.query.pageNum = 1 : null;
      this.loading = true;
      this.getDiffSearchDetailsDataFun()(this.getSearchPayload()).then(res => {
        this.list = res.rows;
        this.total = res.total || 0;
      }).finally(() => {
        this.loading = false;
      });
    },
    // 得到不同的获取维度数据详情的函数
    getDiffSearchDetailsDataFun() {
      switch (this.activeName) {
        case DATA_DIMENSION['date']:
          return getCustomerActivityOfDate;
        case DATA_DIMENSION['staff']:
          return getCustomerActivityOfUser;
        case DATA_DIMENSION['client']:
          return getCustomerActivityOfCustomer;
      }
    },
    // 得到不同维度导出报表的函数
    getDiffExportFormsFun() {
      switch (this.activeName) {
        case DATA_DIMENSION['date']:
          return exportCustomerActivityOfDate;
        case DATA_DIMENSION['staff']:
          return exportCustomerActivityOfUser;
        case DATA_DIMENSION['client']:
          return exportCustomerActivityOfCustomer;
      }
    },
    // 获取echars图表所需数据
    getChartData() {
      getCustomerActivityOfDateChart(this.getSearchPayload()).then(res => {
        const resXAxisData = res.data.map(item => item.time);
        const resClientSendMessageList = res.data.map(item => item.customerSendMessageCnt);
        const resUserSendMessageList = res.data.map(item => item.userSendMessageCnt);
        this.$set(this.echarsOptionsData, 'xAxisData', resXAxisData);
        this.$set(this.echarsOptionsData, 'clientSendMessageList', resClientSendMessageList);
        this.$set(this.echarsOptionsData, 'userSendMessageList', resUserSendMessageList);
      });
    },
    // 查询
    onSearch() {
      this.getList(true);
      this.getChartData();
    },
    // 重置
    resetForm() {
      this.userAndDepartmentList = [];
      this.query = this.$options.data().query;
      this.pickerMinDate = '';
      this.addDateRange = [YESTERDAY_TIME, YESTERDAY_TIME];
      this.sendDateRange = [YESTERDAY_TIME, YESTERDAY_TIME];
      this.getList(true);
      this.getChartData();
    },
    // 导出报表
    exportForms() {
      this.getDiffExportFormsFun()(this.getSearchPayload()).then((res) => {
        this.download(res.data.msg, true);
      }).catch(() => {
        this.msgError('导出失败!');
      });
    },
    // 打开客户活跃度详情
    async openClientDialog(row) {
      this.detailsDialogVisible = true;
      this.showDetailsClient = row;
      await this.$nextTick(); // 保证子组件先拿到props
      this.$refs?.clientDetailsDialogRef?.getList(true);
    },
    // 跳转到客户资料详情页
    skipToUserDetails(row) {
      goRouteWithQuery(
        this.$router,
        '/customerManage/customerCenter/customerDetail',
        {},
        {
          id: row.externalUserId,
          userId: row.userId,
          prePageType: 'customer'
        }
      );
    }
  }
};
</script>
<style scoped lang='scss'>
.tendency-chart {
  margin-bottom: 20px;
  .chart-title {
    font-weight: 900;
    font-size: 24px;
    margin-bottom: 15px;
  }
  .chart-info {
    color: #606266;
    margin-bottom: 15px;
  }
  .echars-show {
    width: 100%;
    height: 350px;
  }
}
.data-details {
  .details-title {
    font-weight: 900;
    font-size: 24px;
    margin-bottom: 15px;
  }
  .forms-handle-btn {
    display: flex;
    justify-content: space-between;
    margin-bottom: 14px;
  }
}
</style>
