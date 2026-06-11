<template>
  <div class="questions-page">
    <div class="page-header">
      <h1 class="page-title">💬 问答区</h1>
      <p class="page-desc">围绕技能提问和回答，轻量互助解决问题</p>
      <el-button type="primary" size="large" @click="showAskDialog = true">
        <el-icon><Edit /></el-icon>我要提问
      </el-button>
    </div>

    <div class="filter-bar">
      <el-select v-model="filters.category" placeholder="技能类别" clearable style="width: 160px">
        <el-option v-for="cat in categories" :key="cat.id" :label="cat.name" :value="cat.id" />
      </el-select>
      <el-select v-model="filters.skill" placeholder="相关技能" clearable style="width: 180px" filterable>
        <el-option v-for="skill in allSkills" :key="skill.id" :label="skill.name" :value="skill.id" />
      </el-select>
      <el-select v-model="filters.status" placeholder="问题状态" clearable style="width: 140px">
        <el-option label="待解决" value="open" />
        <el-option label="已解决" value="resolved" />
      </el-select>
      <el-button type="primary" @click="loadQuestions">
        <el-icon><Search /></el-icon>筛选
      </el-button>
    </div>

    <div v-if="questions.length" class="questions-list">
      <div v-for="q in questions" :key="q.id" class="question-card" @click="goToDetail(q.id)">
        <div class="question-stats">
          <div class="stat-item">
            <span class="stat-num">{{ q.answerCount || 0 }}</span>
            <span class="stat-label">回答</span>
          </div>
          <div class="stat-item">
            <span class="stat-num">{{ q.viewCount || 0 }}</span>
            <span class="stat-label">浏览</span>
          </div>
        </div>
        <div class="question-content">
          <div class="question-header">
            <h3 class="question-title">
              <el-tag v-if="q.status === 'resolved'" type="success" size="small" effect="light">已解决</el-tag>
              <el-tag v-else type="warning" size="small" effect="light">待解决</el-tag>
              {{ q.title }}
            </h3>
          </div>
          <p class="question-desc">{{ q.content }}</p>
          <div class="question-meta">
            <el-tag class="skill-tag" type="info" size="small">{{ q.skillName }}</el-tag>
            <el-avatar :src="q.authorAvatar" :size="24" />
            <span class="author-name">{{ q.authorName }}</span>
            <span class="dot">·</span>
            <span class="time">{{ formatTime(q.createdAt) }}</span>
          </div>
        </div>
      </div>
    </div>
    <el-empty v-else description="暂无问题，快来提问吧" />

    <el-dialog v-model="showAskDialog" title="发布问题" width="600px">
      <el-form :model="askForm" label-position="top">
        <el-form-item label="问题标题" required>
          <el-input v-model="askForm.title" placeholder="请输入问题标题，5-100字" maxlength="100" show-word-limit />
        </el-form-item>
        <el-form-item label="相关技能" required>
          <el-select v-model="askForm.skillId" placeholder="选择相关技能" style="width: 100%" filterable @change="onSkillChange">
            <el-option-group v-for="cat in categories" :key="cat.id" :label="cat.name">
              <el-option
                v-for="skill in getSkillsByCategory(cat.id)"
                :key="skill.id"
                :label="skill.name"
                :value="skill.id"
              />
            </el-option-group>
          </el-select>
        </el-form-item>
        <el-form-item label="问题描述" required>
          <el-input v-model="askForm.content" type="textarea" :rows="5" placeholder="详细描述你的问题，至少10个字" maxlength="2000" show-word-limit />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="showAskDialog = false">取消</el-button>
        <el-button type="primary" @click="submitQuestion" :loading="submitting">发布</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { useRouter } from 'vue-router'
import { questionAPI, skillAPI } from '../api'
import { ElMessage } from 'element-plus'
import { Edit, Search } from '@element-plus/icons-vue'
import dayjs from 'dayjs'

const router = useRouter()
const questions = ref([])
const categories = ref([])
const allSkills = ref([])
const mySkills = ref([])
const filters = ref({ category: '', skill: '', status: '' })
const showAskDialog = ref(false)
const submitting = ref(false)
const askForm = ref({
  title: '',
  skillId: '',
  skillName: '',
  category: '',
  content: ''
})

