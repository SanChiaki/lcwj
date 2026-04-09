<template>
  <div class="approval-flow-container">
    <div ref="container" class="x6-graph"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue';
import { Graph } from '@antv/x6';

const container = ref(null);

const props = defineProps({
  data: { type: Object, required: true }
});

let graph = null;

onMounted(() => {
  if (!container.value) return;
  graph = new Graph({
    container: container.value,
    autoResize: true,
    background: { color: '#ffffff' },
    interacting: false,
    panning: true,
    mousewheel: { enabled: true, modifiers: 'ctrl' },
  });
  renderFlow(graph, props.data);
});

watch(() => props.data, (newData) => {
  if (graph) {
    graph.clearCells();
    renderFlow(graph, newData);
  }
}, { deep: true });

/**
 * 格式化时间差（毫秒 → 可读字符串）
 */
const formatDuration = (ms) => {
  if (ms < 0) ms = 0;
  const totalMinutes = Math.round(ms / 60000);
  if (totalMinutes < 60) return `${totalMinutes}mins`;
  const hours = Math.floor(totalMinutes / 60);
  const mins = totalMinutes % 60;
  if (mins === 0) return `${hours}hour${hours > 1 ? 's' : ''}`;
  return `${hours}h${mins}m`;
};

/**
 * 格式化时间戳用于节点内显示
 */
