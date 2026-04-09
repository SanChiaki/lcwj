<script setup>
import { ref } from 'vue';
import FlowTimeline from './components/FlowTimeline.vue';
import ApprovalFlow from './components/ApprovalFlow.vue';

const activeTab = ref('approval');
const tabs = [
  { key: 'approval', label: '审批流程' },
  { key: 'flow', label: '供应链流转' },
];

const mockApprovalData = {
  steps: [
    { id: 'create', label: '创建' },
    { id: 'review', label: '初审' },
    { id: 'checkin', label: '打卡' },
    { id: 'submit', label: '提交' },
    { id: 'l1', label: 'L1审批' },
  ],
  processes: [
    // Process 1 (index 0)
    {
      entries: [
        { stepId: 'create',  arrivedAt: '2026-03-10T12:00:00' },
        { stepId: 'review',  arrivedAt: '2026-03-10T12:30:00' },
        { stepId: 'checkin', arrivedAt: '2026-03-10T13:15:00' },
        { stepId: 'submit',  arrivedAt: '2026-03-10T14:15:00' },
        { stepId: 'l1',      arrivedAt: '2026-03-10T15:15:00' },
      ],
    },
    // Process 2 (index 1)
    {
      entries: [
        { stepId: 'create',  arrivedAt: '2026-03-10T12:10:00' },
        { stepId: 'review',  arrivedAt: '2026-03-10T12:40:00' },
        { stepId: 'checkin', arrivedAt: '2026-03-10T13:35:00' },
        { stepId: 'submit',  arrivedAt: '2026-03-10T14:25:00' },
        { stepId: 'l1',      arrivedAt: '2026-03-10T15:25:00' },
      ],
    },
    // Process 3 (index 2) — 在L1审批被驳回
    {
      entries: [
        { stepId: 'create',  arrivedAt: '2026-03-10T12:20:00' },
        { stepId: 'review',  arrivedAt: '2026-03-10T12:50:00' },
        { stepId: 'checkin', arrivedAt: '2026-03-10T14:55:00' },
        { stepId: 'submit',  arrivedAt: '2026-03-10T15:05:00' },
        { stepId: 'l1',      arrivedAt: '2026-03-10T15:25:00', rejected: true },
      ],
      rejection: {
        restartFromStepId: 'checkin',
        newProcessIndex: 3,
      },
    },
    // Process 4 (index 3) — 由 Process 3 驳回后重新开始
    {
      entries: [
        { stepId: 'checkin', arrivedAt: '2026-03-10T15:45:00' },
        { stepId: 'submit',  arrivedAt: '2026-03-10T15:55:00' },
        { stepId: 'l1',      arrivedAt: '2026-03-10T16:55:00' },
      ],
    },
    // Process 5 (index 4) — 在L1审批被驳回
    {
      entries: [
        { stepId: 'create',  arrivedAt: '2026-03-10T16:00:00' },
        { stepId: 'review',  arrivedAt: '2026-03-10T16:20:00' },
        { stepId: 'checkin', arrivedAt: '2026-03-10T17:00:00' },
        { stepId: 'submit',  arrivedAt: '2026-03-10T17:10:00' },
        { stepId: 'l1',      arrivedAt: '2026-03-10T17:20:00', rejected: true },
      ],
      rejection: {
        restartFromStepId: 'checkin',
        newProcessIndex: 5,
      },
    },
    // Process 6 (index 5) — 由 Process 5 驳回后重新开始
    {
      entries: [
        { stepId: 'checkin', arrivedAt: '2026-03-10T17:30:00' },
        { stepId: 'submit',  arrivedAt: '2026-03-10T17:40:00' },
        { stepId: 'l1',      arrivedAt: '2026-03-10T18:40:00' },
      ],
    },
  ],
};

