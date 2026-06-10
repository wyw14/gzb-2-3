<template>
  <div class="exchanges">
    <div class="card">
      <h1 class="page-title">交换记录</h1>

      <div class="exchange-tabs">
        <div class="tab" :class="{ active: activeTab === 'negotiating' }" @click="activeTab = 'negotiating'">
          协商中 ({{ negotiatingExchanges.length }})
        </div>
        <div class="tab" :class="{ active: activeTab === 'active' }" @click="activeTab = 'active'">
          进行中 ({{ activeExchanges.length }})
        </div>
        <div class="tab" :class="{ active: activeTab === 'completed' }" @click="activeTab = 'completed'">
          已完成 ({{ completedExchanges.length }})
        </div>
      </div>

      <div v-if="activeTab === 'negotiating'" class="exchange-list">
        <div v-for="exchange in negotiatingExchanges" :key="exchange.id" class="exchange-card">
          <div class="exchange-header">
            <div class="exchange-users">
              <el-avatar :src="getUserAvatar(exchange.initiatorId)" :size="40" />
              <el-icon class="exchange-icon"><Switch /></el-icon>
              <el-avatar :src="getUserAvatar(exchange.partnerId)" :size="40" />
            </div>
            <div class="exchange-status negotiating">
              协商中 · 方案V{{ exchange.currentProposal?.version }}
            </div>
          </div>

          <div class="proposal-section" v-if="exchange.currentProposal">
            <div class="proposal-badge">
              <span class="version-tag">V{{ exchange.currentProposal.version }}</span>
              <span class="proposer-name">{{ getUserName(exchange.currentProposal.proposerId) }} 提出的方案</span>
              <span class="proposal-time">{{ formatTime(exchange.currentProposal.createdAt) }}</span>
            </div>
            <div class="proposal-detail">
              <div class="detail-row">
                <span class="detail-label">能教什么</span>
                <span class="skill-tag skill-teach">{{ exchange.currentProposal.canTeach }}</span>
              </div>
              <div class="detail-row">
                <span class="detail-label">想学什么</span>
                <span class="skill-tag skill-learn">{{ exchange.currentProposal.wantToLearn }}</span>
              </div>
              <div class="detail-row" v-if="exchange.currentProposal.learningGoal">
                <span class="detail-label">学习目标</span>
                <span class="detail-value">{{ exchange.currentProposal.learningGoal }}</span>
              </div>
              <div class="detail-row">
                <span class="detail-label">预计课次</span>
                <span class="detail-value">{{ exchange.currentProposal.expectedSessions }} 次</span>
              </div>
              <div class="detail-row" v-if="exchange.currentProposal.alternativeTimes?.length">
                <span class="detail-label">备选时间</span>
                <div class="time-tags">
                  <el-tag v-for="(t, i) in exchange.currentProposal.alternativeTimes" :key="i" size="small" type="info">{{ t }}</el-tag>
                </div>
              </div>
            </div>
          </div>

          <div class="confirm-status">
            <span :class="{ confirmed: exchange.confirmedBy?.includes(exchange.initiatorId), unconfirmed: !exchange.confirmedBy?.includes(exchange.initiatorId) }">
              {{ getUserName(exchange.initiatorId) }}：{{ exchange.confirmedBy?.includes(exchange.initiatorId) ? '已确认' : '待确认' }}
            </span>
            <span :class="{ confirmed: exchange.confirmedBy?.includes(exchange.partnerId), unconfirmed: !exchange.confirmedBy?.includes(exchange.partnerId) }">
              {{ getUserName(exchange.partnerId) }}：{{ exchange.confirmedBy?.includes(exchange.partnerId) ? '已确认' : '待确认' }}
            </span>
          </div>

          <div class="exchange-footer">
            <el-button size="small" text @click="toggleHistory(exchange.id)">
              {{ expandedHistory === exchange.id ? '收起历史' : `查看方案历史 (${exchange.proposals?.length || 0}版)` }}
            </el-button>
            <div class="exchange-actions">
              <el-button @click="openCounterDialog(exchange)" :disabled="exchange.currentProposal?.proposerId === myId">
                <el-icon><Edit /></el-icon>调整方案
              </el-button>
              <el-button type="primary" @click="confirmProposal(exchange)" :disabled="exchange.confirmedBy?.includes(myId)" :loading="confirmingId === exchange.id">
                <el-icon><Check /></el-icon>
                {{ exchange.confirmedBy?.includes(myId) ? '已确认此方案' : '确认此方案' }}
              </el-button>
              <el-button @click="goToChat(exchange)">
                <el-icon><ChatDotRound /></el-icon>联系对方
              </el-button>
            </div>
          </div>

          <div v-if="expandedHistory === exchange.id && exchange.proposals?.length > 1" class="proposal-history">
            <div class="history-title">方案变更历史</div>
            <el-timeline>
              <el-timeline-item v-for="p in exchange.proposals" :key="p.version" :timestamp="formatTime(p.createdAt)" placement="top">
                <div class="history-item">
                  <span class="version-tag">V{{ p.version }}</span>
                  <span class="proposer-name">{{ getUserName(p.proposerId) }}</span>
                  <div class="history-detail">
                    <span>教: {{ p.canTeach }}</span>
                    <span>学: {{ p.wantToLearn }}</span>
                    <span>{{ p.expectedSessions }}次</span>
                  </div>
                </div>
              </el-timeline-item>
            </el-timeline>
          </div>
        </div>
        <el-empty v-if="negotiatingExchanges.length === 0" description="暂无协商中的交换" />
      </div>

      <div v-if="activeTab === 'active'" class="exchange-list">
        <div v-for="exchange in activeExchanges" :key="exchange.id" class="exchange-card">
          <div class="exchange-header">
            <div class="exchange-users">
              <el-avatar :src="getUserAvatar(exchange.initiatorId)" :size="40" />
              <el-icon class="exchange-icon"><Switch /></el-icon>
              <el-avatar :src="getUserAvatar(exchange.partnerId)" :size="40" />
            </div>
            <div class="exchange-status active-status">
              进行中
            </div>
          </div>

          <div class="proposal-section" v-if="exchange.currentProposal">
            <div class="proposal-detail">
              <div class="detail-row">
                <span class="detail-label">能教什么</span>
                <span class="skill-tag skill-teach">{{ exchange.currentProposal.canTeach }}</span>
              </div>
              <div class="detail-row">
                <span class="detail-label">想学什么</span>
                <span class="skill-tag skill-learn">{{ exchange.currentProposal.wantToLearn }}</span>
              </div>
              <div class="detail-row" v-if="exchange.currentProposal.learningGoal">
                <span class="detail-label">学习目标</span>
                <span class="detail-value">{{ exchange.currentProposal.learningGoal }}</span>
              </div>
              <div class="detail-row">
                <span class="detail-label">预计课次</span>
                <span class="detail-value">{{ exchange.currentProposal.expectedSessions }} 次</span>
              </div>
              <div class="detail-row" v-if="exchange.currentProposal.alternativeTimes?.length">
                <span class="detail-label">备选时间</span>
                <div class="time-tags">
                  <el-tag v-for="(t, i) in exchange.currentProposal.alternativeTimes" :key="i" size="small" type="info">{{ t }}</el-tag>
                </div>
              </div>
            </div>
          </div>

          <div class="exchange-footer">
            <span class="exchange-time">开始于 {{ formatTime(exchange.activeAt) }}</span>
            <div class="exchange-actions">
              <el-button type="success" @click="completeExchange(exchange)" :loading="completingId === exchange.id">
                <el-icon><CircleCheck /></el-icon>标记完成
              </el-button>
              <el-button @click="goToChat(exchange)">
                <el-icon><ChatDotRound /></el-icon>联系对方
              </el-button>
            </div>
          </div>
        </div>
        <el-empty v-if="activeExchanges.length === 0" description="暂无进行中的交换" />
      </div>

      <div v-if="activeTab === 'completed'" class="exchange-list">
        <div v-for="exchange in completedExchanges" :key="exchange.id" class="exchange-card">
          <div class="exchange-header">
            <div class="exchange-users">
              <el-avatar :src="getUserAvatar(exchange.initiatorId)" :size="40" />
              <el-icon class="exchange-icon"><Switch /></el-icon>
              <el-avatar :src="getUserAvatar(exchange.partnerId)" :size="40" />
            </div>
            <div class="exchange-status completed">
              ✅ 已完成
            </div>
          </div>

          <div class="proposal-section" v-if="exchange.currentProposal">
            <div class="proposal-detail">
              <div class="detail-row">
                <span class="detail-label">教授</span>
                <span class="skill-tag skill-teach">{{ exchange.currentProposal.canTeach }}</span>
              </div>
              <div class="detail-row">
                <span class="detail-label">学习</span>
                <span class="skill-tag skill-learn">{{ exchange.currentProposal.wantToLearn }}</span>
              </div>
            </div>
          </div>

          <div class="exchange-footer">
            <span class="exchange-time">完成于 {{ formatTime(exchange.completedAt) }}</span>
            <div class="exchange-actions">
              <el-button type="success" @click="showReviewDialog(exchange)" :disabled="hasReviewed(exchange)">
                <el-icon><Star /></el-icon>
                {{ hasReviewed(exchange) ? '已评价' : '去评价' }}
              </el-button>
            </div>
          </div>
        </div>
        <el-empty v-if="completedExchanges.length === 0" description="暂无已完成的交换" />
      </div>
    </div>

    <el-dialog v-model="showReview" title="评价这次交换" width="500px">
      <div v-if="currentExchange" class="review-form">
        <div class="review-user">
          <el-avatar :src="getUserAvatar(getOtherUserId(currentExchange))" :size="56" />
          <span class="username">{{ getUserName(getOtherUserId(currentExchange)) }}</span>
        </div>
        <el-form :model="reviewForm" label-position="top">
          <el-form-item label="评分">
            <el-rate v-model="reviewForm.rating" size="large" />
          </el-form-item>
          <el-form-item label="评价内容">
            <el-input v-model="reviewForm.comment" type="textarea" :rows="4" placeholder="分享一下这次交换的体验..." maxlength="500" show-word-limit />
          </el-form-item>
        </el-form>
      </div>
      <template #footer>
        <el-button @click="showReview = false">取消</el-button>
        <el-button type="primary" @click="submitReview" :loading="submitting">提交评价</el-button>
      </template>
    </el-dialog>

    <el-dialog v-model="showCounterDialog" title="调整协商方案" width="560px">
      <el-form :model="counterForm" label-position="top">
        <el-form-item label="你能教什么" required>
          <el-input v-model="counterForm.canTeach" placeholder="填写你能教的技能" />
        </el-form-item>
        <el-form-item label="你想学什么" required>
          <el-input v-model="counterForm.wantToLearn" placeholder="填写你想学的技能" />
        </el-form-item>
        <el-form-item label="学习目标">
          <el-input v-model="counterForm.learningGoal" type="textarea" :rows="3" placeholder="描述你希望达到的学习效果..." maxlength="300" show-word-limit />
        </el-form-item>
        <el-form-item label="预计课次">
          <el-input-number v-model="counterForm.expectedSessions" :min="1" :max="50" />
        </el-form-item>
        <el-form-item label="备选时间">
          <div class="time-slots-input">
            <el-input v-model="newCounterTimeSlot" placeholder="例如：每周三晚8点" style="flex:1" @keyup.enter="addCounterTimeSlot" />
            <el-button type="primary" @click="addCounterTimeSlot">添加</el-button>
          </div>
          <div class="time-slots-list">
            <el-tag v-for="(t, i) in counterForm.alternativeTimes" :key="i" closable @close="removeCounterTimeSlot(i)" class="time-tag">
              {{ t }}
            </el-tag>
          </div>
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="showCounterDialog = false">取消</el-button>
        <el-button type="primary" @click="submitCounterProposal" :loading="submittingCounter">提交调整方案</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { useRouter } from 'vue-router'
