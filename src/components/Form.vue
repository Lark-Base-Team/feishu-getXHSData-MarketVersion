<template>
  <el-form ref="form" class="form" label-position="top">
    <!-- 头部标题区域 -->
    <div class="header-title">
      <div class="title-icon"></div>
      <div class="title-text">小红书助手</div>
    </div>
    
    <!-- 简介说明区域 -->
    <div class="description-box">
      <p>依据<span class="underline">小红书笔记链接</span>，获取笔记相关数据。</p>
      <div class="feature-tags">
        <span class="feature-tag"><i class="el-icon-data-analysis"></i> 数据获取</span>
        <!-- <span class="feature-tag"><i class="el-icon-document"></i> 内容保存</span> -->
        <!-- <span class="feature-tag"><i class="el-icon-refresh"></i> 定期更新</span> -->
      </div>
    </div>

    <!-- 免责声明区域 -->
    <el-alert
      type="warning"
      :closable="false"
      style="border-radius: 8px; margin-bottom: 20px;"
    >
      <div class="disclaimer">
        <strong>免责声明：</strong>本工具仅供学习交流使用，不得用于任何商业目的。使用本工具获取的数据，用户应当遵守相关法律法规，不得用于侵犯他人合法权益。开发者对使用本工具产生的任何后果不承担责任。
      </div>
    </el-alert>
    
    <!-- 主配置区域 - 使用卡片样式 -->
    <div class="config-card">
      <div class="card-header">
        <div class="card-title">配置参数</div>
        
      </div>

      <el-form-item :label="$t('labels.link')" size="large" required>
        <el-select v-model="linkFieldId" :placeholder="$t('placeholder.link')" style="width: 100%">
          <el-option v-for="meta in mainFieldListSeView" :key="meta.id" :label="meta.name" :value="meta.id" />
        </el-select>
      </el-form-item>

      
      <!-- 多选框 -->
      <div class="section-title">选择需要获取的数据字段</div>
      <div class="map-fields-checklist">
        <el-checkbox v-model="checkAllToMap" :indeterminate="isIndeterminateToMap" @change="handlecheckAllToMapChange">
          {{ $t('selectGroup.selectAll') }}
        </el-checkbox>
        <el-checkbox-group v-model="checkedFieldsToMap" @change="handleCheckedFieldsToMapChange">
          <el-checkbox v-for="fieldToMap in fieldsToMap" :key="fieldToMap.label" :label="fieldToMap.label" 
            :disabled="fieldToMap.label === 'errorTip'" class="field-checkbox">
            {{ $t(`selectGroup.videoInfo.${fieldToMap.label}`) }}
          </el-checkbox>
        </el-checkbox-group>
      </div>

      <div v-if="checkedFieldsToMap.includes('totalInterCount')" class="interaction-fields">
        <div class="section-title">交互量计算字段</div>
        <el-select v-model="toCalcInterCount" multiple :placeholder="$t('placeholder.interCount')" style="width: 100%;">
          <el-option v-for="item in allToCalcInterCount" :key="item.label"
            :label="$t(`selectGroup.videoInfo.${item.label}`)" :value="item.label" />
        </el-select>
      </div>

      <el-form-item v-if="isDetailMode" :label="$t('labels.cookie')" size="large" required>
        <div class="cookie-tip">Cookie 为必填项，没有有效的 Cookie 将无法获取数据。请参考说明文档获取并填写 Cookie 以获取数据。</div>
        <el-input v-model="cookie" type="text" :placeholder="$t('placeholder.cookie')"></el-input>
      </el-form-item>
    </div>

    <!-- 提交按钮 -->
    <div class="action-area">
      <el-button v-loading="isWritingData" @click="writeData" type="primary" size="large" class="submit-button">
        <i class="el-icon-download"></i> {{ $t('submit') }}
      </el-button>
    </div>

    <!-- 错误信息显示区域 -->
    <div v-if="errorMessages.length > 0" class="error-messages">
      <div class="error-header">
        <i class="el-icon-warning-outline"></i> 错误信息
      </div>
      <el-alert
        v-for="(message, index) in errorMessages"
        :key="index"
        type="error"
        :title="message"
        show-icon
        style="margin-bottom: 10px;"
      />
    </div>

  </el-form>

  <!-- 进度弹窗 -->
  <el-dialog
    title="处理进度"
    v-model="showProgressDialog"
    :show-close="false"
    :close-on-click-modal="false"
    :close-on-press-escape="false"
    width="300px"
    align-center
    class="progress-dialog"
  >
    <div class="progress-container">
      <div id="liquidFillChart" class="liquid-fill-chart"></div>
      <div class="progress-text">
        <span>正在处理数据</span>
        <span>{{ progressText }}</span>
      </div>
    </div>
  </el-dialog>

  <!-- 悬浮辅助菜单 -->
  <div class="floating-menu">
    <div class="floating-menu-item highlight-item" @click="openDocumentation">
      <i class="el-icon-document"></i>
      <span>使用文档</span>
      <span class="highlight-badge">推荐阅读</span>
    </div>
    <div class="floating-menu-item" @click="showQrDialog = true">
      <i class="el-icon-service"></i>
      <span>用户群</span>
    </div>
    <div class="floating-menu-item" @click="openFeedbackForm">
      <i class="el-icon-edit-outline"></i>
      <span>问题反馈</span>
    </div>
  </div>

  <!-- 用户群二维码弹窗 -->
  <el-dialog
    title="加入用户群"
    v-model="showQrDialog"
    width="400px"
    align-center
    :show-close="true"
    custom-class="qr-dialog"
  >
    <div class="qr-content">
      <p class="qr-desc">扫描下方二维码加入用户群，获取更多支持</p>
      <img :src="qrCode" alt="用户群二维码" class="qr-image" />
      <p class="qr-tip">如果无法扫码，请联系开发者获取邀请链接</p>
    </div>
  </el-dialog>
