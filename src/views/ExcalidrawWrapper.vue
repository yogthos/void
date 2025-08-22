<!--
Copyright 2025 The VOID Authors. All Rights Reserved.

  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
<script setup lang="ts">
import { ref, onMounted, watch } from 'vue';
import { applyReactInVue } from 'veaury';
import { appDataDir, join } from '@tauri-apps/api/path';
import { restore, serializeAsJSON } from '@excalidraw/excalidraw';
import type { AppState, BinaryFiles } from '@excalidraw/excalidraw/types/types';
import type { ExcalidrawElement } from '@excalidraw/excalidraw/types/element/types';
import { read_canvas, write_canvas } from '@/lib/logic/utils';
import { useExplorerStore } from '@/lib/logic/explorerstore';


let props = defineProps({
  url: String
});
interface ExcalidrawData {
  elements: ExcalidrawElement[];
  appState: Partial<AppState>;
  files?: BinaryFiles;
}

const ExcalidrawReact = ref<any>(null);
const drawingData = ref<ExcalidrawData>({
  elements: [],
  appState: {
    viewBackgroundColor: 'transparent',
    width: 0,
    height: 0,
    offsetTop: 0,
    offsetLeft: 0,
  }
});
const initializationError = ref<string | null>(null);
const filePath = ref<string>('');
const file_path = ref<string>('');
const waiting = ref<boolean>(false);

const initializeApp = async () => {
  try {
    filePath.value = await join(await appDataDir(), 'drawing.json');
    const module = await import('@excalidraw/excalidraw');
    ExcalidrawReact.value = applyReactInVue(module.Excalidraw);
  } catch (error) {
    initializationError.value = `Ошибка загрузки: ${error instanceof Error ? error.message : String(error)}`; // TODO: add i18n interpolation
  }
};

async function createExcalidrawInstance() {
  if (props.url != undefined) {
    let path = decodeURIComponent(atob(props.url));
    let content = await read_canvas(decodeURIComponent(atob(props.url)));
    if (content == '') {
      await initializeApp();
      file_path.value = useExplorerStore().current + '/' + path.split('/')[path.split('/').length - 1];
      return;
    }
    let obj = JSON.parse(content);
    let restored = restore(obj, null, null);
    drawingData.value = {
      elements: restored.elements,
      appState: {
        ...structuredClone(restored.appState),
        width: 0,
        height: 0,
        offsetTop: 0,
        offsetLeft: 0,
      },
      files: restored.files
    };
    file_path.value = useExplorerStore().current + '/' + path.split('/')[path.split('/').length - 1];
  }
  await initializeApp();
}

onMounted(async () => {
  await createExcalidrawInstance();
});

watch(() => props.url, async () => {
  createExcalidrawInstance();
})


const handleChange = (elements: ExcalidrawElement[], appState: AppState, files?: BinaryFiles) => {
  drawingData.value = { elements, appState, files };
  saveDrawing();
};

const saveDrawing = async () => {
  if (!filePath.value) return;
  let data = serializeAsJSON(drawingData.value.elements, drawingData.value.appState, drawingData.value.files ?? {}, "local");
  if (file_path.value == '' && !waiting.value) {
    waiting.value = true;
    file_path.value = await write_canvas(data);
  }
  else if (!waiting.value) {
    waiting.value = true;
    await write_canvas(data, file_path.value);
  }
  waiting.value = false;

};
</script>

<template>
  <div class="excalidraw-app w-full h-full overflow-hidden rounded-[15px]">
    <div class="canvas-container rounded-[15px] overflow-hidden">
      <component :is="ExcalidrawReact" :initialData="drawingData" @change="handleChange" :UIOptions="{
        canvasActions: {
          loadScene: false,
          saveScene: false,
          export: false,
          changeViewBackgroundColor: false,
          toggleTheme: false,
          saveAsImage: false,
        },
        libraryMenu: false,
      }" />
    </div>
  </div>
</template>

<style scoped>
.excalidraw-app {
  display: flex;
  flex-direction: column;
  background: transparent;
}

.error-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  text-align: center;
  color: #ff4444;
}

.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #f3f3f3;
  border-top: 4px solid #3498db;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 20px;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }

  100% {
    transform: rotate(360deg);
  }
}

.canvas-container {
  flex: 1;
  position: relative;
}

:deep(.excalidraw) {
  --zIndex-popup: 1000;
}

:deep(.App-menu_top .HorizontalMenu) {
  padding-top: env(safe-area-inset-top);
}

:deep(.sidebar-trigger) {
  display: none !important;
}

:deep(.dropdown-menu-button) {
  display: none !important;
}

/* Essential Excalidraw styles */
:deep(.excalidraw) {
  width: 100% !important;
  height: 100% !important;
  position: relative;
  background: transparent;
  overflow: hidden;
}

:deep(.excalidraw__canvas) {
  width: 100% !important;
  height: 100% !important;
  position: relative;
  background: transparent;
}

:deep(.excalidraw__canvas-wrapper) {
  width: 100% !important;
  height: 100% !important;
  position: relative;
  background: transparent;
}

:deep(.layer-ui__wrapper) {
  z-index: var(--zIndex-popup);
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  pointer-events: none;
}

:deep(.layer-ui__wrapper *) {
  pointer-events: auto;
}

:deep(.App-menu_top) {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  z-index: var(--zIndex-popup);
}

:deep(.App-menu_bottom) {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: var(--zIndex-popup);
}

:deep(.FixedSideContainer) {
  position: absolute;
  z-index: var(--zIndex-popup);
}

:deep(.Island) {
  background: rgba(255, 255, 255, 0.9);
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

:deep(.ToolIcon) {
  width: 40px;
  height: 40px;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: background-color 0.2s;
}

:deep(.ToolIcon:hover) {
  background-color: rgba(0, 0, 0, 0.05);
}

:deep(.ToolIcon__icon) {
  width: 24px;
  height: 24px;
}

:deep(.excalidraw-textEditorContainer) {
  position: absolute;
  z-index: var(--zIndex-popup);
  background: white;
  border: 1px solid #d1d5db;
  border-radius: 4px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

:deep(.excalidraw-textEditor) {
  outline: none;
  border: none;
  padding: 8px;
  font-family: inherit;
  font-size: 16px;
  resize: none;
}
</style>
