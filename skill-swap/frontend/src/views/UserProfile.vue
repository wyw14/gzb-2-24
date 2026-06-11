<template>
  <div class="user-profile">
    <div v-if="user" class="card profile-header-card">
      <div class="profile-header">
        <el-avatar :src="user.avatar" :size="100" />
        <div class="profile-info">
          <h1 class="username">{{ user.username }}</h1>
          <div class="user-meta">
            <el-rate :model-value="user.rating" disabled />
            <span class="rating">{{ user.rating }}</span>
            <span class="divider">|</span>
            <span>{{ user.exchangeCount || 0 }} 次交换</span>
            <span class="divider">|</span>
            <span>{{ user.skillPoints || 0 }} 技能点</span>
            <span class="divider">|</span>
            <span>{{ qaStats.questionCount || 0 }} 提问</span>
            <span class="divider">|</span>
            <span>{{ qaStats.answerCount || 0 }} 回答</span>
            <span class="divider">|</span>
            <span class="accepted">{{ qaStats.acceptedAnswerCount || 0 }} 次被采纳</span>
          </div>
          <p class="bio">{{ user.bio || '这个人很懒，什么都没写' }}</p>
        </div>
        <div class="action-buttons">
          <el-button type="primary" @click="goToChat">
            <el-icon><ChatDotRound /></el-icon>发送消息
          </el-button>
        </div>
      </div>
    </div>

    <div v-if="user" class="profile-grid">
      <div class="card">
        <h2 class="section-title">🎓 技能</h2>
        <div class="skills-section">
          <div class="skill-type">
            <h3 class="subsection-title">可教</h3>
            <div class="skills-tags">
              <span v-for="skill in teachSkills" :key="skill.id" class="skill-tag skill-teach">
                {{ skill.name }}
                <small>{{ skill.level }}</small>
              </span>
            </div>
            <el-empty v-if="teachSkills.length === 0" description="暂无" :image-size="60" />
          </div>
          <div class="skill-type">
            <h3 class="subsection-title">想学</h3>
            <div class="skills-tags">
              <span v-for="skill in learnSkills" :key="skill.id" class="skill-tag skill-learn">
                {{ skill.name }}
              </span>
            </div>
            <el-empty v-if="learnSkills.length === 0" description="暂无" :image-size="60" />
          </div>
        </div>
      </div>

      <div class="card">
        <h2 class="section-title">⭐ 评价</h2>
        <div v-if="reviews.length" class="reviews-list">
          <div v-for="review in reviews" :key="review.id" class="review-item">
            <div class="review-header">
              <el-avatar :src="review.reviewerAvatar" :size="40" />
              <div class="reviewer-info">
                <span class="reviewer-name">{{ review.reviewerName }}</span>
                <el-rate :model-value="review.rating" disabled size="small" />
              </div>
            </div>
            <p class="review-content">{{ review.comment }}</p>
          </div>
        </div>
        <el-empty v-else description="暂无评价" />
      </div>
    </div>

    <div class="card qa-section">
      <div class="qa-tabs">
        <span class="tab" :class="{ active: qaTab === 'questions' }" @click="qaTab = 'questions'">
          TA的提问 <span class="count">{{ userQuestions.length }}</span>
        </span>
        <span class="tab" :class="{ active: qaTab === 'answers' }" @click="qaTab = 'answers'">
          TA的回答 <span class="count">{{ userAnswers.length }}</span>
        </span>
      </div>

      <div v-if="qaTab === 'questions'">
        <div v-if="userQuestions.length" class="qa-list">
          <div v-for="q in userQuestions" :key="q.id" class="qa-item" @click="goToQuestion(q.id)">
            <div class="qa-status">
              <el-tag v-if="q.status === 'resolved'" type="success" size="small">已解决</el-tag>
              <el-tag v-else type="warning" size="small">待解决</el-tag>
            </div>
            <div class="qa-content">
              <h4 class="qa-title">{{ q.title }}</h4>
              <div class="qa-meta">
                <el-tag type="info" size="small">{{ q.skillName }}</el-tag>
                <span>{{ q.answerCount || 0 }} 回答</span>
                <span class="dot">·</span>
                <span>{{ formatTime(q.createdAt) }}</span>
              </div>
            </div>
          </div>
        </div>
        <el-empty v-else description="暂无提问" />
      </div>

      <div v-else>
        <div v-if="userAnswers.length" class="qa-list">
          <div v-for="a in userAnswers" :key="a.id" class="qa-item" @click="goToQuestion(a.questionId)">
            <div class="qa-status">
              <el-tag v-if="a.isAccepted" type="success" size="small">已采纳</el-tag>
              <el-tag v-else-if="a.questionStatus === 'resolved'" type="info" size="small">问题已解决</el-tag>
              <el-tag v-else type="warning" size="small">待解决</el-tag>
            </div>
            <div class="qa-content">
              <h4 class="qa-title">{{ a.questionTitle }}</h4>
              <p class="qa-desc">{{ a.content }}</p>
              <div class="qa-meta">
                <span v-if="a.isAccepted" class="accepted-badge">🏆 回答被采纳</span>
                <span>{{ formatTime(a.createdAt) }}</span>
              </div>
            </div>
          </div>
        </div>
        <el-empty v-else description="暂无回答" />
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { authAPI, reviewAPI, questionAPI } from '../api'
import dayjs from 'dayjs'