</template>

<script setup>
import { bitable, FieldType } from '@lark-base-open/js-sdk';
import { useI18n } from 'vue-i18n';
import { ref, onMounted, computed, isShallow, watch, nextTick, onUnmounted } from 'vue';
import axios from 'axios';
import qs from 'qs';
import Reward from '@/components/Reward.vue'; // 确保路径正确
import * as echarts from 'echarts';
import 'echarts-liquidfill';
// -- 可更改区域
// 后端服务基地址列表
const baseUrls = [
  'https://nmblaicdcfba.sealosbja.site',
  'https://rvrctaumolkd.sealosbja.site',
  'https://evuypgxakdox.sealosbja.site'
]

// 随机获取一个基地址
const getRandomBaseUrl = () => {
  const randomIndex = Math.floor(Math.random() * baseUrls.length)
  return baseUrls[randomIndex]
}

// -- 数据区域
const { t } = useI18n();
const mainFieldListSeView = ref([])
const historyFieldListSeView = ref([])
const linkFieldId = ref('')  // 链接字段Id
const isShowReward = ref(false)
const showQrDialog = ref(false) // 控制二维码弹窗的显示
const showProgressDialog = ref(false) // 控制进度弹窗的显示

const qrCode = ref('./src/assets/qrCode.png'); // 你的二维码图片路径
const isWritingData = ref(false)
let historyTable
const isDetailMode = ref(true)
const checkAllToMap = ref(false)
const isIndeterminateToMap = ref(true)
const fieldsToMap = ref([
  {
    "label": "title"
  },
  {
    "label": "uploader"
  },
  {
    "label": "content"
  },
  {
    "label": "tags"
  },
  {
    "label": "releaseTime"
  },
  {
    "label": "lastUpdateTime"
  },
  // {
  //   "label": "viewCount"
  // },
  {
    "label": "collectionCount"
  },
  {
    "label": "likeCount"
  },
  {
    "label": "shareCount"
  },
  {
    "label": "commentCount"
  },
  {
    "label": "totalInterCount"
  },
  {
    "label": "fetchDataTime"
  },
  {
    "label": "errorTip"
  }
])   //  可以创建的字段
const checkedFieldsToMap = ref([
  'title',
  'uploader',
  'content',
  'tags',
  'releaseTime',
  'lastUpdateTime',
  // 'viewCount',
  'collectionCount',
  'likeCount',
  'shareCount',
  'commentCount',
  'fetchDataTime',
  'errorTip'
])   // 默认的to-map的字段

const toCalcInterCount = ref(['likeCount', 'collectionCount', 'shareCount', 'commentCount'])  // 实际用于计算"总交互量"的字段
const allToCalcInterCount = ref({})  // 可用于计算"总交互量"的字段
const cookie = ref('')
const xSCommon = ref('')
const isForcedEnd = ref(false)
const errorCount = ref(0)
const errorMessages = ref([]) // 新增错误信息数组

