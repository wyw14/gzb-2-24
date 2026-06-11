<template>
  <div class="question-detail">
    <el-button @click="$router.back()" type="default" class="back-btn">
      <el-icon><ArrowLeft /></el-icon>返回列表
    </el-button>

    <div v-if="question" class="card question-card">
      <div class="question-header">
        <h1 class="question-title">
          <el-tag v-if="question.status === 'resolved'" type="success" size="large" effect="light">
            ✓ 已解决
          </el-tag>
          <el-tag v-else type="warning" size="large" effect="light">
            待解决
          </el-tag>
          {{ question.title }}
        </h1>
        <div class="question-meta">
          <el-avatar :src="question.authorAvatar" :size="36" />
          <div class="author-info">
            <span class="author-name">{{ question.authorName }}</span>
            <span class="time">{{ formatTime(question.createdAt) }}</span>
          </div>
          <el-tag class="skill-tag" type="info">{{ question.skillName }}</el-tag>
          <div class="stats">
            <span>{{ question.viewCount || 0 }} 浏览</span>
            <span class="dot">·</span>
            <span>{{ question.answerCount || 0 }} 回答</span>
          </div>
        </div>
      </div>
      <div class="question-content">
        {{ question.content }}
      </div>
    </div>

    <div class="answers-section">
      <h2 class="section-title">
        <span>回答</span>
        <span class="count">({{ answers.length }})</span>
      </h2>

      <div v-if="answers.length" class="answers-list">
        <div v-for="answer in answers" :key="answer.id" class="answer-card" :class="{ accepted: answer.isAccepted }">
          <div v-if="answer.isAccepted" class="accepted-badge">
            <el-icon><Check /></el-icon>已采纳的最佳回答
          </div>
          <div class="answer-header">
            <el-avatar :src="answer.authorAvatar" :size="40" />
            <div class="author-info">
              <span class="author-name">{{ answer.authorName }}</span>
              <div class="author-rating">
                <el-rate :model-value="answer.authorRating" disabled size="small" />
                <span>{{ answer.authorRating }}</span>
              </div>
            </div>
            <span class="time">{{ formatTime(answer.createdAt) }}</span>
            <el-button
              v-if="canAccept && !answer.isAccepted && question?.status === 'open' && answer.userId !== userStore.user?.id"
              type="success"
              size="small"
              @click="acceptAnswer(answer.id)"
            >
              <el-icon><Check /></el-icon>采纳为最佳回答
            </el-button>
            <el-tooltip v-else-if="canAccept && !answer.isAccepted && question?.status === 'open' && answer.userId === userStore.user?.id" content="不能采纳自己的回答" placement="top">
              <el-button type="success" size="small" disabled>
                <el-icon><Check /></el-icon>采纳为最佳回答
              </el-button>
            </el-tooltip>
          </div>
          <div class="answer-content">
            {{ answer.content }}
          </div>
        </div>
      </div>
      <el-empty v-else description="暂无回答，快来抢沙发吧" />
    </div>

    <div v-if="question?.status === 'open' && !isQuestionAuthor" class="card answer-form-card">
      <h3 class="form-title">写回答</h3>
      <el-input
        v-model="answerContent"
        type="textarea"
        :rows="5"
        placeholder="分享你的知识和经验，帮助他人解决问题..."
        maxlength="2000"
        show-word-limit
      />
      <div class="form-actions">
        <el-button type="primary" @click="submitAnswer" :loading="submitting" :disabled="!answerContent.trim()">
          <el-icon><Promotion /></el-icon>提交回答
        </el-button>
        <span class="tip">回答被采纳可获得 20 技能点奖励</span>
      </div>
    </div>
    <el-alert v-else-if="question?.status === 'open' && isQuestionAuthor" title="这是你发布的问题，不能回答自己的问题" type="info" :closable="false" show-icon />
    <el-alert v-else title="该问题已解决，不能再回答" type="success" :closable="false" />
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { useRoute } from 'vue-router'
import { questionAPI } from '../api'
import { useUserStore } from '../stores/user'
import { ElMessage, ElMessageBox } from 'element-plus'
import { ArrowLeft, Check, Promotion } from '@element-plus/icons-vue'
import dayjs from 'dayjs'