const route = useRoute()
const router = useRouter()
const user = ref(null)
const reviews = ref([])
const qaTab = ref('questions')
const userQuestions = ref([])
const userAnswers = ref([])
const qaStats = ref({
  questionCount: 0,
  answerCount: 0,
  acceptedAnswerCount: 0,
  resolvedQuestionCount: 0
})

const teachSkills = computed(() => user.value?.skills?.filter(s => s.type === 'teach') || [])
const learnSkills = computed(() => user.value?.skills?.filter(s => s.type === 'learn') || [])

onMounted(async () => {
  await loadUser()
  await loadReviews()
  await loadQAStats()
  await loadUserQuestions()
  await loadUserAnswers()
})

async function loadUser() {
  try {
    const res = await authAPI.getUser(route.params.userId)
    user.value = res.data
  } catch (e) {}
}

async function loadReviews() {
  try {
    const res = await reviewAPI.getReviews(route.params.userId)
    reviews.value = res.data
  } catch (e) {}
}

async function loadQAStats() {
  try {
    const res = await questionAPI.getUserQAStats(route.params.userId)
    qaStats.value = res.data
  } catch (e) {}
}

async function loadUserQuestions() {
  try {
    const res = await questionAPI.getUserQuestions(route.params.userId)
    userQuestions.value = res.data
  } catch (e) {}
}

async function loadUserAnswers() {
  try {
    const res = await questionAPI.getUserAnswers(route.params.userId)
    userAnswers.value = res.data
  } catch (e) {}
}

function goToQuestion(id) {
  router.push(`/questions/${id}`)
}

function goToChat() {
  router.push(`/chat/${route.params.userId}`)
}

function formatTime(time) {
  return dayjs(time).format('YYYY-MM-DD HH:mm')
}
</script>

<style scoped>
.user-profile {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.profile-header-card {
  padding: 32px;
}

.profile-header {
  display: flex;
  align-items: center;
  gap: 24px;
}

.profile-info {
  flex: 1;
}

.username {
  font-size: 28px;
  font-weight: 700;
  color: #333;
  margin-bottom: 8px;
}

.user-meta {
  display: flex;
  align-items: center;
  gap: 12px;
  color: #666;
  font-size: 14px;
  margin-bottom: 12px;
}

.rating {
  font-weight: 600;
  color: #ff9800;
}

.divider {
  color: #ddd;
}

.bio {
  color: #666;
  line-height: 1.6;
}

.profile-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 24px;
}

.skills-section {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.skill-type {
  margin-bottom: 0;
}

.subsection-title {
  font-size: 16px;
  font-weight: 600;
  color: #333;
  margin-bottom: 12px;
}

.skills-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.skill-tag small {
  opacity: 0.8;
  margin-left: 4px;
}

.reviews-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.review-item {
  padding: 16px;
  background: #fafafa;
  border-radius: 8px;
}

.review-header {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 12px;
}

.reviewer-info {
  flex: 1;
}

.reviewer-name {
  font-weight: 600;
  color: #333;
  display: block;
  margin-bottom: 4px;
}

.review-content {
  color: #666;
  line-height: 1.6;
  margin: 0;
}

.accepted {
  color: #67c23a;
  font-weight: 600;
}

.qa-section {
  padding: 24px;
}

.qa-tabs {
  display: flex;
  gap: 24px;
  margin-bottom: 20px;
  border-bottom: 2px solid #eee;
}

.qa-tabs .tab {
  padding: 12px 0;
  cursor: pointer;
  font-weight: 500;
  color: #999;
  border-bottom: 2px solid transparent;
  margin-bottom: -2px;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  gap: 8px;
}

.qa-tabs .tab.active {
  color: #667eea;
  border-bottom-color: #667eea;
}

.qa-tabs .count {
  background: #f0f2f5;
  color: #999;
  padding: 2px 8px;
  border-radius: 10px;
  font-size: 12px;
  font-weight: 400;
}

.qa-tabs .tab.active .count {
  background: #e8ebf5;
  color: #667eea;
}

.qa-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.qa-item {
  display: flex;
  gap: 16px;
  padding: 16px;
  background: #fafafa;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
}

.qa-item:hover {
  background: #f0f2f5;
}

.qa-status {
  min-width: 80px;
}

.qa-content {
  flex: 1;
  min-width: 0;
}

.qa-title {
  font-weight: 600;
  color: #333;
  margin: 0 0 8px 0;
  font-size: 15px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.qa-desc {
  color: #666;
  font-size: 13px;
  margin: 0 0 8px 0;
  display: -webkit-box;
  -webkit-line-clamp: 1;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.qa-meta {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 12px;
  color: #999;
}

.qa-meta .dot {
  color: #ddd;
}

.accepted-badge {
  color: #67c23a;
  font-weight: 500;
}
</style>
