<template>
  <ms-container>
    <ms-main-container>
      <el-card class="table-card" v-loading="result.loading">
        <template v-slot:header>
          <ms-table-header :condition.sync="condition" @search="search"
                           :create-permission="['PROJECT_PERFORMANCE_TEST:READ+CREATE']"
                           @create="create" :createTip="$t('load_test.create')"/>
        </template>

        <ms-table
          :data="tableData"
          :condition="condition"
          :page-size="pageSize"
          :total="total"
          :operators="operators"
          :screenHeight="screenHeight"
          :field-key="tableHeaderKey"
          :remember-order="true"
          :enable-order-drag="enableOrderDrag"
          row-key="id"
          operator-width="190px"
          :screen-height="screenHeight"
          :enable-selection="false"
          @refresh="search"
          :disable-header-config="true"
          ref="table">

          <el-table-column
            prop="num"
            label="ID"
            width="100"
            show-overflow-tooltip>
          </el-table-column>
          <el-table-column
            prop="name"
            :label="$t('commons.name')"
            show-overflow-tooltip>
            <template v-slot:default="scope">
              <span @click="link(scope.row)" style="cursor: pointer;">{{ scope.row.name }}</span>
            </template>
          </el-table-column>
          <el-table-column
            prop="userName"
            sortable="custom"
            :filters="userFilters"
            column-key="user_id"
            :label="$t('load_test.user_name')"
            show-overflow-tooltip>
          </el-table-column>
          <el-table-column
            width="200"
            sortable
            prop="createTime"
            :label="$t('commons.create_time')">
            <template v-slot:default="scope">
              <span>{{ scope.row.createTime | timestampFormatDate }}</span>
            </template>
          </el-table-column>
          <el-table-column
            width="200"
            sortable
            prop="updateTime"
            :label="$t('commons.update_time')">
            <template v-slot:default="scope">
              <span>{{ scope.row.updateTime | timestampFormatDate }}</span>
            </template>
          </el-table-column>
          <el-table-column
            prop="reportCount"
            :label="$t('report.load_test_report')"
            width="150">
            <template v-slot:default="scope">
              <el-link v-if="scope.row.reportCount > 0" @click="reports(scope.row)">
                {{ scope.row.reportCount }}
              </el-link>
              <span v-else> {{ scope.row.reportCount }}</span>
            </template>
          </el-table-column>
          <el-table-column
            prop="status"
            column-key="status"
            :filters="statusFilters"
            :label="$t('commons.status')">
            <template v-slot:default="{row}">
              <ms-performance-test-status :row="row"/>
            </template>
          </el-table-column>
        </ms-table>
        <ms-table-pagination :change="initTableData" :current-page.sync="currentPage" :page-size.sync="pageSize"
                             :total="total"/>
      </el-card>
    </ms-main-container>
  </ms-container>
</template>

<script>
import MsTablePagination from "../../common/pagination/TablePagination";
import MsTableOperator from "../../common/components/MsTableOperator";
import MsContainer from "../../common/components/MsContainer";
import MsMainContainer from "../../common/components/MsMainContainer";
import MsPerformanceTestStatus from "./PerformanceTestStatus";
import MsTableOperators from "../../common/components/MsTableOperators";
import {getCurrentProjectID, getCurrentWorkspaceId} from "@/common/js/utils";
import MsTableHeader from "../../common/components/MsTableHeader";
import {TEST_CONFIGS} from "../../common/components/search/search-components";
import {getLastTableSortField, handleRowDrop} from "@/common/js/tableUtils";
import MsTable from "@/business/components/common/components/table/MsTable";
import {editLoadTestCaseOrder} from "@/network/load-test";