const issubmitAbled = computed(() => {
  return true // 按钮始终可点击
})

const mappedFieldIds = ref({
  "main": {},  // 主表
  "history": {}  // 历史记录 表
})

// 添加监听器
watch(linkFieldId, (newVal) => {
  if (newVal) {
    localStorage.setItem('linkFieldId', newVal)
  }
})

watch(cookie, (newVal) => {
  if (newVal) {
    localStorage.setItem('cookie', newVal)
  }
})

// 进度相关变量
const currentProgress = ref(0)
const progressText = computed(() => `${Math.round(currentProgress.value * 100)}%`)
let liquidFillChart = null

// 初始化水波图
const initLiquidFillChart = () => {
  const chartDom = document.getElementById('liquidFillChart')
  if (!chartDom) return
  
  liquidFillChart = echarts.init(chartDom)
  const option = {
    series: [{
      type: 'liquidFill',
      data: [currentProgress.value],
      radius: '80%',
      amplitude: '8%',
      waveLength: '80%',
      color: ['#3370ff'],
      backgroundStyle: {
        color: '#f7f8fa'
      },
      outline: {
        borderDistance: 2,
        itemStyle: {
          borderColor: '#3370ff',
          borderWidth: 2
        }
      },
      label: {
        show: false
      }
    }]
  }
  liquidFillChart?.setOption(option)
}

// 更新水波图进度
const updateProgress = (progress) => {
  currentProgress.value = progress
  if (liquidFillChart) {
    liquidFillChart.setOption({
      series: [{
        data: [progress]
      }]
    })
  }
}

// -- 核心算法区域
// --001== 写入数据
const writeData = async () => {
  errorMessages.value = []
  isShowReward.value = false
  errorCount.value = 0
  if (isWritingData.value) {
    isForcedEnd.value = true
  }
  isWritingData.value = true
  currentProgress.value = 0
  showProgressDialog.value = true // 显示进度弹窗
  
  // 初始化水波图
  await nextTick()
  initLiquidFillChart()

  // 验证必填字段
  if (!linkFieldId.value) {
    errorMessages.value.push(t('errorTip.noLinkField'))
    isWritingData.value = false
    await bitable.ui.showToast({
      toastType: 'warning',
      message: t('errorTip.noLinkField')
    })
    return
  }

  if (isDetailMode.value && !cookie.value) {
    errorMessages.value.push(t('errorTip.noCookie'))
    isWritingData.value = false
    await bitable.ui.showToast({
      toastType: 'warning',
      message: t('errorTip.noCookie')
    })
    return
  }

  if (checkedFieldsToMap.value.length === 0) {
    errorMessages.value.push(t('errorTip.noFieldsSelected'))
    isWritingData.value = false
    await bitable.ui.showToast({
      toastType: 'warning',
      message: t('errorTip.noFieldsSelected')
    })
    return
  }

  // 加载bitable实例
  const { tableId, viewId } = await bitable.base.getSelection();
  const table = await bitable.base.getActiveTable();
  const view = await table.getViewById(viewId);

  // 错误判断：应当在主表中进行
  if (table.id == historyTable.id) {
    errorMessages.value.push(t('notAllowedInHistoryTable'))
    isWritingData.value = false
    await bitable.ui.showToast({
      toastType: 'warning',
      message: t('notAllowedInHistoryTable')
    })
    return
  }

  // 获取字段数据Ids object类型，匹配已有的字段，创建缺少的字段
  // @status {mappedFieldIds, isWritingData}  表示是命令性质的方法，改变mappedFieldIds对象的状态
  const fieldsMappingResult = await completeMappedFieldIdsValue()
  if (!fieldsMappingResult) {
    await bitable.ui.showToast({
      toastType: 'warning',
      message: t('errorTip.mappingFailed')
    })
    return
  }

  // ## model2: 交互式选择记录 
  const RecordList = await bitable.ui.selectRecordIdList(tableId, viewId);

  console.log("RecordList", RecordList)
  
  // 检查是否选择了记录
  if (!RecordList || RecordList.length === 0) {
    errorMessages.value.push(t('errorTip.noRecordsSelected'))
    isWritingData.value = false
    await bitable.ui.showToast({
      toastType: 'warning',
      message: t('errorTip.noRecordsSelected')
    })
    return
  }

  // 更新本地存储
  localStorage.setItem('isDetailMode', isDetailMode.value.toString())

  // 显示开始处理的Toast
  await bitable.ui.showToast({
    toastType: 'info',
    message: `${t('processingStart')} ${RecordList.length} ${t('records')}`
  })

  let successCount = 0
  const totalRecords = RecordList.length

  for (let recordId of RecordList) {
    if (isForcedEnd.value) {
      showProgressDialog.value = false // 隐藏进度弹窗
      await bitable.ui.showToast({
        toastType: 'warning',
        message: t('processForcedEnd')
      })
      return
    }

    let handleErrorRes = await handleError(recordId)
    if (handleErrorRes.isError)
      continue
    else if (handleErrorRes.isReturn)
      return

    const { noteLink, totalNoteInfo } = handleErrorRes

    await getAndSetRecordValue(totalNoteInfo, table, recordId, mappedFieldIds.value.main)
    await getAndAddRecordValue(totalNoteInfo, historyTable, mappedFieldIds.value.history, noteLink)
    
    successCount++
    // 更新进度
    updateProgress(successCount / totalRecords)
  }

  isWritingData.value = false
  isShowReward.value = true
  showProgressDialog.value = false // 隐藏进度弹窗
  
  // 最终的结果提示
  await bitable.ui.showToast({
    toastType: 'success',
    message: `${t('finishTip')} ${errorCount.value === 0 ? t('noErrors') : t('withErrors', {count: errorCount.value})}`
  })
}





