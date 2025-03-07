<script setup lang="ts">
import DeResourceTree from '@/views/common/DeResourceTree.vue'
import { dvMainStoreWithOut } from '@/store/modules/data-visualization/dvMain'
import { reactive, nextTick, ref, toRefs, onBeforeMount, computed } from 'vue'
import DePreview from '@/components/data-visualization/canvas/DePreview.vue'
import PreviewHead from '@/views/data-visualization/PreviewHead.vue'
import EmptyBackground from '@/components/empty-background/src/EmptyBackground.vue'
import { initCanvasData, initCanvasDataPrepare } from '@/utils/canvasUtils'
import { useAppStoreWithOut } from '@/store/modules/app'
import { useRequestStoreWithOut } from '@/store/modules/request'
import { usePermissionStoreWithOut } from '@/store/modules/permission'
import { useMoveLine } from '@/hooks/web/useMoveLine'
import { Icon } from '@/components/icon-custom'
import { download2AppTemplate, downloadCanvas } from '@/utils/imgUtils'

const dvMainStore = dvMainStoreWithOut()
const previewCanvasContainer = ref(null)
const dashboardPreview = ref(null)
const slideShow = ref(true)
const requestStore = useRequestStoreWithOut()
const permissionStore = usePermissionStoreWithOut()
const appStore = useAppStoreWithOut()
const dataInitState = ref(true)
const downloadStatus = ref(false)
const state = reactive({
  canvasDataPreview: null,
  canvasStylePreview: null,
  canvasViewInfoPreview: null,
  dvInfo: null,
  curPreviewGap: 0
})

const { width, node } = useMoveLine('DASHBOARD')

const props = defineProps({
  showPosition: {
    required: false,
    type: String,
    default: 'preview'
  }
})

const { showPosition } = toRefs(props)

const resourceTreeRef = ref()

const hasTreeData = computed(() => {
  return resourceTreeRef.value?.hasData
})
const isDataEaseBi = computed(() => appStore.getIsDataEaseBi)

const rootManage = computed(() => {
  return resourceTreeRef.value?.rootManage
})
const mounted = computed(() => {
  return resourceTreeRef.value?.mounted
})

function createNew() {
  resourceTreeRef.value?.createNewObject()
}

const loadCanvasData = (dvId, weight?) => {
  // 复用不设置 dvMain 中的componentData 等画布信息
  const initMethod = showPosition.value === 'multiplexing' ? initCanvasDataPrepare : initCanvasData
  dataInitState.value = false
  initMethod(
    dvId,
    'dashboard',
    function ({
      canvasDataResult,
      canvasStyleResult,
      dvInfo,
      canvasViewInfoPreview,
      curPreviewGap
    }) {
      dvInfo['weight'] = weight
      state.canvasDataPreview = canvasDataResult
      state.canvasStylePreview = canvasStyleResult
      state.canvasViewInfoPreview = canvasViewInfoPreview
      state.dvInfo = dvInfo
      state.curPreviewGap = curPreviewGap
      dataInitState.value = true
      nextTick(() => {
        dashboardPreview.value.restore()
      })
    }
  )
}

const downloadH2 = type => {
  downloadStatus.value = true
  nextTick(() => {
    const vueDom = previewCanvasContainer.value.querySelector('.canvas-container')
    downloadCanvas(type, vueDom, state.dvInfo.name, () => {
      downloadStatus.value = false
    })
  })
}

const downloadAsAppTemplate = downloadType => {
  downloadStatus.value = true
  nextTick(() => {
    const vueDom = previewCanvasContainer.value.querySelector('.canvas-container')
    download2AppTemplate(downloadType, vueDom, state.dvInfo.name, () => {
      downloadStatus.value = false
    })
  })
}

const slideOpenChange = () => {
  slideShow.value = !slideShow.value
}

const getPreviewStateInfo = () => {
  return state
}

const reload = id => {
  loadCanvasData(id, state.dvInfo.weight)
}

