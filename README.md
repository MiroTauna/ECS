<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>全景实例运维中心</title>
    <link rel="stylesheet" href="https://unpkg.com/element-plus/dist/index.css" />
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
    <style>
        body { margin: 0; padding: 0; background-color: #f0f2f5; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif; }
        #app { display: flex; height: 100vh; }

        /* 布局样式 (保持统一) */
        .sidebar { width: 220px; background-color: #001529; color: #fff; flex-shrink: 0; display: flex; flex-direction: column; }
        .logo { height: 64px; line-height: 64px; text-align: center; font-size: 20px; font-weight: bold; background-color: #002140; }
        .menu-item { padding: 0 24px; height: 50px; line-height: 50px; cursor: pointer; display: flex; align-items: center; gap: 10px; color: #a6adb4; }
        .menu-item.active { background-color: #1890ff; color: #fff; }

        .main-content { flex: 1; display: flex; flex-direction: column; overflow: hidden; }
        .header { height: 64px; background: #fff; padding: 0 24px; display: flex; align-items: center; justify-content: space-between; box-shadow: 0 1px 4px rgba(0,0,0,0.1); }
        .page-body { padding: 24px; flex: 1; overflow-y: auto; }
        .card { background: #fff; padding: 24px; border-radius: 4px; box-shadow: 0 1px 2px rgba(0,0,0,.05); }
        /* ===== 核心样式：错误信息列 ===== */
        .error-cell { background-color: #fff1f0; border: 1px solid #ffccc7; border-radius: 4px; padding: 8px 12px; color: #cf1322; font-size: 13px; font-family: Consolas, Monaco, monospace; display: flex; justify-content: space-between; align-items: center; }
        .error-text { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; max-width: 400px; }
        .copy-btn { cursor: pointer; color: #1890ff; margin-left: 10px; font-size: 12px; }
        /* ===== 核心样式：全景拓扑图 (Drawer内) ===== */
        .topology-container { display: flex; justify-content: space-between; align-items: center; background: #f5f7fa; padding: 20px; border-radius: 8px; margin-bottom: 20px; border: 1px solid #e4e7ed; }

        .topo-node {
            background: #fff; border: 1px solid #dcdfe6; border-radius: 6px; padding: 15px; width: 160px;
            text-align: center; position: relative; transition: all 0.3s; box-shadow: 0 2px 4px rgba(0,0,0,0.05);
        }
        .topo-node:hover { border-color: #409EFF; box-shadow: 0 4px 12px rgba(0,0,0,0.1); }
        .topo-node.error { border-color: #f56c6c; background: #fef0f0; }

        .node-icon { font-size: 24px; margin-bottom: 8px; color: #606266; }
        .node-title { font-weight: bold; font-size: 14px; margin-bottom: 4px; color: #303133; }
        .node-sub { font-size: 12px; color: #909399; word-break: break-all; }

        /* 箭头连接线 */
        .topo-arrow { flex: 1; height: 2px; background: #dcdfe6; position: relative; margin: 0 20px; }
        .topo-arrow::after { content: ''; position: absolute; right: 0; top: -4px; width: 0; height: 0; border-top: 5px solid transparent; border-bottom: 5px solid transparent; border-left: 8px solid #dcdfe6; }
        /* 日志终端风格 */
        .console-box { background-color: #1e1e1e; color: #d4d4d4; padding: 15px; border-radius: 4px; font-family: 'Menlo', 'Monaco', monospace; font-size: 12px; height: 400px; overflow-y: auto; line-height: 1.5; }
        .log-line { margin-bottom: 2px; }
        .log-info { color: #6a9955; }
        .log-error { color: #f14c4c; }
        .log-warn { color: #cca700; }
    </style>
</head>
<body>
    <div id="app">
        <div class="sidebar">
            <div class="logo">ECS Cloud</div>
            <div class="menu-item"><i class="fas fa-server"></i> 基础设施管理</div>
            <div class="menu-item"><i class="fas fa-network-wired"></i> 网络资源池</div>
            <div class="menu-item active"><i class="fas fa-heartbeat"></i> 全景实例运维</div>
        </div>
        <div class="main-content">
            <div class="header">
                <h3>全景实例运维中心 (O&M Center)</h3>
                <el-input v-model="globalSearch" placeholder="全局搜索 UUID / IP / 报错信息" prefix-icon="el-icon-search" style="width: 300px;"></el-input>
            </div>
            <div class="page-body">
                <div class="card">
                    <!-- 页签区分：正常运维 vs 故障排查 -->
                    <el-tabs v-model="activeTab" type="card">

                        <!-- TAB 1: 所有实例 (常规运维) -->
                        <el-tab-pane label="所有实例列表" name="all">
                            <div style="margin-bottom: 15px;">
                                <el-select v-model="filterTenant" placeholder="按租户筛选" style="width: 150px; margin-right: 10px;"><el-option label="全部" value=""></el-option></el-select>
                                <el-select v-model="filterStatus" placeholder="按状态筛选" style="width: 150px;"><el-option label="Running" value="Running"></el-option></el-select>
                            </div>
                            <el-table :data="instanceList" style="width: 100%" border>
                                <el-table-column prop="name" label="实例名称" width="180" fixed>
                                    <template #default="scope">
                                        <b style="color: #1890ff; cursor: pointer;" @click="openDrawer(scope.row)">{{ scope.row.name }}</b>
                                    </template>
                                </el-table-column>
                                <el-table-column prop="ip" label="IP地址" width="130"></el-table-column>
                                <el-table-column label="状态" width="100">
                                    <template #default="scope">
                                        <el-tag :type="scope.row.status === 'Running' ? 'success' : 'info'">{{ scope.row.status }}</el-tag>
                                    </template>
                                </el-table-column>
                                <el-table-column prop="spec" label="规格" width="120"></el-table-column>
                                <el-table-column prop="host" label="所在宿主机" width="150"></el-table-column>
                                <el-table-column prop="tenant" label="所属租户" width="120"></el-table-column>
                                <el-table-column label="操作" width="150" fixed="right">
                                    <template #default="scope">
                                        <el-button link type="primary" @click="openDrawer(scope.row, 'console')">远程连接</el-button>
                                        <el-button link type="primary" @click="openDrawer(scope.row, 'monitor')">监控</el-button>
                                    </template>
                                </el-table-column>
                            </el-table>
                        </el-tab-pane>
                        <!-- TAB 2: 异常/失败实例 (重点满足需求 4) -->
                        <el-tab-pane name="error">
                            <template #label>
                                <span style="color: #f56c6c;"><i class="fas fa-exclamation-circle"></i> 异常/失败实例 (3)</span>
                            </template>

                            <el-alert title="此列表展示所有 '创建失败' 或 '状态异常' 的实例，请根据报错信息进行排查。" type="error" :closable="false" style="margin-bottom: 15px;"></el-alert>
                            <el-table :data="errorList" style="width: 100%" border>
                                <el-table-column prop="name" label="实例名称" width="180">
                                     <template #default="scope">
                                        <b style="color: #1890ff; cursor: pointer;" @click="openDrawer(scope.row)">{{ scope.row.name }}</b>
                                        <div style="font-size: 12px; color: #999;">ID: {{ scope.row.id }}</div>
                                    </template>
                                </el-table-column>

                                <el-table-column label="失败类型" width="120">
                                    <template #default="scope">
                                        <el-tag type="danger" effect="plain">{{ scope.row.errorType }}</el-tag>
                                    </template>
                                </el-table-column>
                                <!-- 需求4: 直接展示失败原因，不折叠 -->
                                <el-table-column label="失败原因详情 (Failure Message)" min-width="350">
                                    <template #default="scope">
                                        <div class="error-cell">
                                            <div class="error-text" :title="scope.row.errorMessage">{{ scope.row.errorMessage }}</div>
                                            <i class="far fa-copy copy-btn" title="复制报错信息" @click="copyError(scope.row.errorMessage)"></i>
                                        </div>
                                    </template>
                                </el-table-column>
                                <el-table-column prop="createTime" label="操作时间" width="160"></el-table-column>
                                <el-table-column prop="operator" label="操作人" width="100"></el-table-column>

                                <el-table-column label="运维操作" width="200" fixed="right">
                                    <template #default="scope">
                                        <el-button type="primary" size="small" @click="openDrawer(scope.row, 'events')">深度诊断</el-button>
                                        <el-popconfirm title="确定要强制删除该实例吗？资源将立即释放。">
                                            <template #reference>
                                                <el-button type="danger" size="small" plain>强删</el-button>
                                            </template>
                                        </el-popconfirm>
                                    </template>
                                </el-table-column>
                            </el-table>
                        </el-tab-pane>
                    </el-tabs>
                </div>
            </div>
        </div>
        <!-- 核心功能：全景诊断抽屉 (需求 5) -->
        <el-drawer v-model="drawerVisible" :title="
实例诊断: ${currentInstance.name}
" size="80%">
            <div style="padding: 0 20px;">

                <!-- 1. VM-Pod-VMI 映射拓扑图 -->
                <div class="section-title" style="margin-bottom: 10px; font-weight: bold;">资源拓扑映射 (Mapping Topology)</div>
                <div class="topology-container">
                    <!-- Layer 1: VM (逻辑层) -->
                    <div class="topo-node">
                        <i class="fas fa-desktop node-icon" style="color: #409EFF;"></i>
                        <div class="node-title">VM (逻辑)</div>
                        <div class="node-sub">{{ currentInstance.name }}</div>
                        <el-tag size="small" style="margin-top: 5px;">{{ currentInstance.status }}</el-tag>
                    </div>

                    <div class="topo-arrow"></div>
                    <!-- Layer 2: VMI (虚拟化对象) -->
                    <div :class="['topo-node', currentInstance.isError ? 'error' : '']">
                        <i class="fas fa-microchip node-icon" :style="{color: currentInstance.isError ? '#F56C6C' : '#67C23A'}"></i>
                        <div class="node-title">VMI (对象)</div>
                        <div class="node-sub">virt-launcher-xxx</div>
                        <el-tag size="small" :type="currentInstance.isError ? 'danger' : 'success'" style="margin-top: 5px;">
                            {{ currentInstance.isError ? 'CrashLoop' : 'Active' }}
                        </el-tag>
                    </div>
                    <div class="topo-arrow"></div>
                    <!-- Layer 3: Pod (容器层) -->
                    <div :class="['topo-node', currentInstance.isError ? 'error' : '']">
                        <i class="fas fa-box node-icon" :style="{color: currentInstance.isError ? '#F56C6C' : '#67C23A'}"></i>
                        <div class="node-title">Pod (容器)</div>
                        <div class="node-sub">10.244.1.55</div>
                        <el-tag size="small" :type="currentInstance.isError ? 'danger' : 'success'" style="margin-top: 5px;">
                            {{ currentInstance.isError ? 'Error' : 'Running' }}
                        </el-tag>
                    </div>
                    <div class="topo-arrow"></div>
                    <!-- Layer 4: Node (物理层) -->
                    <div class="topo-node">
                        <i class="fas fa-server node-icon"></i>
                        <div class="node-title">Node (宿主机)</div>
                        <div class="node-sub">node-02.prod</div>
                        <el-tag size="small" type="success" style="margin-top: 5px;">Ready</el-tag>
                    </div>
                </div>
                <!-- 2. 诊断详情 Tabs -->
                <el-tabs v-model="drawerTab">
                    <!-- A. 实时日志 (Console) -->
                    <el-tab-pane label="实时日志 (Logs)" name="logs">
                        <div style="margin-bottom: 10px;">
                            <el-radio-group v-model="logSource" size="small">
                                <el-radio-button label="virt-launcher">virt-launcher (组件日志)</el-radio-button>
                                <el-radio-button label="qemu">qemu-kvm (虚拟化日志)</el-radio-button>
                                <el-radio-button label="console">Guest Console (系统启动日志)</el-radio-button>
                            </el-radio-group>
                            <el-button size="small" icon="el-icon-download" style="float: right;">下载日志</el-button>
                        </div>
                        <div class="console-box">
                            <div v-if="currentInstance.isError">
                                <div class="log-line">[INFO] Starting virt-launcher...</div>
                                <div class="log-line">[INFO] Connecting to libvirt daemon...</div>
                                <div class="log-line log-error">[ERROR] Failed to schedule domain: Insufficient resources on node "node-02.prod".</div>
                                <div class="log-line log-error">[FATAL] Domain creation failed. QEMU exited with code 1.</div>
                                <div class="log-line log-warn">[WARN] Retrying in 5 seconds...</div>
                            </div>
                            <div v-else>
                                <div class="log-line">[INFO] Domain started successfully.</div>
                                <div class="log-line">[INFO] Guest IP assigned: 10.233.50.12</div>
                                <div class="log-line log-info">Cloud-init: User data script executed successfully.</div>
                                <div class="log-line">Login prompt: Welcome to CentOS 7.</div>
                            </div>
                        </div>
                    </el-tab-pane>
                    <!-- B. 关键事件 (Events) -->
                    <el-tab-pane label="关键事件 (K8s Events)" name="events">
                        <el-timeline>
                            <el-timeline-item timestamp="2023-10-24 10:05:00" type="primary" placement="top">
                                <div style="font-weight: bold;">Scheduled</div>
                                <div>Successfully assigned default/vm-test-01 to node-02.prod</div>
                            </el-timeline-item>
                            <el-timeline-item timestamp="2023-10-24 10:05:02" type="primary" placement="top">
                                <div style="font-weight: bold;">Pulling</div>
                                <div>Pulling image "kubevirt/virt-launcher:v0.58.0"</div>
                            </el-timeline-item>

                            <!-- 模拟错误事件 -->
                            <el-timeline-item v-if="currentInstance.isError" timestamp="2023-10-24 10:05:10" type="danger" placement="top">
                                <div style="font-weight: bold; color: #f56c6c;">FailedMount</div>
                                <div style="color: #f56c6c;">MountVolume.SetUp failed for volume "data-disk-1": rpc error: code = Internal desc = ceph connection timeout</div>
                            </el-timeline-item>
                            <el-timeline-item v-if="!currentInstance.isError" timestamp="2023-10-24 10:05:15" type="success" placement="top">
                                <div style="font-weight: bold;">Started</div>
                                <div>Started container compute</div>
                            </el-timeline-item>
                        </el-timeline>
                    </el-tab-pane>
                    <!-- C. 元数据 (YAML) -->
                    <el-tab-pane label="元数据 (YAML)" name="yaml">
                        <el-input type="textarea" :rows="15" readonly v-model="mockYaml" style="font-family: monospace;"></el-input>
                    </el-tab-pane>
                </el-tabs>
            </div>
        </el-drawer>
    </div>
    <script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
    <script src="https://unpkg.com/element-plus/dist/index.full.js"></script>

    <script>
        const { createApp, ref, reactive } = Vue;
        const app = createApp({
            setup() {
                const activeTab = ref('all');
                const globalSearch = ref('');
                const filterTenant = ref('');
                const filterStatus = ref('');
                const drawerVisible = ref(false);
                const drawerTab = ref('logs');
                const logSource = ref('virt-launcher');
                const currentInstance = ref({});
                // 模拟正常实例列表
                const instanceList = ref([
                    { name: 'vm-web-prod-01', ip: '10.233.50.12', status: 'Running', spec: '4C8G', host: 'node-01.prod', tenant: '电商部', isError: false },
                    { name: 'vm-db-slave-02', ip: '10.233.50.13', status: 'Running', spec: '8C16G', host: 'node-03.prod', tenant: '金融部', isError: false },
                ]);
                // 模拟异常实例列表 (需求 4 核心数据)
                const errorList = ref([
                    {
                        id: 'i-xz98fd2',
                        name: 'vm-bigdata-worker-05',
                        createTime: '2023-10-24 10:00',
                        operator: 'admin',
                        errorType: '资源不足', // 失败类型
                        errorMessage: '0/24 nodes are available: 3 Insufficient cpu, 20 Insufficient memory. Preemption: 0/24 nodes are available.', // 具体原因，不折叠
                        isError: true
                    },
                    {
                        id: 'i-lk29s8d',
                        name: 'vm-middleware-kafka',
                        createTime: '2023-10-24 09:45',
                        operator: 'zhangsan',
                        errorType: '存储挂载',
                        errorMessage: 'MountVolume.SetUp failed for volume "pvc-29a8": rpc error: code = Internal desc = rbd: map failed exit status 110',
                        isError: true
                    },
                    {
                        id: 'i-po01k2d',
                        name: 'vm-test-image-01',
                        createTime: '2023-10-24 09:30',
                        operator: 'lisi',
                        errorType: '镜像拉取',
                        errorMessage: 'Failed to pull image "harbor.local/os/centos-99:latest": rpc error: code = Unknown desc = Error response from daemon: manifest unknown',
                        isError: true
                    }
                ]);
                const mockYaml = ref(`apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: vm-web-prod-01
  namespace: default
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
      interfaces:
      - masquerade: {}
        name: default
    resources:
      requests:
        memory: 8Gi`);
                const openDrawer = (row, tabName = 'logs') => {
                    currentInstance.value = row;
                    drawerTab.value = tabName;
                    drawerVisible.value = true;
                };
                const copyError = (text) => {
                    // 模拟复制
                    navigator.clipboard.writeText(text).then(() => {
                        ElementPlus.ElMessage.success('报错信息已复制到剪贴板');
                    });
                };
                return {
                    activeTab, globalSearch, filterTenant, filterStatus,
                    instanceList, errorList,
                    drawerVisible, drawerTab, currentInstance, logSource,
                    mockYaml,
                    openDrawer, copyError
                }
            }
        });
        app.use(ElementPlus);
        app.mount('#app');
    </script>
</body>
</html>