// -- 辅助算法区域
// --001== 获取所勾选字段的字段Id
const getSelectedFieldsId = (fieldList, checkedFields) => {
  const mappedFields = {};
  for (let field of checkedFields) {
    // 查找与checkedFields相匹配的fieldListSeView项目
    const foundField = fieldList.find(f => f.name === t(`selectGroup.videoInfo.${field}`));

    if (field.endsWith('Count') && foundField && foundField.type !== 2)
      return ` [${t(`selectGroup.videoInfo.${field}`)}] ${t(`checks.number`)}`
    else if ((field == t('selectGroup.videoInfo.uploader') || field == t('selectGroup.videoInfo.content') || field == t('selectGroup.videoInfo.tags')) && foundField && foundField.type !== 1)
      return ` [${t(`selectGroup.videoInfo.${field}`)}] ${t(`checks.text`)}`
    else if (field == t('selectGroup.videoInfo.releaseTime') && foundField && foundField.type !== 5)
      return ` [${t(`selectGroup.videoInfo.${field}`)}] ${t(`checks.datetime`)}`
    else if (field == t('selectGroup.videoInfo.lastUpdateTime') && foundField && foundField.type !== 5)
      return ` [${t(`selectGroup.videoInfo.${field}`)}] ${t(`checks.datetime`)}`


    // 如果找到了相应的项目，就使用其id，否则设置为-1
    mappedFields[field] = foundField ? foundField.id : -1;
  }



  return mappedFields;
}


// --002== 请求在replit写的flask框架的接口，获取基本数据
/* @param:path
* get_xhs_detail_data-获取详细数据
*/
const getXHSdatabylink = async (noteLink, baseId) => {
  const path = 'get_xhs_detail_data';
  const baseUrl = getRandomBaseUrl();
  const url = `${baseUrl}/${path}`;
  const data = {
    url: noteLink,
    cookie: cookie.value,
    "base_id": baseId
  };

  try {
    const response = await axios.post(url, qs.stringify(data));

    const noteInfo = response.data.info;
    const res = {
      status: 200,
      info: {
        title: noteInfo.title,
        uploader: noteInfo.uploader,
        content: noteInfo.content,
        tags: noteInfo.tags,
        releaseTime: noteInfo.releaseTime,
        lastUpdateTime: noteInfo.lastUpdateTime,
        collectionCount: noteInfo.collectionCount,
        likeCount: noteInfo.likeCount,
        shareCount: noteInfo.shareCount,
        commentCount: noteInfo.commentCount,
        images: noteInfo.images
      }
    };

    return res.status === 200 ? res.info : { status: "-100", result: res };
  } catch (error) {
    console.error(error);
    const errorResponse = error.response ? error.response.data.error : error.message;
    console.log("getXHSdatabylink() >> errorResponse", errorResponse)
    return { status: "-100", result: errorResponse };
  }
};


// --003== 创建 mappedFieldIds 中 value 为 -1 的字段
/**
 * 命令式，改变mappedFieldIds.value的状态
 * @param {param} mappedFieldIds 现有的映射字段ids
 * @param {param} table 当前要处理的table
 */
