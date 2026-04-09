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
  const NODE_PADDING_X = 20;
  const NODE_PADDING_TOP = 14;
  const TITLE_HEIGHT = 24;
  const LINE_HEIGHT = 20;
  const NODE_PADDING_BOTTOM = 12;
  const STEP_GAP_Y = 200;       // 主节点间垂直间距
  const START_X = 100;
  const START_Y = 60;
  const REJECTION_OFFSET_X = 200; // 驳回节点与主节点的间距

  // 文字宽度估算
  const calcTextWidth = (text, fontSize, bold = false) => {
    let width = 0;
    for (const char of (text || '')) {
      width += (char.charCodeAt(0) > 127) ? fontSize : fontSize * 0.62;
    }
    if (bold) width *= 1.05;
    return width;
  };

  // --- 1. 收集每个 step 的 process 信息 ---
  const stepProcesses = new Map();
  steps.forEach(s => stepProcesses.set(s.id, []));

  processes.forEach((proc, pIdx) => {
    proc.entries.forEach(entry => {
      if (stepProcesses.has(entry.stepId) && !entry.rejected) {
        stepProcesses.get(entry.stepId).push({
          processIndex: pIdx,
          arrivedAt: entry.arrivedAt,
          rejected: !!entry.rejected
        });
      }
    });
  });

  // --- 2. 计算主节点尺寸和位置 ---
  const stepPositions = new Map();
  const calcNodeHeight = (lineCount) => {
    return NODE_PADDING_TOP + TITLE_HEIGHT + lineCount * LINE_HEIGHT + NODE_PADDING_BOTTOM;
  };

  // 先计算每个节点需要的宽度
  const stepWidths = new Map();
  steps.forEach((step) => {
    const procs = stepProcesses.get(step.id);
    const titleWidth = calcTextWidth(step.label, 15, true);
    let maxLineWidth = titleWidth;
    procs.forEach((p) => {
      const text = `Process ${p.processIndex + 1}: ${formatTime(p.arrivedAt)}`;
      maxLineWidth = Math.max(maxLineWidth, calcTextWidth(text, 12));
    });
    stepWidths.set(step.id, maxLineWidth + NODE_PADDING_X * 2);
  });

  // 所有主节点使用统一宽度（取最大值），保持对齐
  const nodeWidth = Math.max(...stepWidths.values(), 200);

  let currentY = START_Y;
  steps.forEach((step) => {
    const procs = stepProcesses.get(step.id);
    const height = calcNodeHeight(procs.length);
    stepPositions.set(step.id, {
      x: START_X,
      y: currentY,
      width: nodeWidth,
      height: height
    });
    currentY += height + STEP_GAP_Y;
  });

  // --- 3. 收集驳回节点信息 ---
  const rejectionNodes = [];
  processes.forEach((proc, pIdx) => {
    if (proc.rejection) {
      const rejectedEntry = proc.entries.find(e => e.rejected);
      if (rejectedEntry) {
        const step = steps.find(s => s.id === rejectedEntry.stepId);
        const stepIdx = steps.findIndex(s => s.id === rejectedEntry.stepId);
        // 前一个步骤（连线的出发节点）
        const prevStepId = stepIdx > 0 ? steps[stepIdx - 1].id : rejectedEntry.stepId;
        rejectionNodes.push({
          processIndex: pIdx,
          stepId: rejectedEntry.stepId,
          prevStepId: prevStepId,
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

    const markup = [
      { tagName: 'rect', selector: 'body' },
      { tagName: 'text', selector: 'title' },
    ];
    const attrs = {
      body: {
        fill: '#7B9FC7',
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
  const rejectionsByStep = new Map();
  rejectionNodes.forEach(rn => {
    if (!rejectionsByStep.has(rn.prevStepId)) {
      rejectionsByStep.set(rn.prevStepId, []);
    }
    rejectionsByStep.get(rn.prevStepId).push(rn);
  });

  // 计算驳回节点宽度
  const calcRejectionWidth = (rn) => {
    const titleWidth = calcTextWidth(rn.stepLabel, 15, true);
    const processLineWidth = calcTextWidth(`Process ${rn.processIndex + 1} [被驳回]`, 12);
    const timeLineWidth = calcTextWidth(`${formatTime(rn.arrivedAt)} – Rejected`, 12);
    return Math.max(titleWidth, processLineWidth, timeLineWidth) + NODE_PADDING_X * 2;
  };

  rejectionsByStep.forEach((rns, stepId) => {
    const sourcePos = stepPositions.get(stepId);
    if (!sourcePos) return;

    // 统一同组驳回节点宽度
    const rNodeWidth = Math.max(...rns.map(calcRejectionWidth), 160);
    const rHeight = calcNodeHeight(2);
    const rX = sourcePos.x + sourcePos.width + REJECTION_OFFSET_X;
    const REJECTION_GAP = 20;
    const totalHeight = rns.length * rHeight + (rns.length - 1) * REJECTION_GAP;
    const startY = sourcePos.y + (sourcePos.height - totalHeight) / 2;

    rns.forEach((rn, idx) => {
      const rY = startY + idx * (rHeight + REJECTION_GAP);

      const markup = [
        { tagName: 'rect', selector: 'body' },
        { tagName: 'text', selector: 'title' },
        { tagName: 'text', selector: 'processLine' },
        { tagName: 'text', selector: 'timeLine' },
      ];
      const attrs = {
        body: {
          fill: '#C87878',
          stroke: 'none',
          rx: 6,
          ry: 6,
          width: rNodeWidth,
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
        width: rNodeWidth,
        height: rHeight,
        markup,
        attrs,
        zIndex: 5,
      });
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
    // 动态计算间距，确保连线不超出节点宽度
    const maxSpread = fromPos.width * 0.8; // 最多占用节点宽度的80%
    const edgeSpacing = Math.min(60, maxSpread / Math.max(totalEdges - 1, 1));
    const baseX = fromPos.x + fromPos.width / 2;
    const startOffsetX = -(totalEdges - 1) * edgeSpacing / 2;

    edgeProcesses.forEach((_, edgeIdx) => {
      const offsetX = startOffsetX + edgeIdx * edgeSpacing;
      const srcX = baseX + offsetX;
      const srcY = fromPos.y + fromPos.height;
      const tgtY = toPos.y;
      // 弧线：用中间控制点向外偏移，越靠边的线弧度越大
      const curveOffset = (edgeIdx - (totalEdges - 1) / 2) * 15;
      graph.addEdge({
        source: { x: srcX, y: srcY },
        target: { x: srcX, y: tgtY },
        vertices: [{ x: srcX + curveOffset, y: (srcY + tgtY) / 2 }],
        connector: { name: 'smooth' },
        attrs: {
          line: {
            stroke: '#7B9FC7',
            strokeWidth: 1.5,
            targetMarker: { name: 'classic', size: 6 },
          },
        },
        zIndex: 2,
      });
    });

    // 标签沿连线纵向堆叠排列（用 rect + text markup，确保字号生效）
    const gapTop = fromPos.y + fromPos.height;
    const gapBottom = toPos.y;
    const gapHeight = gapBottom - gapTop;
    const LINE_H = 16;

    if (totalEdges === 1) {
      const labelText = `Process ${edgeProcesses[0].processIndex + 1}: ${formatDuration(edgeProcesses[0].duration)}`;
      const labelWidth = calcTextWidth(labelText, 11) + 16;
      graph.addNode({
        x: baseX - labelWidth / 2, y: gapTop + gapHeight / 2 - LINE_H / 2,
        width: labelWidth, height: LINE_H,
        markup: [
          { tagName: 'rect', selector: 'body' },
          { tagName: 'text', selector: 'label' },
        ],
        attrs: {
          body: { fill: '#fff', stroke: 'none', rx: 2, ry: 2, refWidth: '100%', refHeight: '100%' },
          label: { text: labelText, fill: '#7B9FC7', fontSize: 11, textAnchor: 'middle', refX: 0.5, refY: 0.5, yAlignment: 'middle' },
        },
        zIndex: 10,
      });
    } else {
      const labelGap = Math.max(4, Math.floor((gapHeight - totalEdges * LINE_H) / (totalEdges + 1)));
      edgeProcesses.forEach((ep, idx) => {
        const labelText = `Process ${ep.processIndex + 1}: ${formatDuration(ep.duration)}`;
        const labelWidth = calcTextWidth(labelText, 11) + 16;
        const edgeOffsetX = startOffsetX + idx * edgeSpacing;
        graph.addNode({
          x: baseX + edgeOffsetX - labelWidth / 2,
          y: gapTop + labelGap + idx * (LINE_H + labelGap),
          width: labelWidth, height: LINE_H,
          markup: [
            { tagName: 'rect', selector: 'body' },
            { tagName: 'text', selector: 'label' },
          ],
          attrs: {
            body: { fill: '#fff', stroke: 'none', rx: 2, ry: 2, refWidth: '100%', refHeight: '100%' },
            label: { text: labelText, fill: '#7B9FC7', fontSize: 11, textAnchor: 'middle', refX: 0.5, refY: 0.5, yAlignment: 'middle' },
          },
          zIndex: 10,
        });
      });
    }
  }

  // --- 7. 渲染驳回连线 ---
  // 需要知道每个驳回节点的实际位置，重新计算
  const rejectionPositions = new Map(); // rejectionNodeId → { x, y, width, height }
  rejectionsByStep.forEach((rns, stepId) => {
    const sourcePos = stepPositions.get(stepId);
    if (!sourcePos) return;
    const rNodeWidth = Math.max(...rns.map(calcRejectionWidth), 160);
    const rHeight = calcNodeHeight(2);
    const rX = sourcePos.x + sourcePos.width + REJECTION_OFFSET_X;
    const REJECTION_GAP = 20;
    const totalHeight = rns.length * rHeight + (rns.length - 1) * REJECTION_GAP;
    const startY = sourcePos.y + (sourcePos.height - totalHeight) / 2;
    rns.forEach((rn, idx) => {
      rejectionPositions.set(rn.rejectionNodeId, {
        x: rX, y: startY + idx * (rHeight + REJECTION_GAP),
        width: rNodeWidth, height: rHeight,
      });
    });
  });

  // 计算所有驳回节点的最右边界，回连线需要绕过
  let maxRejectionRight = 0;
  rejectionPositions.forEach(pos => {
    maxRejectionRight = Math.max(maxRejectionRight, pos.x + pos.width);
  });

  rejectionNodes.forEach((rn, rnIdx) => {
    const prevStepPos = stepPositions.get(rn.prevStepId);
    const rPos = rejectionPositions.get(rn.rejectionNodeId);
    if (!rPos || !prevStepPos) return;

    // 从上一个步骤节点到驳回节点的连线
    const rejectedEntry = processes[rn.processIndex].entries.find(e => e.rejected);
    const prevEntry = processes[rn.processIndex].entries.find(e => e.stepId === rn.prevStepId);
    const duration = prevEntry
      ? new Date(rejectedEntry.arrivedAt) - new Date(prevEntry.arrivedAt)
      : 0;

    // 源：上一个步骤节点右侧中心，目标：驳回节点左侧中心
    const srcX = prevStepPos.x + prevStepPos.width;
    const srcY = prevStepPos.y + prevStepPos.height / 2;
    const tgtX = rPos.x;
    const tgtY = rPos.y + rPos.height / 2;

    graph.addEdge({
      source: { x: srcX, y: srcY },
      target: { x: tgtX, y: tgtY },
      connector: { name: 'smooth' },
      labels: duration > 0 ? [
        {
          position: 0.5,
          attrs: {
            label: {
              text: `Process ${rn.processIndex + 1}: ${formatDuration(duration)}`,
              fill: '#C87878',
              fontSize: 11,
            },
            rect: { fill: '#fff', stroke: 'none', rx: 2, ry: 2, refWidth: 8, refHeight: 4, refX: -4, refY: -2 },
          },
        },
      ] : [],
      attrs: {
        line: {
          stroke: '#C87878',
          strokeWidth: 1.5,
          targetMarker: { name: 'classic', size: 6 },
        },
      },
      zIndex: 10,
    });

    // 从驳回节点回到重新开始的步骤（弧线绕右上方）
    const restartPos = stepPositions.get(rn.restartFromStepId);
    if (restartPos) {
      const newProc = processes[rn.newProcessIndex];
      let restartDuration = 0;
      if (newProc && newProc.entries.length > 0) {
        restartDuration = new Date(newProc.entries[0].arrivedAt) - new Date(rejectedEntry.arrivedAt);
      }

      // 用 manhattan 自动避障 + rounded 圆角拐弯
      // 用坐标而非 cell 引用，确保从右侧出发、到达右侧
      const arcSrcX = rPos.x + rPos.width;
      const arcSrcY = rPos.y + rPos.height / 2;
      const arcTgtX = restartPos.x + restartPos.width;
      const arcTgtY = restartPos.y + restartPos.height / 2;
      graph.addEdge({
        source: { x: arcSrcX, y: arcSrcY },
        target: { x: arcTgtX, y: arcTgtY },
        router: {
          name: 'manhattan',
          args: {
            padding: 30 + rnIdx * 15,
            startDirections: ['right'],
            endDirections: ['right'],
          },
        },
        connector: { name: 'rounded', args: { radius: 10 } },
        labels: restartDuration > 0 ? [
          {
            position: 0.5,
            attrs: {
              label: {
                text: `Process ${rn.newProcessIndex + 1}: ${formatDuration(restartDuration)}`,
                fill: '#C87878',
                fontSize: 11,
              },
              rect: { fill: '#fff', stroke: 'none', rx: 2, ry: 2, refWidth: 8, refHeight: 4, refX: -4, refY: -2 },
            },
          },
        ] : [],
        attrs: {
          line: {
            stroke: '#C87878',
            strokeWidth: 1.5,
            strokeDasharray: '6,3',
            targetMarker: { name: 'classic', size: 6 },
          },
        },
        zIndex: 10,
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
