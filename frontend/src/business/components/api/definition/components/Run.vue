<template>
  <span></span>
</template>
<script>
import {getBodyUploadFiles, getCurrentProjectID, strMapToObj} from "@/common/js/utils";
import ThreadGroup from "./jmeter/components/thread-group";
import TestPlan from "./jmeter/components/test-plan";
import {TYPE_TO_C} from "@/business/components/api/automation/scenario/Setting";

export default {
  name: 'MsRun',
  components: {},
  props: {
    environment: Object,
    debug: Boolean,
    reportId: String,
    runData: Array,
    type: String,
    envMap: Map,
    isStop: Boolean,
  },
  data() {
    return {
      result: {},
      loading: false,
      runId: "",
      reqNumber: 0,
      websocket: {}
    }
  },

  watch: {
    // 初始化
    reportId() {
      this.run()
    },
    isStop() {
      if (!this.isStop && this.websocket && this.websocket.close instanceof Function) {
        this.websocket.close();
      }
    }
  },
  methods: {
    initWebSocket() {
      let protocol = "ws://";
      if (window.location.protocol === 'https:') {
        protocol = "wss://";
      }
      let runMode = this.debug ? "debug" : "run";
      const uri = protocol + window.location.host + "/api/definition/run/report/" + this.runId + "/" + runMode;
      this.websocket = new WebSocket(uri);
      this.websocket.onmessage = this.onMessage;
    },
    onMessage(e) {
      if (e.data) {
        let data = JSON.parse(e.data);
        this.websocket.close();
        this.$emit('runRefresh', data);
      }
    },
    sort(stepArray) {
      if (stepArray) {
        for (let i in stepArray) {
          if (!stepArray[i].clazzName) {
            stepArray[i].clazzName = TYPE_TO_C.get(stepArray[i].type);
          }
          if (stepArray[i].hashTree && stepArray[i].hashTree.length > 0) {
            this.sort(stepArray[i].hashTree);
          }
        }
      }
    },
    run() {
      let projectId = getCurrentProjectID();
      // 如果envMap不存在，是单接口调用
      if (!this.envMap || this.envMap.size === 0) {
        projectId = getCurrentProjectID();
      } else {
        // 场景步骤下接口调用
        if (this.runData.projectId) {
          projectId = this.runData.projectId;
        }
      }

      let testPlan = new TestPlan();
      testPlan.clazzName = TYPE_TO_C.get(testPlan.type);
      let threadGroup = new ThreadGroup();
      threadGroup.clazzName = TYPE_TO_C.get(threadGroup.type);
      threadGroup.hashTree = [];
      testPlan.hashTree = [threadGroup];
      this.runData.forEach(item => {
        item.projectId = projectId;
        if (!item.clazzName) {
          item.clazzName = TYPE_TO_C.get(item.type);
        }
        threadGroup.hashTree.push(item);
      })
      this.sort(testPlan.hashTree);
      let reqObj = {id: this.reportId, testElement: testPlan, type: this.type, clazzName: this.clazzName ? this.clazzName : TYPE_TO_C.get(this.type), projectId: projectId, environmentMap: strMapToObj(this.envMap)};
      let bodyFiles = getBodyUploadFiles(reqObj, this.runData);
      if (this.runData[0].url) {
        reqObj.name = this.runData[0].url;
      } else {
        reqObj.name = this.runData[0].path;
      }
      let url = "";
      if (this.debug) {
        reqObj.reportId = this.reportId;
        url = "/api/definition/run/debug";
      } else {
        url = "/api/definition/run";
      }
      this.$fileUpload(url, null, bodyFiles, reqObj, response => {
        this.runId = response.data;
        this.initWebSocket();
        this.$emit('autoCheckStatus');  //   执行结束后，自动更新计划状态
      }, error => {
        this.$emit('errorRefresh', {});
      });
    }
  }
}
</script>