const createFields = async (mappedFieldIds, table) => {

  for (let key in mappedFieldIds) {
    if (mappedFieldIds[key] === -1) {
      switch (key) {

        case "title":  // 视频名称
          mappedFieldIds[key] = await table.addField({
            type: FieldType.Text,
            name: t(`selectGroup.videoInfo.${key}`),
          })
          break;
        case "link":  // 链接
          mappedFieldIds[key] = await table.addField({
            type: FieldType.Text,
            name: t(`selectGroup.videoInfo.link`),
          })
          break;
        case "errorTip":
          mappedFieldIds[key] = await table.addField({
            type: FieldType.Text,
            name: t(`selectGroup.videoInfo.errorTip`),
          })
          break;
        case "uploader":
        case "content":
        case "tags":
          mappedFieldIds[key] = await table.addField({
            type: FieldType.Text,
            name: t(`selectGroup.videoInfo.${key}`),
          })
          break;
        case "releaseTime":
        case "lastUpdateTime":
          mappedFieldIds[key] = await table.addField({
            type: FieldType.DateTime,
            name: t(`selectGroup.videoInfo.${key}`),
          })
          break;
        case "danmuCount":
        case "coinCount":
        case "viewCount":
        case "collectionCount":
        case "likeCount":
        case "commentCount":  // 评论量
        case "totalInterCount":  // 总互动量
        case "shareCount":
          mappedFieldIds[key] = await table.addField({
            type: FieldType.Number,
            name: t(`selectGroup.videoInfo.${key}`),
          })
          break;
        case "commentWc":
        case "danmuWc":
          mappedFieldIds[key] = await table.addField({
            type: FieldType.Attachment,
            name: t(`selectGroup.videoInfo.${key}`),
          })
          break;
        case "fetchDataTime":
          mappedFieldIds[key] = await table.addField({
            type: FieldType.DateTime,
            name: t(`selectGroup.videoInfo.${key}`),
          })
          break;
      }
    }


  }

}

// --004== 依据 recordId & field 获取 cell 值
const getCellValueByRFIDS = async (recordId, field) => {
  const cellValue = await field.getValue(recordId)
  if (typeof cellValue == 'object')
    return cellValue[0].link

  return cellValue
}

// --005== 依据 checkedFiledsToMap 做不同的请求处理
const getDataByCheckedFields = async (noteLink, baseId) => {
  let basicInfo = await getXHSdatabylink(noteLink, baseId)
  return { basicInfo }
}

/** --006== 补全mappedFieldIds对象的值，使得拿到所有字段数据Ids object类型
 * 匹配已有的字段，创建缺少的字段
 * @status {mappedFieldIds, isWritingData}  表示是命令性质的方法，改变mappedFieldIds对象的状态
 */
const completeMappedFieldIdsValue = async () => {
  const selection = await bitable.base.getSelection()
  const table = await bitable.base.getTableById(selection.tableId)

  // 匹配已有的字段
  const mainMappedFields = getSelectedFieldsId(mainFieldListSeView.value, checkedFieldsToMap.value)
  // 历史记录表的 "视频链接" 字段
  let historyCheckedFields = JSON.parse(JSON.stringify(checkedFieldsToMap.value))
  historyCheckedFields.push('link')
  let result = historyCheckedFields.filter(item => item != 'errorTip')

  const historyMappedFields = getSelectedFieldsId(historyFieldListSeView.value, result)


  if (typeof mainMappedFields == 'string' || typeof historyMappedFields == 'string') {// 错误处理，提示格式错误 
    // 使用错误信息区域显示而不是toast
    errorMessages.value.push(typeof mainMappedFields === 'string' ? mainMappedFields : historyMappedFields)
    isWritingData.value = false
    return false
  }

  // 创建缺少的字段
  mappedFieldIds.value.main = mainMappedFields
  mappedFieldIds.value.history = historyMappedFields
  await createFields(mappedFieldIds.value.main, table)
  await createFields(mappedFieldIds.value.history, historyTable)
  return true
}

/** --007== 错误处理 
 * @status {} 命令性质，写入 errorTip Field
 * 
 * 
*/
const handleErrorTip = async (errorMsg, recordId) => {
  const table = await bitable.base.getActiveTable();
  errorCount.value++
  errorMessages.value.push(errorMsg) // 添加错误信息到显示区域
  await table.setCellValue(mappedFieldIds.value.main['errorTip'], recordId, [{ type: 'text', text: errorMsg }])
}

