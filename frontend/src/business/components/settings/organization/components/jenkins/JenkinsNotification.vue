<template>
  <div>
    <el-row>
      <el-col :span="24">
        <h3>{{ $t('organization.message.jenkins_task_notification') }}</h3>
        <el-button icon="el-icon-circle-plus-outline" plain size="mini" @click="handleAddTaskModel"
                   v-permission="['ORGANIZATION_MESSAGE:READ+EDIT']">
          {{ $t('organization.message.create_new_notification') }}
        </el-button>
        <el-popover
          placement="right-end"
          title="示例"
          width="600"
          trigger="click">
          <ms-code-edit :read-only="true" height="400px" :data.sync="title" :modes="modes" :mode="'html'"/>
          <el-button icon="el-icon-warning" plain size="mini" slot="reference">
            {{ $t('organization.message.mail_template_example') }}
          </el-button>
        </el-popover>
        <el-popover
          placement="right-end"
          title="示例"
          width="600"
          trigger="click">
          <ms-code-edit :read-only="true" height="200px" :data.sync="robotTitle" :modes="modes" :mode="'text'"/>
          <el-button icon="el-icon-warning" plain size="mini" slot="reference">
            {{ $t('organization.message.robot_template') }}
          </el-button>
        </el-popover>


      </el-col>
    </el-row>
    <el-row>
      <el-col :span="24">
        <el-table
          :data="jenkinsTask"
          class="tb-edit"
          border
          :cell-style="rowClass"
          :header-cell-style="headClass">
          <el-table-column :label="$t('schedule.event')" min-width="15%" prop="events">
            <template slot-scope="scope">
              <el-select v-model="scope.row.event"
                         :placeholder="$t('organization.message.select_events')"
                         size="mini"
                         prop="events" :disabled="!scope.row.isSet">
                <el-option
                  v-for="item in jenkinsEventOptions"
                  :key="item.value"
                  :label="item.label"
                  :value="item.value">
                </el-option>
              </el-select>
            </template>
          </el-table-column>
          <el-table-column :label="$t('schedule.receiver')" prop="userIds" min-width="20%">
            <template v-slot:default="{row}">
              <el-select v-model="row.userIds" filterable multiple size="mini"
                         :placeholder="$t('commons.please_select')" style="width: 100%;" :disabled="!row.isSet">
                <el-option
                  v-for="item in jenkinsReceiverOptions"
                  :key="item.id"
                  :label="item.name"
                  :value="item.id">
                </el-option>
              </el-select>
            </template>
          </el-table-column>
          <el-table-column :label="$t('schedule.receiving_mode')" min-width="20%" prop="type">
            <template slot-scope="scope">
              <el-select v-model="scope.row.type" :placeholder="$t('organization.message.select_receiving_method')"
                         size="mini"
                         :disabled="!scope.row.isSet" @change="handleEdit(scope.$index, scope.row)"
              >
                <el-option
                  v-for="item in receiveTypeOptions"
                  :key="item.value"
                  :label="item.label"
                  :value="item.value">
                </el-option>
              </el-select>
            </template>
          </el-table-column>
          <el-table-column label="webhook" min-width="20%" prop="webhook">
            <template v-slot:default="scope">
              <el-input v-model="scope.row.webhook" placeholder="webhook地址"
                        size="mini"
                        :disabled="!scope.row.isSet||!scope.row.isReadOnly"></el-input>
            </template>
          </el-table-column>
          <el-table-column :label="$t('commons.operating')" min-width="25%" prop="result">
            <template v-slot:default="scope">
              <ms-tip-button
                circle
                type="success"
                size="mini"
                v-if="scope.row.isSet"
                v-xpack
                @click="handleTemplate(scope.$index,scope.row)"
                :tip="$t('organization.message.template')"
                icon="el-icon-tickets"/>
              <ms-tip-button
                circle
                type="primary"
                size="mini"
                v-show="scope.row.isSet"
                @click="handleAddTask(scope.$index,scope.row)"
                :tip="$t('commons.add')"
                icon="el-icon-check"/>
              <ms-tip-button
                circle
                size="mini"
                v-show="scope.row.isSet"
                @click="removeRowTask(scope.$index,jenkinsTask)"
                :tip="$t('commons.cancel')"
                icon="el-icon-refresh-left"/>
              <ms-tip-button
                el-button
                circle
                type="primary"
                size="mini"
                icon="el-icon-edit"
                v-show="!scope.row.isSet"
                :tip="$t('commons.edit')"
                @click="handleEditTask(scope.$index,scope.row)"
                v-permission="['ORGANIZATION_MESSAGE:READ+EDIT']"/>
              <ms-tip-button
                circle
                type="danger"
                icon="el-icon-delete"
                size="mini"
                v-show="!scope.row.isSet"
                @click="deleteRowTask(scope.$index,scope.row)"
                :tip="$t('commons.delete')"
                v-permission="['ORGANIZATION_MESSAGE:READ+EDIT']"/>
            </template>
          </el-table-column>
        </el-table>
      </el-col>
    </el-row>
    <notice-template v-xpack ref="noticeTemplate"/>
  </div>
</template>

<script>
import {hasLicense} from "@/common/js/utils";
import MsCodeEdit from "@/business/components/common/components/MsCodeEdit";
import MsTipButton from "@/business/components/common/components/MsTipButton";

const TASK_TYPE = 'JENKINS_TASK';

const requireComponent = require.context('@/business/components/xpack/', true, /\.vue$/);
const noticeTemplate = requireComponent.keys().length > 0 ? requireComponent("./notice/NoticeTemplate.vue") : {};