const route = useRoute()
const userStore = useUserStore()
const question = ref(null)
const answers = ref([])
const answerContent = ref('')
const submitting = ref(false)

const isQuestionAuthor = computed(() => {
  return question.value && question.value.userId === userStore.user?.id
})

const canAccept = computed(() => {
  return isQuestionAuthor.value
})

onMounted(async () => {
  await loadQuestion()
})

async function loadQuestion() {
  try {
    const res = await questionAPI.getQuestion(route.params.id)
    question.value = res.data.question
    answers.value = res.data.answers
  } catch (e) {
    ElMessage.error('加载失败')
  }
}

async function submitAnswer() {
  if (!answerContent.value.trim() || answerContent.value.length < 5) {
    ElMessage.warning('回答内容至少5个字')
    return
  }

  try {
    submitting.value = true
    await questionAPI.createAnswer(route.params.id, { content: answerContent.value })
    ElMessage.success('回答成功')
    answerContent.value = ''
    await loadQuestion()
  } catch (e) {
    ElMessage.error(e.message || '回答失败')
  } finally {
    submitting.value = false
  }
}

async function acceptAnswer(answerId) {
  try {
    await ElMessageBox.confirm('确定采纳此回答为最佳回答吗？采纳后问题将标记为已解决，且无法撤销。', '提示', {
      confirmButtonText: '确定采纳',
      cancelButtonText: '再想想',
      type: 'warning'
    })
    await questionAPI.acceptAnswer(route.params.id, answerId)
    ElMessage.success('采纳成功，回答者获得20技能点奖励')
    await loadQuestion()
  } catch (e) {
    if (e !== 'cancel') {
      ElMessage.error(e.message || '采纳失败')
    }
  }
}

function formatTime(time) {
  return dayjs(time).format('YYYY-MM-DD HH:mm')
}
</script>

<style scoped>
.question-detail {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.back-btn {
  align-self: flex-start;
}

.question-card {
  padding: 32px;
}

.question-header {
  margin-bottom: 24px;
}

.question-title {
  font-size: 24px;
  font-weight: 700;
  color: #333;
  margin: 0 0 16px 0;
  display: flex;
  align-items: center;
  gap: 12px;
}

.question-meta {
  display: flex;
  align-items: center;
  gap: 16px;
  flex-wrap: wrap;
}

.author-info {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.author-name {
  font-weight: 600;
  color: #333;
}

.time {
  font-size: 13px;
  color: #999;
}

.skill-tag {
  margin-left: auto;
}

.stats {
  font-size: 13px;
  color: #999;
}

.dot {
  margin: 0 8px;
  color: #ddd;
}

.question-content {
  line-height: 1.8;
  color: #444;
  font-size: 15px;
  background: #f8f9fa;
  padding: 20px;
  border-radius: 8px;
  white-space: pre-wrap;
}

.answers-section .section-title {
  font-size: 20px;
  font-weight: 600;
  color: #333;
  margin: 0 0 16px 0;
  display: flex;
  align-items: center;
  gap: 8px;
}

.answers-section .count {
  color: #999;
  font-weight: 400;
  font-size: 16px;
}

.answers-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.answer-card {
  background: white;
  border-radius: 12px;
  padding: 24px;
  position: relative;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.06);
  transition: all 0.3s;
}

.answer-card.accepted {
  border: 2px solid #67c23a;
  background: #f0f9eb;
}

.accepted-badge {
  position: absolute;
  top: 16px;
  right: 16px;
  background: #67c23a;
  color: white;
  padding: 4px 12px;
  border-radius: 16px;
  font-size: 13px;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 4px;
}

.answer-header {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 16px;
}

.answer-header .author-info {
  flex: 1;
}

.author-rating {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 13px;
  color: #ff9800;
}

.answer-content {
  line-height: 1.8;
  color: #444;
  font-size: 15px;
  white-space: pre-wrap;
}

.answer-form-card {
  padding: 24px;
}

.form-title {
  font-size: 18px;
  font-weight: 600;
  color: #333;
  margin: 0 0 16px 0;
}

.form-actions {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 16px;
}

.tip {
  font-size: 13px;
  color: #999;
}
</style>