const formatTime = (isoStr) => {
  if (!isoStr) return '';
  const d = new Date(isoStr);
  const pad = (n) => String(n).padStart(2, '0');
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())} ${pad(d.getHours())}:${pad(d.getMinutes())}:${pad(d.getSeconds())}`;
};

const renderFlow = (graph, data) => {
  const { steps, processes } = data;

  // --- 配置 ---
  const NODE_WIDTH = 320;
  const NODE_PADDING_X = 16;
  const NODE_PADDING_TOP = 14;
  const TITLE_HEIGHT = 24;
  const LINE_HEIGHT = 20;
  const NODE_PADDING_BOTTOM = 12;
  const STEP_GAP_Y = 200;       // 主节点间垂直间距
  const START_X = 100;
  const START_Y = 60;
  const REJECTION_OFFSET_X = 500; // 驳回节点水平偏移

  // --- 1. 收集每个 step 的 process 信息 ---
  // stepProcesses[stepId] = [{ processIndex, arrivedAt, rejected }]
  const stepProcesses = new Map();
  steps.forEach(s => stepProcesses.set(s.id, []));

  processes.forEach((proc, pIdx) => {
    proc.entries.forEach(entry => {
      if (stepProcesses.has(entry.stepId)) {
        stepProcesses.get(entry.stepId).push({
          processIndex: pIdx,
          arrivedAt: entry.arrivedAt,
          rejected: !!entry.rejected
        });
      }
    });
  });

  // --- 2. 计算主节点尺寸和位置 ---
  const stepPositions = new Map(); // stepId → { x, y, width, height }
  const calcNodeHeight = (processCount) => {
    return NODE_PADDING_TOP + TITLE_HEIGHT + processCount * LINE_HEIGHT + NODE_PADDING_BOTTOM;
  };

  let currentY = START_Y;
  steps.forEach((step) => {
    const procs = stepProcesses.get(step.id);
    const height = calcNodeHeight(procs.length);
    stepPositions.set(step.id, {
      x: START_X,
      y: currentY,
      width: NODE_WIDTH,
      height: height
    });
    currentY += height + STEP_GAP_Y;
  });

  // --- 3. 收集驳回节点信息 ---
  const rejectionNodes = []; // { processIndex, stepId, stepLabel, arrivedAt, rejectionNodeId }
  processes.forEach((proc, pIdx) => {
    if (proc.rejection) {
      const rejectedEntry = proc.entries.find(e => e.rejected);
      if (rejectedEntry) {
        const step = steps.find(s => s.id === rejectedEntry.stepId);
        rejectionNodes.push({
          processIndex: pIdx,
          stepId: rejectedEntry.stepId,
          stepLabel: step ? step.label : rejectedEntry.stepId,
          arrivedAt: rejectedEntry.arrivedAt,
          rejectionNodeId: `rejection-p${pIdx}`,
          restartFromStepId: proc.rejection.restartFromStepId,
          newProcessIndex: proc.rejection.newProcessIndex
        });
      }
    }
  });

  // --- 4. 渲染主节点 ---
  steps.forEach((step) => {
    const pos = stepPositions.get(step.id);
    const procs = stepProcesses.get(step.id);

    // 构建 markup：body + title + 每个 process 一行
    const markup = [
      { tagName: 'rect', selector: 'body' },
      { tagName: 'text', selector: 'title' },
    ];
    const attrs = {
      body: {
        fill: '#4285F4',
        stroke: 'none',
        rx: 6,
        ry: 6,
        width: pos.width,
        height: pos.height,
      },
      title: {
        text: step.label,
        fill: '#fff',
        fontSize: 15,
        fontWeight: 'bold',
        textAnchor: 'middle',
        refX: 0.5,
        refY: NODE_PADDING_TOP,
      },
    };

    procs.forEach((p, idx) => {
      const selector = `proc${idx}`;
      markup.push({ tagName: 'text', selector });
      const text = `Process ${p.processIndex + 1}: ${formatTime(p.arrivedAt)}`;
      attrs[selector] = {
        text,
        fill: '#fff',
        fontSize: 12,
        textAnchor: 'middle',
        refX: 0.5,
        refY: NODE_PADDING_TOP + TITLE_HEIGHT + idx * LINE_HEIGHT,
      };
    });

    graph.addNode({
      id: step.id,
      x: pos.x,
      y: pos.y,
      width: pos.width,
      height: pos.height,
      markup,
      attrs,
      zIndex: 5,
    });
  });

  // --- 5. 渲染驳回节点 ---
  rejectionNodes.forEach((rn) => {
    const sourcePos = stepPositions.get(rn.stepId);
    if (!sourcePos) return;

    const rHeight = calcNodeHeight(1);
    const rX = sourcePos.x + REJECTION_OFFSET_X;
    const rY = sourcePos.y + (sourcePos.height - rHeight) / 2;

    const markup = [
      { tagName: 'rect', selector: 'body' },
      { tagName: 'text', selector: 'title' },
      { tagName: 'text', selector: 'processLine' },
      { tagName: 'text', selector: 'timeLine' },
    ];
    const attrs = {
      body: {
        fill: '#FF4D6A',
        stroke: 'none',
        rx: 6,
        ry: 6,
        width: NODE_WIDTH,
        height: rHeight,
      },
      title: {
        text: rn.stepLabel,
        fill: '#fff',
        fontSize: 15,
        fontWeight: 'bold',
        textAnchor: 'middle',
        refX: 0.5,
        refY: NODE_PADDING_TOP,
      },
      processLine: {
        text: `Process ${rn.processIndex + 1} [被驳回]`,
        fill: '#fff',
        fontSize: 12,
        textAnchor: 'middle',
        refX: 0.5,
        refY: NODE_PADDING_TOP + TITLE_HEIGHT,
      },
      timeLine: {
        text: `${formatTime(rn.arrivedAt)} – Rejected`,
        fill: '#fff',
        fontSize: 12,
        textAnchor: 'middle',
        refX: 0.5,
        refY: NODE_PADDING_TOP + TITLE_HEIGHT + LINE_HEIGHT,
      },
    };

    graph.addNode({
      id: rn.rejectionNodeId,
      x: rX,
      y: rY,
      width: NODE_WIDTH,
      height: rHeight,
      markup,
      attrs,
      zIndex: 5,
    });
  });

  // --- 6. 渲染主流程连线（相邻步骤之间） ---
  // 对于每对相邻步骤，找出经过两个步骤的 process，画连线
  for (let i = 0; i < steps.length - 1; i++) {
    const fromStep = steps[i];
    const toStep = steps[i + 1];

    // 找出经过两个步骤的 process
    const edgeProcesses = [];
    processes.forEach((proc, pIdx) => {
      const fromEntry = proc.entries.find(e => e.stepId === fromStep.id);
      const toEntry = proc.entries.find(e => e.stepId === toStep.id);
      if (fromEntry && toEntry) {
        const duration = new Date(toEntry.arrivedAt) - new Date(fromEntry.arrivedAt);
        edgeProcesses.push({ processIndex: pIdx, duration });
      }
    });

    if (edgeProcesses.length === 0) continue;

    const fromPos = stepPositions.get(fromStep.id);
    const toPos = stepPositions.get(toStep.id);

    // 渲染连线（无标签）
    const totalEdges = edgeProcesses.length;
    const edgeSpacing = 60;
    const baseX = fromPos.x + fromPos.width / 2;
    const startOffsetX = -(totalEdges - 1) * edgeSpacing / 2;

    edgeProcesses.forEach((_, edgeIdx) => {
      const offsetX = startOffsetX + edgeIdx * edgeSpacing;
      graph.addEdge({
        source: { x: baseX + offsetX, y: fromPos.y + fromPos.height },
        target: { x: baseX + offsetX, y: toPos.y },
        connector: { name: 'normal' },
        attrs: {
          line: {
            stroke: '#4A86E8',
            strokeWidth: 1.5,
            targetMarker: { name: 'classic', size: 6 },
          },
        },
        zIndex: 2,
      });
    });

    // 标签沿连线纵向堆叠排列
    const gapTop = fromPos.y + fromPos.height;
    const gapBottom = toPos.y;
    const gapHeight = gapBottom - gapTop;
    const LINE_H = 14;

    if (totalEdges === 1) {
      graph.addNode({
        x: baseX - 55, y: gapTop + gapHeight / 2 - LINE_H / 2,
        width: 110, height: LINE_H,
        shape: 'text-block',
        text: `Process ${edgeProcesses[0].processIndex + 1}: ${formatDuration(edgeProcesses[0].duration)}`,
        attrs: {
          body: { fill: '#fff', stroke: 'none', rx: 2 },
          text: { fill: '#4A86E8', fontSize: 10, textAlign: 'center' },
        },
        zIndex: 3,
      });
    } else {
      const labelGap = Math.max(6, Math.floor((gapHeight - totalEdges * LINE_H) / (totalEdges + 1)));
      edgeProcesses.forEach((ep, idx) => {
        const edgeOffsetX = startOffsetX + idx * edgeSpacing;
        graph.addNode({
          x: baseX + edgeOffsetX - 55,
          y: gapTop + labelGap + idx * (LINE_H + labelGap),
          width: 110, height: LINE_H,
          shape: 'text-block',
          text: `Process ${ep.processIndex + 1}: ${formatDuration(ep.duration)}`,
          attrs: {
            body: { fill: '#fff', stroke: 'none', rx: 2 },
            text: { fill: '#4A86E8', fontSize: 10, textAlign: 'center' },
          },
          zIndex: 3,
        });
      });
    }
  }

  // --- 7. 渲染驳回连线 ---
  rejectionNodes.forEach((rn) => {
    const sourcePos = stepPositions.get(rn.stepId);
    const rHeight = calcNodeHeight(1);
    const rX = sourcePos.x + REJECTION_OFFSET_X;
    const rY = sourcePos.y + (sourcePos.height - rHeight) / 2;

    // 从主节点到驳回节点的连线
    const rejectedEntry = processes[rn.processIndex].entries.find(e => e.rejected);
    const prevEntry = processes[rn.processIndex].entries[
      processes[rn.processIndex].entries.indexOf(rejectedEntry) - 1
    ];
    const duration = prevEntry
      ? new Date(rejectedEntry.arrivedAt) - new Date(prevEntry.arrivedAt)
      : 0;

    graph.addEdge({
      source: { cell: rn.stepId, anchor: { name: 'right' } },
      target: { cell: rn.rejectionNodeId, anchor: { name: 'left' } },
      connector: { name: 'rounded', args: { radius: 8 } },
      labels: duration > 0 ? [
        {
          position: 0.5,
          attrs: {
            label: {
              text: `Process ${rn.processIndex + 1}: ${formatDuration(duration)}`,
              fill: '#FF4D6A',
              fontSize: 11,
            },
            rect: { fill: '#fff', stroke: 'none' },
          },
        },
      ] : [],
      attrs: {
        line: {
          stroke: '#FF4D6A',
          strokeWidth: 1.5,
          targetMarker: { name: 'classic', size: 6 },
        },
      },
      zIndex: 2,
    });

    // 从驳回节点回到重新开始的步骤
    const restartPos = stepPositions.get(rn.restartFromStepId);
    if (restartPos) {
      // 计算新 process 的耗时（从驳回到新 process 的第一个 entry）
      const newProc = processes[rn.newProcessIndex];
      let restartDuration = 0;
      if (newProc && newProc.entries.length > 0) {
        restartDuration = new Date(newProc.entries[0].arrivedAt) - new Date(rejectedEntry.arrivedAt);
      }

      graph.addEdge({
        source: { cell: rn.rejectionNodeId, anchor: { name: 'top' } },
        target: { cell: rn.restartFromStepId, anchor: { name: 'right' } },
        router: {
          name: 'manhattan',
          args: { padding: 40 },
        },
        connector: { name: 'rounded', args: { radius: 20 } },
        labels: restartDuration > 0 ? [
          {
            position: 0.5,
            attrs: {
              label: {
                text: `Process ${rn.newProcessIndex + 1}: ${formatDuration(restartDuration)}`,
                fill: '#FF4D6A',
                fontSize: 11,
              },
              rect: { fill: '#fff', stroke: 'none' },
            },
          },
        ] : [],
        attrs: {
          line: {
            stroke: '#FF4D6A',
            strokeWidth: 1.5,
            strokeDasharray: '6,3',
            targetMarker: { name: 'classic', size: 6 },
          },
        },
        zIndex: 2,
      });
    }
  });

  graph.centerContent();
};
</script>

<style scoped>
.approval-flow-container { width: 100%; height: 100%; min-height: 600px; background: #fff; }
.x6-graph { width: 100%; height: 100%; }
</style>