const mockFlowData = {
  layers: [
    { id: 'material', label: '物料', type: 'timeline' },
    { id: 'warehouse_center', label: '中心仓', type: 'lane' },
    { id: 'xd', label: 'XD', type: 'lane' },
    { id: 'temp_store', label: '齐套暂存点', type: 'lane' },
    { id: 'team', label: '队伍', type: 'timeline' },
  ],
  nodes: [
    // 物料层 (Plan) - 含单号和物料内容
    { id: 'm1', layerId: 'material', label: '初始计划', orderNo: 'PO-20251231-001', material: '钢板/螺栓/线缆', date: '2025-12-31', styleType: 'plan' },
    { id: 'm2', layerId: 'material', label: 'MR1创建', orderNo: 'MR-20251231-001', material: '液压泵组件', date: '2025-12-31', styleType: 'plan' },
    { id: 'm3', layerId: 'material', label: 'MR2创建', orderNo: 'MR-20260105-002', material: '控制面板', date: '2026-01-05', styleType: 'plan' },
    { id: 'm4', layerId: 'material', label: 'MR3创建', orderNo: 'MR-20260201-003', material: '发动机总成', date: '2026-02-01', styleType: 'plan' },

    // 路径 1 节点 - 含单号
    { id: 'e1', layerId: 'warehouse_center', orderNo: 'WH-001', label: '入库登记', date: '2025-12-31', styleType: 'event' },
    { id: 'e1-2', layerId: 'warehouse_center', orderNo: 'WH-002', label: '质检完成', date: '2025-12-31', styleType: 'event' },
    { id: 'e2', layerId: 'xd', orderNo: 'XD-101', label: '越库分拣', date: '2026-01-02', styleType: 'event' },
    { id: 'ex', layerId: 'xd', orderNo: 'XD-102', label: '异常物料暂扣超长处理', date: '2026-01-02', styleType: 'event' },
    { id: 'ex2', layerId: 'xd', orderNo: 'XD-103', label: '补单重发流程', date: '2026-01-02', styleType: 'event' },
    { id: 'e3', layerId: 'xd', orderNo: 'XD-104', label: '配送出库', date: '2026-01-05', styleType: 'event' },
    { id: 'e4', layerId: 'temp_store', orderNo: 'TS-201', label: '齐套暂存', date: '2026-01-08', styleType: 'event' },
    { id: 't3', layerId: 'team', label: '第一次签收', orderNo: 'RC-20260116-001', material: '液压泵/钢板', date: '2026-01-16', styleType: 'plan' },

    // 路径 2 节点
    { id: 'e5', layerId: 'warehouse_center', orderNo: 'WH-003', label: '二次入库', date: '2026-01-10', styleType: 'event' },
    { id: 'e6', layerId: 'xd', orderNo: 'XD-105', label: '加急越库', date: '2026-01-14', styleType: 'event' },

    // 路径 3 节点
    { id: 'e7', layerId: 'warehouse_center', orderNo: 'WH-004', label: '批次入库', date: '2026-02-05', styleType: 'event' },
    { id: 'e8', layerId: 'xd', orderNo: 'XD-106', label: '整批分拣', date: '2026-02-08', styleType: 'event' },

    // 共享节点与结尾
    { id: 'e9', layerId: 'xd', orderNo: 'XD-107', label: '末端配送', date: '2026-02-12', styleType: 'event' },
    { id: 't4', layerId: 'team', label: '第二次签收', orderNo: 'RC-20260216-002', material: '控制面板/发动机', date: '2026-02-16', styleType: 'plan' },

    // 辅助节点 (添加日期)
    { id: 't1', layerId: 'team', label: '计划变更', orderNo: 'CHG-20260105-001', material: '钢板规格调整', date: '2026-01-05', styleType: 'plan' },
    { id: 't2', layerId: 'team', label: '第一次打卡', date: '2026-01-15', styleType: 'checkin' },
  ],
  links: [
    // Path 1
    { source: 'm1', target: 'e1' },
    { source: 'm2', target: 'e1-2' },
    { source: 'e1', target: 'e2' },
    { source: 'e2', target: 'e3' },
    { source: 'e3', target: 'e4' },
    { source: 'e4', target: 't3' },
    // Path 2
    { source: 'm3', target: 'e5' },
    { source: 'e5', target: 'e6' },
    { source: 'e6', target: 'e9' },
    // Path 3
    { source: 'm4', target: 'e7' },
    { source: 'e7', target: 'e8' },
    { source: 'e8', target: 'e9' },
    // Path Shared
    { source: 'e9', target: 't4' },
  ]
};

</script>

<template>
  <div class="app-container">
    <header>
      <h1>流程图</h1>
    </header>
    <div class="tabs">
      <button
        v-for="tab in tabs"
        :key="tab.key"
        :class="['tab-btn', { active: activeTab === tab.key }]"
        @click="activeTab = tab.key"
      >
        {{ tab.label }}
      </button>
    </div>
    <main>
      <ApprovalFlow v-if="activeTab === 'approval'" :data="mockApprovalData" />
      <FlowTimeline v-else :data="mockFlowData" />
    </main>
  </div>
</template>

<style>
body {
  margin: 0;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  background-color: #f0f2f5;
}

.app-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  padding: 20px;
  box-sizing: border-box;
}

header {
  margin-bottom: 20px;
}

header h1 {
  margin: 0;
  font-size: 24px;
  color: #1f1f1f;
}

main {
  flex: 1;
  overflow: hidden;
  background: white;
  border-radius: 8px;
}

.tabs {
  display: flex;
  gap: 4px;
  margin-bottom: 12px;
}

.tab-btn {
  padding: 8px 20px;
  border: 1px solid #ddd;
  background: #f5f5f5;
  border-radius: 6px 6px 0 0;
  cursor: pointer;
  font-size: 14px;
  color: #666;
}

.tab-btn.active {
  background: #fff;
  color: #1f1f1f;
  font-weight: bold;
  border-bottom-color: #fff;
}
</style>
