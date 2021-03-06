<template>
  <el-tabs v-model="activeName" style="width:100vw;min-width: 1300px;">
    <el-tab-pane :label="$t('bare_metal')" name="bare-metal">
      <div class="machine-title2">
        <el-button-group class="batch-button">
          <el-button type="primary" icon="el-icon-delete" @click="delAllSelection">{{ $t('del') }}
          </el-button>
          <el-button type="primary" icon="el-icon-refresh" @click="getData">{{ $t('refresh') }}</el-button>
          <el-button type="primary" icon="el-icon-discover" @click="openDiscover">{{ $t('Discovery') }}</el-button>
          <el-button type="primary" icon="el-icon-caret-right" @click="runWorkflow">{{ $t('Run') }}</el-button>
        </el-button-group>
      </div>
      <div id="control" style="display: flex;">

        <div id="workflow-selector" style="display: flex;">
          <div id="select-workflow">
            <div class="el-icon-menu h25" style="border-bottom: yellowgreen 1px solid;    width: 100%;">
              {{ $t('Workflow') }}
            </div>
            <div class="run-splitter h25"></div>
            <el-select v-model="wfRequest.workflow" filterable :placeholder="$t('please_select')"
                       @change="getParamsTemplate">
              <el-option
                  v-for="g in supportedWorkflow"
                  :label="g.friendlyName"
                  :value="g.id"></el-option>
            </el-select>

            <el-button :disabled="this.multipleSelection.length == 0 || !wfRequest.workflow"
                       @click="addToSelectedWorkflow"
                       class="h50 ml10"><span
                class="el-icon-circle-plus"></span>{{ $t('add_to_selected_wf_list') }}
            </el-button>
          </div>
        </div>

        <div id="action-list">
          <div class="el-icon-s-operation h25"
               style="border-bottom: yellowgreen 1px solid;width: 100%;">
            <el-badge :value="selectedWorkflow.length" class="item" type="primary" v-show="selectedWorkflow.length">
              {{ $t('selected_workflows') }}
            </el-badge>
            <span v-show="selectedWorkflow.length == 0">
              {{ $t('selected_workflows') }}
              </span>
          </div>
          <div>
            <el-card v-for="(w, $index) in selectedWorkflow" style="height:100%;">
              <el-row>
                <el-col :span="6">
                  <el-button @click="deleteSelectedWorkflow($index)" class="h50 ml10"><span
                      class="el-icon-remove"></span>{{ $t('del') }}
                  </el-button>
                </el-col>

                <el-col :span="10">
                  {{ w.machineModel + ' ' + w.machineSn }}
                  <br>
                  {{ w.friendlyName }}
                </el-col>

                <el-col :span="8">
                  <el-button @click="editWfParams($index)" v-if="w.settable">
                    {{ $t('set_workflow_param') }}
                  </el-button>
                  <span v-if="!w.settable">
                                {{ $t('no_nessary_to_set') }}
                              </span>
                </el-col>
              </el-row>
            </el-card>
          </div>
        </div>
      </div>

      <el-table
          :data="tableData"
          class="table"
          ref="multipleTable"
          v-loading="loadingList"
          header-cell-class-name="table-header"
          style="width: 100%"
          @sort-change="sortChange($event)"
          @selection-change="handleSelectionChange"
      >
        <el-table-column type="selection" align="left"></el-table-column>

        <el-table-column prop="machine_model" :label="$t('machine_model')" align="left"
                         sortable="custom" style="overflow: scroll" width="180">
          <template slot-scope="scope">
            <el-tooltip class="item" effect="dark" :content="$t('detail')" placement="right-end">
              <el-link type="primary" @click="showDetail(scope.row)" target="_blank">
                  <span style="display: block; word-break:keep-all;
  white-space:nowrap;overflow: hidden">{{ scope.row.machineModel }}</span>
              </el-link>
            </el-tooltip>
          </template>
        </el-table-column>

        <el-table-column prop="machine_sn" :label="$t('machine_sn')" align="left"
                         sortable="custom">
          <template slot-scope="scope">
            {{ scope.row.machineSn }}
          </template>
        </el-table-column>

        <el-table-column prop="management_ip" :label="$t('management_ip')" align="left"
                         sortable="custom">
          <template slot-scope="scope">
            {{ scope.row.managementIp }}
          </template>
        </el-table-column>

        <el-table-column prop="ip_array" :label="$t('IP')" align="left"
                         sortable="custom">
          <template slot-scope="scope">
            {{ scope.row.ipArray }}
          </template>
        </el-table-column>

        <el-table-column :prop="c.prop" :label="$t(c.label)" align="left" v-for="c in columns" sortable="custom"
                         :width="resizeWith(c)">
          <template slot-scope="scope">
            <span v-if="!c.custom">{{ scope.row[c.prop] }}</span>
            <span v-if="c.custom">{{ c.formatter(scope.row[c.prop]) }}</span>
          </template>
        </el-table-column>

        <el-table-column prop="status" :label="$t('machine_status')" align="left">
          <template slot-scope="scope">
            <i class="el-icon-loading" v-if="scope.row.status && scope.row.status.indexOf('ing') != -1"></i>
            <span v-html="statusFilter(scope.row)"></span>
          </template>
        </el-table-column>

        <el-table-column prop="status" :label="$t('power')" align="left">
          <template slot-scope="scope">
            <span v-html="powerFilter(scope.row.power)"></span>
          </template>
        </el-table-column>

        <el-table-column prop="" :label="$t('opt')" align="left">
          <template slot="header" slot-scope="scope">
            <el-input
                v-model="search"
                size="mini"
                :placeholder="$t('input_key_search')" v-on:change="getData"/>
          </template>
          <template slot-scope="scope">
            <el-dropdown>
              <el-button type="primary">
                {{ $t('opt') }}<i class="el-icon-arrow-down el-icon--right"></i>
              </el-button>
              <el-dropdown-menu slot="dropdown">
                <el-dropdown-item @click.native="power('on', scope.row)">{{ $t('poweron') }}
                </el-dropdown-item>
                <el-dropdown-item @click.native="power('off', scope.row)">{{ $t('poweroff') }}
                </el-dropdown-item>
                <el-dropdown-item @click.native="power('reset', scope.row)">{{ $t('reset') }}
                </el-dropdown-item>
                <el-dropdown-item @click.native="power('pxe', scope.row)">{{ $t('pxeboot') }}
                </el-dropdown-item>
                <el-dropdown-item @click.native="fillOBM(scope.row)"> {{ $t('OBM') + $t('info') }}
                </el-dropdown-item>
              </el-dropdown-menu>
            </el-dropdown>
          </template>
        </el-table-column>
      </el-table>
      <div class="pagination">
        <el-pagination
            @size-change="handleSizeChange"
            @current-change="handlePageChange"
            :current-page="query.pageIndex"
            :page-sizes="[10, 20, 50, 100]"
            :page-size="10"
            layout="total, sizes, prev, pager, next, jumper"
            :total="pageTotal">
        </el-pagination>
      </div>

      <el-drawer
          size="50%"
          :title="logTitle"
          :visible.sync="executionLogDrawer"
          direction="ttb"
          min-height="40vh"
          :wrapperClosable="false"
          :before-close="handleCloseExecutionLog">
        <el-card>
          <table class="detail-info">
            <tr>
              <td>{{ $t('time') }}</td>
              <td>{{ $t('user') }}</td>
              <td>{{ $t('operation') }}</td>
              <td>{{ $t('output') }}</td>
            </tr>

            <tr v-for="l in logs">
              <td>{{ l.createTime | dateFormat }}</td>
              <td>{{ l.user }}</td>
              <td>{{ l.operation }}</td>
              <td>{{ l.outPut }}</td>
            </tr>
          </table>
        </el-card>
      </el-drawer>

      <!--obm-->
      <el-dialog :title="$t('obms')" :visible.sync="fillOutObms" width="40vw">
        <el-form :model="form">
          <el-form-item :label="$t('ip')" :label-width="formLabelWidth">
            <el-input v-model="curObm.ip" autocomplete="off"></el-input>
          </el-form-item>
          <el-form-item :label="$t('username')" :label-width="formLabelWidth">
            <el-input v-model="curObm.userName" autocomplete="off"></el-input>
          </el-form-item>
          <el-form-item :label="$t('pwd')" :label-width="formLabelWidth">
            <el-input v-model="curObm.pwd" autocomplete="off" show-password></el-input>
          </el-form-item>
        </el-form>
        <template v-slot:footer>
          <div class="dialog-footer">
            <el-button @click="fillOutObms = false">{{ $t('cancel') }}</el-button>
            <el-button type="primary" @click="submitOBM" :loading="obmLoading">{{ $t('confirm') }}</el-button>
          </div>
        </template>
      </el-dialog>

      <!--参数配置-->
      <el-dialog :title="currentParamConfig" :visible.sync="fillWfParams" ref="paramDialog" width="80vw">
        <keep-alive>
          <component v-if="editWorkflowIndex != -1 && selectedWorkflow.length > 0"
                     :is="currentWfParamTemplate"
                     :params="workflowParam"
                     :extraParams="extraParams"
                     :currentWorkflowIndex="editWorkflowIndex"
                     :bareMetalId="selectedWorkflow[editWorkflowIndex].bareMetalId"
                     :workflow="selectedWorkflow[editWorkflowIndex]"
                     ref="currentWfParamTemplate"></component>
        </keep-alive>
        <template v-slot:footer>
          <el-button @click="restoreParams">
            <i class="el-icon-refresh"></i>
            {{ $t('reset') }}
          </el-button>
          <el-button type="primary" @click="saveParams" :loading="fillWfParamsLoading">{{ $t('confirm') }}</el-button>
        </template>
      </el-dialog>

      <el-drawer
          :title="editType == 'edit' ? $t('edit_discovery') : $t('add_discovery')"
          :visible.sync="discoveryVisible"
          direction="rtl"
          size="100%"
          :with-header="false"
          :wrapperClosable="false"

          :before-close="handleClose">
        <Discovery class="bare-discovery" @back="discoveryVisible =false" @queryByRuleId="queryByRuleId"></Discovery>
      </el-drawer>

      <!--详情页-->
      <drawer
          :visible.sync="detailDrawer"
          direction="rtl"
          :with-header="false"
          size="50%"
          :before-close="handleClose">

        <div class="demo-drawer__content">
          <el-tabs v-model="detailShowName">
            <el-tab-pane :label="$t('detail')" name="detail">
              <el-form ref="form" :model="machine" label-position="top">
                <el-form-item :label="$t('machine_model')">
                  {{ machine.machineModel }}
                </el-form-item>
                <el-form-item :label="$t('machine_sn')">
                  {{ machine.machineSn }}
                </el-form-item>
                <el-form-item :label="$t('management_ip')">
                  {{ machine.managementIp }}
                </el-form-item>
                <el-form-item :label="$t('bmc_mac')">
                  {{ machine.bmcMac }}
                </el-form-item>
                <el-form-item :label="$t('cpu')">
                  <span v-if="machine.cpu">{{ machine.cpuType }} X {{ machine.cpu }}</span>
                  <span v-if="!machine.cpu">{{ $t('not_exists') }}</span>
                </el-form-item>
                <el-form-item :label="$t('memory')">
                  <span v-if="machine.memory">{{ machine.memory }}{{ ' ' + $t('GB') }}</span>
                  <span v-if="!machine.cpu">{{ $t('not_exists') }}</span>
                </el-form-item>
                <el-form-item :label="$t('disk')">
                  <span v-if="machine.disk">{{ machine.disk }}{{ ' ' + $t('GB') }}</span>
                  <span v-if="!machine.disk">{{ $t('not_exists') }}</span>
                </el-form-item>
              </el-form>
            </el-tab-pane>
            <el-tab-pane :label="$t('cpu_detail')" name="cpuDetail" class="detail-pane">
              <table class="detail-info">
                <tr>
                  <td>{{ $t('proc_name') }}</td>
                  <td>{{ $t('proc_socket') }}</td>
                  <td>{{ $t('proc_speed') }}</td>
                  <td>{{ $t('proc_num_cores') }}</td>
                  <td>{{ $t('proc_num_threads') }}</td>
                  <td>{{ $t('sync_time') }}</td>
                </tr>
                <tr v-for="c in cpus">
                  <td>{{ c.procName }}</td>
                  <td>{{ c.procSocket }}</td>
                  <td>{{ c.procSpeed }}</td>
                  <td>{{ c.procNumCores }}</td>
                  <td>{{ c.procNumThreads }}</td>
                  <td>{{ c.syncTime | dateFormat }}</td>
                </tr>
              </table>
            </el-tab-pane>
            <el-tab-pane :label="$t('memory_detail')" name="memoryDetail" class="detail-pane">
              <table class="detail-info">
                <tr>
                  <td>{{ $t('mem_cpu_num') }}</td>
                  <td>{{ $t('mem_mod_num') }}</td>
                  <td>{{ $t('mem_mod_size') }}</td>
                  <td>{{ $t('mem_mod_type') }}</td>
                  <td>{{ $t('mem_mod_num') }}</td>
                  <td>{{ $t('mem_mod_frequency') }}</td>
                  <td>{{ $t('mem_mod_part_num') }}</td>
                  <td>{{ $t('mem_mod_min_volt') }}</td>
                  <td>{{ $t('sync_time') }}</td>
                </tr>
                <tr v-for="c in memories">
                  <td>{{ c.memCpuNum }}</td>
                  <td>{{ c.memModNum }}</td>
                  <td>{{ c.memModSize + ' GB' }}</td>
                  <td>{{ c.memModType }}</td>
                  <td>{{ c.memModNum }}</td>
                  <td>{{ c.memModFrequency + ' MHz' }}</td>
                  <td>{{ c.memModPartNum }}</td>
                  <td>{{ c.memModMinVolt }}</td>
                  <td>{{ c.syncTime | dateFormat }}</td>
                </tr>
              </table>
            </el-tab-pane>
            <el-tab-pane :label="$t('disk_detail')" name="diskDetail" class="detail-pane">
              <table class="detail-info">
                <tr>
                  <td>{{ $t('enclosure_id') }}</td>
                  <td>{{ $t('controller_id') }}</td>
                  <td>{{ $t('drive') }}</td>
                  <td>{{ $t('type') }}</td>
                  <td>{{ $t('size') }}</td>
                  <td>{{ $t('raid') }}</td>
                  <td>{{ $t('manufactor') }}</td>
                  <td>{{ $t('sn') }}</td>
                  <td>{{ $t('sync_time') }}</td>
                </tr>
                <tr v-for="c in disks">
                  <td>{{ c.enclosureId }}</td>
                  <td>{{ c.controllerId }}</td>
                  <td>{{ c.drive }}</td>
                  <td>{{ c.type }}</td>
                  <td>{{ c.size }}</td>
                  <td>{{ c.raid }}</td>
                  <td>{{ c.manufactor }}</td>
                  <td>{{ c.sn }}</td>
                  <td>{{ c.syncTime | dateFormat }}</td>
                </tr>
              </table>
            </el-tab-pane>

            <el-tab-pane :label="$t('nic_detail')" name="nicDetail" class="detail-pane">
              <table class="detail-info">
                <tr>
                  <td>{{ $t('nic_number') }}</td>
                  <td>{{ $t('mac') }}</td>
                  <td>{{ $t('sync_time') }}</td>
                </tr>
                <tr v-for="c in nics">
                  <td>{{ c.number }}</td>
                  <td>{{ c.mac }}</td>
                  <td>{{ c.syncTime | dateFormat }}</td>
                </tr>
              </table>
            </el-tab-pane>
          </el-tabs>
        </div>
      </drawer>
    </el-tab-pane>
  </el-tabs>
