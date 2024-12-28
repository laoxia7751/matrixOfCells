<template>
    <div class="chip-container">
      <div ref="chipImageRef" class="chipImage">
        <canvas ref="canvasRef" id="chipCanvas"></canvas>
      </div>
      <div class="zoom-control" v-if="zoomControl">
        <input 
          type="range" 
          v-model="zoomLevel" 
          :min="minZoom" 
          :max="maxZoom" 
          :step="0.1"
          @input="handleZoomChange"
        />
        <span class="zoom-text">{{ Math.round(zoomLevel * 100) }}%</span>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, onMounted, onUnmounted, nextTick, watch, computed, defineExpose } from 'vue';
  
  const chipImageRef = ref(null);
  const canvasRef = ref(null);
  
  const props = defineProps({
    defectData: {
      type: Object,
      required: true
    },
    zoomControl: {
      type: Boolean,
      default: false
    }
  });
  
  const emit = defineEmits(['click-die'])
  
  const codeColorMap = new Map([
      ['___','#000000'],
      ['000','#2ecc71'],
      ['255','#ff7675'],
      ['256','#d63031'],
      ['@@@','#f9ca24'],
  ])
  
  const chipData = computed(() => props.defectData?.data || []);
  const defectCodes = computed(() => props.defectData?.sampleImgDefect || []);
  const center = computed(() => props.defectData?.center);
  const offset = computed(() => props.defectData?.offset);
  
  let ctx;
  let scale = 1;
  let translateX = 0;
  let translateY = 0;
  let isDragging = false;
  let lastX = 0;
  let lastY = 0;
  let cellSize = 10;
  let animationFrameId = null;
  let isDrawingRequested = false;
  
  const minZoom = 0.1;
  const maxZoom = 10;
  const zoomLevel = ref(1);
  
  // 添加 ResizeObserver 相关变量
  let resizeObserver = null;
  
  // 添加高亮单元格的状态
  let highlightCell = null;
  
  const getDefectColor = (row, col) => {
    // 计算相对于中心点的坐标
    const centerRow = chipData.value.length / 2;
    const centerCol = chipData.value[0].length / 2;
    
    // 计算相对坐标
    let coordX = -(Math.ceil(col - centerCol)); // 反转 X 轴方向
    let coordY = -(Math.ceil(row - centerRow));
    
    // 坐标修正
    if (coordY <= 0) {
      coordY -= 1;
    }
    if (coordX <= 0) { // 修改为处理 X 轴负值
      coordX -= 1;
    }
    
    // 使用相对坐标查找缺陷
    const defect = defectCodes.value.find(d => 
      d.DieCol === -coordX && d.DieRow === coordY // 再次反转 X 轴以匹配数据
    );
    
    if (!defect) return null;
    
    const defectCode = defect.DefectCodes[0];
    return defectCode ? defectCode.ColorMark : '#ff0000';
  };
  
  const fitMatrixToCanvas = () => {
    if (!ctx) return;
    
    const rows = chipData.value.length;
    const cols = chipData.value[0].length;
    const matrixWidth = cols * cellSize;
    const matrixHeight = rows * cellSize;
    
    const canvasWidth = ctx.canvas.width;
    const canvasHeight = ctx.canvas.height;
    
    // 计算宽高比
    const matrixRatio = matrixWidth / matrixHeight;
    const canvasRatio = canvasWidth / canvasHeight;
    
    // 根据宽高比决定缩放基准
    if (matrixRatio > canvasRatio) {
      // 如果矩阵更宽，以宽度为基准
      scale = canvasWidth / matrixWidth;
    } else {
      // 如果矩阵更高，以高度为基准
      scale = canvasHeight / matrixHeight;
    }
    
    // 重置平移位置到中心
    translateX = 0;
    translateY = 0;
    
    // 重新绘制
    drawMatrix();
  };
  
  // 计算可见区域
  const getVisibleArea = (rows, cols) => {
    const canvas = canvasRef.value;
    const viewportWidth = canvas.width;
    const viewportHeight = canvas.height;
    
    const cellSizeScaled = cellSize * scale;
    const viewportCenterX = -translateX / scale;
    const viewportCenterY = -translateY / scale;
    
    const visibleCols = Math.ceil(viewportWidth / cellSizeScaled);
    const visibleRows = Math.ceil(viewportHeight / cellSizeScaled);
    
    const startCol = Math.max(0, Math.floor((cols/2 + viewportCenterX/cellSize - visibleCols/2)));
    const endCol = Math.min(cols, Math.ceil((cols/2 + viewportCenterX/cellSize + visibleCols/2)));
    const startRow = Math.max(0, Math.floor((rows/2 + viewportCenterY/cellSize - visibleRows/2)));
    const endRow = Math.min(rows, Math.ceil((rows/2 + viewportCenterY/cellSize + visibleRows/2)));
    
    return { startRow, endRow, startCol, endCol };
  };
  
  // 添加高亮单元格的方法
  const highlightCenterCell = (coordinates) => {
    if (coordinates) {
      // 保存新的高亮坐标
      highlightCell = {
        x: coordinates[0],
        y: coordinates[1]
      };
    } else {
      // 清除高亮
      highlightCell = null;
    }
    // 重新绘制
    drawMatrix();
  };
  
  // 绘制矩阵
  const drawMatrix = () => {
    if (isDrawingRequested) return;
    
    isDrawingRequested = true;
    animationFrameId = requestAnimationFrame(() => {
      if (!ctx) return;
      
      ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
      
      // 设置白色背景
      ctx.fillStyle = '#FFFFFF';
      ctx.fillRect(0, 0, ctx.canvas.width, ctx.canvas.height);
      
      ctx.save();
      
      ctx.translate(ctx.canvas.width / 2 + translateX, ctx.canvas.height / 2 + translateY);
      ctx.scale(scale, scale);
      
      const rows = chipData.value.length;
      const cols = chipData.value[0].length;
      
      const visibleArea = getVisibleArea(rows, cols);
      
      // 绘制矩阵单元格
      for (let row = visibleArea.startRow; row < visibleArea.endRow; row++) {
        for (let col = visibleArea.startCol; col < visibleArea.endCol; col++) {
          const cellValue = chipData.value[row][col];
          const x = (col - cols/2) * cellSize;
          const y = (row - rows/2) * cellSize;
          
          // 计算当前单元格的坐标系坐标
          let coordX = -(Math.ceil(col - cols/2));
          let coordY = -(Math.ceil(row - rows/2));
          
          if (coordY <= 0) coordY -= 1;
          if (coordX <= 0) coordX -= 1;
          
          // 检查是否是高亮单元格
          const isHighlighted = highlightCell && 
            highlightCell.x === -coordX && 
            highlightCell.y === coordY;
          
          if (isHighlighted) {
            // 绘制白色高亮
            ctx.fillStyle = '#FFFFFF';
          } else {
            // 检查是否存在缺陷
            const defectColor = getDefectColor(row, col);
            // 使用正常颜色
            ctx.fillStyle = defectColor || codeColorMap.get(cellValue) || '#000000';
          }
          
          ctx.fillRect(x, y, cellSize, cellSize);
          ctx.strokeStyle = '#333';
          ctx.strokeRect(x, y, cellSize, cellSize);
        }
      }
      
      // 绘制坐标轴
      const axisLength = Math.max(rows, cols) * cellSize; // X轴长度
      const yAxisLength = axisLength * 1.2; // Y轴长度改为矩阵高度的1.2倍
      
      // 设置坐标轴样式
      ctx.strokeStyle = '#FF0000';
      ctx.lineWidth = 2 / scale;
      
      // 绘制 X 轴
      ctx.beginPath();
      ctx.moveTo(-axisLength/2, 0);
      ctx.lineTo(axisLength/2, 0);
      ctx.stroke();
      
      // 绘制 X 轴箭头
      ctx.beginPath();
      const arrowSize = 10 / scale;
      ctx.moveTo(axisLength/2 - arrowSize, -arrowSize/2);
      ctx.lineTo(axisLength/2, 0);
      ctx.lineTo(axisLength/2 - arrowSize, arrowSize/2);
      ctx.stroke();
      
      // 绘制 Y 轴
      ctx.beginPath();
      ctx.moveTo(0, yAxisLength/2);
      ctx.lineTo(0, -yAxisLength/2);
      ctx.stroke();
      
      // 绘制 Y 轴箭头
      ctx.beginPath();
      ctx.moveTo(-arrowSize/2, -yAxisLength/2 + arrowSize);
      ctx.lineTo(0, -yAxisLength/2);
      ctx.lineTo(arrowSize/2, -yAxisLength/2 + arrowSize);
      ctx.stroke();
      
      ctx.restore();
      isDrawingRequested = false;
    });
  };
  
  // 初始化画布
  const initCanvas = () => {
    const canvas = canvasRef.value;
    ctx = canvas.getContext('2d');
    
    const resizeCanvas = () => {
      canvas.width = chipImageRef.value.clientWidth;
      canvas.height = chipImageRef.value.clientHeight;
      // 只在有数据时执行 fitMatrixToCanvas
      if (props.defectData?.data) {
        fitMatrixToCanvas();
      }
    };
    
    // 设置 ResizeObserver
    resizeObserver = new ResizeObserver((entries) => {
      for (const entry of entries) {
        if (entry.target === chipImageRef.value) {
          resizeCanvas();
          break;
        }
      }
    });
    
    // 开始观察元素尺寸变化
    resizeObserver.observe(chipImageRef.value);
    
    // 初始化画布尺寸
    resizeCanvas();
    
    canvas.addEventListener('wheel', handleWheel);
    canvas.addEventListener('mousedown', handleMouseDown);
    canvas.addEventListener('mousemove', handleMouseMove);
    canvas.addEventListener('mouseup', handleMouseUp);
    canvas.addEventListener('click', handleClick);
  };
  
  // 事件处理函数
  const handleWheel = (e) => {
    e.preventDefault();
    const delta = e.deltaY > 0 ? 0.9 : 1.1;
    scale *= delta;
    scale = Math.max(minZoom, Math.min(scale, maxZoom));
    zoomLevel.value = scale; // 更新滑块值
    drawMatrix();
  };
  
  const handleMouseDown = (e) => {
    isDragging = true;
    lastX = e.clientX;
    lastY = e.clientY;
  };
  
  const handleMouseMove = (e) => {
    if (!isDragging) return;
    
    const deltaX = e.clientX - lastX;
    const deltaY = e.clientY - lastY;
    
    translateX += deltaX;
    translateY += deltaY;
    
    lastX = e.clientX;
    lastY = e.clientY;
    
    drawMatrix();
  };
  
  const handleMouseUp = () => {
    isDragging = false;
  };
  console.log(defectCodes.value)
  const handleClick = (e) => {
    const rect = canvasRef.value.getBoundingClientRect();
    const x = e.clientX - rect.left;
    const y = e.clientY - rect.top;
    
    const transformedX = (x - ctx.canvas.width / 2 - translateX) / scale;
    const transformedY = (y - ctx.canvas.height / 2 - translateY) / scale;
    
    const col = Math.floor(transformedX / cellSize + chipData.value[0].length/2);
    const row = Math.floor(transformedY / cellSize + chipData.value.length/2);
    
    if (row >= 0 && row < chipData.value.length && col >= 0 && col < chipData.value[0].length) {
      // 修正坐标系计算
      let coordX = Math.ceil(col - chipData.value[0].length/2);
      let coordY = -(Math.ceil(row - chipData.value.length/2)); // 反转 Y 轴方向
      
      // 对 Y 轴负值进行修正
      if (coordY <= 0) {
        coordY -= 1;
      }
      if(coordX >= 0){
        coordX += 1;
      }
      
      console.log(`单元格:{row:${row},col:${col}},坐标轴:[${coordX},${coordY}],值: ${chipData.value[row][col]}`);
      
      // 添加缺陷信息的输出
      const defect = defectCodes.value.find(d => d.DieCol === coordX && d.DieRow === coordY);
      if (defect && defect.DefectCodes.length > 0) {
        console.log('缺陷信息:', defect.DefectCodes[0]);
        emit('click-die', defect);
      }
    }
  };
  
  // 添加滑块变化处理方法
  const handleZoomChange = () => {
    scale = zoomLevel.value;
    drawMatrix();
  };
  
  // 监听 defectData
  watch(
    () => props.defectData,
    (newData) => {
      if (newData?.data) {
        nextTick(() => {
          fitMatrixToCanvas();
        });
      }
    },
    { immediate: true, deep: true }
  );
  
  // 生命周期钩子
  onMounted(() => {
    initCanvas();
  });
  
  onUnmounted(() => {
    if (canvasRef.value) {
      const canvas = canvasRef.value;
      canvas.removeEventListener('wheel', handleWheel);
      canvas.removeEventListener('mousedown', handleMouseDown);
      canvas.removeEventListener('mousemove', handleMouseMove);
      canvas.removeEventListener('mouseup', handleMouseUp);
      canvas.removeEventListener('click', handleClick);
    }
    
    // 清理 ResizeObserver
    if (resizeObserver) {
      resizeObserver.disconnect();
      resizeObserver = null;
    }
    
    // 取消可能正在进行的动画帧
    if (animationFrameId) {
      cancelAnimationFrame(animationFrameId);
    }
  });
  
  // 暴露方法
  defineExpose({
    highlightCenterCell
  });
  </script>
  
  <style lang="scss">
  .chip-container {
    display: flex;
    flex-direction: column;
    gap: 10px;
    height: 100%;
  }
  
  .chipImage {
    position: relative;
    width: 100%;
    flex: 1;
    border: 1px dashed #ccc;
    cursor: pointer;
  
    #chipCanvas {
      display: block;
      width: 100%;
      height: 100%;
      background-color: #FFFFFF;
    }
  }
  
  .zoom-control {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 0 10px;
    
    input[type="range"] {
      flex: 1;
      min-width: 100px;
    }
    
    .zoom-text {
      min-width: 60px;
      text-align: right;
      font-size: 14px;
      color: #666;
    }
  }
  </style>
  