import { exchangeAPI, reviewAPI, authAPI } from '../api'
import { useUserStore } from '../stores/user'
import { ElMessage } from 'element-plus'
import dayjs from 'dayjs'
import { Switch, Check, ChatDotRound, Star, Edit, CircleCheck } from '@element-plus/icons-vue'

const router = useRouter()
const userStore = useUserStore()
const myId = userStore.user?.id

const exchanges = ref([])
const users = ref({})
const activeTab = ref('negotiating')
const confirmingId = ref(null)
const completingId = ref(null)
const expandedHistory = ref(null)
const showReview = ref(false)
const currentExchange = ref(null)
const submitting = ref(false)
const reviewForm = ref({ rating: 5, comment: '' })

const showCounterDialog = ref(false)
const counterExchange = ref(null)
const newCounterTimeSlot = ref('')
const submittingCounter = ref(false)
const counterForm = ref({
  canTeach: '',
  wantToLearn: '',
  learningGoal: '',
  expectedSessions: 4,
  alternativeTimes: []
})

const negotiatingExchanges = computed(() =>
  exchanges.value.filter(e => e.status === 'negotiating')
)

const activeExchanges = computed(() =>
  exchanges.value.filter(e => e.status === 'active')
)

const completedExchanges = computed(() =>
  exchanges.value.filter(e => e.status === 'completed')
)