/**
 * --008== 处理数据写入中的异常
 * @param {string} recordId
 * @return {Promise<noteLink、totalNoteInfo、isError、isReturn>} 
 */
const handleError = async (recordId) => {
  // 错误处理，链接字段格式错误，应为文本类型
  const selection = await bitable.base.getSelection();
  const baseId = selection.baseId
  const table = await bitable.base.getActiveTable();


  const linkFieldMeta = await table.getFieldMetaById(linkFieldId.value)
  const linkField = await table.getFieldById(linkFieldId.value)
  if (linkFieldMeta.type !== 1 && linkFieldMeta.type !== 15) {
    await handleErrorTip(`[${linkFieldMeta.name}] ${t('errorTip.errorLinkType')}`, recordId)
    isWritingData.value = false
    return { "isReturn": true }
  }

  // 错误处理：链接地址为空
  let noteLink
  try {
    noteLink = await getCellValueByRFIDS(recordId, linkField)

  } catch (error) {

    return { "isError": true, "isReturn": false }
  }

  console.log(540, noteLink)
  // 错误处理：链接格式错误
  if (!noteLink.includes('https://www.xiaohongshu.com/') && !noteLink.includes('xhslink.com/')) {

    await handleErrorTip(t('errorTip.errorLink'), recordId)
    return { "isError": true }
  } 
  console.log("noteLink", noteLink)


  let totalNoteInfo = await getDataByCheckedFields(noteLink, baseId)

  if (totalNoteInfo.basicInfo.status == -100) {
    await handleErrorTip(totalNoteInfo.basicInfo.result, recordId)
    return { "isError": true }
  }

  return { noteLink, totalNoteInfo }
}

/**
 * --009== 获取并设置特定 table 的特定 record 的特定字段集合的 Value
 * @param {object} totalNoteInfo 
 * @param {object} table 数据表
 * @param {string} recordId 记录ID
 * @param {object} mappedFieldIds 映射字段Ids 
 */
const getAndSetRecordValue = async (totalNoteInfo, table, recordId, mappedFieldIds) => {
  const recordFields = getRecordFields(totalNoteInfo, mappedFieldIds)
  console.log(recordFields)


  await table.setRecord(recordId, {
    fields: recordFields
  })
}

const getAndAddRecordValue = async (totalNoteInfo, table, mappedFieldIds, noteLink) => {
  console.log("111111", noteLink)

  const recordFields = getRecordFields(totalNoteInfo, mappedFieldIds, noteLink)
  console.log(recordFields)


  await table.addRecord({
    fields: recordFields
  })
}

/**
 * 查询式，获取 recordFields
 * @param {object} totalNoteInfo 笔记数据
 * @param {object} mappedFieldIds 字段链接
 */
const getRecordFields = (totalNoteInfo, mappedFieldIds, noteLink = 0) => {
  let recordFields = {}
  let key = ''
  let value = ''
  let fetchDataTimeValue = Date.now()
  console.log(mappedFieldIds)
  let checkedFields = JSON.parse(JSON.stringify(checkedFieldsToMap.value))

  checkedFields = checkedFields.filter(item => item != 'errorTip')
  for (let field of checkedFields) {

    key = mappedFieldIds[field]

    if (field === 'fetchDataTime')
      value = fetchDataTimeValue
    else if (field === 'totalInterCount')
      value = toCalcInterCount.value.reduce((total, key) => total + totalNoteInfo.basicInfo[key], 0);
    else
      value = totalNoteInfo.basicInfo[field]


    recordFields[key] = value
  }

  // 打个补丁，链接字段
  if (noteLink)
    recordFields[mappedFieldIds['link']] = noteLink


  return recordFields
}


// Map==全选事件
const handlecheckAllToMapChange = (val) => {
  const data = JSON.parse(JSON.stringify(fieldsToMap.value))
  if (val) {
    checkedFieldsToMap.value = []
    for (const item of data)
      checkedFieldsToMap.value.push(item.label);
  } else {
    checkedFieldsToMap.value = ['errorTip'] // 即使取消全选，也保留 errorTip
  }
  isIndeterminateToMap.value = false
}
// Map==字段选择事件
const handleCheckedFieldsToMapChange = (value) => {
  // 确保 errorTip 始终被选中
  if (!value.includes('errorTip')) {
    value.push('errorTip')
  }
  
  const checkedCount = value.length
  checkAllToMap.value = checkedCount === fieldsToMap.value.length
  isIndeterminateToMap.value = checkedCount > 0 && checkedCount < fieldsToMap.value.length
}


