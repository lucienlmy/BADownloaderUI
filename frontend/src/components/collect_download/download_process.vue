<template>
    <FramePage title="开始下载">
        <var-space justify="center">
            <var-button type="primary" @click="startDownload" size="large" v-if="flow.download === 'idle'"><var-icon name="download" />开始下载</var-button>
        </var-space>



        <div v-if="flow.download === 'running'">
            <var-space direction="column" :size="[12, 12]">
                <var-progress type="info" :value="progress.progressPercent" :line-width="6" label :indeterminate="progress.downFinished == 0" />
            </var-space>

            <var-space justify="center">
                <var-chip type="info" class="mt-2.5">正在下载：{{ progress.downloadingTitle }} ( {{ progress.downFinished }} / {{ flow.listCount }} )</var-chip>
                <var-chip type="danger" class="mt-2.5" v-if="progress.failedCount > 0">失败：{{ progress.failedCount }}</var-chip>
            </var-space>
        </div>

        <template v-if="flow.download === 'done'">
            <var-result
                :title="failed.length > 0 ? `${succeeded} 下载完成，${failed.length} 内容下载失败` : '下载完成'"
                description="请点击下一步回到主页"
            >
                <template #image />
                <template #footer>
                    <var-button @click="OpenDownloadFolader" type="primary">打开下载文件夹</var-button>
                </template>
            </var-result>

            <!-- 失败任务列表，默认收起 -->
            <template v-if="failed.length > 0">
                <var-space justify="center" class="my-4">
                    <var-button text type="primary" @click="showFailed = !showFailed">
                        {{ showFailed ? '收起' : '查看' }}失败项（{{ failed.length }}）
                        <var-icon :name="showFailed ? 'chevron-up' : 'chevron-down'" />
                    </var-button>
                </var-space>
                <var-collapse-transition :expand="showFailed">
                    <div class="flex flex-col gap-2 mx-5 mb-5">
                        <var-paper v-for="item in failed" :key="item.index" class="surface-container px-4 py-2.5">
                            <div class="text-sm font-semibold">{{ item.index + 1 }}. {{ item.title }}</div>
                            <div class="mt-1 text-xs break-all whitespace-pre-wrap opacity-70">{{ item.error }}</div>
                        </var-paper>
                    </div>
                </var-collapse-transition>
            </template>
        </template>
    </FramePage>
</template>

<script setup lang="ts">
import FramePage from '../modules/frame_page.vue'
import { reactive, ref } from 'vue'
import { DownloadTaskList, GetTaskListAll, OpenDownloadFolader } from '../../../wailsjs/go/wails_api/WailsApi'
import { task } from '../../../wailsjs/go/models'
import { onEvent } from '../../events'
import { flow } from '../../state'

// 进度条相关
const progress = reactive({
    downFinished: 0,    // 任务完成数量
    progressPercent: 0, // 任务完成百分比
    downloadingTitle: "", // 下载中标题
    failedCount: 0,     // 失败任务数量
})

// 下载结束后返回的失败任务，以及成功数量（未删除的任务减去失败的任务）
const failed = ref<task.Progress[]>([])
const succeeded = ref(0)
const showFailed = ref(false)

async function startDownload() {
    flow.download = 'running';
    failed.value = (await DownloadTaskList()) ?? [];
    const active = (await GetTaskListAll()).filter(t => !t.IsDelete).length;
    succeeded.value = Math.max(active - failed.value.length, 0);
    flow.download = 'done';
}


// 下载完成事件
onEvent("downloadFinish", (task) => {
    progress.downFinished++;
    progress.downloadingTitle = task.title;
    progress.progressPercent = (progress.downFinished / flow.listCount) * 100;
    if (task.error) {
        progress.failedCount++;
    }
})
</script>
