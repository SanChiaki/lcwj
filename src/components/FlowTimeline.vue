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
  startX: 150,
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

  // 1. 计算坐标 - 按日期分配X坐标，同日期节点中心对齐，保证间隙
  const nodePositions = new Map();

  // 收集所有唯一日期并排序
  const allDates = [...new Set(nodes.map(n => n.date).filter(d => d))].sort();

  // 为每个日期分配X坐标
  const dateToX = new Map();
  const DATE_GAP = 130; // 日期之间的间距
  allDates.forEach((date, index) => {
    dateToX.set(date, CONFIG.startX + index * DATE_GAP);
  });

  // 按层处理节点
  layers.forEach((layer, layerIdx) => {
    const centerY = CONFIG.startY + layerIdx * CONFIG.rowHeight + CONFIG.rowHeight / 2;

    // 按日期分组
    const dateGroups = new Map();
    nodes.filter(n => n.layerId === layer.id).forEach(node => {
      const date = node.date || '';
      if (!dateGroups.has(date)) dateGroups.set(date, []);
      dateGroups.get(date).push(node);
    });

    // 处理每个日期组
    dateGroups.forEach((dateNodes, date) => {
      const baseX = dateToX.get(date) || CONFIG.startX;
      const actualWidth = dateNodes[0].styleType === 'plan' ? 120 : 80;

      // 同层同日期节点水平偏移避免重叠
      dateNodes.forEach((node, idx) => {
        const offsetX = (idx - (dateNodes.length - 1) / 2) * (actualWidth + 30);
        nodePositions.set(node.id, { x: baseX + offsetX, y: centerY, width: actualWidth });
      });
    });
  });

  let maxRightX = CONFIG.startX + 800;
  nodePositions.forEach(pos => { if (pos.x + pos.width / 2 > maxRightX) maxRightX = pos.x + pos.width / 2; });
  const lineTotalWidth = maxRightX - (CONFIG.startX - 100) + 100;

  // 2. 渲染背景线和时间轴线 (使用 Node 替代 Edge 以防不显示)
  const isFirstLayer = (idx) => idx === 0;
  const isLastLayer = (idx) => idx === layers.length - 1;
  const separatorNodes = []; // 收集分隔线节点用于排除

  layers.forEach((layer, index) => {
    const topY = CONFIG.startY + index * CONFIG.rowHeight;
    const bottomY = topY + CONFIG.rowHeight;
    const centerY = topY + CONFIG.rowHeight / 2;
    const isTimeline = layer.type === 'timeline';

    // 绘制顶部边界/分隔线 (跳过最顶层)
    if (!isFirstLayer(index)) {
      if (!isTimeline || (layers[index-1] && layers[index-1].type !== 'timeline')) {
        const sepNode = graph.addNode({
          x: CONFIG.startX - 100, y: topY, width: lineTotalWidth, height: 1,
          shape: 'rect', attrs: { body: { fill: '#000', stroke: 'none' } }, zIndex: 1,
          data: { isSeparator: true }
        });
        separatorNodes.push(sepNode);
      }
    }

    // 绘制底部边界线 (跳过最底层)
    if (!isLastLayer(index)) {
      const sepNode = graph.addNode({
        x: CONFIG.startX - 100, y: bottomY, width: lineTotalWidth, height: 1,
        shape: 'rect', attrs: { body: { fill: '#000', stroke: 'none' } }, zIndex: 1,
        data: { isSeparator: true }
      });
      separatorNodes.push(sepNode);
    }

    // 绘制时间轴 (顶层蓝色, 底层紫色)
    const layerColor = isLastLayer(index) ? '#9f45fc' : '#4A86E8';

    if (isTimeline) {
      graph.addEdge({
        source: { x: CONFIG.startX - 100, y: centerY },
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
        attrs: { body: { fill: 'none', stroke: 'none' }, text: { textAlign: 'left', fontSize: 14, fill: '#333' } },
        zIndex: 10
      });
    }
  });

  // 3. 渲染业务节点
  nodes.forEach(node => {
    const pos = nodePositions.get(node.id);
    if (!pos) return;

    const layer = layers.find(l => l.id === node.layerId);
    const isTimelineNode = layer && layer.type === 'timeline';

    if (isTimelineNode) {
      const layerIndex = layers.findIndex(l => l.id === layer.id);
      const isBottomTimeline = isLastLayer(layerIndex);
      const nodeColor = isBottomTimeline ? '#9f45fc' : '#4A86E8';

      // 1. 圆点
      graph.addNode({
        x: pos.x - 6, y: pos.y - 6, width: 12, height: 12,
        shape: 'circle',
        attrs: { body: { fill: nodeColor, stroke: '#fff', strokeWidth: 2 } },
        zIndex: 5
      });
      // 2. 日期
      if (node.date) {
        graph.addNode({
          x: pos.x - 60, y: pos.y - 45, width: 120, height: 30,
          shape: 'text-block', text: node.date,
          attrs: { body: { fill: 'none', stroke: 'none' }, text: { fontSize: 13, textAlign: 'center', fill: '#333', fontWeight: 'bold' } },
          zIndex: 5
        });
      }
      // 3. 业务方框 (ID 用于连线) - 圆角矩形, 边框固定颜色 #e3e4e6
      graph.addNode({
        id: node.id, x: pos.x - 60, y: pos.y + 15, width: 120, height: 40,
        label: node.label,
        attrs: { body: { stroke: '#e3e4e6', strokeWidth: 2, fill: '#fff', rx: 6, ry: 6 }, label: { fill: '#333', fontSize: 13 } },
        zIndex: 5
      });

    } else {
      // 泳道内节点: 两行显示 (日期 + 描述)
      // 使用text-block形状,通过样式实现两行效果
      const dateText = node.date || '';
      const labelText = node.label || '';

      graph.addNode({
        id: node.id,
        x: pos.x - 40,
        y: pos.y - 22,
        width: 80,
        height: 44,
        shape: 'rect',
        attrs: {
          body: {
            fill: '#D3D3D3',
            stroke: 'none',
            rx: 6,
            ry: 6
          },
          label: {
            text: dateText,
            fill: '#666',
            fontSize: 9,
            refY: 12
          },
          label2: {
            text: labelText,
            fill: '#333',
            fontSize: 11,
            fontWeight: 'bold',
            refY: 28
          }
        },
        markup: [
          { tagName: 'rect', selector: 'body' },
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
      connector: { name: 'smooth', args: { radius: 8 } },
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
