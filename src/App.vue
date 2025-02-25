<template>
    <div class="container">
        <BlueprintEditor ref="renderer" :blueprintData="data" v-model:selectedBuildingIndex="selectedBuildingIndex"
                         @update:selectedBuildingIndex="i => buildingFocused(i !== null)" />
        <div class="sidebar" :class="{ expanded: expandSidebar }">
            <div>
                <div class="info-tab tab"
                    :class="{ active: activeTab === 'info'}"
                    @click="activeTab = 'info'"></div>
                <div class="operations-tab tab"
                    :class="{ active: activeTab === 'operations'}"
                    @click="activeTab = 'operations'"></div>
            </div>
            <div v-if="activeTab === 'info'">
                <section style="display: flex; flex-direction: row; gap: 5px;">
                    <div>
                        <label for="iconLayout">图标布局</label>
                        <select id="iconLayout" :disabled="!data"
                                v-model="iconLayout">
                            <option v-for="[id, s] of allIconLayouts" :key="id" :value="id">{{s}}</option>
                        </select>
                        <label for="shortDesc">缩略图文字</label>
                        <input type="text" id="shortDesc" :disabled="!data"
                            :value="data?.header.shortDesc"
                            @input="e => data!.header.shortDesc = (e.target as HTMLInputElement).value">
                    </div>
                    <BlueprintIcon style="height: 90px; flex: none;" :layout-id="iconLayout" :icons="data?.header.icons"/>
                </section>
                <section>
                    <label for="desc">蓝图介绍</label>
                    <textarea rows="2" id="desc" :disabled="!data"
                            :value="data?.header.desc"
                            @input="e => data!.header.desc = (e.target as HTMLInputElement).value"
                    ></textarea>
                </section>
                <section>
                    <div class="row">
                        <label for="bp-str" style="margin-right: auto;">代码</label>
                        <span class="error" v-if="notSupport">不支持，请手动复制</span>
                        <button style="margin-left: 4px;" @click="copy" :disabled="working || !bpStr">复制</button>
                        <button style="margin-left: 4px;" @click="paste" :disabled="working">粘贴</button>
                    </div>
                    <textarea rows="3" id="bp-str" v-model="bpStrInput"
                            @copy="onCopy" @cut="onCut" @paste="onPaste"
                            @focus="encodeBp" @change="e => parseBp((e.target as HTMLTextAreaElement).value)">
                    </textarea>
                    <div class="row" style="align-items: stretch; column-gap: 4px;">
                        <button @click="parseBp('')" :disabled="working"
                                style="position: relative; flex: auto; width: 50px;" >
                            {{ bpStr ? "清空" : "选择文件" }}
                            <input v-if="!bpStr" @change="onBpFile" :disabled="working"
                                type="file" accept="text/plain" id="blueprint-file"
                                style="position: absolute; inset: 0; opacity: 0;" />
                        </button>
                        <a v-if="bpStr" class="button" @click="prepareSave"
                            :href="bpUrl" :download="data?.header.shortDesc + '.txt'"
                            style="flex: auto; width: 50px;" >
                            <!-- TODO: data? to workaround rerender -->
                            保存文件
                        </a>
                    </div>
                    <div class="error">{{ parseErrorMessage }}</div>
                </section>
                <section>
                    <div class="row">
                        <span style="margin-right: auto;">创建版本号</span>
                        <span>{{ data?.header.gameVersion }}</span>
                    </div>
                    <div class="row">
                        <span style="margin-right: auto;">创建时间</span>
                        <span>{{ data?.header.time.toLocaleString([], { timeZone: 'UTC' }) }}</span>
                    </div>
                </section>
                <section v-if="selectedBuilding !== null">
                    <BuildingInfoPanel :building="selectedBuilding" @change="() => {rerender(); codeExpired = true}"
                                       @change:position="updateBuildings"/>
                </section>
            </div>
            <div v-else-if="activeTab === 'operations'">
                <ul class="operations">
                    <li v-if="data" @click="replaceModal!.open()">
                        <img src="@/assets/icons/find_replace.svg">
                        批量替换
                        <ReplaceModal :blueprint="data" @change="() => {rerender(); codeExpired = true}" ref="replaceModal"/>
                    </li>
                </ul>
            </div>
            <footer>
                {{ version }}<SWStatus />
            </footer>
        </div>
        <button class="expand-btn" :class="{ expanded: expandSidebar }"
                @click="expandSidebar = !expandSidebar" ></button>
    </div>
