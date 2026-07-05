<script setup lang="ts">
import { computed, reactive, ref } from 'vue'
import type { FormInstance, FormRules, TabsPaneContext } from 'element-plus'
import { ElMessage } from 'element-plus'
import { CopyDocument, Refresh, Search } from '@element-plus/icons-vue'

type Market = 'EU' | 'US' | 'TH'
type Platform = 'amazon' | 'alibaba' | 'tiktok'

type FormModel = {
  productName: string
  ingredient: string
  effect: string
  market: Market | ''
  platforms: Platform[]
}

type PlatformOption = {
  label: string
  value: Platform
}

type ResultItem = {
  platform: Platform
  title: string
  bullets: string[]
  updatedAt: string
}

type TextSegment = {
  text: string
  blocked?: boolean
  suggestion?: string
}

const formRef = ref<FormInstance>()
const loading = ref(false)
const activeTab = ref<Platform>('amazon')
const results = ref<ResultItem[]>([])

const form = reactive<FormModel>({
  productName: '',
  ingredient: '',
  effect: '',
  market: '',
  platforms: ['amazon'],
})

const marketOptions = [
  { label: '欧盟', value: 'EU' },
  { label: '美国', value: 'US' },
  { label: '泰国', value: 'TH' },
]

const platformOptions: PlatformOption[] = [
  { label: '亚马逊', value: 'amazon' },
  { label: '国际站', value: 'alibaba' },
  { label: 'TikTok', value: 'tiktok' },
]

const blockedWords = [
  { word: '美白', suggestion: '提亮肤色' },
  { word: '祛斑', suggestion: '淡化暗沉' },
  { word: '抗衰', suggestion: '紧致呵护' },
  { word: '防癌', suggestion: '健康防护' },
  { word: '速效', suggestion: '快速吸收' },
]

const rules = reactive<FormRules<FormModel>>({
  productName: [{ required: true, message: '请输入产品名称', trigger: 'blur' }],
  ingredient: [{ required: true, message: '请输入核心成分', trigger: 'blur' }],
  effect: [{ required: true, message: '请输入主打功效', trigger: 'blur' }],
  market: [{ required: true, message: '请选择目标市场', trigger: 'change' }],
  platforms: [
    {
      type: 'array',
      required: true,
      min: 1,
      message: '请至少选择一个目标平台',
      trigger: 'change',
    },
  ],
})

const hasResults = computed(() => results.value.length > 0)

function getPlatformLabel(platform: Platform) {
  return platformOptions.find((item) => item.value === platform)?.label ?? platform
}

function getMarketLabel(market: Market | '') {
  return marketOptions.find((item) => item.value === market)?.label ?? '-'
}

function getTime() {
  return new Date().toLocaleTimeString('zh-CN', {
    hour: '2-digit',
    minute: '2-digit',
  })
}

function makeResult(platform: Platform): ResultItem {
  const productName = form.productName.trim()
  const ingredient = form.ingredient.trim()
  const effect = form.effect.trim()
  const market = getMarketLabel(form.market)

  const templates: Record<Platform, ResultItem> = {
    amazon: {
      platform,
      title: `${productName} with ${ingredient} | Gentle Care for ${market}`,
      bullets: [
        `${ingredient} 配方，帮助日常维持肌肤水润与光泽。`,
        `${effect}，适合跨境美妆详情页演示场景。`,
        '轻盈肤感，适合通勤、旅行和日常护理。',
        `面向 ${market} 市场，文案已标记需人工复核的敏感表达。`,
        'Mock 生成结果，仅用于前端交互演示。',
      ],
      updatedAt: getTime(),
    },
    alibaba: {
      platform,
      title: `${productName} Beauty Care Supply for ${market} Buyers`,
      bullets: [
        `核心成分：${ingredient}，适合外贸询盘页面展示。`,
        `卖点方向：${effect}，支持品牌方按市场再编辑。`,
        '包装、容量和认证信息可在正式系统中补充。',
        '当前版本仅模拟内容生成和合规提示。',
      ],
      updatedAt: getTime(),
    },
    tiktok: {
      platform,
      title: `${productName} Short Video Script - ${market}`,
      bullets: [
        `开场：展示 ${ingredient} 质地和上脸延展性。`,
        `卖点：突出 ${effect}，语气轻快但避免绝对化承诺。`,
        '镜头：产品特写、肤感对比、购物车口播。',
        '结尾：引导查看详情页，保留人工审核环节。',
      ],
      updatedAt: getTime(),
    },
  }

  return templates[platform]
}

function splitWithBlockedWords(text: string): TextSegment[] {
  const matches = blockedWords
    .flatMap((item) => {
      const found: Array<TextSegment & { index: number; length: number }> = []
      let start = text.indexOf(item.word)

      while (start >= 0) {
        found.push({
          text: item.word,
          blocked: true,
          suggestion: item.suggestion,
          index: start,
          length: item.word.length,
        })
        start = text.indexOf(item.word, start + item.word.length)
      }

      return found
    })
    .sort((a, b) => a.index - b.index)

  if (!matches.length) return [{ text }]

  const segments: TextSegment[] = []
  let cursor = 0

  matches.forEach((match) => {
    if (match.index < cursor) return
    if (match.index > cursor) {
      segments.push({ text: text.slice(cursor, match.index) })
    }

    segments.push({
      text: match.text,
      blocked: true,
      suggestion: match.suggestion,
    })
    cursor = match.index + match.length
  })

  if (cursor < text.length) {
    segments.push({ text: text.slice(cursor) })
  }

  return segments
}