onMounted(async () => {
  await loadExchanges()
})

async function loadExchanges() {
  const res = await exchangeAPI.getExchanges()
  exchanges.value = res.data

  const userIds = new Set()
  exchanges.value.forEach(e => {
    userIds.add(e.initiatorId)
    userIds.add(e.partnerId)
  })

  for (const id of userIds) {
    if (!users.value[id]) {
      try {
        const userRes = await authAPI.getUser(id)
        users.value[id] = userRes.data
      } catch (e) {}
    }
  }
}

function getUserAvatar(id) {
  return users.value[id]?.avatar || ''
}

function getUserName(id) {
  return users.value[id]?.username || '未知用户'
}

function getOtherUserId(exchange) {
  return exchange.initiatorId === myId ? exchange.partnerId : exchange.initiatorId
}

function formatTime(time) {
  return dayjs(time).format('YYYY-MM-DD HH:mm')
}

function toggleHistory(id) {
  expandedHistory.value = expandedHistory.value === id ? null : id
}

async function confirmProposal(exchange) {
  try {
    confirmingId.value = exchange.id
    await exchangeAPI.confirmExchange(exchange.id)
    ElMessage.success('确认成功')
    await loadExchanges()
  } catch (e) {
    ElMessage.error(e.message || '确认失败')
  } finally {
    confirmingId.value = null
  }
}