</template>

<script>
import HttpUtil from "../../common/utils/HttpUtil";
import {isAnyBlank, toLine} from "../../common/utils/CommonUtil";
import OBM from "../obm/Obm"
import Discovery from "../discovery/Discovery"
import Vue from "vue"
import i18n from "@/i18n/i18n";
import {WebSocketUtil} from "@/common/utils/WebSocket";

Vue.filter('statusFilter', function (row) {
  return i18n.t('PXE') + ' ' + i18n.t(row.status);
});
let _ = require('lodash');
export default {
  data() {
    return {
      discoveryVisible: false,
      search: null,
      fillWfParams: false,
      executionLogDrawer: false,
      activeName: 'bare-metal',
      logTitle: '',
      loadingLog: false,
      currentMachine: {},
      form: {},
      logs: [],
      query: {
        address: '',
        name: '',
        pageIndex: 1,
        pageSize: 10
      },
      tableData: [],
      multipleSelection: [],
      delList: [],
      editVisible: false,
      formLabelWidth: '80px',
      pageTotal: 0,
      loading: false,
      columns: [
        {
          label: 'CPU',
          prop: "cpu"
        },
        {
          label: 'memory',
          prop: "memory",
          expandLanguage: "en_US",
          custom: true,
          formatter: function (item) {
            if (item) {
              return parseInt(item) + ' GB'
            }
            return item;
          }
        },
        {
          label: 'disk',
          prop: "disk",
          custom: true,
          formatter: function (item) {
            if (item) {
              return parseInt(item) / 1000 + ' TB'
            }
            return item;
          }
        },
      ],
      detailDrawer: false,
      editType: 'edit',
      editObj: {
        name: null,
        id: null,
        email: null,
        phone: null,
        roles: [],
        rolesIds: []
      },
      allGraphDefinitions: [],
      supportedWorkflow: [],
      wfRequest: {},
      workflowParamList: [],
      workflowParam: {},
      extraParams: {},
      paramEditable: false,
      selectedWorkflow: [],
      fillOutObms: false,
      curObm: {
        ip: null,
        userName: null,
        pwd: null,
        bareMetalId: null
      },
      obmLoading: false,
      fillWfParamsLoading: false,
      machine: {},
      detailShowName: 'detail',
      cpuLoading: false,
      cpus: [],
      memories: [],
      disks: [],
      nics: [],
      paramComponent: {},
      currentWfParamTemplate: {},
      editWorkflowIndex: -1,
      queryVO: {
        searchKey: this.search
      },
      logPoller: null,
      currentParamConfig: '',
      loadingList: false,
      websocket: null
    }
  },
  components: {
    OBM, Discovery
  },
  computed: {},
  destroyed() {
    if (this.websocket) {
      this.websocket.close();
    }
  },
  mounted() {
    if (!this.websocket) {
      this.websocket = new WebSocketUtil();
      this.websocket.openSocket('lifecycle', this.notify);
      this.getData();
    }
    this.getAllGraphDefinitions();
  }
  ,
  methods: {
    notify(msg) {
      this.getData();
      this.$notify({
        title: this.$t('server_message'),
        message: msg,
      });
    },
    queryByRuleId(e) {
      if (e) {
        this.search = e;
        this.getData(true);
      }
    },
    powerFilter(status) {
      return this.$t(status);
    },
    openDiscover() {
      this.discoveryVisible = true;
    },
    statusFilter(row) {
      if (row.status.indexOf("ing") == -1) {
        if (row.serverId)
          return '<span style="display: inline-block;">' +
              this.$t('PXE') + ' ' + this.$t(row.status) + '<i class="el-icon-check" style="color:greenyellow;margin-left:5px;"></i><br>'
              + this.$t('OBM') + ' ' + this.$t('info') + (row.outBandList.length > 0 ? '<i class="el-icon-check" style="color:greenyellow;margin-left:5px;"></i>' : '<i class="el-icon-close" style="margin-left:5px;color: red;"></i>') + '</span>';
        else
          return '<span style="display: inline-block;">' +
              this.$t('PXE') + ' ' + this.$t('status') + '<i class="el-icon-close" style="margin-left:5px;color: red;"></i><br>'
              + this.$t('OBM') + ' ' + this.$t('info') + (row.outBandList.length > 0 ? '<i class="el-icon-check" style="color:greenyellow;margin-left:5px;"></i>' : '<i class="el-icon-close" style="margin-left:5px;color: red;"></i>') + '</span>';
      } else {
        return '<span style="display: inline-block;">' +
            this.$t('PXE') + ' ' + this.$t(row.status) + '<br>';
      }
    },
    resizeWith(c) {
      return (c.expandLanguage && c.expandLanguage == localStorage.getItem('lang')) ? '100px' : '90px';
    },
    restoreParams() {
      this.$refs.currentWfParamTemplate.restoreParams();
    },
    expandChange(a) {
      this.executionLogDrawer = true;
      this.currentMachine = a;
      this.logTitle = this.$t('execution_log') + ' ' + this.currentMachine.machineModel + ' ' + this.currentMachine.machineSn + ' ' + this.currentMachine.managementIp;
      this.logPoller = window.setInterval(this.loadLogs, 1000);
    },
    loadLogs() {
      HttpUtil.post("/execution-log/detaillist/" + 1 + "/" + 1000, {bareMetalId: this.currentMachine.id}, (res) => {
        this.$set(this, 'logs', res.data.listObject);
      });
    },
    buildRequest(workflow) {
      let request = JSON.parse(JSON.stringify(workflow));
      delete request.componentId;
      delete request.machineModel;
      delete request.machineSn;
      delete request.settable;
      delete request.brands;
      delete request.friendlyName;
      return request;
    },
    saveParams() {
      if (this.$refs.currentWfParamTemplate.valid()) {
        this.selectedWorkflow[this.editWorkflowIndex].params = this.$refs.currentWfParamTemplate.payLoad;
        this.selectedWorkflow[this.editWorkflowIndex].extraParams = this.$refs.currentWfParamTemplate.extraParams;
        this.fillWfParamsLoading = true;
        HttpUtil.post("/workflow/params", this.buildRequest(this.selectedWorkflow[this.editWorkflowIndex]), (res) => {
          this.fillWfParamsLoading = false;
          this.fillWfParams = false;
        });
      }
    },
    copy(obj) {
      return JSON.parse(JSON.stringify(obj));
    },
    editWfParams(index) {
      this.editWorkflowIndex = index;
      this.fillWfParams = true;
      this.currentWfParamTemplate = this.selectedWorkflow[index].componentId;
      this.currentParamConfig = this.selectedWorkflow[index].machineModel + ' ' + this.selectedWorkflow[index].friendlyName + " " + this.$t('param_config');
      if (this.workflowParamList.length) {
        this.workflowParam = this.selectedWorkflow[index].params;
        this.extraParams = this.selectedWorkflow[index].extraParams;
      }
    },
    power(opt, row) {
      this.$confirm(this.$t('confirm') + this.$t('power_' + opt) + '?', this.$t('tips'), {
        type: "warning"
      }).then(() => {
        HttpUtil.get("/bare-metal/power/" + row.id + "/" + opt, null, (res) => {
          if (res.success) {
            this.$message.success(this.$t('success'));
          } else {
            this.$message.error(this.$t('error'));
          }
        });
      });
    },
    getHardware() {
      HttpUtil.get("/bare-metal/hardwares/" + this.machine.id, null, (res) => {
        this.cpus = res.data.cpus;
        this.memories = res.data.memories;
        this.disks = res.data.disks;
        this.nics = res.data.nics;
      })
    },
    showDetail(machine) {
      this.machine = machine;
      this.detailDrawer = true;
      this.getHardware();
    },
    fillOBM(val) {
      if (val.outBandList.length > 0) {
        this.curObm = {
          ip: val.managementIp,
          userName: val.outBandList[0].userName,
          pwd: val.outBandList[0].pwd,
          bareMetalId: val.id,
        };
      } else {
        this.curObm = {
          ip: val.managementIp,
          userName: null,
          pwd: null,
          bareMetalId: val.id,
        };
      }
      this.fillOutObms = true;
    },
    submitOBM() {
      this.obmLoading = true;
      if (isAnyBlank(this.curObm.ip, this.curObm.userName, this.curObm.pwd)) {
        this.$message.error(this.$t('pls_fill_in_blanks'));
        this.obmLoading = false;
        return;
      }
      HttpUtil.post("/outband/save?bareMetalId=" + this.curObm.bareMetalId, this.curObm, (res) => {
        this.obmLoading = false;
        this.curObm = {};
        this.fillOutObms = false;
        this.getData();
      }, (res) => {
        this.obmLoading = false;
      });
    },
    sortChange(val) {
      console.log(val);
      console.log(val.order);
      if (val.order) {
        this.queryVO = {
          searchKey: '%' + this.search + '%',
          sort: val.prop + " " + val.order.replace("ending", "")
        }
      } else {
        delete this.queryVO.sort;
      }
      this.getData();
    },
    getData(accurate, callback) {
      this.loadingList = true;
      if (this.search) {
        if (!accurate)
          this.queryVO.searchKey = '%' + this.search + '%';
        else
          this.queryVO.searchKey = this.search;

      } else {
        this.queryVO.searchKey = null;
      }
      HttpUtil.post("/bare-metal/list/" + this.query.pageIndex + "/" + this.query.pageSize, this.queryVO, (res) => {
        this.tableData = res.data.listObject;
        this.pageTotal = res.data.itemCount;
        this.loadingList = false;
        if (callback) {
          callback();
        }
      });
    },
    handleSizeChange(val) {
      this.query.pageSize = val;
      this.handlePageChange(this.query.pageIndex);
    },
    handleClose() {
      this.detailDrawer = false;
      this.discoveryVisible = false;
    },
    handleCloseExecutionLog() {
      this.executionLogDrawer = false;
      window.clearInterval(this.logPoller);
    },
    cancelForm() {
      this.loading = false;
      this.detailDrawer = false;
    },
    confirmEdit() {
      this.loading = true;
      if (this.editType == 'edit') {
        HttpUtil.post("/bare-metal/update", this.editObj, (res) => {
          this.detailDrawer = false;
          this.$message.success(this.$t('edit_success'));
          this.cancelForm();
          this.getData();
        })
      } else {
        HttpUtil.post("/bare-metal/add", this.editObj, (res) => {
          this.detailDrawer = false;
          this.$message.success(this.$t('add_success'));
          this.cancelForm();
          this.getData();
        })
      }
    },
    // 多选操作
    handleSelectionChange(val) {
      this.multipleSelection = val;
    },
    getSelectedIds: function () {
      this.delList = [].concat(this.multipleSelection);
      let ids = _.map(this.delList, (item) => item.id);
      return ids;
    },
    delAllSelection() {
      this.$confirm(this.$t('confirm_to_del'), this.$t('tips'), {
        type: 'warning'
      }).then(() => {
        let ids = this.getSelectedIds();
        if (!ids || ids.length == 0) {
          this.$message.error(this.$t('pls_select_bare_metal') + "!");
          return;
        }
        HttpUtil.post("/bare-metal/del", ids, (res) => {
          this.$message.success(this.$t('delete_success'));
          this.getData();
        });
        this.multipleSelection = [];
      });
    },
    // 编辑操作
    handleEdit(row, type) {
      if (type == 'edit') {
        this.detailDrawer = true;
        this.editType = type;
        this.editObj = JSON.parse(JSON.stringify(row));
        this.editObj.rolesIds = _.map(this.editObj.roles, (item) => item.id);

      } else if (type == 'del') {
        this.$confirm(this.$t('confirm_to_del'), this.$t('tips'), {
          type: 'warning'
        }).then(() => {
          HttpUtil.get("/bare-metal/del/" + row.id, {}, (res) => {
            this.getData();
            this.$message.success(this.$t('delete_success!'));
          });
        })
      } else {
        this.detailDrawer = true;
        this.editType = type;
        this.editObj = {};
      }
    },
    // 分页导航
    handlePageChange(val) {
      this.$set(this.query, 'pageIndex', val);
      this.getData();
    },
    getAllGraphDefinitions(name) {
      HttpUtil.get("/workflow/listall", {name: name}, (res) => {
        if (res.data && res.data.length > 0) {
          res.data.forEach(w => {
            w.brands = eval(w.brands);
            w.settable = eval(w.settable);
            if (w.defaultParams) w.defaultParams = JSON.parse(w.defaultParams);
          });
        }
        this.supportedWorkflow = res.data;
      });
    },
    runWorkflow() {
      if (!this.selectedWorkflow.length) {
        this.$message.error(this.$t('pls_select_workflow') + "!");
        return;
      }
      for (let i = 0; i < this.selectedWorkflow.length; i++) {
        let obj = _.find(this.tableData, (o) => o.id == this.selectedWorkflow[i].bareMetalId);
        if (!obj.outBandList.length) {
          this.$message.error(obj.machineModel + '' + obj.machineSn + ' ' + this.$t('obm_not_exists') + "!");
          return;
        }
      }
      let that = this;
      let reqList = _.map(this.copy(this.selectedWorkflow), (wf) => {
        return that.buildRequest(wf);
      });
      if (reqList.length == 0) {
        this.$message.error(this.$t('pls_select_node') + "!");
        return;
      }
      if (_.findIndex(reqList, r => r.settable && !r.params) != -1) {
        this.$message.error(this.$t('pls_set_params') + "!");
        return;
      }
      HttpUtil.post("/workflow/run", reqList, (res) => {
        this.selectedWorkflow = [];
        this.getData();
      });
    },
    getWorkflowById() {
      return _.find(this.supportedWorkflow, (o) => o.id == this.wfRequest.workflow);
    }
    ,
    getParamsTemplate() {
      HttpUtil.get("/workflow/params/" + this.getWorkflowById().injectableName, {}, (res) => {
        if (res.data[0]) {
          this.workflowParamList = res.data;
        } else {
          this.workflowParamList = [];
          this.workflowParam = null;
        }
      })
    }
    ,
    createWorkflowParamComponent(workflowParam) {
      //动态异步
      if (!this.paramComponent[workflowParam.componentId]) {

        let comPointer = Vue.component(workflowParam.componentId,
            // 这个动态导入会返回一个 `Promise` 对象。
            () => import("./../../rackparams/" + workflowParam.workflowName)
        )

        this.paramComponent[workflowParam.componentId] = comPointer;
      }
    },
    addToSelectedWorkflow() {
      let that = this;
      if (that.getWorkflowById().injectableName) {
        let originWf = _.find(that.supportedWorkflow, s => s.injectableName == that.getWorkflowById().injectableName);
        for (let k = 0; k < that.multipleSelection.length; k++) {
          let duplicated = false;
          let componentId = that.getWorkflowById().injectableName + "-" + that.multipleSelection[k].id;
          for (let j = 0; j < that.selectedWorkflow.length; j++) {
            if (that.selectedWorkflow[j].componentId == componentId) {
              duplicated = true;
            }
          }

          if (duplicated) continue;
          if (_.findIndex(originWf.brands, w => w == that.multipleSelection[k].machineBrand) == -1) {
            that.$message.error(originWf.friendlyName + that.$t('not_supported_brand!') + ' ' + that.multipleSelection[k].machineBrand);
            continue;
          }

          that.selectedWorkflow.push(
              {
                componentId: that.getWorkflowById().injectableName + "-" + that.multipleSelection[k].id,
                bareMetalId: that.multipleSelection[k].id,
                machineModel: that.multipleSelection[k].machineModel,
                machineSn: that.multipleSelection[k].machineSn,
                workflowName: that.getWorkflowById().injectableName,
                friendlyName: originWf.friendlyName,
                settable: originWf.settable,
              }
          );

          that.createWorkflowParamComponent(that.selectedWorkflow[that.selectedWorkflow.length - 1]);

          that.currentWfParamTemplate = that.selectedWorkflow[that.selectedWorkflow.length - 1].componentId;
          if (that.workflowParamList.length) {
            let paramTemplate = _.find(that.workflowParamList, function (p) {
              return p.bareMetalId == that.selectedWorkflow[that.selectedWorkflow.length - 1].bareMetalId;
            });
            if (paramTemplate == null) {
              that.$set(that, 'workflowParam', null);
              that.$set(that, 'extraParams', null);
            } else {
              that.$set(that, 'workflowParam', JSON.parse(paramTemplate.paramsTemplate));
              that.$set(that, 'extraParams', JSON.parse(paramTemplate.extraParams));
            }

            that.selectedWorkflow[that.selectedWorkflow.length - 1].params = JSON.parse(paramTemplate.paramsTemplate);
            that.selectedWorkflow[that.selectedWorkflow.length - 1].extraParams = JSON.parse(paramTemplate.extraParams);
          } else {
            that.selectedWorkflow[that.selectedWorkflow.length - 1].params = originWf.defaultParams;
            that.$set(that, 'workflowParam', originWf.defaultParams);
            that.$set(that, 'extraParams', null);
          }
        }
        console.log("device : " + this.selectedWorkflow[this.selectedWorkflow.length - 1].params.options.defaults.networkDevices[0].device);
        // that.$refs.multipleTable.clearSelection();
      }
    }
    ,
    deleteSelectedWorkflow(index) {
      if (this.$refs.currentWfParamTemplate)
        this.$refs.currentWfParamTemplate.$destroy(true);
      this.selectedWorkflow.splice(index, 1);
    },
  }
}
</script>