const resourceNodeClick = data => {
  loadCanvasData(data.id, data.weight)
  if (showPosition.value === 'multiplexing') {
    dvMainStore.initCurMultiplexingComponents()
  }
}

const previewShowFlag = computed(() => !!dvMainStore.dvInfo?.name)

onBeforeMount(() => {
  if (showPosition.value === 'preview') {
    dvMainStore.canvasDataInit()
  }
})

defineExpose({
  getPreviewStateInfo
})
</script>

<template>
  <div class="dv-preview dv-teleport-query">
    <el-aside
      class="resource-area"
      :class="{ 'close-side': !slideShow }"
      ref="node"
      :style="{ width: width + 'px' }"
    >
      <de-resource-tree
        ref="resourceTreeRef"
        v-show="slideShow"
        :cur-canvas-type="'dashboard'"
        :show-position="showPosition"
        @node-click="resourceNodeClick"
      />
    </el-aside>
    <el-container
      class="preview-area"
      :class="{ 'no-data': !hasTreeData }"
      v-loading="requestStore.loadingMap[permissionStore.currentPath]"
    >
      <div
        @click="slideOpenChange"
        v-if="showPosition === 'preview' && false"
        class="flexible-button-area"
      >
        <el-icon v-if="slideShow"><ArrowLeft /></el-icon>
        <el-icon v-else><ArrowRight /></el-icon>
      </div>
      <!--从store中判断当前是否有点击仪表板 复用时也符合-->
      <template v-if="previewShowFlag">
        <preview-head
          v-if="showPosition === 'preview'"
          @reload="reload"
          @download="downloadH2"
          @downloadAsAppTemplate="downloadAsAppTemplate"
        />
        <div ref="previewCanvasContainer" class="content">
          <de-preview
            ref="dashboardPreview"
            v-if="state.canvasStylePreview && dataInitState"
            :dv-info="state.dvInfo"
            :cur-gap="state.curPreviewGap"
            :component-data="state.canvasDataPreview"
            :canvas-style-data="state.canvasStylePreview"
            :canvas-view-info="state.canvasViewInfoPreview"
            :show-position="showPosition"
            :download-status="downloadStatus"
          ></de-preview>
        </div>
      </template>
      <template v-else-if="hasTreeData && mounted">
        <empty-background description="请在左侧选择仪表板" img-type="select" />
      </template>
      <template v-else-if="mounted">
        <empty-background description="暂无仪表板" img-type="none">
          <el-button v-if="rootManage && !isDataEaseBi" @click="createNew" type="primary">
            <template #icon>
              <Icon name="icon_add_outlined" />
            </template>
            {{ $t('commons.create') }}{{ $t('chart.dashboard') }}
          </el-button>
        </empty-background>
      </template>
    </el-container>
  </div>
</template>

<style lang="less">
.dv-preview {
  width: 100%;
  height: 100%;
  overflow: hidden;
  display: flex;
  background: #ffffff;
  .resource-area {
    position: relative;
    height: 100%;
    width: 279px;
    padding: 0;
    border-right: 1px solid #d7d7d7;
    overflow: visible;
  }
  .preview-area {
    flex: 1;
    display: flex;
    flex-direction: column;
    overflow-x: hidden;
    overflow-y: auto;
    position: relative;
    //transition: 0.5s;

    &.no-data {
      background-color: rgba(245, 246, 247, 1);
    }

    .content {
      display: flex;
      width: 100%;
      height: 100%;
      overflow-x: hidden;
      overflow-y: auto;
      align-items: center;
    }
  }
}

.close-side {
  width: 0px !important;
  padding: 0px !important;
}

.flexible-button-area {
  position: absolute;
  height: 60px;
  width: 16px;
  left: 0;
  top: calc(50% - 30px);
  background-color: #ffffff;
  border-radius: 0 4px 4px 0;
  cursor: pointer;
  z-index: 10;
  display: flex;
  align-items: center;
  border-top: 1px solid #d7d7d7;
  border-right: 1px solid #d7d7d7;
  border-bottom: 1px solid #d7d7d7;
}
</style>
