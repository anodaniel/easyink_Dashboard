<script>
import * as api from '@/api/customer';
import store from '@/store';
import { getList as getListTag } from '@/api/customer/tag';
import RightContainer from '@/components/RightContainer';
import EmptyDefaultIcon from '@/components/EmptyDefaultIcon';
import AddTag from '@/components/AddTag';
import SelectUser from '@/components/SelectUser/index.vue';
import SelectTag from '@/components/SelectTag';
import { EventBus } from '@/event-bus.js';
import { goRouteWithQuery } from '@/utils';
import { PAGE_LIMIT, CUSTOMER_PROPERTY_VALUE, WX_TYPE } from '@/utils/constant';
import PropertySetting from './components/propertySetting.vue';
import { dealShowText, dealAtInfo } from '@/utils/common';
import ExportCustomerModal from './components/exportCustomerModal.vue';
import { download } from '@/utils/download';

const COMMON_TYPE = '0';
const LOSS_TYPE = '1';
const TRANSFER_TYPE = '4';
const TO_INHERIT_TYPE = '3';
const ALL_TYPE = '-1';
const CUSTOMER_STATUS = {
  [COMMON_TYPE]: '正常',
  [LOSS_TYPE]: '已流失',
  [TO_INHERIT_TYPE]: '待继承',
  [TRANSFER_TYPE]: '转接中',
  [ALL_TYPE]: '全部'
};

const fieldObj = {
  '邮箱': 'email',
  '地址': 'address',
  '描述': 'description',
  '电话': 'remarkMobiles',
  '出生日期': 'birthday'
};

