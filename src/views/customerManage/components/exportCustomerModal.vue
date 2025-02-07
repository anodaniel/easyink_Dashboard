<!--
 * @Description: 导出客户弹窗
 * @Author: broccoli
 * @LastEditors: broccoli
-->
<template>
  <el-dialog v-bind="$attrs" title="导出客户" class="export-customer-modal" v-on="$listeners" @close="onClose">
    <el-alert
      title="请设置要导出的客户信息字段"
      type="warning"
      show-icon
      :closable="false"
    />
    <div class="check-list">
      <span v-for="(item, index) in list" :key="index" class="check-item">
        <el-checkbox :value="checked(item)" @change="(value) => onCheck(value, item)">{{ item.name }}</el-checkbox>
      </span>
    </div>
    <div slot="footer">
      <el-button @click="onClose">取消</el-button>
      <el-button type="primary" :loading="loading" @click="handelConfirm">确定</el-button>
    </div>
  </el-dialog>
</template>
<script>
import * as api from '@/api/customer';
import store from '@/store';
export default {
  name: '',
  components: {},
  props: {
    list: {
      type: Array,
      default: () => []
    },
    query: {
      type: Object,
      default: () => {}
    }
  },
  data() {
    return {
      selected: [],
      loading: false
    };
  },
  computed: {
    /**
     * 所有已经启用的系统字段+自定义字段object格式
     */
    allCustomerPropertyIds() {
      const allList = this.$store.state.customerProperty.customerProperList.filter(item => item.status);
      const ids = allList.map(item => item.id) || [];
      return ids;
    },
    loginUserId() {
      return store.getters.userId;
    }
  },
  watch: {
    list(val) {
      this.dealSelected(val);
    }
  },
  mounted() {
    this.dealSelected(this.list);
  },
  methods: {
    dealSelected(list) {
      const defaultProperty = window.localStorage.getItem(`exportProperty-${this.loginUserId}-${store.getters.corpId}`);
      let defaultPropertyList;
      if (defaultProperty || defaultProperty === '') {
        defaultPropertyList = JSON.parse(defaultProperty)?.list;
      } else {
        defaultPropertyList = [...list];
      }
      // 过滤掉未启用的字段
      defaultPropertyList = defaultPropertyList.filter(item => [
        'customer', 'remark', 'source', 'createTime', 'userName', 'department', 'tag', 'status',
        ...this.allCustomerPropertyIds
      ].includes(item.id));
      this.selected = defaultPropertyList || [];
    },
    /**
     * 关闭弹窗
     */
    onClose() {
      this.$emit('update:visible', false);
    },
    handelConfirm() {
      const selectedProperties = this.selected && this.selected.map(item => item.name);
      if (!this.selected || this.selected.length === 0) {
        this.msgWarn('请至少选择一个字段');
        return;
      }
      const queryParams = {
        ...this.query,
        selectedProperties: selectedProperties.join(',')
      };
      window.localStorage.setItem(`exportProperty-${this.loginUserId}-${store.getters.corpId}`, JSON.stringify({ list: this.selected }));
      this.loading = true;
      api.exportCustomer(queryParams)
        .then((response) => {
          this.loading = false;
          this.download(response.msg);
          this.onClose();
        }).catch(e => { this.loading = false; });
    },
    checked(item) {
      return this.selected && this.selected.findIndex(selectItem => selectItem.id === item.id) > -1;
    },
    onCheck(value, item) {
      let list = [...this.selected];
      if (value) {
        list.push({ id: item.id, name: item.name });
      } else {
        list = list.filter(listItem => listItem.id !== item.id);
      }
      this.selected = [...list];
    }
  }

};
</script>
<style scoped lang='scss'>
.export-customer-modal {
    .el-alert {
        margin-bottom: 15px;
    }
    .check-item {
        margin-right: 20px;
        line-height: 32px;
    }
    .check-list {
      max-height: 300px;
      overflow: auto;
    }
}
</style>
