<template>
    <div class="container">
        <!--        <div class="handle-box">-->
        <!--            <el-button-group>-->
        <!--                <el-button-->
        <!--                        type="primary"-->
        <!--                        icon="el-icon-delete"-->
        <!--                        class="handle-del mr10"-->
        <!--                        @click="delAllSelection"-->
        <!--                >{{$t('batch_del')}}-->
        <!--                </el-button>-->
        <!--                <el-button-->
        <!--                        type="primary"-->
        <!--                        icon="el-icon-delete"-->
        <!--                        class="handle-del mr10"-->
        <!--                        @click="handleEdit({}, 'add')"-->
        <!--                >{{$t('add')}}-->
        <!--                </el-button>-->
        <!--            </el-button-group>-->
        <!--        </div>-->
        <div class="machine-title">
            <i class="el-icon-user-solid">Machines</i>

            <el-button-group class="batch-button">
                <el-button type="primary" icon="el-icon-edit"></el-button>
                <el-button type="primary" icon="el-icon-share"></el-button>
                <el-button type="primary" icon="el-icon-delete"></el-button>
            </el-button-group>
        </div>
        <div id="control" style="display: flex;">
            <div id="run-workflow">
                <div class="el-icon-caret-right h25" style="border-bottom: yellowgreen 1px solid;    width: 100%;">Run
                </div>
                <div class="run-splitter h25"></div>
                <el-button class="el-icon-caret-right h50"></el-button>
                <el-button class="el-icon-close h50"></el-button>
            </div>

            <div id="workflow-selector" style="display: flex;">
                <div id="select-workflow">
                    <div class="el-icon-menu h25" style="border-bottom: yellowgreen 1px solid;    width: 100%;">Workflow
                    </div>
                    <div class="run-splitter h25"></div>
                    <!--                    <el-button class="h50">Workflow <span class="el-icon-caret-bottom"></span></el-button>-->
                    <el-select v-model="wfRequest.workflow">
                        <el-option v-for="g in allGraphDefinitions" :label="g.injectableName" :value="g.id"></el-option>
                    </el-select>
                    <el-button class="el-icon-close h50"></el-button>
                </div>
                <div id="select-params">
                    <div class="el-icon-s-operation h25" style="border-bottom: yellowgreen 1px solid;    width: 100%;">
                        Params
                    </div>
                    <div class="run-splitter h25"></div>
                    <el-button class="h50">Workflow <span class="el-icon-caret-bottom"></span></el-button>

                </div>
            </div>
        </div>

        <el-table
                :data="tableData"
                class="table"
                ref="multipleTable"
                header-cell-class-name="table-header"
                style="width: 100%"
                @selection-change="handleSelectionChange"
        >
            <el-table-column type="selection" align="center"></el-table-column>
            <el-table-column :prop="c.prop" :label="c.label" align="center"
                             v-for="c in columns" sortable="custom"></el-table-column>

            <el-table-column prop="" :label="$t('opt')" align="center">

                <template slot-scope="scope">
                    <el-dropdown>
                        <el-button type="primary">
                            操作<i class="el-icon-arrow-down el-icon--right"></i>
                        </el-button>
                        <el-dropdown-menu slot="dropdown">
                            <el-dropdown-item @click.native="power('on', scope.row)">poweron</el-dropdown-item>
                            <el-dropdown-item @click.native="power('off', scope.row)">poweroff</el-dropdown-item>
                            <el-dropdown-item @click.native="power('reset', scope.row)">powercycle
                            </el-dropdown-item>
                            <el-dropdown-item @click.native="power('pxe', scope.row)">pxeboot
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

    </div>
</template>

<script>
    import HttpUtil from "../../common/utils/HttpUtil";

    let _ = require('lodash');
    export default {
        data() {
            return {
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
                form: {},
                loading:false,
                idx: -1,
                id: -1,
                columns: [
                    {
                        label: this.$t('machine_model'),
                        prop: "machineModel"
                    },
                    {
                        label: this.$t('machine_sn'),
                        prop: "machineSn"
                    },
                    {
                        label: this.$t('management_ip'),
                        prop: "managementIp"
                    },
                    {
                        label: this.$t('cpu'),
                        prop: "cpu"
                    },
                    {
                        label: this.$t('memory'),
                        prop: "memory"
                    },
                    {
                        label: this.$t('disk'),
                        prop: "disk"
                    },
                    {
                        label: this.$t('status'),
                        prop: "status"
                    },
                ],
                drawer: false,
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
                wfRequest: {}
            };
        },
        mounted() {
            this.getData();
            this.getAllGraphDefinitions();
        },
        methods: {
            power(opt, row) {
                HttpUtil.get("/bare-metal/power/" + row.id + "/" + opt, null, (res) => {
                    this.$message.success($t('success!'));
                });
            },
            getData() {
                HttpUtil.post("/bare-metal/list/" + this.query.pageIndex + "/" + this.query.pageSize, {}, (res) => {
                    this.tableData = res.data.listObject;
                    this.pageTotal = res.data.itemCount;
                });
            },
            handleSizeChange(val) {
                this.query.pageSize = val;
                this.handlePageChange(this.query.pageIndex);
            },
            handleClose() {
                this.drawer = false;
            },
            cancelForm() {
                this.loading = false;
                this.drawer = false;
            },
            add() {
                this.drawer = true;
                this.editType = 'add';
            },
            confirmEdit() {
                this.loading = true;
                if (this.editType == 'edit') {
                    HttpUtil.post("/bare-metal/update", this.editObj, (res) => {
                        this.drawer = false;
                        this.$message.success('编辑成功');
                        this.cancelForm();
                        this.getData();
                    })
                } else {
                    HttpUtil.post("/bare-metal/add", this.editObj, (res) => {
                        this.drawer = false;
                        this.$message.success('新增成功');
                        this.cancelForm();
                        this.getData();
                    })
                }
            },
            // 多选操作
            handleSelectionChange(val) {
                this.multipleSelection = val;
            },
            delAllSelection() {
                const length = this.multipleSelection.length;
                let str = '';
                this.delList = this.delList.concat(this.multipleSelection);
                for (let i = 0; i < length; i++) {
                    str += this.multipleSelection[i].name + ' ';
                }
                let ids = _.map(this.delList, (item) => item.id);
                HttpUtil.post("/bare-metal/del", ids, (res) => {
                    this.$message.success(`删除成功！删除了${str}！`);
                    this.getData();
                });
                this.multipleSelection = [];
            },
            // 编辑操作
            handleEdit(row, type) {
                if (type == 'edit') {
                    this.drawer = true;
                    this.editType = type;
                    this.editObj = JSON.parse(JSON.stringify(row));
                    ;
                    this.editObj.rolesIds = _.map(this.editObj.roles, (item) => item.id);

                } else if (type == 'del') {
                    this.$confirm('确定要删除吗？', '提示', {
                        type: 'warning'
                    }).then(() => {
                        HttpUtil.get("/bare-metal/del/" + row.id, {}, (res) => {
                            this.getData();
                            this.$message.success('删除成功');
                        });
                    })
                } else {
                    this.drawer = true;
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
                HttpUtil.get("/rackhd/graphdefinitions/1/1000", {name: name}, (res) => {
                    this.allGraphDefinitions = res.data.listObject;
                })
            }
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
        width: 141px;
        height: 120px;
        padding: 10px 10px 15px 10px;
        border-radius: 5px;
    }

    #workflow-selector {
        border: solid #d7d2d2 1px;
        /*width: 100%;*/
        height: 120px;
        padding: 10px 10px 15px 10px;
        border-radius: 5px;
        margin-left: 10px;
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

</style>