const getSkillsByCategory = (categoryId) => {
  return [...allSkills.value, ...mySkills.value].filter(s => s.category === categoryId)
}

onMounted(async () => {
  await loadCategories()
  await loadAllSkills()
  await loadMySkills()
  await loadQuestions()
})

async function loadCategories() {
  const res = await skillAPI.getCategories()
  categories.value = res.data
}

async function loadAllSkills() {
  const res = await skillAPI.getSkills({ type: 'teach' })
  const skillMap = new Map()
  res.data.forEach(s => {
    if (!skillMap.has(s.name)) {
      skillMap.set(s.name, s)
    }
  })
  allSkills.value = Array.from(skillMap.values())
}

async function loadMySkills() {
  try {
    const res = await skillAPI.getSkills()
    mySkills.value = res.data
  } catch (e) {}
}

async function loadQuestions() {
  const params = {}
  if (filters.value.category) params.category = filters.value.category
  if (filters.value.skill) params.skill = filters.value.skill
  if (filters.value.status) params.status = filters.value.status
  const res = await questionAPI.getQuestions(params)
  questions.value = res.data
}

function onSkillChange(skillId) {
  const skill = [...allSkills.value, ...mySkills.value].find(s => s.id === skillId)
  if (skill) {
    askForm.value.skillName = skill.name
    askForm.value.category = skill.category
  }
}

async function submitQuestion() {
  if (!askForm.value.title || askForm.value.title.length < 5) {
    ElMessage.warning('标题至少5个字')
    return
  }
  if (!askForm.value.skillId) {
    ElMessage.warning('请选择相关技能')
    return
  }
  if (!askForm.value.content || askForm.value.content.length < 10) {
    ElMessage.warning('问题描述至少10个字')
    return
  }

  try {
    submitting.value = true
    await questionAPI.createQuestion(askForm.value)
    ElMessage.success('发布成功')
    showAskDialog.value = false
    askForm.value = { title: '', skillId: '', skillName: '', category: '', content: '' }
    await loadQuestions()
  } catch (e) {
    ElMessage.error(e.message || '发布失败')
  } finally {
    submitting.value = false
  }
}

function goToDetail(id) {
  router.push(`/questions/${id}`)
}

function formatTime(time) {
  return dayjs(time).format('YYYY-MM-DD HH:mm')
}
</script>

<style scoped>
.questions-page {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.page-header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-radius: 16px;
  padding: 32px;
  color: white;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.page-title {
  font-size: 32px;
  font-weight: 700;
  margin: 0 0 8px 0;
}

.page-desc {
  margin: 0;
  opacity: 0.9;
  font-size: 16px;
}

.filter-bar {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
}

.questions-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.question-card {
  background: white;
  border-radius: 12px;
  padding: 24px;
  display: flex;
  gap: 24px;
  cursor: pointer;
  transition: all 0.3s;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.06);
}

.question-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 24px rgba(102, 126, 234, 0.15);
}

.question-stats {
  display: flex;
  flex-direction: column;
  gap: 8px;
  min-width: 80px;
}

.stat-item {
  text-align: center;
  padding: 8px;
  background: #f5f7fa;
  border-radius: 8px;
}

.stat-num {
  display: block;
  font-size: 20px;
  font-weight: 700;
  color: #667eea;
}

.stat-label {
  font-size: 12px;
  color: #999;
}

.question-content {
  flex: 1;
}

.question-header {
  margin-bottom: 8px;
}

.question-title {
  font-size: 18px;
  font-weight: 600;
  color: #333;
  margin: 0;
  display: flex;
  align-items: center;
  gap: 8px;
}

.question-desc {
  color: #666;
  line-height: 1.6;
  margin: 0 0 12px 0;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.question-meta {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  color: #999;
}

.skill-tag {
  margin-right: 8px;
}

.author-name {
  color: #666;
  font-weight: 500;
}

.dot {
  color: #ddd;
}

.time {
  color: #bbb;
}
</style>