onMounted(async () => {
  // 初始化勾选字段
  // const language = await bitable.bridge.getLanguage();

  // 获取字段列表 -- start
  const selection = await bitable.base.getSelection()
  const table = await bitable.base.getTableById(selection.tableId)
  const view = await table.getViewById(selection.viewId)
  mainFieldListSeView.value = await view.getFieldMetaList()
  // 获取字段列表 -- end


  // 历史记录表
  try {
    historyTable = await bitable.base.getTableByName(t('history.tableName'));

  } catch (error) {
    const { tableId, index } = await bitable.base.addTable({
      name: t('history.tableName')
    })

    historyTable = await bitable.base.getTableById(tableId)

  }

  historyFieldListSeView.value = await historyTable.getFieldMetaList()


  // 初始化可参与计算 "总交互量" 的对象数组，以"Count"结尾的
  allToCalcInterCount.value = fieldsToMap.value
    .filter(item => item.label.endsWith('Count'));

  // 读取缓存内容
  if (localStorage.getItem('isDetailMode') !== null) {
    isDetailMode.value = localStorage.getItem('isDetailMode') === 'true'
  }
  
  // 从缓存中读取链接字段ID
  const cachedLinkFieldId = localStorage.getItem('linkFieldId')
  if (cachedLinkFieldId) {
    // 验证缓存的字段ID是否在当前表格中存在
    const fieldExists = mainFieldListSeView.value.some(field => field.id === cachedLinkFieldId)
    if (fieldExists) {
      linkFieldId.value = cachedLinkFieldId
    } else {
      // 如果字段不存在，清除缓存
      localStorage.removeItem('linkFieldId')
    }
  }

  // 从缓存中读取cookie
  const cachedCookie = localStorage.getItem('cookie')
  if (cachedCookie) {
    cookie.value = cachedCookie
  }
  
  if (localStorage.getItem('xSCommon') !== null) {
    xSCommon.value = localStorage.getItem('xSCommon')
  }

  // 确保 errorTip 字段始终被选中
  if (!checkedFieldsToMap.value.includes('errorTip')) {
    checkedFieldsToMap.value.push('errorTip')
  }
});

// 悬浮菜单功能
const openDocumentation = () => {
  window.open('https://jfsq6znqku.feishu.cn/wiki/T4z9wWK88inr5zkQqhtcwUCSnSf?from=from_copylink', '_blank');
}

const openFeedbackForm = () => {
  window.open('https://jfsq6znqku.feishu.cn/share/base/form/shrcnpUQeYcjCLK9IDPHX18seEf', '_blank');
}

// 在组件卸载时销毁图表
onUnmounted(() => {
  if (liquidFillChart) {
    liquidFillChart.dispose()
    liquidFillChart = null
  }
})

</script>



<style scoped>
.form {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.header-title {
  display: flex;
  align-items: center;
  margin-bottom: 20px;
}

.title-icon {
  font-size: 28px;
  margin-right: 12px;
}

.title-text {
  font-size: 24px;
  font-weight: bold;
  color: #333;
}

.description-box {
  background-color: #f7f8fa;
  border-radius: 8px;
  padding: 16px;
  margin-bottom: 20px;
}

.description-box p {
  margin: 0 0 12px 0;
  color: #606266;
  line-height: 1.5;
}

.feature-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
}

.feature-tag {
  display: inline-flex;
  align-items: center;
  padding: 4px 10px;
  background-color: #e6f1fc;
  color: #409eff;
  border-radius: 4px;
  font-size: 12px;
}

.feature-tag i {
  margin-right: 4px;
}

.config-card {
  background-color: #fff;
  border-radius: 8px;
  padding: 20px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.05);
  margin-bottom: 20px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.card-title {
  font-size: 18px;
  font-weight: bold;
  color: #333;
}

.section-title {
  margin: 16px 0 8px 0;
  font-weight: 500;
  color: #606266;
}

.helper-doc {
  font-size: 15px;
}

.helper-doc a.doc-link {
  color: #3370ff;
  text-decoration: none;
  display: flex;
  align-items: center;
  padding: 6px 12px;
  border: 1px solid #3370ff;
  border-radius: 4px;
  transition: all 0.3s ease;
  background-color: rgba(51, 112, 255, 0.08);
  font-weight: 500;
}

