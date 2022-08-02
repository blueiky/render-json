<script setup lang="ts">
import { onMounted, ref, watch, toRaw } from "vue";
import * as monaco from "monaco-editor";
import type { UploadProps } from "element-plus";
import * as xlsx from "xlsx";

import editorWorker from "monaco-editor/esm/vs/editor/editor.worker?worker";
import jsonWorker from "monaco-editor/esm/vs/language/json/json.worker?worker";
import cssWorker from "monaco-editor/esm/vs/language/css/css.worker?worker";
import htmlWorker from "monaco-editor/esm/vs/language/html/html.worker?worker";
import tsWorker from "monaco-editor/esm/vs/language/typescript/ts.worker?worker";


(self as any).MonacoEnvironment = {
    getWorker(_: any, label: any) {
        if (label === "json") {
            return new jsonWorker();
        }
        if (label === "css" || label === "scss" || label === "less") {
            return new cssWorker();
        }
        if (label === "html" || label === "handlebars" || label === "razor") {
            return new htmlWorker();
        }
        if (label === "typescript" || label === "javascript") {
            return new tsWorker();
        }
        return new editorWorker();
    }
};

const common = ref<HTMLDivElement>();
const customEdit = ref<HTMLDivElement>();
const output = ref<HTMLDivElement>();

const edit = ref<monaco.editor.IStandaloneCodeEditor>();
const customEditor = ref<monaco.editor.IStandaloneCodeEditor>();
const outputEditor = ref<monaco.editor.IStandaloneCodeEditor>();

const dataSource = ref("excel");
// const dataSource = ref("custom");
const batchData = ref();

onMounted(() => {
    if (common.value) {
        edit.value = monaco.editor.create(common.value, {
            language: "json",
            theme: "vs-dark",
            formatOnPaste: true,
            automaticLayout: true,
            fontSize: 16,
            minimap: {
                enabled: false,
            },
        });
    }
    if (dataSource.value && dataSource.value === "custom") {
        if (customEdit.value && !customEditor.value) {
            customEditor.value = monaco.editor.create(customEdit.value, {
                value: `[
    {
        "key": "value",
        ...
    },
    ...
]`,
                language: "json",
                theme: "vs-dark",
                formatOnPaste: true,
                automaticLayout: true,
                fontSize: 16,
                minimap: {
                    enabled: false,
                },
            });
        }
    }
    if (output.value) {
        outputEditor.value = monaco.editor.create(output.value, {
            value: "",
            language: "json",
            theme: "vs-dark",
            formatOnPaste: true,
            automaticLayout: true,
            fontSize: 16,
            minimap: {
                enabled: false,
            },
        });
    }
});

watch(dataSource, async (newValue, oldValue) => {
    if (newValue === "custom") {
        batchData.value = undefined;
        if (customEdit.value && !customEditor.value) {
            customEditor.value = monaco.editor.create(customEdit.value, {
                value: `[
    {
        "key": "value",
        ...
    },
    ...
]`,
                language: "json",
                theme: "vs-dark",
                formatOnPaste: true,
                automaticLayout: true,
                fontSize: 16,
                minimap: {
                    enabled: false,
                },
            });
        }
    }
    if (newValue === "excel") {
        batchData.value = undefined;
        customEdit.value = undefined;
    }
});

const handleChange: UploadProps["onChange"] = (uploadFile, uploadFiles) => {
    const fileReader = new FileReader();
    fileReader.onload = function(ev) {
        const workbook = xlsx.read(fileReader.result, { type: "binary", cellDates: true });
        const sheetName = workbook.SheetNames[0];
        batchData.value = xlsx.utils.sheet_to_json(workbook.Sheets[sheetName]);
    };
    // @ts-ignore
    fileReader.readAsBinaryString(new Blob([ uploadFile["raw"] ]));
};

const handleDataReader = () => {
    let commonObj = {};

    if (edit.value) {
        const commonVal = toRaw(edit.value).getValue();
        if (commonVal && commonVal !== "") {
            commonObj = JSON.parse(commonVal);
        }
    }
    if (outputEditor.value) {
        if (dataSource.value === "excel") {
            const data = batchData.value;
            const temp = [];
            for (let item of data) {
                if (item["date"] instanceof Date) {
                    item["date"] = formatDate(item["date"]);
                }
                temp.push({ ...commonObj, ...item });
            }
            toRaw(outputEditor.value).setValue(JSON.stringify(temp, undefined, 4));
        }
        if (dataSource.value === "custom") {
            if (customEditor.value) {
                const data = toRaw(customEditor.value).getValue();
                const json = JSON.parse(data);
                const temp = [];
                for (let item of json) {
                    if (item["date"] instanceof Date) {
                        item["date"] = formatDate(item["date"]);
                    }
                    temp.push({ ...commonObj, ...item });
                }
                toRaw(outputEditor.value).setValue(JSON.stringify(temp, undefined, 4));
            }
        }
    }
};

const formatDate = (date: Date) => {
    const year = date.getFullYear();
    const month = date.getMonth();
    const day = date.getDay();
    const hour = date.getHours();
    const minute = date.getMinutes();
    const sec = date.getSeconds();

    function app(num: number) {
        return num < 10 ? "0" + num : num;
    }

    return `${year}-${app(month)}-${app(day)} ${app(hour)}:${minute}:${sec}`;
};

</script>

<template>
    <div class="editor-container">
        <el-row :gutter="20">
            <el-col :span="14">
                <div class="common">
                    <p>公共参数</p>
                    <div style="width: 100%; height: 260px;" ref="common"/>
                </div>
                <div class="batch">
                    <p>批量参数</p>
                    <div class="select">
                        <el-radio-group v-model="dataSource">
                            <el-radio-button label="excel">Excel文件</el-radio-button>
                            <el-radio-button label="custom">自行输入</el-radio-button>
                        </el-radio-group>
                        <div class="extra">
                            <el-button type="primary" @click="handleDataReader" :disabled="dataSource === 'excel' && !batchData">转化</el-button>
                        </div>
                    </div>

                    <el-upload
                        accept=".xlsx, .xls"
                        :auto-upload="false"
                        :on-change="handleChange"
                        v-if="dataSource === 'excel'"
                    >
                        <template #trigger>
                            <el-button>选择文件</el-button>
                        </template>
                    </el-upload>

                    <div v-show="dataSource === 'custom'" style="width: 100%; height: 460px;" ref="customEdit"/>

                </div>
            </el-col>
            <el-col :span="10">
                <div class="result">
                    <p>结果数据</p>
                    <div style="width: 100%; height: calc(100vh - 64px)" ref="output"/>
                </div>
            </el-col>
        </el-row>
    </div>
</template>

<style scoped>
.editor-container {
    padding: 12px;
}

.common {
    display: flex;
    flex-direction: column;
}

.common p {
    margin-bottom: 12px;
}

.batch {
    padding: 12px 0;
}

.batch p {
    margin-bottom: 8px;
}

.batch .select {
    display: flex;
    margin-bottom: 12px;
}

.select .el-radio-group {
    flex: 1;
}

.result p {
    margin-bottom: 12px;
}
</style>
