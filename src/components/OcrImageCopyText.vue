<template>
  <div class="image-container"
       :style="{
         backgroundImage: `url(${imagePath})`,
         width: imageWidth + 'px',
         height: imageHeight + 'px'
       }"
       @mousedown="startSelection"
       @mousemove="updateSelection"
       @mouseup="endSelection"
       ref="imageContainer">
    <img :src="imagePath" @load="setImageDimensions" style="display: none;" />

    <div class="selection" v-if="isSelecting" :style="selectionStyle"></div>
    <div class="text-overlay" v-for="(item, index) in ocrData" :key="index" :style="getTextStyle(item.poly)">
      {{ item.text }}
    </div>
  </div>
</template>

<script>
import { ref } from 'vue';

export default {
  props: {
    imagePath: {
      type: String,
      required: true,
    },
    ocrData: {
      type: Array,
      required: true,
    }
  },
  setup(props) {
    const isSelecting = ref(false);
    const selectionStart = ref({x: 0, y: 0});
    const selectionEnd = ref({x: 0, y: 0});
    const selectionStyle = ref({});

    // 用于存储图片的宽度和高度
    const imageWidth = ref(0);
    const imageHeight = ref(0);

    const startSelection = (event) => {
      isSelecting.value = true;
      selectionStart.value = {
        x: event.offsetX,
        y: event.offsetY,
      };
      selectionEnd.value = {...selectionStart.value}; // 初始化结束点
    };

    const updateSelection = (event) => {
      if (!isSelecting.value) return;
      selectionEnd.value = {
        x: event.offsetX,
        y: event.offsetY,
      };
      updateSelectionStyle();
    };

    const endSelection = () => {
      if (!isSelecting.value) return;
      isSelecting.value = false;
      copyTextWithinSelection();
    };

    const updateSelectionStyle = () => {
      const left = Math.min(selectionStart.value.x, selectionEnd.value.x);
      const top = Math.min(selectionStart.value.y, selectionEnd.value.y);
      const width = Math.abs(selectionStart.value.x - selectionEnd.value.x);
      const height = Math.abs(selectionStart.value.y - selectionEnd.value.y);

      selectionStyle.value = {
        position: 'absolute',
        left: `${left}px`,
        top: `${top}px`,
        width: `${width}px`,
        height: `${height}px`,
        border: '1px dashed rgba(0, 0, 255, 0.5)', // 选中区域的样式
        pointerEvents: 'none', // 禁止事件穿透
      };
    };

    const copyTextWithinSelection = () => {
      const selectedText = props.ocrData.filter(item => {
        const poly = item.poly;
        const itemX = poly[0][0];
        const itemY = poly[0][1];

        return (
            itemX >= Math.min(selectionStart.value.x, selectionEnd.value.x) &&
            itemX <= Math.max(selectionStart.value.x, selectionEnd.value.x) &&
            itemY >= Math.min(selectionStart.value.y, selectionEnd.value.y) &&
            itemY <= Math.max(selectionStart.value.y, selectionEnd.value.y)
        );
      }).map(item => item.text).join('');

      if (selectedText) {
        navigator.clipboard.writeText(selectedText).then(() => {
          console.log('Copied to clipboard: ', selectedText);
        }).catch(err => {
          console.error('Could not copy text: ', err);
        });
      }
    };

    const setImageDimensions = (event) => {
      const img = event.target;
      imageWidth.value = img.naturalWidth; // 获取图片的自然宽度
      imageHeight.value = img.naturalHeight; // 获取图片的自然高度
    };

    const getTextStyle = (poly) => {
      const width = poly[1][0] - poly[0][0];
      const height = poly[2][1] - poly[0][1];
      const left = poly[0][0] + 'px';
      const top = poly[0][1] + 'px';

      return {
        position: 'absolute',
        left: left,
        top: top,
        color: 'transparent', // 设置文本颜色为透明
      };
    };
    const imagePath = props.imagePath;
    const ocrData = props.ocrData;

    return {
      imagePath,
      ocrData,
      isSelecting,
      selectionStyle,
      startSelection,
      updateSelection,
      endSelection,
      getTextStyle,
      imageWidth,
      imageHeight,
      setImageDimensions,
    };
  },
};
</script>

<style>
.image-container {
  position: relative;
  display: inline-block;
}

.selection {
  background-color: rgba(0, 0, 255, 0.2); /* 半透明蓝色 */
  position: absolute;
}

.text-overlay {
  pointer-events: none; /* 防止文本遮挡响应 */
}
</style>