</template>

<script setup lang="ts">
import { computed, defineAsyncComponent, nextTick, onUnmounted, provide, reactive, ref, shallowReactive, shallowRef, watch, watchEffect } from 'vue';
import { BlueprintBuilding, BlueprintData, fromStr, toStr } from '@/blueprint/parser';
import { version, rendererKey, buildingInfoKey } from '@/define';
import { BuildingInfo } from './blueprint/buildingInfo';

import BuildingInfoPanel from './components/BuildingInfoPanel.vue';
import SWStatus from '@/swStatus.vue';
import BlueprintIcon from './components/BlueprintIcon.vue';
import ReplaceModal from './components/ReplaceModal.vue';
const BlueprintEditor = defineAsyncComponent(() => import(/* webpackChunkName: "renderer" */'./components/BlueprintEditor.vue'));

const renderer = ref<null | InstanceType<typeof BlueprintEditor>>(null);
const replaceModal = ref<null | InstanceType<typeof ReplaceModal>>(null);
provide(rendererKey, renderer);

const bpStr = ref('');
const data = shallowRef(null as BlueprintData | null);
const expandSidebar = ref(true);
const activeTab = ref<'info' | 'operations'>('info')
const working = ref(false);
const notSupport = ref(false);
const codeExpired = ref(false);
const parseErrorMessage = ref('');
const selectedBuildingIndex = ref(null as number | null);

const rerender = () => {
    // TODO: find a more efficient way
    const d = data.value;
    data.value = null;
    nextTick(() => {
        data.value = d;
    });
}

const buildingFocused = (selected: boolean) => {
    expandSidebar.value = selected;
    if (selected)
        activeTab.value = 'info';
}

const selectedBuilding = computed(() => {
    if (data.value === null || selectedBuildingIndex.value === null)
        return null;
    return data.value.buildings[selectedBuildingIndex.value];
})
const buildingInfo = computed(() => {
    if (data.value === null)
        return null;
    return new BuildingInfo(data.value.buildings);
})
provide(buildingInfoKey, buildingInfo);

const allIconLayouts = new Map<number, string>([
    [ 1, '无'],
    [10, '1-1'], [11, '1-2'],
    [20, '2-1'], [21, '2-2'], [22, '2-3'], [23, '2-4'], [24, '2-5'],
    [30, '3-1'], [31, '3-2'], [32, '3-3'], [33, '3-4'],
    [40, '4-1'], [41, '4-2'],
    [50, '5-1'], [51, '5-2'],
]);
const iconLayout = computed<number>({
    get() { return data.value?.header.layout ?? 0; },
    set(v) { data.value!.header.layout = v; },
})

const encodeBp = () => {
    if (!codeExpired.value || !data.value)
        return;
    bpStr.value = toStr(data.value)
    codeExpired.value = false;
}
const bpStrLatest = () => {
    encodeBp();
    return bpStr.value;
}

const bpStrInput = ref('');
const bpStrDisplay = () => {
    if (codeExpired.value)
        return 'BLUEPRINT:...';
    const len = bpStr.value.length
    if (len < 1000)
        return bpStr.value;
    return bpStr.value.substring(0, 400) + '\n...\n' + bpStr.value.substring(len - 400, len);
}
watchEffect(() => {
    bpStrInput.value = bpStrDisplay();
});

const bpUrl = ref('');
watchEffect(() => {
    if (codeExpired.value && bpUrl.value) {
        URL.revokeObjectURL(bpUrl.value);
        bpUrl.value = '';
    }
})
const prepareSave = () => {
    if (!bpUrl.value) {
        bpUrl.value = URL.createObjectURL(new Blob([bpStrLatest()], { type: 'text/plain' }));
    }
}
onUnmounted(() => {
    if (bpUrl.value) {
        URL.revokeObjectURL(bpUrl.value);
        bpUrl.value = '';
    }
})

const parseBp = (s: string) => {
    if (s) {
        try {
            data.value = shallowReactive(fromStr(s.trim()));
            data.value.header = reactive(data.value.header);
            parseErrorMessage.value = '';
            watch(data.value, () => codeExpired.value = true);
        } catch (e) {
            parseErrorMessage.value = String(e);
        }
    } else {
        data.value = null;
        parseErrorMessage.value = '';
    }
    selectedBuildingIndex.value = null;
    bpStr.value = s;
    codeExpired.value = false;
}