export default {
  name: 'Customer',
  components: { AddTag, SelectUser, SelectTag, RightContainer, EmptyDefaultIcon, PropertySetting, ExportCustomerModal },
  props: {},
  data() {
    return {
      query: {
        pageNum: 1,
        pageSize: PAGE_LIMIT,
        name: '', // "客户名称",
        userIds: '', // "添加人id",
        tagIds: '', // "标签id,多个标签，id使用逗号隔开",
        beginTime: '', // "开始时间",
        endTime: '', // "结束时间"
        status: null
      },
      queryTag: [], // 搜索框选择的标签
      queryUser: [], // 搜索框选择的添加人
      dateRange: [], // 添加日期
      loading: false,
      isMoreFilter: false,
      total: 0,
      // 添加标签表单
      form: {
        groupName: '',
        weTags: []
      },
      list: [], // 客户列表
      multipleSelection: [], // 多选数组
      dialogVisible: false, // 选择标签弹窗显隐
      dialogVisibleSelectUser: false, // 选择添加人弹窗显隐
      dialogVisibleAddTag: false, // 添加标签弹窗显隐
      selectedGroup: '', // 选择的标签分组
      selectedTags: [], // 选择的标签
      removeTag: [], // 可移除的标签
      tagDialogType: {
        title: '', // 选择标签弹窗标题
        type: '' // 弹窗类型
      },
      removeTags: [],
      customerStatus: [{
        label: CUSTOMER_STATUS[ALL_TYPE],
        value: ALL_TYPE
      }, {
        label: CUSTOMER_STATUS[COMMON_TYPE],
        value: COMMON_TYPE
      }, {
        label: CUSTOMER_STATUS[LOSS_TYPE],
        value: LOSS_TYPE
      }, {
        label: CUSTOMER_STATUS[TRANSFER_TYPE],
        value: TRANSFER_TYPE
      }, {
        label: CUSTOMER_STATUS[TO_INHERIT_TYPE],
        value: TO_INHERIT_TYPE
      }],
      // 显示字段设置弹窗
      properSettingVisible: false,
      showCustomerPropertyList: [],
      // 自定义显示的列
      columnList: [],
      CUSTOMER_PROPERTY_VALUE,
      tagDialogLoading: false, // 选择标签弹窗的加载状态
      // 导出客户弹窗显示
      exportVisible: false,
      // 客户状态查询传参枚举
      CUSTOMER_STATUS,
      // 去重后的客户总数
      ignoreDuplicateCount: 0,
      // 客户总数
      totalCount: 0,
      fieldObj,
      wxType: WX_TYPE
    };
  },
  computed: {
    /**
     * 导出客户显示的字段
     */
    exportPropertyList() {
      const allPropertyList = [...this.$store.state.customerProperty.customerProperList] || [];
      const customList = allPropertyList.filter(item => item.type !== CUSTOMER_PROPERTY_VALUE['sysField'] && item.status);
      const sysList = allPropertyList.filter(item => item.type === CUSTOMER_PROPERTY_VALUE['sysField'] && item.status);
      // 默认字段
      const defaultList = [{ id: 'customer', name: '客户' },
        { id: 'remark', name: '备注' },
        { id: 'source', name: '来源' },
        { id: 'createTime', name: '添加时间' },
        { id: 'userName', name: '所属员工' },
        { id: 'department', name: '所属部门' },
        { id: 'tag', name: '标签' },
        { id: 'status', name: '客户状态' }];
        // 系统字段，根据是否启用来决定是否展示
      const extraList = [
        { name: '出生日期' },
        { name: '电话' },
        { name: '邮箱' },
        { name: '地址' },
        { name: '描述' }];
      const filterExtraList = [];
      extraList.map(item => {
        const allIndex = sysList.findIndex(allItem => allItem.name === item.name);
        if (allIndex > -1) {
          filterExtraList.push(sysList[allIndex]);
        }
      });
      const list = [
        ...defaultList,
        ...filterExtraList,
        ...customList
      ];
      return list;
    }
  },
  watch: {
    '$store.state.customerProperty.customerProperList'(val) {
      this.dealShowList(val);
    }
  },
  created() {
    if (this.$route.query) {
      Object.keys(this.query).forEach(key => {
        if (this.$route.query[key]) {
          this.query[key] = this.$route.query[key];
        }
      });
    }
    this.getPropertyList();
    this.getList();
    this.getCount();
    this.getListTag();

    this.$store.dispatch(
      'app/setBusininessDesc',
      `
        <div>查看当前企业下所有客户信息，可对客户批量添加标签</div>
      `
    );
  },
  mounted() {},
  methods: {
    /**
     * 获取客户自定义字段
     */
    getPropertyList() {
      if (!this.$store.state.customerProperty.customerProperList || this.$store.state.customerProperty.customerProperList.length === 0) {
        this.$store.dispatch('GetCustomerProperty').catch(e => {
          this.dealShowList(this.$store.state.customerProperty.customerProperList);
        });
      } else {
        this.dealShowList(this.$store.state.customerProperty.customerProperList);
      }
    },
    deleteTag(tag) {
      const index = this.queryTag.findIndex(tag_ => tag_.tagId === tag.tagId);
      this.queryTag.splice(index, 1);
      this.query.tagIds = this.queryTag.map((tag) => tag.tagId) + '';
    },
    /**
     * 获取客户去重统计数
     */
    getCount() {
      const params = {
        ...this.query,
        status: this.query.status === ALL_TYPE ? null : this.query.status
      };
      api.getCustomerSum(params).then(({ data }) => {
        this.ignoreDuplicateCount = data?.ignoreDuplicateCount || 0;
        this.totalCount = data?.totalCount || 0;
      });
    },
    getList(page) {
      if (this.dateRange) {
        this.query.beginTime = this.dateRange[0];
        this.query.endTime = this.dateRange[1];
      } else {
        this.query.beginTime = '';
        this.query.endTime = '';
      }
      page && (this.query.pageNum = page);
      this.loading = true;
      const params = {
        ...this.query,
        status: this.query.status === ALL_TYPE ? null : this.query.status
      };
      api
        .getList(params)
        .then(({ rows, total }) => {
          this.list = rows;
          this.total = +total;
          this.loading = false;
          this.multipleSelection = [];
        })
        .catch(() => {
          this.loading = false;
        });
    },
    getListTag(refresh, selectedTagFn) {
      getListTag().then(({ rows }) => {
        this.$store.dispatch('listInfo/setTagList', { list: rows });
        this.listTagOneArray = [];
        rows.forEach((element) => {
          element.weTags.forEach((tag) => {
            const newTag = { ...tag, groupName: element.groupName };
            this.listTagOneArray.push(newTag);
            if (selectedTagFn && selectedTagFn(newTag)) {
              this.selectedTags.push(newTag);
            }
          });
        });
        if (refresh) {
          this.$refs.selectTag.getList();
          this.form = {
            groupName: '',
            weTags: []
          };
        }
      });
    },
    showTagDialog() {
      this.selectedTags = [...this.queryTag];
      this.tagDialogType = {
        title: '选择标签',
        type: 'query'
      };
      this.dialogVisible = true;
      // this.$refs.selectTag.$forceUpdate()
    },
    makeTag(type) {
      if (!this.multipleSelection.length) {
        let msg = '请选择一位客户';
        if (type === 'add') msg = '请至少选择一位客户';
        this.msgWarn(msg);
        return;
      }
      this.selectedTags = [];
      const hasErrorTag = [];
      const repeat = [];
      this.multipleSelection.forEach((element) => {
        element.weFlowerCustomerTagRels.forEach((grandchild) => {
          // 判断是否有重复标签
          const isRepeat = this.selectedTags.some((tag) => {
            return tag.tagId === grandchild.tagId;
          });
          // 去重
          if (isRepeat) {
            repeat.push(grandchild.tagName);
            return;
          }

          const filter = this.listTagOneArray.find((tag) => {
            return tag.tagId === grandchild.tagId;
          });
          // 如果没有匹配到，则说明该便签处于异常状态，可能已被删除或破坏
          if (!filter) {
            hasErrorTag.push(grandchild.tagName);
            // isError = true
            return;
          }

          type !== 'add' && this.selectedTags.push(filter);
        });
      });
      if (hasErrorTag.length > 0) {
        this.msgError('已有标签[' + hasErrorTag + ']不在标签库中，或存在异常');
        return;
      }
      if (type === 'remove' && this.selectedTags.length === 0) {
        this.msgError('该客户没有标签，不可进行移除');
        return;
      }
      if (type === 'remove') {
        this.removeTags = [...this.selectedTags];
        this.selectedTags = [];
      }

      this.tagDialogType = {
        title:
          (type === 'add' ? '批量打标签' : '批量移除标签'),
        infoMsg:
          (type === 'add' ? '请选择要为已勾选客户打上的标签' : '以下选中的标签，将从已勾选客户下移除'),
        type: type
      };
      this.dialogVisible = true;
      // this.$refs.selectTag.$forceUpdate()
    },
    sync() {
      const loading = this.$loading({
        lock: true,
        text: 'Loading',
        spinner: 'el-icon-loading',
        background: 'rgba(0, 0, 0, 0.7)'
      });
      api
        .sync()
        .then(() => {
          loading.close();
          this.msgSuccess('后台开始同步数据，请稍后关注进度');
        })
        .catch((fail) => {
          loading.close();
          console.log(fail);
        });
    },
    /**
     * 导出按钮操作
     */
    exportCustomer() {
      this.exportVisible = true;
    },
    selectedUser(list) {
      this.queryUser = list;
      this.query.userIds = list.map(user => user.userId) + '';
    },
    submitSelectTag(selected) {
      if (this.tagDialogType.type === 'query') {
        this.query.tagIds = selected.map(tag => tag.tagId) + '';
        // debugger;
        this.queryTag = [...selected];
        this.dialogVisible = false;
      } else {
        this.tagDialogLoading = true;
        if (this.tagDialogType.type === 'add') {
          const data = this.multipleSelection.map(cus => {
            const allTag = [...cus.weFlowerCustomerTagRels];
            selected.forEach(sTag => {
              if (!allTag.find(tag => tag.tagId === sTag.tagId)) {
                allTag.push(sTag);
              }
            });
            return {
              externalUserid: cus.externalUserid,
              userId: cus.userId,
              addTag: allTag
            };
          });
          api.makeLabelbatch(data).then(() => {
            this.tagDialogLoading = false;
            this.msgSuccess('操作成功');
            this.dialogVisible = false;
            this.getList();
          }).catch(() => {
            this.tagDialogLoading = false;
          });
        } else {
          const data = {
            customerList: this.multipleSelection.map(cus => ({ externalUserid: cus.externalUserid, userId: cus.userId })),
            weTagIdList: selected.map(ele => ele.tagId)
          };
          api.removeLabel(data).then(() => {
            this.tagDialogLoading = false;
            this.msgSuccess('操作成功');
            this.dialogVisible = false;
            this.getList();
          }).catch(() => {
            this.tagDialogLoading = false;
          });
        }
      }
    },
    resetForm(formName) {
      this.dateRange = [];
      this.queryTag = [];
      this.queryUser = [];
      this.selectedTags = [];
      this.$refs['queryForm'].resetFields();
      this.query.userIds = '';
      this.query.tagIds = '';
      this.getList(1);
      this.getCount();
      EventBus.$emit('resetTag');
      EventBus.$emit('resetUser');
    },
    // 多选框选中数据
    handleSelectionChange(selection) {
      this.multipleSelection = selection;
    },
    // 用于多选变单选
    handleSelection(selection, row) {
      this.$nextTick(() => {
        this.$refs.table.clearSelection();
        this.$refs.table.toggleRowSelection(row, true);
      });
    },
    goRoute(row) {
      goRouteWithQuery(
        this.$router,
        'customerDetail',
        this.query,
        {
          id: row.externalUserid,
          userId: row.userId,
          dimissionTime: row.dimissionTime,
          prePageType: 'customer'
        }
      );
    },
    showAddTagDialog(data) {
      this.form = JSON.parse(
        JSON.stringify({ weTags: [], ...data })
      );
      this.dialogVisibleAddTag = true;
    },
    addTagFn(addTags) {
      const addTagFilterFn = (tag) => addTags.find(addTag => addTag.groupName === tag.groupName && addTag.name === tag.name);
      this.getListTag(true, addTagFilterFn);
    },
    /**
     * 打开字段显示设置弹窗
     */
    openSetting() {
      this.properSettingVisible = true;
    },
    dealShowList(list = this.$store.state.customerProperty.customerProperList) {
      const extraList = [{ id: 'createTime', name: '添加时间' },
        { id: 'userName', name: '所属员工' },
        { id: 'tag', name: '标签' },
        { id: 'status', name: '客户状态' }];
      this.showCustomerPropertyList = [
        ...extraList,
        ...list.filter(customerProperItem => customerProperItem.status)];
      const columnList = [];
      let newIdList = [];
      // 是否展示默认字段（用户未自定义设置展示字段则显示默认字段）
      const idList = window.localStorage.getItem(`showCustomerProperty-${store.getters.userId}-${store.getters.corpId}`);
      if (idList || idList === '') {
        newIdList = idList.split(',');
      } else {
        newIdList = ['createTime', 'userName', 'tag', 'status'];
      }
      if (this.showCustomerPropertyList && this.showCustomerPropertyList.length > 0) {
        this.showCustomerPropertyList.map(item => {
          if (newIdList.includes(item.id)) {
            columnList.push(item);
          }
        });
      } else {
        newIdList.map(item => {
          columnList.push({ id: item });
        });
      }
      this.columnList = [...columnList];
      this.$nextTick(() => {
        this.$refs.table && this.$refs.table.doLayout();
      });
    },
    getColumnList() {
      this.dealShowList();
    },
    dealShowText(colunmItem, row, type) {
      return dealShowText(colunmItem, row, type)?.value;
    },
    /**
     * 点击查询
     */
    onSearch() {
      this.getList(1);
      this.getCount();
    },
    /**
     * 下载文件
     */
    downloadFile(url) {
      download(url);
    },
    renderUserInfo(item) {
      return dealAtInfo(item);
    }
  }
};
</script>