export default {
  components: {
    MsTable,
    MsTableHeader,
    MsPerformanceTestStatus,
    MsTablePagination,
    MsTableOperator,
    MsContainer,
    MsMainContainer,
    MsTableOperators
  },
  data() {
    return {
      tableHeaderKey:"PERFORMANCE_TEST_TABLE",
      result: {},
      deletePath: "/performance/delete",
      condition: {
        components: TEST_CONFIGS
      },
      projectId: null,
      tableData: [],
      multipleSelection: [],
      currentPage: 1,
      pageSize: 10,
      total: 0,
      loading: false,
      testId: null,
      enableOrderDrag: true,
      operators: [
        {
          tip: this.$t('commons.edit'), icon: "el-icon-edit",
          exec: this.handleEdit,
          permissions: ['PROJECT_PERFORMANCE_TEST:READ+EDIT']
        }, {
          tip: this.$t('commons.copy'), icon: "el-icon-copy-document", type: "success",
          exec: this.handleCopy,
          permissions: ['PROJECT_PERFORMANCE_TEST:READ+COPY']
        }, {
          tip: this.$t('commons.delete'), icon: "el-icon-delete", type: "danger",
          exec: this.handleDelete,
          permissions: ['PROJECT_PERFORMANCE_TEST:READ+DELETE']
        }
      ],
      statusFilters: [
        {text: 'Saved', value: 'Saved'},
        {text: 'Starting', value: 'Starting'},
        {text: 'Running', value: 'Running'},
        {text: 'Reporting', value: 'Reporting'},
        {text: 'Completed', value: 'Completed'},
        {text: 'Error', value: 'Error'}
      ],
      userFilters: [],
      screenHeight: 'calc(100vh - 200px)',
    };
  },
  watch: {
    '$route'(to) {
      if (to.name !== 'perPlan') {
        return;
      }
      this.projectId = to.params.projectId;
      this.initTableData();
    }
  },
  created: function () {
    this.projectId = this.$route.params.projectId;
    this.initTableData();
    this.getMaintainerOptions();
  },
  methods: {
    getMaintainerOptions() {
      let workspaceId = getCurrentWorkspaceId();
      this.$post('/user/project/member/tester/list', {projectId: getCurrentProjectID()}, response => {
        this.userFilters = response.data.map(u => {
          return {text: u.name, value: u.id};
        });
      });
    },
    initTableData() {
      this.condition.orders = getLastTableSortField(this.tableHeaderKey);

      this.enableOrderDrag = this.condition.orders.length > 0 ? false : true;

      this.condition.projectId = getCurrentProjectID();
      this.condition.workspaceId = getCurrentWorkspaceId();
      this.result = this.$post(this.buildPagePath('/performance/list'), this.condition, response => {
        let data = response.data;
        this.total = data.itemCount;
        this.tableData = data.listObject;
        // 查询报告数量
        this.tableData.forEach(test => {
          this.result = this.$get('/performance/test/report-count/' + test.id, response => {
            this.$set(test, 'reportCount', response.data);
          });
        });

        this.$nextTick(() => {
          handleRowDrop(this.tableData, (param) => {
            param.groupId = getCurrentProjectID();
            editLoadTestCaseOrder(param);
          });
          if (this.$refs.table) {
            this.$refs.table.clear();
          }
        });
      });
    },
    search(combine) {
      this.initTableData(combine);
    },
    buildPagePath(path) {
      return path + "/" + this.currentPage + "/" + this.pageSize;
    },
    handleSelectionChange(val) {
      this.multipleSelection = val;
    },
    handleEdit(test) {
      this.$router.push({
        path: '/performance/test/edit/' + test.id,
      });
    },
    handleCopy(test) {
      this.result = this.$post("/performance/copy", {id: test.id}, () => {
        this.$success(this.$t('commons.copy_success'));
        this.search();
      });
    },
    handleDelete(test) {
      this.$alert(this.$t('load_test.delete_confirm') + test.name + "？", '', {
        confirmButtonText: this.$t('commons.confirm'),
        callback: (action) => {
          if (action === 'confirm') {
            this._handleDelete(test);
          }
        }
      });
    },
    _handleDelete(test) {
      let data = {
        id: test.id
      };

      this.result = this.$post(this.deletePath, data, () => {
        this.$success(this.$t('commons.delete_success'));
        this.initTableData();
      });
    },
    link(row) {
      this.$router.push({
        path: '/performance/test/edit/' + row.id,
      });
    },
    reports(row) {
      this.$router.push({
        path: '/performance/report/' + row.id,
      });
    },
    create() {
      if (!getCurrentProjectID()) {
        this.$warning(this.$t('commons.check_project_tip'));
        return;
      }
      this.$router.push('/performance/test/create');
    }
  }
};
</script>

<style scoped>

.test-content {
  width: 100%;
}

.table-page {
  padding-top: 20px;
  margin-right: -9px;
  float: right;
}

</style>