.helper-doc a.doc-link:hover {
  background-color: #3370ff;
  color: white;
  box-shadow: 0 2px 8px rgba(51, 112, 255, 0.3);
}

.helper-doc a.doc-link i {
  margin-right: 6px;
  font-size: 16px;
}

.map-fields-checklist {
  margin-bottom: 16px;
}

.field-checkbox {
  margin-right: 16px;
  margin-bottom: 8px;
}

.interaction-fields {
  background-color: #f5f7fa;
  padding: 16px;
  border-radius: 6px;
  margin-bottom: 16px;
}

.cookie-tip {
  font-size: 14px;
  color: #909399;
  line-height: 1.5;
  margin-bottom: 10px;
}

.disclaimer {
  font-size: 14px;
  line-height: 1.5;
  text-align: left;
  color: #5f350e;
}

.action-area {
  display: flex;
  justify-content: center;
  margin: 30px 0;
}

.submit-button {
  width: 180px;
  height: 50px;
  font-size: 16px;
  box-shadow: 0 4px 12px rgba(51, 112, 255, 0.2);
}

.error-messages {
  margin-top: 20px;
  padding: 16px;
  border-radius: 8px;
  background-color: #fff2f0;
  border: 1px solid #fde2e2;
}

.error-header {
  display: flex;
  align-items: center;
  font-size: 16px;
  font-weight: 500;
  color: #f56c6c;
  margin-bottom: 12px;
}

.error-header i {
  margin-right: 8px;
}

.underline {
  text-decoration: underline;
  font-weight: 500;
  color: #333;
}

/* 悬浮菜单样式 */
.floating-menu {
  position: fixed;
  right: 20px;
  bottom: 100px;
  display: flex;
  flex-direction: column;
  gap: 12px;
  z-index: 1000;
}

.floating-menu-item {
  display: flex;
  align-items: center;
  padding: 10px 16px;
  background-color: white;
  border-radius: 20px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.15);
  cursor: pointer;
  transition: all 0.3s ease;
  color: #333;
  font-size: 15px;
  min-width: 110px;
  position: relative;
}

.highlight-item {
  background-color: #ECF5FF;
  border: 1px solid #3370ff;
  box-shadow: 0 4px 20px rgba(51, 112, 255, 0.25);
}

.highlight-badge {
  position: absolute;
  top: -10px;
  right: 10px;
  background-color: #3370ff;
  color: white;
  padding: 2px 8px;
  border-radius: 10px;
  font-size: 12px;
  font-weight: bold;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

.floating-menu-item:hover {
  background-color: #f0f7ff;
  color: #3370ff;
  transform: translateX(-8px);
  box-shadow: 0 6px 20px rgba(51, 112, 255, 0.25);
}

.highlight-item:hover {
  background-color: #3370ff;
  color: white;
  transform: translateX(-10px);
  box-shadow: 0 6px 20px rgba(51, 112, 255, 0.4);
}

/* 响应式调整 */
@media (max-width: 768px) {
  .floating-menu {
    bottom: 30px;
    right: 15px;
  }
  
  .floating-menu-item {
    padding: 8px 14px;
    font-size: 14px;
    min-width: 100px;
  }
}

/* 二维码弹窗样式 */
.qr-dialog {
  border-radius: 12px;
  overflow: hidden;
}

.qr-content {
  text-align: center;
  padding: 20px;
}

.qr-desc {
  margin-bottom: 20px;
  color: #606266;
  font-size: 15px;
  line-height: 1.5;
}

.qr-image {
  width: 200px;
  height: 200px;
  border-radius: 8px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
  margin: 20px 0;
}

.qr-tip {
  margin-top: 15px;
  font-size: 12px;
  color: #909399;
}

/* 进度弹窗样式 */
.progress-dialog :deep(.el-dialog__header) {
  padding-bottom: 0;
  margin-right: 0;
}

.progress-dialog :deep(.el-dialog__body) {
  padding-top: 10px;
}

.progress-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px 0;
}

.liquid-fill-chart {
  width: 150px;
  height: 150px;
  margin-bottom: 15px;
}

.progress-text {
  display: flex;
  flex-direction: column;
  align-items: center;
  color: #606266;
  font-size: 14px;
}

.progress-text span:first-child {
  margin-bottom: 6px;
  color: #606266;
}

.progress-text span:last-child {
  font-size: 18px;
  font-weight: 500;
  color: #3370ff;
}
</style>
