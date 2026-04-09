<template>
  <div class="flow-container">
    <div ref="container" class="x6-graph"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue';
import { Graph } from '@antv/x6';

const container = ref(null);

const CONFIG = {
  columnWidth: 200,
  rowHeight: 120, 
  startX: 220,
  startY: 60,
  minGap: 60,
};

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

const renderFlow = (graph, data) => {
  const { layers, nodes, links } = data;

  // 辅助函数：解析日期字符串
  const parseDate = (dateStr) => {
    if (!dateStr) return null;
    return dateStr;
  };

  // 1. 计算坐标 - 按日期分配X坐标，同日期节点垂直堆叠，泳道高度动态计算
  const nodePositions = new Map();

  // 收集所有唯一日期并排序
  const allDates = [...new Set(nodes.map(n => n.date).filter(d => d))].sort();

  // 为每个日期分配X坐标
  const dateToX = new Map();
  const DATE_GAP = 160; // 日期之间的间距
  allDates.forEach((date, index) => {
    dateToX.set(date, CONFIG.startX + index * DATE_GAP);
  });

  // 计算每层的高度需求（基于该层最大堆叠节点数）
  const layerHeights = [];
  const layerDateGroups = [];

  layers.forEach((layer, layerIdx) => {
    // 按日期分组
    const dateGroups = new Map();
    const layerNodes = nodes.filter(n => n.layerId === layer.id);

    layerNodes.forEach(node => {
      const date = node.date || '';
      if (!dateGroups.has(date)) dateGroups.set(date, []);
      dateGroups.get(date).push(node);
    });

    layerDateGroups.push(dateGroups);

    // 计算该层最大堆叠高度和最大节点宽度
    let maxStackHeight = 1;
    let maxNodeWidth = 80;
    dateGroups.forEach((dateNodes) => {
      const nodeHeight = 60;
      const stackHeight = dateNodes.length * nodeHeight + (dateNodes.length - 1) * 10;
      maxStackHeight = Math.max(maxStackHeight, stackHeight);

      // 计算该日期组的最大节点宽度
      dateNodes.forEach(node => {
        if (node.styleType !== 'plan') {
          const MAX_NODE_WIDTH = 160;
          const calcTextWidth = (text, fontSize) => {
            let width = 0;
            for (const char of (text || '')) {
              width += (char.charCodeAt(0) > 127) ? fontSize : fontSize * 0.6;
            }
            return width;
          };
          const dateWidth = calcTextWidth(node.date, 9);
          const labelWidth = calcTextWidth(node.label, 11);
          const nodeWidth = Math.min(MAX_NODE_WIDTH, Math.max(80, Math.max(dateWidth, labelWidth) + 16));
          maxNodeWidth = Math.max(maxNodeWidth, nodeWidth);
        } else {
          maxNodeWidth = Math.max(maxNodeWidth, 120);
        }
      });
    });

    // 计算层高度
    // lane层: 对称分布，上下都需要空间
    // timeline层: 节点从中心向下堆叠，只需要下方空间
    const isTimeline = layer.type === 'timeline';
    const layerHeight = isTimeline
      ? Math.max(CONFIG.rowHeight, maxStackHeight + 80) // 80=圆点上方空间(45) + 圆点到第一个节点(35)
      : Math.max(CONFIG.rowHeight, maxStackHeight + 40); // 40=上下边距各20
    layerHeights.push(layerHeight);
  });

  // 计算每层的累积Y位置
  const layerYPositions = [];
  let currentY = CONFIG.startY;
  layerHeights.forEach((height, idx) => {
    const isTimeline = layers[idx].type === 'timeline';
    // timeline层: 圆点在层顶部向下45px处(给日期标签留空间)
    // lane层: 中心在层中间
    const centerY = isTimeline ? currentY + 45 : currentY + height / 2;
    layerYPositions.push({
      topY: currentY,
      centerY: centerY,
      height: height
    });
    currentY += height;
  });

  // 放置节点
  layers.forEach((layer, layerIdx) => {
    const { centerY } = layerYPositions[layerIdx];
    const dateGroups = layerDateGroups[layerIdx];

    // 处理每个日期组
    const isTimeline = layer.type === 'timeline';
    dateGroups.forEach((dateNodes, date) => {
      const baseX = dateToX.get(date) || CONFIG.startX;
      const actualHeight = 60;

      // 同层同日期节点垂直堆叠
      dateNodes.forEach((node, idx) => {
        // 根据文字长度计算节点宽度
        let actualWidth;
        if (node.styleType === 'plan') {
          actualWidth = 120; // plan节点固定宽度
        } else {
          // lane节点根据文字计算宽度（限制最大宽度，超出换行）
          const MAX_NODE_WIDTH = 160; // 最大宽度限制
          const calcTextWidth = (text, fontSize) => {
            let width = 0;
            for (const char of (text || '')) {
              width += (char.charCodeAt(0) > 127) ? fontSize : fontSize * 0.6;
            }
            return width;
          };
          const dateWidth = calcTextWidth(node.date, 9);
          const labelWidth = calcTextWidth(node.label, 11);
          actualWidth = Math.min(MAX_NODE_WIDTH, Math.max(80, Math.max(dateWidth, labelWidth) + 16));
        }

        let offsetY;
        if (isTimeline) {
          // timeline层: 从圆点(即centerY)下方开始向下堆叠
          // 圆点半径6，再留10px间距，第一个节点中心在圆点下方46px
          offsetY = idx * (actualHeight + 10) + 46;
        } else {
          // lane层: 以中心对称分布
          offsetY = (idx - (dateNodes.length - 1) / 2) * (actualHeight + 10);
        }
        nodePositions.set(node.id, { x: baseX, y: centerY + offsetY, width: actualWidth });
      });
    });
  });

  let maxRightX = CONFIG.startX + 800;
  nodePositions.forEach(pos => { if (pos.x + pos.width / 2 > maxRightX) maxRightX = pos.x + pos.width / 2; });
  const lineStartX = 80;
  const lineTotalWidth = maxRightX - lineStartX + 100;

  // 2. 渲染背景线和时间轴线 (使用 Node 替代 Edge 以防不显示)
  const isFirstLayer = (idx) => idx === 0;
  const isLastLayer = (idx) => idx === layers.length - 1;
  const separatorNodes = []; // 收集分隔线节点用于排除

  layers.forEach((layer, index) => {
    const { topY, centerY, height: rowHeight } = layerYPositions[index];
    const bottomY = topY + rowHeight;
    const isTimeline = layer.type === 'timeline';

    // 绘制顶部边界/分隔线 (跳过最顶层)
    if (!isFirstLayer(index)) {
      if (!isTimeline || (layers[index-1] && layers[index-1].type !== 'timeline')) {
        graph.addEdge({
          source: { x: lineStartX, y: topY },
          target: { x: lineStartX + lineTotalWidth, y: topY },
          attrs: { line: { stroke: '#d0d0d0', strokeWidth: 1, strokeDasharray: '5,5', targetMarker: null } },
          zIndex: 1,
          data: { isSeparator: true }
        });
      }
    }

    // 绘制时间轴 (顶层蓝色, 底层紫色)
    const layerColor = isLastLayer(index) ? '#9f45fc' : '#4A86E8';

    if (isTimeline) {
      graph.addEdge({
        source: { x: 80, y: centerY },
        target: { x: maxRightX + 100, y: centerY },
        attrs: { line: { stroke: layerColor, strokeWidth: 2, targetMarker: 'classic' } },
        zIndex: 1
      });
    }

    // 左侧垂直标签
    if (isTimeline) {
      graph.addNode({
        x: 30, y: centerY - 40, width: 30, height: 80,
        shape: 'rect', label: layer.label,
        attrs: { body: { fill: layerColor, stroke: 'none' }, label: { fill: '#fff', textWrap: { width: 20 }, fontSize: 13 } },
        zIndex: 10
      });
    } else {
      graph.addNode({
        x: 30, y: centerY - 15, width: 100, height: 30,
        shape: 'text-block', text: layer.label,
        attrs: { body: { fill: 'none', stroke: 'none' }, text: { textAlign: 'left', fontSize: 14, fill: '#999' } },
        zIndex: 10
      });
    }
  });

  // 3. 渲染业务节点
  // 先按 timeline 层和日期分组，用于渲染圆点和日期（每个日期只渲染一次）
  const timelineDateMarkers = new Map();

  nodes.forEach(node => {
    const layer = layers.find(l => l.id === node.layerId);
    if (layer && layer.type === 'timeline' && node.date) {
      const key = `${layer.id}-${node.date}`;
      if (!timelineDateMarkers.has(key)) {
        const layerIndex = layers.findIndex(l => l.id === layer.id);
        const isBottomTimeline = isLastLayer(layerIndex);
        const nodeColor = isBottomTimeline ? '#9f45fc' : '#4A86E8';

        const baseX = dateToX.get(node.date) || CONFIG.startX;
        const centerY = layerYPositions[layerIndex].centerY;

        // 1. 圆点（渲染在坐标轴上，每个日期只一个）
        graph.addNode({
          x: baseX - 6, y: centerY - 6, width: 12, height: 12,
          shape: 'circle',
          attrs: { body: { fill: nodeColor, stroke: '#fff', strokeWidth: 2 } },
          zIndex: 5
        });
        // 2. 日期标签（渲染在圆点上方）
        graph.addNode({
          x: baseX - 60, y: centerY - 45, width: 120, height: 30,
          shape: 'text-block', text: node.date,
          attrs: { body: { fill: 'none', stroke: 'none' }, text: { fontSize: 13, textAlign: 'center', fill: '#333', fontWeight: 'bold' } },
          zIndex: 5
        });

        timelineDateMarkers.set(key, true);
      }
    }
  });

  // 渲染所有节点（包括 timeline 和 lane）
  nodes.forEach(node => {
    const pos = nodePositions.get(node.id);
    if (!pos) return;

    const layer = layers.find(l => l.id === node.layerId);
    const isTimelineNode = layer && layer.type === 'timeline';

    if (isTimelineNode) {
      // 里程碑节点: 3行 - 单号 + 标题 + 物料
      const orderNoText = node.orderNo || '';
      const labelText = node.label || '';
      const materialText = node.material || '';
      graph.addNode({
        id: node.id, x: pos.x - 60, y: pos.y - 30, width: 120, height: 60,
        attrs: {
          body: { stroke: '#e3e4e6', strokeWidth: 2, fill: '#fff', rx: 6, ry: 6 },
          orderNo: { text: orderNoText, fill: '#999', fontSize: 10, refY: 12, textWrap: { width: 108, ellipsis: true } },
          label: { text: labelText, fill: '#333', fontSize: 13, fontWeight: 'bold', refY: 30, textWrap: { width: 108, ellipsis: true } },
          material: { text: materialText, fill: '#888', fontSize: 10, refY: 48, textWrap: { width: 108, ellipsis: true } },
        },
        markup: [
          { tagName: 'rect', selector: 'body' },
          { tagName: 'text', selector: 'orderNo' },
          { tagName: 'text', selector: 'label' },
          { tagName: 'text', selector: 'material' },
        ],
        zIndex: 5
      });

    } else {
      // 泳道内节点: 3行显示 (单号 + 日期 + 标签)
      const orderNoText = node.orderNo || '';
      const dateText = node.date || '';
      const labelText = node.label || '';

      // 根据文字长度计算节点宽度
      const calcTextWidth = (text, fontSize) => {
        let width = 0;
        for (const char of text) {
          width += (char.charCodeAt(0) > 127) ? fontSize : fontSize * 0.6;
        }
        return width;
      };

      const MAX_NODE_WIDTH = 120;
      const orderNoWidth = calcTextWidth(orderNoText, 9);
      const dateWidth = calcTextWidth(dateText, 9);
      const labelWidth = calcTextWidth(labelText, 11);
      const contentWidth = Math.max(orderNoWidth, dateWidth, labelWidth);
      const nodeWidth = Math.min(MAX_NODE_WIDTH, Math.max(80, contentWidth + 16));

      graph.addNode({
        id: node.id,
        x: pos.x - nodeWidth / 2,
        y: pos.y - 30,
        width: nodeWidth,
        height: 60,
        shape: 'rect',
        attrs: {
          body: { fill: '#D3D3D3', stroke: 'none', rx: 6, ry: 6 },
          orderNo: {
            text: orderNoText,
            fill: '#888',
            fontSize: 9,
            refY: 12,
            textWrap: { width: nodeWidth - 12, ellipsis: true }
          },
          label: {
            text: dateText,
            fill: '#666',
            fontSize: 9,
            refY: 28,
            textWrap: { width: nodeWidth - 12, ellipsis: true }
          },
          label2: {
            text: labelText,
            fill: '#333',
            fontSize: 11,
            fontWeight: 'bold',
            refY: 45,
            textWrap: { width: nodeWidth - 12, ellipsis: true }
          }
        },
        markup: [
          { tagName: 'rect', selector: 'body' },
          { tagName: 'text', selector: 'orderNo' },
          { tagName: 'text', selector: 'label' },
          { tagName: 'text', selector: 'label2' }
        ],
        zIndex: 5
      });

    }
  });

  // 4. 渲染依赖连线
  // 根据源节点和目标节点所在层决定连接方向
  links.forEach(link => {
    const sourceNode = nodes.find(n => n.id === link.source);
    const targetNode = nodes.find(n => n.id === link.target);
    const sourceLayer = sourceNode?.layerId;
    const targetLayer = targetNode?.layerId;

    // 同层优先左右连接，不同层优先上下连接
    const isSameLayer = sourceLayer === targetLayer;
    const startDirections = isSameLayer ? ['right', 'left'] : ['bottom', 'right'];
    const endDirections = isSameLayer ? ['left', 'right'] : ['top', 'left'];

    graph.addEdge({
      source: { cell: link.source, anchor: { name: 'center' } },
      target: { cell: link.target, anchor: { name: 'center' } },
      router: {
        name: 'metro',
        args: {
          padding: 5,
          excludeNodes: separatorNodes,
          step: 10,
          startDirections,
          endDirections
        }
      },
      connector: { name: 'rounded', args: { radius: 8 } },
      attrs: { line: { stroke: '#4A86E8', strokeWidth: 1.5, targetMarker: { name: 'classic', size: 6 } } },
      zIndex: 2
    });
  });

  graph.centerContent();
};
</script>

<style scoped>
.flow-container { width: 100%; height: 100%; min-height: 800px; background: #fff; }
.x6-graph { width: 100%; height: 100%; }
</style>
