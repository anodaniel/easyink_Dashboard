<!--
 * @Description: 选择兑换活动弹窗
 * @Author: wJiaaa
 * @LastEditors: wJiaaa
-->
<template>
  <div class="exchange-activities">
    <!-- 触发详情弹窗 -->
    <el-dialog title="选择兑换活动" :visible="Pvisible" :close-on-click-modal="false" :before-close="onClose">
      <RightContainer>
        <template v-slot:search>
          <el-form
            ref="queryForm"
            :inline="true"
            :model="query"
            label-width="100px"
            style="height:40px"
            size="small"
          >
            <el-form-item>
              <el-input v-model="query.activityName" placeholder="请输入活动名称" />
            </el-form-item>
            <el-form-item label=" ">
              <el-button type="primary" @click="onSearch">查询</el-button>
              <el-button class="btn-reset" @click="resetForm">重置</el-button>
            </el-form-item>
          </el-form>
        </template>
        <template v-slot:data>
          <el-table
            ref="multipleTable"
            v-loading="loading"
            height="300"
            style="font-size: 14px"
            :data="list"
            @selection-change="handleSelectionChange"
          >
            <template slot="empty">
              <empty-default-icon
                text="暂无数据"
                :length="list.length"
              />
            </template>
            <el-table-column type="selection" width="55" align="center" />
            <el-table-column prop="activityName" label="活动名称" align="center" width="200">
              <template #default="{ row }">
                <el-popover
                  :open-delay="600"
                  placement="top-start"
                  trigger="hover"
                  popper-class="tip-popover"
                  :content="`${row.activityName}`"
                >
                  <div slot="reference" class="intwoline">{{ row.activityName }}</div>
                </el-popover>
              </template>
            </el-table-column>
            <el-table-column label="活动时间" align="center" width="200">
              <template #default="{ row }">
                <div v-if="row.effectStartTime">{{ row.effectStartTime }}~{{ row.effectEndTime }}</div>
              </template>
            </el-table-column>
            <el-table-column align="center" prop="status" width="200">
              <template #default="{ row }">
                <div>{{ row.remainInventory }}/{{ row.sumInventory }}</div>
              </template>
              <template #header>
                <div>
                  <span>库存</span>
                  <el-popover
                    placement="top-start"
                    trigger="hover"
                    popper-class="tip-popover"
                    content="库存即剩余未领取的兑换码"
                  >
                    <i
                      slot="reference"
                      class="iconfont icon-question"
                      style="font-size: 14px"
                    />
                  </el-popover>
                </div>
              </template>
            </el-table-column>
          </el-table>
        </template>
      </RightContainer>
      <div class="footer">
        <pagination
          v-if="total > 0"
          class="page"
          :total="total"
          :page.sync="query.pageNum"
          :limit.sync="query.pageSize"
          :pager-count="7"
          @pagination="getList()"
        />
        <div v-else class="footer-text">
          <div v-if="checkPermi(['redeeomCode:activity:add'])" class="use-active data-count-num" @click="openHowTouseActiveDrawer = true">
            如何使用兑换活动？
          </div>
          <!-- 无权限 -->
          <div v-else>
            没有可用的兑换活动，请联系上级创建
          </div>
        </div>
        <div class="btn">
          <el-button @click="Pvisible = false">取 消</el-button>
          <el-button type="primary" @click="submit">确 定</el-button>
        </div>
      </div>
    </el-dialog>
    <HowToUse :visible.sync="openHowTouseActiveDrawer" :pvisible.sync="Pvisible" />
  </div>
</template>

<script>
import EmptyDefaultIcon from '@/components/EmptyDefaultIcon';
import RightContainer from '@/components/RightContainer';
import { PAGE_LIMIT } from '@/utils/constant';
import { getConversionCodeActiveList } from '@/api/redeem';
import { checkPermi } from '@/utils/permission';
import HowToUse from '@/views/marketingActivities/components/HowToUse.vue';
const DEFAULT_PAGE_NUM = 1;
export default {
  components: { EmptyDefaultIcon, RightContainer, HowToUse },
  props: {
    visible: {
      type: Boolean,
      default: false
    },
    selectedActiveList: {
      type: Array,
      default: () => []
    }
  },
  data() {
    return {
      list: [],
      loading: false,
      query: {
        activityName: '',
        pageNum: DEFAULT_PAGE_NUM,
        pageSize: PAGE_LIMIT
      },
      openHowTouseActiveDrawer: false,
      total: 0,
      activeList: []
    };
  },
  computed: {
    Pvisible: {
      get() {
        return this.visible;
      },
      set(val) {
        this.$emit('update:visible', val);
      }
    }
  },
  watch: {
    /**
     * 监听原先已被选择的活动列表变化
     */
    selectedActiveList: {
      // 深度监听变化
      handler(curVal, oldVal) {
        this.activeList = curVal;
      },
      deep: true,
      immediate: true
    }
  },
  created() {
    this.getList();
  },
  updated() {
    this.dealSelectActive();
  },
  methods: {
    checkPermi(key) {
      return checkPermi(key);
    },
    /**
     * 处理显示已选中的兑换活动
     */
    dealSelectActive() {
      this.list.map((item) => {
        if (item.id === this.activeList[0]?.id) {
          this.$nextTick(() => {
            this.$refs.multipleTable && this.$refs.multipleTable.toggleRowSelection(item, true);
          });
        }
      });
    },
    /**
     * 获取活动列表
     */
    getList(pageNum) {
      this.loading = true;
      pageNum && (this.query.pageNum = pageNum);
      getConversionCodeActiveList(this.query).then(({ rows, total }) => {
        this.loading = false;
        this.list = rows;
        this.total = total;
      }).catch(() => {
        this.loading = false;
      });
    },
    /**
     * 多选框选中数据
     */
    handleSelectionChange(selection) {
      this.list.forEach(item => {
        if (selection[selection.length - 1] === item) {
          this.$refs.multipleTable.toggleRowSelection(item, true);
          this.activeList = [item];
        } else {
          this.$refs.multipleTable.toggleRowSelection(item, false);
        }
      });
    },
    /**
     * 查询
     */
    onSearch() {
      this.getList(1);
    },
    /**
     * 重置
     */
    resetForm() {
      this.query = {
        activeName: '',
        pageNum: 1,
        pageSize: PAGE_LIMIT
      };
      this.getList();
    },
    submit() {
      this.$emit('success', [...this.activeList]);
      this.Pvisible = false;
    },
    /**
     * 关闭弹窗
     */
    onClose() {
      this.$emit('update:visible', false);
    }
  }
};
</script>

<style scoped lang="scss">
.exchange-activities {
  /deep/ .el-dialog {
   width: auto;
  }
  /deep/ .el-dialog__wrapper {
    display: flex;
    align-items: center;
    justify-content: center;
  }
  /deep/.el-dialog__body {
    padding: 6px 20px;
  }
  .footer {
    height: 40px;
    margin-bottom: 10px;
    padding:0 15px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    .pagination-container {
      padding-bottom: 5px;
    }
    .btn {
      height: 20px;
    }
    .footer-text{
      margin-top: 20px;
      .use-active {
        color:#3AB2A6;
        cursor:pointer;
      }
    }
  }
}
</style>
