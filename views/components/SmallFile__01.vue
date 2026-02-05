<template>
  <div class="pipeline-item" :class="{'cursor-pointer': pipelineItem?.status}">
    <!-- <div :class="['pipeline-item-cir', 'pipeline-item-cir-before']" v-if="!pipelineItem.isFirst"><span></span></div> -->
    <Icon v-if="pipelineItem.status" class="pipeline-item-icon" :name="STATUS_MAP[pipelineItem.status.toLowerCase()]?.icon" />
    <span class="pipeline-item-name" :title="pipelineItem.name">{{ pipelineItem.name }}</span>
    <span class="pipeline-item-time">{{ pipelineItem.time }}</span>
    <!-- <div :class="['pipeline-item-cir', 'pipeline-item-cir-after']" v-if="!pipelineItem.isLast"><span></span></div> -->
  </div>
</template>
<script setup lang="ts" name="pipelineItem">
import { inject, onMounted, ref } from 'vue';
import { STATUS_MAP } from '@/constant/pipeline';
import Icon from '@/components/Icon/index.vue';
// x6的自定义组件中不能获取到router
interface PipelineItem {
  id: string
  status: string,
  name: string,
  time: string,
  isFirst: boolean,
  isLast: boolean,
}
const getNode = inject('getNode');
// @ts-ignore 工具类的方法直接内置了
const node = getNode();
const currentData = node.store.data.data;
const pipelineItem = ref<PipelineItem>(currentData);

// 数据变化监听
onMounted(() => {
  // previous current
  node.on('change:data', (data: any) => {
    pipelineItem.value = data.current;
  });
});

</script>

<style lang="scss" scoped>
$pipeline-item-width: 310px;
$pipeline-item-height: 54px;
$pipeline-item-cir-width: 20px;
$pipeline-item-cir-height: 20px;
$pipeline-item-cir-span-width: 18px;
$pipeline-item-cir-span-height: 18px;
.pipeline-item {
  width: $pipeline-item-width;
  height: $pipeline-item-height;
  display: flex;
  align-items: center;
  border: 1px solid var(--devui-line, #E3E3EE);
  border-radius: 4px;
  padding-left: 16px;
  position: relative;
  background-color: var(--devui-base-bg, #ffffff);
  .pipeline-item-icon{
    margin-right: 8px;
  }
  .pipeline-item-name{
    flex: 1;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  .pipeline-item-time{
    margin-left: auto;
    margin-right: 16px;
  }
  .pipeline-item-cir{
    width: $pipeline-item-cir-width;
    height: $pipeline-item-cir-height;
    display: flex;
    box-sizing: border-box;
    border: 1px solid var(--devui-line, #E3E3EE);
    background-color: var(--devui-base-bg, #ffffff);
    border-radius: 50%;
    position: absolute;
    top: 50%;
    transform: translateY(-50%);
    &.pipeline-item-cir-before {
      left: -10px;
    }
    &.pipeline-item-cir-after{
      right: -10px;
    }
    > span{
      width: $pipeline-item-cir-span-width;
      height: $pipeline-item-cir-span-height;
      margin: auto;
      border: 5px solid var(--devui-base-bg, #ffffff);
      background-color: var(--devui-disabled-text, #BCBCD0);
      border-radius: 50%;
    }
  }
}
</style>