export default {
  name: "JenkinsNotification",
  components: {
    MsTipButton,
    MsCodeEdit,
    "NoticeTemplate": noticeTemplate.default
  },
  props: {
    jenkinsReceiverOptions: {
      type: Array
    },
    receiveTypeOptions: {
      type: Array
    }
  },
  data() {
    return {
      modes: ['text', 'html'],
      title: '<!DOCTYPE html>\n' +
        '<html lang="en">\n' +
        '<head>\n' +
        '    <meta charset="UTF-8">\n' +
        '    <title>MeterSphere</title>\n' +
        '</head>\n' +
        '<body>\n' +
        '<div>\n' +
        '    <div style="text-align: left">\n' +
        '        <p>尊敬的用户：</p>\n' +
        '        <p style="margin-left: 60px">您好:\n' +
        '    </div>\n' +
        '    <div style="margin-left: 100px">\n' +
        '        <p>${testName} 接口测试运行失败/成功<br/>\n' +
        '        <p>执行人:${executor}</p>' +
        '        <p>执行环境:${executionEnvironment}</p>' +
        '        <p>执行时间:${executionTime}</p>' +
        '            请点击下面链接进入测试报告页面</p>\n' +
        '        <a href="${url}/#/${type}/report/view/${id}">${url}/#/${type}/report/view/${id}</a>\n' +
        '        <p>新版接口测试报告路径</p>\n' +
        '        <a href="${url}/#/api/automation/report/view/">${url}/#/api/automation/report/view/${id}</a>\n' +
        '    </div>\n' +
        '\n' +
        '</div>\n' +
        '</body>\n' +
        '</html>',
      robotTitle:
        "测试【任务通知】:'${executor}所执行的 ${testName} ${type}测试运行${status}\n" +
        "测试环境为:${executionEnvironment}\n" +
        "执行时间：${executionTime}\n" +
        "请点击下面链接进入测试报告页面\n" +
        "${url}/#/${type}/report/view/${id}" +
        "新版接口测试报告路径\n" +
        "${url}/#/api/automation/report/view/${id}",
      jenkinsTask: [{
        taskType: "jenkinsTask",
        event: "",
        userIds: [],
        type: [],
        webhook: "",
        isSet: true,
        identification: "",
        isReadOnly: false,
      }],
      jenkinsEventOptions: [
        {value: 'EXECUTE_SUCCESSFUL', label: this.$t('schedule.event_success')},
        {value: 'EXECUTE_FAILED', label: this.$t('schedule.event_failed')}
      ],
    };
  },
  activated() {
    this.initForm();
  },
  methods: {
    initForm() {
      this.result = this.$get('/notice/search/message/type/' + TASK_TYPE, response => {
        this.jenkinsTask = response.data;
        // 上报通知数
        this.$emit("noticeSize", {module: 'jenkins', data: this.jenkinsTask, taskType:TASK_TYPE});
      });
    },
    handleEdit(index, data) {
      data.isReadOnly = true;
      if (data.type === 'EMAIL' || data.type === 'IN_SITE') {
        data.isReadOnly = !data.isReadOnly;
        data.webhook = '';
      }
    },
    handleAddTaskModel() {
      let Task = {};
      Task.event = [];
      Task.userIds = [];
      Task.type = '';
      Task.webhook = '';
      Task.isSet = true;
      Task.identification = '';
      Task.taskType = TASK_TYPE;
      this.jenkinsTask.unshift(Task);
    },
    handleAddTask(index, data) {
      if (data.event && data.userIds.length > 0 && data.type) {
        // console.log(data.type)
        if (data.type === 'NAIL_ROBOT' || data.type === 'WECHAT_ROBOT' || data.type === 'LARK') {
          if (!data.webhook) {
            this.$warning(this.$t('organization.message.message_webhook'));
          } else {
            this.addTask(data);
          }
        } else {
          this.addTask(data);
        }
      } else {
        this.$warning(this.$t('organization.message.message'));
      }
    },
    handleEditTask(index, data) {
      data.isSet = true;
      if (data.type === 'EMAIL' || data.type === 'IN_SITE') {
        data.isReadOnly = false;
        data.webhook = '';
      } else {
        data.isReadOnly = true;
      }
    },
    addTask(data) {
      this.result = this.$post("/notice/save/message/task", data, () => {
        this.initForm();
        this.$success(this.$t('commons.save_success'));
      });
    },
    removeRowTask(index, data) { //移除
      if (!data[index].identification) {
        data.splice(index, 1);
      } else {
        data[parseInt(index)].isSet = false;
      }

    },
    deleteRowTask(index, data) { //删除
      this.result = this.$get("/notice/delete/message/" + data.identification, response => {
        this.$success(this.$t('commons.delete_success'));
        this.initForm();
      });
    },
    rowClass() {
      return "text-align:center";
    },
    headClass() {
      return "text-align:center;background:'#ededed'";
    },
    handleTemplate(index, row) {
      if (hasLicense()) {
        this.$refs.noticeTemplate.open(row);
      }
    }
  }
};
</script>

<style scoped>
.el-row {
  margin-bottom: 10px;
}

.el-button {
  margin-right: 10px;
}

/deep/ .el-select .el-input.is-disabled .el-input__inner {
  background-color: #F5F7FA;
  border-color: #E4E7ED;
  color: #0a0a0a;
  cursor: not-allowed;
}

/deep/ .el-input.is-disabled .el-input__inner {
  background-color: #F5F7FA;
  border-color: #E4E7ED;
  color: #0a0a0a;
  cursor: not-allowed;
}
</style>