<style scoped>
.handle-box {
  margin-bottom: 20px;
}

.table {
  width: 100%;
  font-size: 14px;
}

.mr10 {
  margin-right: 10px;
}

#run-workflow {
  border: solid #d7d2d2 1px;
  height: 120px;
  padding: 10px 10px 15px 10px;
  border-radius: 5px;
  min-width: 120px;
}

#workflow-selector {
  border: solid #d7d2d2 1px;
  min-width: 450px;
  height: 120px;
  padding: 10px 10px 15px 10px;
  border-radius: 5px;
  margin-left: 10px;
}

#action-list {
  border: solid #d7d2d2 1px;
  width: 100%;
  height: 120px;
  padding: 10px 10px 15px 10px;
  border-radius: 5px;
  margin: 0 10px;
  overflow: auto;
}

.h25 {
  height: 25%;
}

#control {
  display: flex;
  padding-top: 10px;
  padding-bottom: 20px;
  /*border: solid #d7d2d2 1px;*/
  padding-left: 10px;
  border-left: none;
}

.ml10 {
  margin-left: 10px;
}

.demo-table-expand {
  font-size: 0;
}

.demo-table-expand label {
  width: 90px;
  color: #99a9bf;
}

.demo-table-expand .el-form-item {
  margin-right: 0;
  margin-bottom: 0;
  width: 50%;
}

.detail-pane {
  overflow-x: scroll;
  color: #303133;
}

.detail-info {
  border: 1px solid #EBEEF5;
  border-spacing: 0px !important;
  width: 100%;
}

.detail-info td {
  border: 1px solid #EBEEF5;
  border-bottom: none;
  border-right: none;
  padding: 12px 0;
  min-width: 0;
  box-sizing: border-box;
  text-overflow: ellipsis;
  vertical-align: middle;
  position: relative;
  text-align: center;
}

.el-tabs__nav-wrap .is-top {
  padding-left: 10px;
}

.bare-discovery {
  padding: 20px;
}
</style>