<script>
import * as api from '@/api/task';
import { PAGE_LIMIT } from '@/utils/constant';
const defaultFissionType = 2;
export default {
  name: 'Group',
  data() {
    return {
      query: {
        pageNum: 1,
        pageSize: PAGE_LIMIT,
        taskName: '',
        startTime: '',
        overTime: '',
        fissionType: defaultFissionType
      },
      dateRange: [],
      tableData: [],
      total: 0,
      loading: false
    };
  },
  created() {
    this.getList();

    this.$store.dispatch(
      'app/setBusininessDesc',
      `
        <div>用于查看当前企业所有的客户列表及详细信息，支持对客户进行打标签。</div>
      `
    );
  },
  methods: {
    getList(data) {
      this.loading = true;
      const params = Object.assign({}, this.query, data);
      api.getList(params).then(({ rows, total }) => {
        this.tableData = rows;
        this.total = +total;
        this.loading = false;
      });
    },
    resetForm() {},
    toDetail(row) {
      this.$router.push({
        path: `fissionDetail?id=${row.id}`
      });
    },
    newAdd() {
      this.$router.push({
        path: 'addFission'
      });
    },
    toEdit(row) {
      this.$router.push({
        path: `addFission?id=${row.id}`
      });
    }
  }
};
</script>

<template>
  <div class="app-container">
    <el-form
      ref="queryForm"
      :inline="true"
      :model="query"
      class="top-search"
      size="small"
    >
      <el-form-item label="裂变名" prop="taskName">
        <el-input v-model="query.taskName" placeholder="请输入" />
      </el-form-item>
      <el-form-item label="添加日期">
        <el-date-picker
          v-model="dateRange"
          type="daterange"
          range-separator="至"
          start-placeholder="开始日期"
          end-placeholder="结束日期"
          align="right"
        />
      </el-form-item>
      <el-form-item label=" ">
        <el-button
          type="primary"
          :loading="loading"
          @click="getList({ pageNum: 1 })"
        >查询</el-button>
        <el-button
          type="primary"
          style="background: #FA7216;color: #FFFFFF;border-color:#FA7216"
          @click="newAdd()"
        >新增任务</el-button>
      </el-form-item>
    </el-form>
    <el-table :data="tableData" border style="width: 100%">
      <el-table-column prop="taskName" label="群裂变名称" />
      <el-table-column prop="fissNum" label="裂变客户数量" />
      <el-table-column prop="fissStatus" label="活动状态">
        <template slot-scope="scope">
          <p>{{ scope.row.fissStatus === 1 ? '进行中' : '已结束' }}</p>
        </template>
      </el-table-column>
      <el-table-column prop="startTime" label="活动时间">
        <template slot-scope="scope">
          <p>{{ scope.row.startTime }}-{{ scope.row.overTime }}</p>
        </template>
      </el-table-column>
      <el-table-column prop="operation" label="操作">
        <template slot-scope="scope">
          <el-button
            v-hasPermi="['enterpriseWechat:view']"
            size="medium"
            type="text"
            icon="el-icon-view"
            @click="toDetail(scope.row)"
          />
          <el-button
            v-if="scope.row.fissStatus != 2"
            v-hasPermi="['enterpriseWechat:edit']"
            size="medium"
            type="text"
            icon="el-icon-edit-outline"
            style="color:#E74E59"
            @click="toEdit(scope.row)"
          />
        </template>
      </el-table-column>
    </el-table>
    <pagination
      v-show="total > 0"
      :total="total"
      :page.sync="query.pageNum"
      :limit.sync="query.pageSize"
      @pagination="getList()"
    />
  </div>
</template>

<style lang="scss" scoped></style>