function resultToText(result: ResultItem) {
  return [result.title, ...result.bullets.map((item) => `- ${item}`)].join('\n')
}

async function copyResult(result: ResultItem) {
  const text = resultToText(result)

  try {
    await navigator.clipboard.writeText(text)
  } catch {
    const textarea = document.createElement('textarea')
    textarea.value = text
    textarea.style.position = 'fixed'
    textarea.style.opacity = '0'
    document.body.appendChild(textarea)
    textarea.select()
    document.execCommand('copy')
    document.body.removeChild(textarea)
  }

  ElMessage.success('复制成功')
}

async function handleGenerate() {
  if (!formRef.value) return

  const valid = await formRef.value.validate().catch(() => false)
  if (!valid) {
    ElMessage.warning('请先补全必填信息')
    return
  }

  loading.value = true
  window.setTimeout(() => {
    results.value = form.platforms.map((platform) => makeResult(platform))
    activeTab.value = results.value[0]?.platform ?? 'amazon'
    loading.value = false
    ElMessage.success('生成完成')
  }, 2000)
}

function handleReset() {
  formRef.value?.resetFields()
  form.platforms = ['amazon']
  results.value = []
  activeTab.value = 'amazon'
}

function handleTabClick(tab: TabsPaneContext) {
  activeTab.value = tab.paneName as Platform
}
</script>

<template>
  <main class="workspace">
    <header class="topbar">
      <div>
        <p class="eyebrow">Demo / Mock</p>
        <h1>美妆外贸内容生成台</h1>
      </div>
      <el-tag type="warning" effect="plain" round>演示版</el-tag>
    </header>

    <section class="layout">
      <aside class="panel form-panel">
        <div class="panel-head">
          <h2>信息填写</h2>
          <span>Beauty Content</span>
        </div>

        <el-form
          ref="formRef"
          :model="form"
          :rules="rules"
          label-position="top"
          class="input-form"
        >
          <el-form-item label="产品名称" prop="productName">
            <el-input v-model="form.productName" placeholder="例如：VC 精华乳" clearable />
          </el-form-item>

          <el-form-item label="核心成分" prop="ingredient">
            <el-input v-model="form.ingredient" placeholder="例如：烟酰胺、玻尿酸" clearable />
          </el-form-item>

          <el-form-item label="主打功效" prop="effect">
            <el-input
              v-model="form.effect"
              placeholder="例如：美白、抗衰、补水"
              type="textarea"
              :rows="3"
              resize="none"
            />
          </el-form-item>

          <el-form-item label="目标市场" prop="market">
            <el-select v-model="form.market" placeholder="请选择" class="full-width">
              <el-option
                v-for="item in marketOptions"
                :key="item.value"
                :label="item.label"
                :value="item.value"
              />
            </el-select>
          </el-form-item>

          <el-form-item label="目标平台" prop="platforms">
            <el-checkbox-group v-model="form.platforms" class="platforms">
              <el-checkbox-button
                v-for="item in platformOptions"
                :key="item.value"
                :value="item.value"
              >
                {{ item.label }}
              </el-checkbox-button>
            </el-checkbox-group>
          </el-form-item>
        </el-form>
      </aside>

      <section class="panel result-panel" v-loading="loading">
        <div class="panel-head">
          <h2>生成结果</h2>
          <span>{{ hasResults ? 'Ready' : 'Waiting' }}</span>
        </div>

        <el-empty v-if="!hasResults" description="等待生成结果" :image-size="120" />

        <el-tabs v-else v-model="activeTab" class="result-tabs" @tab-click="handleTabClick">
          <el-tab-pane
            v-for="result in results"
            :key="result.platform"
            :label="getPlatformLabel(result.platform)"
            :name="result.platform"
          >
            <article class="result-card">
              <div class="result-card-head">
                <div>
                  <p class="result-meta">
                    {{ getPlatformLabel(result.platform) }} / {{ result.updatedAt }}
                  </p>
                  <h3>
                    <template v-for="(segment, index) in splitWithBlockedWords(result.title)" :key="index">
                      <el-tooltip
                        v-if="segment.blocked"
                        :content="`建议替换为：${segment.suggestion}`"
                        placement="top"
                      >
                        <mark>{{ segment.text }}</mark>
                      </el-tooltip>
                      <span v-else>{{ segment.text }}</span>
                    </template>
                  </h3>
                </div>

                <el-button :icon="CopyDocument" plain @click="copyResult(result)">复制</el-button>
              </div>

              <ol class="bullet-list">
                <li v-for="(bullet, bulletIndex) in result.bullets" :key="bulletIndex">
                  <template v-for="(segment, index) in splitWithBlockedWords(bullet)" :key="index">
                    <el-tooltip
                      v-if="segment.blocked"
                      :content="`建议替换为：${segment.suggestion}`"
                      placement="top"
                    >
                      <mark>{{ segment.text }}</mark>
                    </el-tooltip>
                    <span v-else>{{ segment.text }}</span>
                  </template>
                </li>
              </ol>
            </article>
          </el-tab-pane>
        </el-tabs>
      </section>
    </section>

    <footer class="actions">
      <el-button :icon="Refresh" size="large" @click="handleReset">重置</el-button>
      <el-button :icon="Search" type="primary" size="large" :loading="loading" @click="handleGenerate">
        生成
      </el-button>
    </footer>
  </main>
</template>