<template>
  <RightContainer :config-keys="['contactSecret', 'customSecret']" page-title="查看客户信息">
    <template v-slot:search>
      <el-form
        ref="queryForm"
        :inline="true"
        :model="query"
        label-width="100px"
        class="top-search"
        size="small"
      >
        <el-form-item prop="name">
          <el-input v-model="query.name" placeholder="请输入客户名称" />
        </el-form-item>
        <el-form-item prop="status">
          <el-select v-model="query.status" placeholder="请选择客户状态" clearable>
            <el-option
              v-for="item in customerStatus"
              :key="item.value"
              :label="item.label"
              :value="item.value"
            />
          </el-select>
        </el-form-item>
        <el-form-item>
          <div class="tag-input" @click="dialogVisibleSelectUser = true">
            <span v-if="!queryUser.length" class="tag-place">请选择员工</span>
            <template v-else>
              <el-tag
                v-for="(unit, unique) in queryUser"
                :key="unique"
                type="info"
              >{{ unit.name }}</el-tag>
            </template>
          </div>
        </el-form-item>
        <el-form-item>
          <div class="tag-input" @click="showTagDialog">
            <span v-if="!queryTag.length" class="tag-place">请选择标签</span>
            <template v-else>
              <el-tag
                v-for="(unit, unique) in queryTag"
                :key="unique"
                type="info"
                closable
                @close="deleteTag(unit)"
              >{{ unit.name }}</el-tag>
            </template>
          </div>
        </el-form-item>
        <el-form-item>
          <el-date-picker
            v-model="dateRange"
            value-format="yyyy-MM-dd"
            type="daterange"
            :picker-options="pickerOptions"
            range-separator="至"
            start-placeholder="开始日期"
            end-placeholder="结束日期"
            align="right"
          />
        </el-form-item>
        <el-form-item label=" ">
          <el-button
            type="primary"
            @click="onSearch"
          >查询</el-button>
          <el-button
            class="btn-reset"
            @click="resetForm()"
          >重置</el-button>
        </el-form-item>
      </el-form>
    </template>
    <template v-slot:data-stat>
      <div class="total-text">
        <div>
          共 <span class="theme-text-color">{{ totalCount }}</span> 个客户
        </div>
        <div>
          去重后 <span class="theme-text-color">{{ ignoreDuplicateCount }}</span> 个客户
          <el-popover
            placement="top-start"
            trigger="hover"
            popper-class="tip-popover"
          >
            <div>
              若客户添加多个员工，不重复计数
            </div>
            <i slot="reference" class="iconfont icon-question" style="font-size: 14px;" />
          </el-popover>
        </div>
      </div>
    </template>
    <template v-slot:operate-btn>
      <el-button
        v-hasPermi="['customerManage:customer:sync']"
        v-preventReClick
        @click="sync"
      >同步客户</el-button>
      <el-button
        v-hasPermi="['customerManage:customer:makeTag']"
        @click="makeTag('add')"
      >批量打标签</el-button>
      <el-button
        v-hasPermi="['customerManage:customer:removeTag']"
        @click="makeTag('remove')"
      >批量移除标签</el-button>
      <el-button
        v-hasPermi="['customerManage:customer:export']"
        @click="exportCustomer"
      >导出客户</el-button>
    </template>
    <template v-slot:data>
      <el-table
        ref="table"
        :data="list"
        tooltip-effect="dark"
        highlight-current-row
        @selection-change="handleSelectionChange"
      >
        <template slot="empty">
          <empty-default-icon
            text="暂无数据"
            desc="若首次加载，请耐心等待后台数据同步，稍后回来"
            :desc-show-condition="{ name: query.name, userIds: query.userIds, tagIds: query.tagIds, beginTime: query.beginTime, endTime: query.endTime }"
            :desc-config="['customSecret']"
          />
        </template>
        <el-table-column
          key="row-1"
          type="selection"
          align="center"
          width="55"
        />
        <el-table-column key="row-2" label="客户" prop="name" align="left" fixed min-width="280">
          <template slot-scope="{ row }">
            <div class="customer-info cp" @click="goRoute(row)">
              <img :src="row.avatar">
              <div class="customer-name">
                <div class="inoneline" :title="(row.remark || row.name || row.alias) + renderUserInfo(row)">
                  <span>{{ row.remark || row.name || row.alias }}</span>
                  <span :class="row.type === wxType ? 'wx-type-flag' : 'corp-type-name'">{{
                    renderUserInfo(row)
                  }}</span>
                </div>
                <div class="customer-real-name inoneline" :title="row.name">{{ row.name }}</div>
              </div>
            </div>
          </template>
        </el-table-column>
        <el-table-column
          v-for="(colunmItem, columnIndex) in columnList"
          :key="`${columnIndex}-${colunmItem.id}`"
          :label="colunmItem.name"
          prop="name"
          align="left"
          min-width="200"
        >
          <template #header>
            <div v-if="colunmItem.id === 'status'">
              <span>客户状态</span>
              <el-popover
                placement="top-start"
                trigger="hover"
                popper-class="tip-popover"
              >
                <div>
                  <div>1. 已流失：客户已将所属员工删除或拉黑</div>
                  <div>2. 转接中：在职员工或离职员工的客户已分配给其他员工，待添加被继承人</div>
                  <div>3. 待继承：离职员工的客户，等待被分配给在职员工</div>
                </div>
                <i slot="reference" class="iconfont icon-question" style="font-size: 14px;" />
              </el-popover>
            </div>
            <span v-else>
              {{ colunmItem.name }}
            </span>
          </template>
          <template slot-scope="{ row }">
            <div v-if="colunmItem.id === 'createTime'">
              {{ row.createTime || '' }}
            </div>
            <div v-else-if="colunmItem.id === 'userName'">
              <div class="al">
                <div class="cus-owner">{{ row.userName || '' }}</div>
                <div class="cus-dept">{{ row.department || '' }}</div>
              </div>
            </div>
            <div v-else-if="colunmItem.id === 'tag'">
              <el-tag
                v-for="(unit, unique) in row.weFlowerCustomerTagRels"
                :key="unique"
                type="info"
              >{{ unit.tagName }}</el-tag>
            </div>
            <div v-else-if="colunmItem.id === 'status'">
              {{ CUSTOMER_STATUS[row.status] || '' }}
            </div>
            <div v-else>
              <div v-if="colunmItem.type === CUSTOMER_PROPERTY_VALUE['sysField']">
                {{ row[fieldObj[colunmItem.name]] }}
              </div>
              <div
                v-if="[CUSTOMER_PROPERTY_VALUE['singleText'],
                       CUSTOMER_PROPERTY_VALUE['mulText'],
                       CUSTOMER_PROPERTY_VALUE['dateField']
                ].includes(colunmItem.type)"
              >
                {{ dealShowText(colunmItem, row, 'textType') }}
              </div>
              <div
                v-if="[
                  CUSTOMER_PROPERTY_VALUE['singleRadio'],
                  CUSTOMER_PROPERTY_VALUE['selectSingle']
                ].includes(colunmItem.type)"
              >{{ dealShowText(colunmItem, row, 'singleType') }}</div>
              <div
                v-if="[
                  CUSTOMER_PROPERTY_VALUE['mulRadio'],
                ].includes(colunmItem.type)"
              >
                <el-tag
                  v-for="(mulItem, mulIndex) in dealShowText(colunmItem, row, 'mulType')"
                  :key="mulIndex"
                  type="info"
                >{{ mulItem }}</el-tag>
              </div>
              <div v-if="colunmItem.type === CUSTOMER_PROPERTY_VALUE['img']">
                <el-image
                  v-for="(imgItem, imgIndex) in dealShowText(colunmItem, row, 'imgFile')"
                  :key="imgIndex"
                  :src="imgItem"
                  style="height: 70px; width: 70px;margin-right: 5px;"
                  fit="cover"
                  :preview-src-list="[imgItem]"
                />
              </div>
              <div v-if="colunmItem.type === CUSTOMER_PROPERTY_VALUE['file']" class="file-preview-list">
                <div
                  v-for="(fileItem, fileIndex) in dealShowText(colunmItem, row, 'fileType')"
                  :key="fileIndex"
                  @click="downloadFile(fileItem.url)"
                >{{ fileItem.name }}</div>
              </div>
            </div>
          </template>
        </el-table-column>
        <el-table-column key="row-3" label="操作" width="80" fixed="right">
          <template #header>
            <div class="customer-status">
              <span>操作</span>
              <el-popover
                placement="top-start"
                trigger="hover"
                popper-class="tip-popover"
                content="设置客户显示信息"
              >
                <i
                  slot="reference"
                  class="property-setting el-icon-setting"
                  style="font-size: 16px; vertical-align: text-bottom; cursor:pointer;"
                  @click="openSetting"
                />
              </el-popover>
            </div>
          </template>
          <template slot-scope="{ row }">
            <el-button
              v-hasPermi="['customerManage:customer:view']"
              type="text"
              size="small"
              @click="goRoute(row)"
            >详情</el-button>
            <!-- <el-button type="text" size="small">编辑</el-button> -->
          </template>
        </el-table-column>
      </el-table>
      <pagination
        v-show="total > 0"
        :total="total"
        :page.sync="query.pageNum"
        :limit.sync="query.pageSize"
        :select-data-len="multipleSelection.length"
        @pagination="getList()"
      />

      <!-- 选择标签弹窗 -->
      <SelectTag
        ref="selectTag"
        :visible.sync="dialogVisible"
        :title="tagDialogType.title"
        :selected="selectedTags"
        :type="tagDialogType.type"
        :info-msg="tagDialogType.infoMsg"
        :add-tag-fn="showAddTagDialog"
        :remove-tags="removeTags"
        :loading="tagDialogLoading"
        @success="submitSelectTag"
      />

      <!-- 选择添加人弹窗 -->
      <SelectUser
        :visible.sync="dialogVisibleSelectUser"
        title="选择添加人"
        @success="selectedUser"
      />

      <!-- 添加标签弹窗 -->
      <AddTag
        :visible.sync="dialogVisibleAddTag"
        :is-add-tag="true"
        :form="form"
        @success="addTagFn"
      />
      <PropertySetting :visible.sync="properSettingVisible" :data.sync="showCustomerPropertyList" @getList="getColumnList" />
      <ExportCustomerModal :visible.sync="exportVisible" :list="exportPropertyList" :query="query" />
    </template>
  </RightContainer>

</template>

<style lang="scss" scoped>
.iconfont {
  font-size: 16px;
  margin-left: 4px;
  color: #707070;
  &.icon-boy {
    color: #1296db;
  }
  &.icon-girl {
    color: #d62f81;
  }
}
.bfc-d + .bfc-d .el-checkbox:first-child {
  margin-left: 10px;
}
.cp {
  display: flex;
  align-items: center;
  justify-content: center;
}

.cus-owner {
  font-size: 14px;
  color: #333;
}
.cus-dept {
  font-size: 12px;
  color: #666;
}
.el-tag {
  height: 18px;
  line-height: 12px;
  font-size: 12px;
  padding: 3px;
  border-radius: 3px;
  margin-bottom: 2px;
}
.total-text {
  div {
    line-height: 18px;
    span {
      font-weight: bold;
    }
  }
}
.customer-status {
  display: flex;
  align-items: center;
  .property-setting {
    margin-left: 10px;
  }
}
.file-preview-list {
  div {
    color: #377AFF;
    cursor: pointer;
  }
}
</style>
