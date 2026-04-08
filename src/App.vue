<script setup>
import FlowTimeline from './components/FlowTimeline.vue';

const mockFlowData = {
  layers: [
    { id: 'material', label: '物料', type: 'timeline' },
    { id: 'warehouse_center', label: '中心仓', type: 'lane' },
    { id: 'xd', label: 'XD', type: 'lane' },
    { id: 'temp_store', label: '齐套暂存点', type: 'lane' },
    { id: 'team', label: '队伍', type: 'timeline' },
  ],
  nodes: [
    // 物料层 (Plan)
    { id: 'm1', layerId: 'material', label: '初始计划nodeWidthnodeWidth', date: '2025-12-31', styleType: 'plan' },
    { id: 'm2', layerId: 'material', label: 'MR1创建', date: '2025-12-31', styleType: 'plan' },
    { id: 'm3', layerId: 'material', label: 'MR2创建', date: '2026-01-05', styleType: 'plan' },
    { id: 'm4', layerId: 'material', label: 'MR3创建', date: '2026-02-01', styleType: 'plan' },

    // 路径 1 节点
    { id: 'e1', layerId: 'warehouse_center', label: '事件1超长超长长', date: '2025-12-31', styleType: 'event' },
    { id: 'e2', layerId: 'xd', label: '事件2', date: '2026-01-02', styleType: 'event' },
    { id: 'ex', layerId: 'xd', label: '事件x超长超长长超长超长长超长超长长超长超长长超长超长长', date: '2026-01-02', styleType: 'event' },
    { id: 'ex2', layerId: 'xd', label: '事件x2超长超长长超长超长长超长超长长超长超长长', date: '2026-01-02', styleType: 'event' },
    { id: 'e3', layerId: 'xd', label: '事件3', date: '2026-01-05', styleType: 'event' },
    { id: 'e4', layerId: 'temp_store', label: '事件4', date: '2026-01-08', styleType: 'event' },
    { id: 't3', layerId: 'team', label: '第一次签收', date: '2026-01-16', styleType: 'plan' },

    // 路径 2 节点
    { id: 'e5', layerId: 'warehouse_center', label: '事件5', date: '2026-01-10', styleType: 'event' },
    { id: 'e6', layerId: 'xd', label: '事件6', date: '2026-01-14', styleType: 'event' },

    // 路径 3 节点
    { id: 'e7', layerId: 'warehouse_center', label: '事件7', date: '2026-02-05', styleType: 'event' },
    { id: 'e8', layerId: 'xd', label: '事件8', date: '2026-02-08', styleType: 'event' },

    // 共享节点与结尾
    { id: 'e9', layerId: 'xd', label: '事件9', date: '2026-02-12', styleType: 'event' },
    { id: 't4', layerId: 'team', label: '第二次签收', date: '2026-02-16', styleType: 'plan' },

    // 辅助节点 (添加日期)
    { id: 't1', layerId: 'team', label: '计划变更', date: '2026-01-05', styleType: 'plan' },
    { id: 't2', layerId: 'team', label: '第一次打卡', date: '2026-01-15', styleType: 'plan' },
  ],
  links: [
    // Path 1
    { source: 'm2', target: 'e1' },
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
      <h1>物料供应链协同流转图</h1>
    </header>
    <main>
      <FlowTimeline :data="mockFlowData" />
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
</style>