async function completeExchange(exchange) {
  try {
    completingId.value = exchange.id
    await exchangeAPI.completeExchange(exchange.id)
    ElMessage.success('已标记完成')
    await loadExchanges()
  } catch (e) {
    ElMessage.error(e.message || '操作失败')
  } finally {
    completingId.value = null
  }
}

function goToChat(exchange) {
  const otherId = getOtherUserId(exchange)
  router.push(`/chat/${otherId}`)
}

function openCounterDialog(exchange) {
  counterExchange.value = exchange
  const cp = exchange.currentProposal || {}
  counterForm.value = {
    canTeach: cp.canTeach || '',
    wantToLearn: cp.wantToLearn || '',
    learningGoal: cp.learningGoal || '',
    expectedSessions: cp.expectedSessions || 4,
    alternativeTimes: cp.alternativeTimes ? [...cp.alternativeTimes] : []
  }
  newCounterTimeSlot.value = ''
  showCounterDialog.value = true
}

function addCounterTimeSlot() {
  const slot = newCounterTimeSlot.value.trim()
  if (!slot) return
  if (counterForm.value.alternativeTimes.includes(slot)) {
    ElMessage.warning('该时间已添加')
    return
  }
  counterForm.value.alternativeTimes.push(slot)
  newCounterTimeSlot.value = ''
}

function removeCounterTimeSlot(index) {
  counterForm.value.alternativeTimes.splice(index, 1)
}

async function submitCounterProposal() {
  if (!counterForm.value.canTeach) {
    ElMessage.warning('请填写你能教什么')
    return
  }
  if (!counterForm.value.wantToLearn) {
    ElMessage.warning('请填写你想学什么')
    return
  }
  try {
    submittingCounter.value = true
    await exchangeAPI.counterProposal(counterExchange.value.id, {
      canTeach: counterForm.value.canTeach,
      wantToLearn: counterForm.value.wantToLearn,
      learningGoal: counterForm.value.learningGoal,
      expectedSessions: counterForm.value.expectedSessions,
      alternativeTimes: counterForm.value.alternativeTimes
    })
    ElMessage.success('调整方案已提交')
    showCounterDialog.value = false
    await loadExchanges()
  } catch (e) {
    ElMessage.error(e.message || '提交失败')
  } finally {
    submittingCounter.value = false
  }
}

function hasReviewed(exchange) {
  return exchange.reviewedBy?.includes(myId)
}