const onBpFile = async (e: Event) => {
    const input = e.target as HTMLInputElement;
    if (input.files && input.files[0]) {
        working.value = true;
        parseBp(await input.files[0].text());
        working.value = false;
    }
    input.value = '';
}

const onCopy = (e: ClipboardEvent) => {
    if (!e.clipboardData)
        return;
    e.preventDefault();
    e.clipboardData.setData('text/plain', bpStrLatest());
}

const onCut = (e: ClipboardEvent) => {
    if (!e.clipboardData)
        return;
    e.preventDefault();
    e.clipboardData.setData('text/plain', bpStrLatest());
    parseBp('');
}

const onPaste = (e: ClipboardEvent) => {
    if (!e.clipboardData)
        return;
    e.preventDefault();
    (e.target as HTMLElement).blur();
    parseBp(e.clipboardData.getData('text/plain'));
}

const copy = async () => {
    working.value = true;
    try {
        await navigator.clipboard.writeText(bpStrLatest());
        notSupport.value = false;
    } catch {
        notSupport.value = true;
    }
    working.value = false;
}

const paste = async () => {
    working.value = true;
    try {
        parseBp(await navigator.clipboard.readText());
        notSupport.value = false;
    } catch {
        notSupport.value = true;
    }
    working.value = false;
}

function updateBuildingPosition(buildingInfo: BlueprintBuilding): void {
    if (data?.value?.buildings) {
        for (let i = 0, len = data.value.buildings.length; i < len; i++) {
            if (data.value.buildings[i].index === buildingInfo.index) {
                // TODO 此处改为单独重新渲染
                data.value.buildings[i] = buildingInfo
                return
            }
        }
    }
}

function updateBuildings(buildings: BlueprintBuilding[]) {
    buildings.forEach(buildingInfo => {
        updateBuildingPosition(buildingInfo)
    })
    // 暂时全局重新渲染
    rerender()
}
</script>

<style lang="scss">
body {
    margin: 0;
    background: black;
    width: 100vw;
    height: 100vh;
    overflow: hidden;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen,
        Ubuntu, Cantarell, "Open Sans", "Helvetica Neue", sans-serif;
}

.container {
    width: 100vw;
    height: 100vh;
    position: relative;
}

.expand-btn {
    position: absolute;
    right: 0;
    top: 0;
    height: 60px;
    width: 60px;
    background: url(@/assets/icons/menu.svg) center no-repeat, #00000060;
    border: 0;

    &.expanded {
        background: url(@/assets/icons/close.svg) center no-repeat;
    }
}

.tab {
    height: 60px;
    width: 60px;
    opacity: 0.5;
    display: inline-block;

    &.active {
        opacity: 1.0;
    }
}
.info-tab {
    background: url(@/assets/icons/menu.svg) center no-repeat;
}
.operations-tab {
    background: url(@/assets/icons/settings.svg) center no-repeat;
}

.row {
    display: flex;
    flex-direction: row;
    align-items: baseline;
}

.sidebar {
    position: absolute;
    right: 0;
    top: 0;
    bottom: 0;
    width: 300px;
    overflow-y: auto;
    box-sizing: border-box;
    background: #000000b0;
    color: white;
    display: none;
    padding-left: 10px;
    padding-right: 10px;

    @media screen and (max-width: 360px) {
        width: 100%;
    }

    section {
        margin: 5px 0;
    }

    footer {
        margin-top: auto;
        margin-bottom: 5px;
        color: gray;
    }

    &.expanded {
        display: flex;
        flex-direction: column;
    }

    .error {
        color: red;
    }

    button,
    .button {
        color: inherit;
        background: #64a0dc;
        border: 0;
        box-sizing: border-box;
        font-size: 0.9rem;
        padding: 1px 4px;
        margin: 0;
        text-decoration: none;
        text-align: center;
        user-select: none;
        cursor: pointer;

        &[disabled] {
            background: gray;
            cursor: default;
        }
    }

    textarea, input[type="text"], select {
        background: #ffffff40;
        border: 0;
        width: 100%;
        font-size: 0.9rem;
        box-sizing: border-box;
        resize: none;
        color: inherit;
        &:focus {
            background: #4f6671;
        }
    }
}

ul.operations {
    padding: 0;
    >li {
        display: block;
        list-style: none;
        user-select: none;
        cursor: pointer;
        border-bottom: 1px solid #fff6;
        padding: 5px;
        margin: 0;

        img {
            vertical-align: middle;
        }
    }
}
</style>