function showReviewDialog(exchange) {
  currentExchange.value = exchange
  reviewForm.value = { rating: 5, comment: '' }
  showReview.value = true
}

async function submitReview() {
  if (!currentExchange.value) return
  if (!reviewForm.value.rating) {
    ElMessage.warning('请选择评分')
    return
  }

  try {
    submitting.value = true
    await reviewAPI.createReview({
      exchangeId: currentExchange.value.id,
      targetUserId: getOtherUserId(currentExchange.value),
      rating: reviewForm.value.rating,
      comment: reviewForm.value.comment
    })

    if (!currentExchange.value.reviewedBy) {
      currentExchange.value.reviewedBy = []
    }
    currentExchange.value.reviewedBy.push(myId)

    ElMessage.success('评价成功')
    showReview.value = false
  } catch (e) {
    ElMessage.error(e.message || '评价失败')
  } finally {
    submitting.value = false
  }
}
</script>

<style scoped>
.exchanges {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.exchange-tabs {
  display: flex;
  gap: 16px;
  margin-bottom: 24px;
  border-bottom: 2px solid #eee;
}

.tab {
  padding: 12px 24px;
  cursor: pointer;
  font-weight: 500;
  color: #999;
  border-bottom: 2px solid transparent;
  margin-bottom: -2px;
  transition: all 0.2s;
}

.tab.active {
  color: #667eea;
  border-bottom-color: #667eea;
}

.exchange-list {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.exchange-card {
  background: #fafafa;
  border-radius: 16px;
  padding: 24px;
  transition: all 0.3s;
  border: 2px solid transparent;
}

.exchange-card:hover {
  border-color: #667eea;
}

.exchange-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.exchange-users {
  display: flex;
  align-items: center;
  gap: 12px;
}

.exchange-icon {
  font-size: 24px;
  color: #667eea;
}

.exchange-status {
  padding: 6px 16px;
  border-radius: 20px;
  font-weight: 600;
  font-size: 14px;
}

.exchange-status.negotiating {
  background: #e6f7ff;
  color: #1890ff;
}

.exchange-status.active-status {
  background: #fff7e6;
  color: #fa8c16;
}

.exchange-status.completed {
  background: #f6ffed;
  color: #52c41a;
}

.proposal-section {
  background: white;
  border-radius: 12px;
  padding: 20px;
  margin-bottom: 16px;
}

.proposal-badge {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 16px;
  padding-bottom: 12px;
  border-bottom: 1px solid #f0f0f0;
}

.version-tag {
  display: inline-block;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 2px 10px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 600;
}

.proposer-name {
  font-size: 14px;
  color: #333;
  font-weight: 500;
}

.proposal-time {
  font-size: 12px;
  color: #999;
  margin-left: auto;
}

.proposal-detail {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.detail-row {
  display: flex;
  align-items: flex-start;
  gap: 12px;
}

.detail-label {
  min-width: 80px;
  color: #666;
  font-size: 13px;
  font-weight: 500;
  line-height: 28px;
}

.detail-value {
  color: #333;
  font-size: 14px;
  line-height: 28px;
}

.time-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.confirm-status {
  display: flex;
  gap: 24px;
  padding: 12px 0;
  margin-bottom: 8px;
  font-size: 14px;
}

.confirm-status .confirmed {
  color: #52c41a;
  font-weight: 500;
}

.confirm-status .unconfirmed {
  color: #999;
}

.exchange-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 8px;
}

.exchange-time {
  color: #999;
  font-size: 13px;
}

.exchange-actions {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.proposal-history {
  margin-top: 16px;
  padding-top: 16px;
  border-top: 1px dashed #e8e8e8;
}

.history-title {
  font-weight: 600;
  color: #333;
  margin-bottom: 16px;
  font-size: 15px;
}

.history-item {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}

.history-detail {
  display: flex;
  gap: 12px;
  font-size: 13px;
  color: #666;
}

.review-form {
  text-align: center;
}

.review-user {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  margin-bottom: 24px;
}

.review-user .username {
  font-weight: 600;
  font-size: 18px;
  color: #333;
}

.time-slots-input {
  display: flex;
  gap: 8px;
  margin-bottom: 8px;
}

.time-slots-list {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.time-tag {
  margin: 0;
}
</style>
