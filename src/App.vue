<template>
  <div class="app-shell">
    <header class="app-header">
      <button class="brand" @click="currentView = 'books'">
        <span class="brand-mark">书</span>
        <span>
          <strong>Bookstore</strong>
          <small>在线图书商城</small>
        </span>
      </button>

      <nav class="decor-nav" aria-label="商城特色">
        <span>精选书单</span>
        <span>新书上架</span>
        <span>订单可评价</span>
        <span>在线客服</span>
      </nav>

      <div class="account-area">
        <el-dropdown v-if="isLoggedIn" trigger="hover" @command="handleUserCommand">
          <button class="user-menu-button">
            <span class="avatar">{{ session.username.slice(0, 1).toUpperCase() }}</span>
            <span class="user-menu-text">
              <strong>{{ session.nickname || session.username }}</strong>
              <small v-if="session.role === 'ADMIN'">管理员</small>
              <small v-else>普通用户</small>
            </span>
            <span class="chevron">⌄</span>
          </button>
          <template #dropdown>
            <el-dropdown-menu>
              <el-dropdown-item command="books">商城首页</el-dropdown-item>
              <el-dropdown-item command="orders">{{ isAdmin ? '订单管理' : '我的订单' }}</el-dropdown-item>
              <el-dropdown-item command="support">{{ isAdmin ? '客服工单' : '客服中心' }}</el-dropdown-item>
              <el-dropdown-item command="reviews">我的评论</el-dropdown-item>
              <el-dropdown-item command="logout" divided>退出登录</el-dropdown-item>
            </el-dropdown-menu>
          </template>
        </el-dropdown>
        <el-button v-else round type="primary" @click="authDialog = true">登录 / 注册</el-button>
      </div>
    </header>

    <main class="main-content">
      <section v-if="currentView === 'books'" class="store-home">
        <div class="home-hero">
          <div class="hero-copy">
            <el-tag effect="light" round>今日推荐</el-tag>
            <h1>发现下一本值得读的书</h1>
            <p>搜索、筛选、加入购物车，再从右侧结算。订单完成后可以直接在订单详情里评价图书。</p>
            <div class="hero-search">
              <el-input v-model="bookSearch" size="large" placeholder="搜索书名、作者、分类" clearable @keyup.enter="searchBooks">
                <template #prefix>⌕</template>
              </el-input>
              <el-button type="primary" size="large" round @click="searchBooks">搜索</el-button>
            </div>
          </div>

          <div class="hero-showcase" aria-hidden="true">
            <div class="floating-book blue">
              <span>Java</span>
              <strong>后端开发</strong>
            </div>
            <div class="floating-book yellow">
              <span>Vue</span>
              <strong>前端设计</strong>
            </div>
            <div class="floating-book green">
              <span>AI</span>
              <strong>智能客服</strong>
            </div>
          </div>
        </div>

        <div class="category-strip">
          <button
            v-for="category in categories"
            :key="category"
            :class="{ active: activeCategory === category }"
            @click="selectCategory(category)"
          >
            {{ category }}
          </button>
        </div>

        <div class="section-title">
          <div>
            <h2>全部图书</h2>
            <p>共 {{ bookPage.total }} 本，当前第 {{ bookPage.pageNum }} / {{ bookPage.pages || 1 }} 页</p>
          </div>
          <el-button v-if="isAdmin" type="primary" round @click="bookDialog = true">新增图书</el-button>
        </div>

        <div class="catalog-layout">
          <div class="book-list">
            <article v-for="book in filteredBooks" :key="book.id" class="book-card">
              <div class="cover" :style="coverStyle(book)">
                <img v-if="book.coverUrl" :src="book.coverUrl" :alt="book.title" />
                <span v-else>{{ (book.title || '书').slice(0, 1) }}</span>
              </div>
              <div class="book-info">
                <div class="book-title-row">
                  <div>
                    <h3>{{ book.title }}</h3>
                    <p>{{ book.author || '未知作者' }}</p>
                  </div>
                  <el-tag size="small" effect="plain">{{ book.category || '未分类' }}</el-tag>
                </div>
                <p class="desc">{{ book.description || '暂无简介' }}</p>
                <div class="book-meta">
                  <strong>￥{{ money(book.price) }}</strong>
                  <span>库存 {{ book.stock ?? 0 }}</span>
                </div>
                <div class="book-actions">
                  <el-button type="primary" round @click="addToCart(book)">加入购物车</el-button>
                  <el-button round @click="openBookReviews(book)">查看评论</el-button>
                  <el-button v-if="isAdmin" text type="danger" @click="deleteBook(book)">删除</el-button>
                </div>
              </div>
            </article>
            <el-empty v-if="filteredBooks.length === 0" description="暂无图书，请确认后端已启动或点击刷新" />
            <el-pagination
              v-if="bookPage.total > 0"
              v-model:current-page="bookPage.pageNum"
              v-model:page-size="bookPage.pageSize"
              class="pagination-bar"
              background
              layout="total, sizes, prev, pager, next, jumper"
              :page-sizes="[6, 8, 12, 16]"
              :total="bookPage.total"
              @current-change="loadBooks"
              @size-change="handleBookPageSizeChange"
            />
          </div>

          <aside class="cart-card">
            <div class="side-title">
              <div>
                <h2>购物车</h2>
                <p>{{ cart.length }} 件商品</p>
              </div>
              <el-button v-if="cart.length" text @click="cart = []">清空</el-button>
            </div>

            <div v-if="cart.length" class="cart-list">
              <div v-for="item in cart" :key="item.book.id" class="cart-item">
                <div>
                  <strong>{{ item.book.title }}</strong>
                  <small>￥{{ money(item.book.price) }} × {{ item.quantity }}</small>
                </div>
                <div class="quantity-box">
                  <el-button circle size="small" @click="changeQuantity(item.book.id, -1)">-</el-button>
                  <span>{{ item.quantity }}</span>
                  <el-button circle size="small" @click="changeQuantity(item.book.id, 1)">+</el-button>
                </div>
              </div>
            </div>
            <el-empty v-else description="还没有选择图书" />

            <div class="cart-total">
              <span>合计</span>
              <strong>￥{{ money(cartTotal) }}</strong>
            </div>
            <el-button type="primary" round size="large" :disabled="cart.length === 0" @click="openCheckout">
              去结算
            </el-button>
          </aside>
        </div>
      </section>

      <section v-if="currentView === 'orders'" class="page-card">
        <div class="page-header">
          <div>
            <h1>{{ isAdmin ? '订单管理' : '我的订单' }}</h1>
            <p>{{ isAdmin ? '查看和更新用户订单状态。' : '查看订单进度，待支付订单可继续支付或取消。' }}</p>
          </div>
          <el-button round @click="loadOrders">刷新订单</el-button>
        </div>

        <el-table :data="orderRows" class="soft-table" row-key="id">
          <el-table-column type="expand">
            <template #default="{ row }">
              <div v-if="row.items && row.items.length" class="order-items">
                <div v-for="item in row.items" :key="item.id" class="order-line">
                  <span>{{ item.bookTitle }}</span>
                  <span>￥{{ money(item.price) }} × {{ item.quantity }}</span>
                  <strong>￥{{ money(item.subtotal) }}</strong>
                  <el-button
                    v-if="canReviewOrder(row)"
                    size="small"
                    type="primary"
                    plain
                    round
                    @click="openOrderItemReview(item)"
                  >
                    评价
                  </el-button>
                </div>
              </div>
              <p v-else class="muted">暂无订单明细。</p>
            </template>
          </el-table-column>
          <el-table-column prop="id" label="订单号" width="90" />
          <el-table-column prop="receiverName" label="收货人" min-width="120" />
          <el-table-column prop="receiverPhone" label="电话" min-width="140" />
          <el-table-column label="金额" width="120">
            <template #default="{ row }">￥{{ money(row.totalAmount) }}</template>
          </el-table-column>
          <el-table-column label="状态" width="120">
            <template #default="{ row }">
              <el-tag :type="statusType(row.status)" effect="light">{{ statusText(row.status) }}</el-tag>
            </template>
          </el-table-column>
          <el-table-column label="操作" width="240">
            <template #default="{ row }">
              <el-button v-if="!isAdmin && row.status === 'PENDING'" size="small" type="primary" @click="payOrder(row)">
                支付
              </el-button>
              <el-button v-if="!isAdmin && row.status === 'PENDING'" size="small" @click="cancelOrder(row)">
                取消
              </el-button>
              <el-select
                v-if="isAdmin"
                :model-value="row.status"
                size="small"
                class="status-select"
                @change="value => updateOrderStatus(row, value)"
              >
                <el-option label="待支付" value="PENDING" />
                <el-option label="已支付" value="PAID" />
                <el-option label="已发货" value="SHIPPED" />
                <el-option label="已完成" value="COMPLETED" />
                <el-option label="已取消" value="CANCELLED" />
              </el-select>
            </template>
          </el-table-column>
        </el-table>
      </section>

      <section v-if="currentView === 'reviews'" class="page-card">
        <div class="page-header">
          <div>
            <h1>我的评论</h1>
            <p>管理你发布过的图书评价。</p>
          </div>
          <el-button round @click="loadMyReviews">刷新评论</el-button>
        </div>
        <div class="review-list">
          <article v-for="review in myReviews" :key="review.id" class="review-card">
            <el-rate :model-value="review.rating" disabled />
            <p>{{ review.content }}</p>
            <small>图书 #{{ review.bookId }} · {{ review.status }}</small>
            <el-button text type="danger" @click="deleteReview(review)">删除</el-button>
          </article>
          <el-empty v-if="myReviews.length === 0" description="暂无评论" />
        </div>
      </section>

      <section v-if="currentView === 'support'" class="page-card">
        <div class="page-header">
          <div>
            <h1>{{ isAdmin ? '客服工单' : '客服中心' }}</h1>
            <p>{{ isAdmin ? '查看用户问题并进行回复。' : '提交问题并查看客服回复。' }}</p>
          </div>
          <div class="header-actions">
            <el-button v-if="!isAdmin" type="primary" round @click="ticketDialog = true">新建工单</el-button>
            <el-button round @click="refreshSupportCenter">刷新</el-button>
          </div>
        </div>

        <div class="faq-panel">
          <div class="side-title">
            <div>
              <h2>客服知识库</h2>
              <p>常见问题会同步提供给智能客服作为回答依据。</p>
            </div>
            <el-input v-model="faqSearch" clearable placeholder="搜索 FAQ" @keyup.enter="loadFaqs" />
            <el-button round @click="loadFaqs">搜索</el-button>
          </div>
          <div class="faq-list">
            <article v-for="faq in faqs" :key="faq.id" class="faq-card">
              <el-tag size="small" effect="light">{{ faq.category }}</el-tag>
              <strong>{{ faq.question }}</strong>
              <p>{{ faq.answer }}</p>
            </article>
            <el-empty v-if="faqs.length === 0" description="暂无知识库内容" />
          </div>
        </div>

        <div class="support-layout">
          <div class="ticket-list">
            <article
              v-for="ticket in ticketRows"
              :key="ticket.id"
              class="ticket-card"
              :class="{ active: activeTicket?.id === ticket.id }"
              @click="openTicket(ticket)"
            >
              <div>
                <strong>{{ ticket.subject }}</strong>
                <small>{{ ticket.username }} · #{{ ticket.id }}</small>
              </div>
              <el-tag :type="ticket.status === 'CLOSED' ? 'info' : 'success'" effect="light">
                {{ ticket.status }}
              </el-tag>
            </article>
            <el-empty v-if="ticketRows.length === 0" description="暂无工单" />
          </div>

          <div class="message-board">
            <template v-if="activeTicket">
              <div class="message-head">
                <div>
                  <h2>{{ activeTicket.subject }}</h2>
                  <p>工单 #{{ activeTicket.id }}</p>
                </div>
                <el-select
                  v-if="isAdmin"
                  :model-value="activeTicket.status"
                  size="large"
                  class="status-select"
                  @change="value => updateTicketStatus(value)"
                >
                  <el-option label="OPEN" value="OPEN" />
                  <el-option label="IN_PROGRESS" value="IN_PROGRESS" />
                  <el-option label="CLOSED" value="CLOSED" />
                </el-select>
              </div>
              <div class="messages">
                <div
                  v-for="message in activeMessages"
                  :key="message.id"
                  class="message-bubble"
                  :class="{ admin: message.senderRole === 'ADMIN' }"
                >
                  <small>{{ message.senderUsername }} · {{ message.senderRole }}</small>
                  <p>{{ message.content }}</p>
                </div>
              </div>
              <div class="reply-box">
                <el-input v-model="replyContent" type="textarea" :rows="3" placeholder="输入回复内容" />
                <el-button type="primary" round @click="replyTicket">发送</el-button>
              </div>
            </template>
            <el-empty v-else description="选择一个工单查看对话" />
          </div>
        </div>
      </section>
    </main>

    <div class="ai-chat-widget" :class="{ open: aiChatOpen }">
      <button v-if="!aiChatOpen" class="ai-chat-fab" @click="openAiChat">
        <span>AI</span>
        <strong>智能客服</strong>
      </button>

      <section v-else class="ai-chat-panel" aria-label="智能客服">
        <div class="ai-chat-head">
          <div>
            <strong>Bookstore 智能客服</strong>
            <small>{{ aiChatLoading ? '正在整理推荐...' : '可咨询选书、订单、评论入口' }}</small>
          </div>
          <el-button text circle @click="aiChatOpen = false">×</el-button>
        </div>

        <div ref="aiMessageList" class="ai-chat-messages">
          <div
            v-for="(message, index) in aiMessages"
            :key="index"
            class="ai-chat-message"
            :class="message.role"
          >
            <div class="ai-message-bubble">
              <p>{{ message.content }}</p>
              <small v-if="message.source === 'LOCAL_FALLBACK'">本地书库推荐</small>
            </div>
            <div v-if="message.toolResults?.length" class="ai-tool-results">
              <span v-for="result in message.toolResults" :key="`${index}-${result.type}`">
                <strong>{{ result.title }}</strong>
                <small>{{ result.content }}</small>
              </span>
            </div>
            <div v-if="message.books?.length" class="ai-book-suggestions">
              <article v-for="book in message.books" :key="book.id" class="ai-book-card">
                <div>
                  <strong>{{ book.title }}</strong>
                  <small>{{ book.category || '未分类' }} · ￥{{ money(book.price) }} · 库存 {{ book.stock ?? 0 }}</small>
                </div>
                <el-button size="small" round type="primary" :disabled="!book.stock" @click="addAiBookToCart(book)">
                  加入购物车
                </el-button>
              </article>
            </div>
            <div v-if="message.faqs?.length" class="ai-faq-suggestions">
              <article v-for="faq in message.faqs" :key="faq.id" class="ai-faq-card">
                <small>{{ faq.category }}</small>
                <strong>{{ faq.question }}</strong>
                <p>{{ faq.answer }}</p>
              </article>
            </div>
            <div v-if="message.toolActions?.length" class="ai-tool-actions">
              <el-button
                v-for="action in message.toolActions"
                :key="`${index}-${action.type}-${action.label}`"
                size="small"
                round
                type="primary"
                plain
                @click="executeAiAction(action, message)"
              >
                {{ action.label }}
              </el-button>
            </div>
          </div>
          <div v-if="aiChatLoading" class="ai-chat-message assistant">
            <div class="ai-message-bubble typing">
              <span></span>
              <span></span>
              <span></span>
            </div>
          </div>
        </div>

        <div class="ai-quick-questions">
          <button v-for="question in aiQuickQuestions" :key="question" @click="askQuickAi(question)">
            {{ question }}
          </button>
        </div>

        <div class="ai-chat-input">
          <el-input
            v-model="aiChatInput"
            type="textarea"
            :autosize="{ minRows: 2, maxRows: 4 }"
            placeholder="例如：我最近想学 Java 开发，买什么书？"
            @keydown.enter.prevent="sendAiMessage()"
          />
          <el-button type="primary" round :loading="aiChatLoading" @click="sendAiMessage()">发送</el-button>
        </div>
      </section>
    </div>

    <el-dialog v-model="authDialog" width="420px" class="rounded-dialog" title="账号登录">
      <el-tabs v-model="authMode" stretch>
        <el-tab-pane label="用户登录" name="user" />
        <el-tab-pane label="用户注册" name="register" />
        <el-tab-pane label="管理员" name="admin" />
      </el-tabs>
      <el-form label-position="top">
        <el-form-item label="用户名">
          <el-input v-model="authForm.username" size="large" placeholder="请输入用户名" />
        </el-form-item>
        <el-form-item label="密码">
          <el-input v-model="authForm.password" size="large" type="password" show-password placeholder="请输入密码" />
        </el-form-item>
        <el-form-item v-if="authMode === 'register'" label="昵称">
          <el-input v-model="authForm.nickname" size="large" placeholder="可选" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button round @click="authDialog = false">取消</el-button>
        <el-button type="primary" round @click="submitAuth">
          {{ authMode === 'register' ? '注册' : '登录' }}
        </el-button>
      </template>
    </el-dialog>

    <el-dialog v-model="bookDialog" width="560px" class="rounded-dialog" title="新增图书">
      <el-form label-position="top" class="two-column-form">
        <el-form-item label="书名">
          <el-input v-model="bookForm.title" />
        </el-form-item>
        <el-form-item label="作者">
          <el-input v-model="bookForm.author" />
        </el-form-item>
        <el-form-item label="分类">
          <el-input v-model="bookForm.category" />
        </el-form-item>
        <el-form-item label="价格">
          <el-input-number v-model="bookForm.price" :min="0" :precision="2" />
        </el-form-item>
        <el-form-item label="库存">
          <el-input-number v-model="bookForm.stock" :min="0" />
        </el-form-item>
        <el-form-item label="封面 URL">
          <el-input v-model="bookForm.coverUrl" />
        </el-form-item>
        <el-form-item label="简介" class="span-2">
          <el-input v-model="bookForm.description" type="textarea" :rows="3" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button round @click="bookDialog = false">取消</el-button>
        <el-button type="primary" round @click="submitBook">保存</el-button>
      </template>
    </el-dialog>

    <el-dialog v-model="checkoutDialog" width="520px" class="rounded-dialog" title="确认订单">
      <el-form label-position="top">
        <el-form-item label="收货人">
          <el-input v-model="checkoutForm.receiverName" />
        </el-form-item>
        <el-form-item label="联系电话">
          <el-input v-model="checkoutForm.receiverPhone" />
        </el-form-item>
        <el-form-item label="收货地址">
          <el-input v-model="checkoutForm.receiverAddress" type="textarea" :rows="3" />
        </el-form-item>
      </el-form>
      <div class="checkout-summary">
        <span>{{ cart.length }} 件商品</span>
        <strong>￥{{ money(cartTotal) }}</strong>
      </div>
      <template #footer>
        <el-button round @click="checkoutDialog = false">取消</el-button>
        <el-button type="primary" round @click="submitOrder">提交订单</el-button>
      </template>
    </el-dialog>

    <el-dialog v-model="reviewDialog.visible" width="680px" class="rounded-dialog" :title="reviewDialog.book?.title || '图书评论'">
      <div class="review-summary">
        <span>平均评分</span>
        <el-rate :model-value="Number(reviewDialog.averageRating)" disabled allow-half />
        <strong>{{ Number(reviewDialog.averageRating || 0).toFixed(1) }}</strong>
      </div>
      <div class="review-dialog-list">
        <article v-for="review in reviewDialog.reviews" :key="review.id" class="review-card">
          <el-rate :model-value="review.rating" disabled />
          <p>{{ review.content }}</p>
          <small>{{ review.username }}</small>
        </article>
        <el-empty v-if="reviewDialog.reviews.length === 0" description="这本书还没有评论" />
      </div>
      <el-divider />
      <el-form label-position="top">
        <el-form-item label="我的评分">
          <el-rate v-model="reviewForm.rating" />
        </el-form-item>
        <el-form-item label="评论内容">
          <el-input v-model="reviewForm.content" type="textarea" :rows="3" placeholder="购买并支付后可以评论" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button round @click="reviewDialog.visible = false">关闭</el-button>
        <el-button type="primary" round @click="submitReview">发布评论</el-button>
      </template>
    </el-dialog>

    <el-dialog v-model="ticketDialog" width="520px" class="rounded-dialog" title="新建客服工单">
      <el-form label-position="top">
        <el-form-item label="问题主题">
          <el-input v-model="ticketForm.subject" />
        </el-form-item>
        <el-form-item label="问题描述">
          <el-input v-model="ticketForm.content" type="textarea" :rows="4" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button round @click="ticketDialog = false">取消</el-button>
        <el-button type="primary" round @click="submitTicket">提交</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { computed, nextTick, onMounted, reactive, ref, watch } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'

const storedSession = JSON.parse(localStorage.getItem('bookstore-session') || '{}')
const session = reactive({
  username: storedSession.username || '',
  role: storedSession.role || '',
  nickname: storedSession.nickname || ''
})

const currentView = ref('books')
const books = ref([])
const allCategories = ref([])
const bookSearch = ref('')
const activeCategory = ref('全部')
const bookPage = reactive({
  pageNum: 1,
  pageSize: 8,
  total: 0,
  pages: 0
})
const cart = ref([])
const myOrders = ref([])
const adminOrders = ref([])
const myReviews = ref([])
const myTickets = ref([])
const adminTickets = ref([])
const faqs = ref([])
const faqSearch = ref('')
const activeTicket = ref(null)
const activeMessages = ref([])
const replyContent = ref('')
const aiChatOpen = ref(false)
const aiChatInput = ref('')
const aiChatLoading = ref(false)
const aiMessageList = ref(null)
const aiQuickQuestions = ['Java 后端怎么入门？', '推荐几本算法书', '订单怎么评价？']
const aiMessages = ref([
  {
    role: 'assistant',
    content: '你好，我是 Bookstore 智能客服。你可以问我选书建议、学习路线、订单和评论入口。'
  }
])

const authDialog = ref(false)
const authMode = ref('user')
const authForm = reactive({ username: '', password: '', nickname: '' })
const bookDialog = ref(false)
const checkoutDialog = ref(false)
const ticketDialog = ref(false)
const bookForm = reactive({
  title: '',
  author: '',
  category: '',
  price: 0,
  stock: 0,
  coverUrl: '',
  description: ''
})
const checkoutForm = reactive({ receiverName: '', receiverPhone: '', receiverAddress: '' })
const ticketForm = reactive({ subject: '', content: '' })
const reviewForm = reactive({ rating: 5, content: '' })
const reviewDialog = reactive({
  visible: false,
  book: null,
  reviews: [],
  averageRating: 0
})

const isLoggedIn = computed(() => Boolean(session.username))
const isAdmin = computed(() => session.role === 'ADMIN')

const categories = computed(() => {
  return ['全部', ...allCategories.value]
})

const filteredBooks = computed(() => books.value)

const cartTotal = computed(() =>
  cart.value.reduce((sum, item) => sum + Number(item.book.price || 0) * item.quantity, 0)
)

const myOrderRows = computed(() =>
  myOrders.value.map(entry => ({
    ...(entry.order || {}),
    items: entry.items || []
  }))
)

const orderRows = computed(() => {
  if (isAdmin.value) {
    return adminOrders.value.map(order => ({ ...order, items: [] }))
  }
  return myOrderRows.value
})

const ticketRows = computed(() => (isAdmin.value ? adminTickets.value : myTickets.value))

watch(
  session,
  value => {
    if (value.username) {
      localStorage.setItem('bookstore-session', JSON.stringify(value))
    } else {
      localStorage.removeItem('bookstore-session')
    }
  },
  { deep: true }
)

onMounted(async () => {
  await Promise.all([loadCategories(), loadBooks(), loadFaqs()])
  if (isLoggedIn.value) {
    await Promise.all([loadOrders(), loadMyReviews(), loadTickets()])
  }
})

async function api(path, options = {}) {
  const response = await fetch(path, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...(options.headers || {})
    },
    body: options.body && typeof options.body !== 'string' ? JSON.stringify(options.body) : options.body
  })
  if (!response.ok) {
    const text = await response.text()
    throw new Error(text || response.statusText)
  }
  return response.json()
}

async function loadBooks() {
  try {
    const params = new URLSearchParams({
      pageNum: String(bookPage.pageNum),
      pageSize: String(bookPage.pageSize)
    })
    const keyword = bookSearch.value.trim()
    if (keyword) {
      params.set('keyword', keyword)
    }
    if (activeCategory.value && activeCategory.value !== '全部') {
      params.set('category', activeCategory.value)
    }
    const result = await api(`/api/books?${params.toString()}`)
    books.value = result.list || []
    bookPage.total = Number(result.total || 0)
    bookPage.pageNum = Number(result.pageNum || bookPage.pageNum)
    bookPage.pageSize = Number(result.pageSize || bookPage.pageSize)
    bookPage.pages = Number(result.pages || 0)
  } catch (error) {
    books.value = []
    bookPage.total = 0
    ElMessage.warning(`后端接口暂不可用：${readError(error)}`)
  }
}

async function loadCategories() {
  try {
    allCategories.value = await api('/api/books/categories')
  } catch {
    allCategories.value = []
  }
}

async function searchBooks() {
  bookPage.pageNum = 1
  await loadBooks()
}

async function handleBookPageSizeChange() {
  bookPage.pageNum = 1
  await loadBooks()
}

async function submitAuth() {
  if (!authForm.username || !authForm.password) {
    ElMessage.warning('请输入用户名和密码')
    return
  }
  const path =
    authMode.value === 'admin'
      ? '/api/admin/login'
      : authMode.value === 'register'
        ? '/api/user/register'
        : '/api/user/login'
  const result = await api(path, {
    method: 'POST',
    body: {
      username: authForm.username,
      password: authForm.password,
      nickname: authForm.nickname
    }
  })
  if (!result.success) {
    ElMessage.error(result.message || '操作失败')
    return
  }
  if (authMode.value === 'register') {
    ElMessage.success('注册成功，请登录')
    authMode.value = 'user'
    return
  }
  session.username = result.username || authForm.username
  session.role = result.role || (authMode.value === 'admin' ? 'ADMIN' : 'USER')
  session.nickname = result.nickname || result.username || authForm.username
  authDialog.value = false
  ElMessage.success('登录成功')
  await Promise.all([loadOrders(), loadMyReviews(), loadTickets()])
}

function logout() {
  session.username = ''
  session.role = ''
  session.nickname = ''
  cart.value = []
  myOrders.value = []
  adminOrders.value = []
  myReviews.value = []
  myTickets.value = []
  adminTickets.value = []
  activeTicket.value = null
  activeMessages.value = []
  currentView.value = 'books'
}

function requireLogin() {
  if (isLoggedIn.value) {
    return true
  }
  authDialog.value = true
  ElMessage.info('请先登录')
  return false
}

async function handleUserCommand(command) {
  if (command === 'books') {
    currentView.value = 'books'
    await loadBooks()
    return
  }
  if (command === 'orders') {
    await openOrders()
    return
  }
  if (command === 'support') {
    await openSupport()
    return
  }
  if (command === 'reviews') {
    await openReviewsHub()
    return
  }
  if (command === 'logout') {
    logout()
  }
}

async function openAiChat() {
  aiChatOpen.value = true
  await scrollAiMessages()
}

async function askQuickAi(question) {
  if (aiChatLoading.value) {
    return
  }
  await sendAiMessage(question)
}

async function sendAiMessage(presetQuestion = '') {
  const question = (presetQuestion || aiChatInput.value).trim()
  if (!question || aiChatLoading.value) {
    return
  }
  aiChatOpen.value = true
  aiChatInput.value = ''
  aiMessages.value.push({ role: 'user', content: question })
  aiChatLoading.value = true
  await scrollAiMessages()

  try {
    const history = aiMessages.value
      .slice(-9, -1)
      .filter(message => ['user', 'assistant'].includes(message.role))
      .map(message => ({ role: message.role, content: message.content }))
    const params = new URLSearchParams()
    if (session.username) {
      params.set('username', session.username)
    }
    params.set('role', session.role || 'USER')
    const result = await api(`/api/ai/chat?${params.toString()}`, {
      method: 'POST',
      body: {
        message: question,
        history
      }
    })
    aiMessages.value.push({
      role: 'assistant',
      content: result.reply || result.message || '我暂时没有整理出合适答案，请稍后再试。',
      books: result.recommendedBooks || [],
      faqs: result.matchedFaqs || [],
      toolActions: result.toolActions || [],
      toolResults: result.toolResults || [],
      source: result.source
    })
  } catch (error) {
    aiMessages.value.push({
      role: 'assistant',
      content: `智能客服暂时连接不上：${readError(error)}。你可以先用搜索框查找图书，或到客服中心提交工单。`
    })
  } finally {
    aiChatLoading.value = false
    await scrollAiMessages()
  }
}

function addAiBookToCart(book) {
  addToCart(book)
}

async function executeAiAction(action, message) {
  if (!action) {
    return
  }
  if (action.type === 'CREATE_ORDER_DRAFT') {
    if (!requireLogin()) {
      return
    }
    if (action.confirmRequired) {
      await ElMessageBox.confirm('将推荐图书加入购物车，并打开结算窗口。是否继续？', '确认推荐订单', {
        type: 'info'
      })
    }
    const items = action.payload?.items || []
    const added = addAiItemsToCart(items, message.books || [])
    if (added > 0) {
      ElMessage.success(`已加入 ${added} 本推荐图书`)
      checkoutDialog.value = true
    }
    return
  }
  if (action.type === 'CREATE_SUPPORT_TICKET') {
    if (!requireLogin()) {
      return
    }
    await ElMessageBox.confirm('将根据这次咨询内容创建客服工单，管理员可在后台回复。是否继续？', '创建客服工单', {
      type: 'info'
    })
    const result = await api(`/api/support/tickets?username=${encodeURIComponent(session.username)}`, {
      method: 'POST',
      body: {
        subject: action.payload?.subject || '智能客服转人工',
        content: action.payload?.content || message.content
      }
    })
    if (!result.success) {
      ElMessage.error(result.message || '创建工单失败')
      return
    }
    ElMessage.success(`客服工单 #${result.ticketId} 已创建`)
    await openSupport()
    return
  }
  if (action.type === 'OPEN_ORDERS') {
    await openOrders()
    return
  }
  if (action.type === 'OPEN_SUPPORT') {
    await openSupport()
  }
}

function addAiItemsToCart(items, suggestedBooks = []) {
  let added = 0
  for (const item of items) {
    const book = findAiBook(item, suggestedBooks)
    if (!book || !book.stock) {
      continue
    }
    const existing = cart.value.find(entry => entry.book.id === book.id)
    const quantity = Math.max(1, Number(item.quantity || 1))
    if (existing) {
      if (existing.quantity + quantity <= book.stock) {
        existing.quantity += quantity
        added += 1
      }
    } else {
      cart.value.push({ book, quantity: Math.min(quantity, book.stock) })
      added += 1
    }
  }
  if (added === 0) {
    ElMessage.warning('推荐图书暂无库存或已在购物车中')
  }
  return added
}

function findAiBook(item, suggestedBooks) {
  const bookId = Number(item.bookId)
  return (
    suggestedBooks.find(book => Number(book.id) === bookId) ||
    books.value.find(book => Number(book.id) === bookId) || {
      id: bookId,
      title: item.title,
      price: item.price,
      stock: item.stock,
      category: item.category
    }
  )
}

async function scrollAiMessages() {
  await nextTick()
  if (aiMessageList.value) {
    aiMessageList.value.scrollTop = aiMessageList.value.scrollHeight
  }
}

async function selectCategory(category) {
  activeCategory.value = category
  await searchBooks()
}

async function submitBook() {
  if (!bookForm.title) {
    ElMessage.warning('请输入书名')
    return
  }
  const result = await api(`/api/books?username=${encodeURIComponent(session.username)}&role=${session.role}`, {
    method: 'POST',
    body: { ...bookForm }
  })
  if (!result) {
    ElMessage.error('新增失败')
    return
  }
  ElMessage.success('已新增图书')
  bookDialog.value = false
  Object.assign(bookForm, { title: '', author: '', category: '', price: 0, stock: 0, coverUrl: '', description: '' })
  await Promise.all([loadCategories(), loadBooks()])
}

async function deleteBook(book) {
  await ElMessageBox.confirm(`确定删除《${book.title}》吗？`, '删除图书', { type: 'warning' })
  const result = await api(`/api/books/${book.id}?username=${encodeURIComponent(session.username)}&role=${session.role}`, {
    method: 'DELETE'
  })
  if (!result) {
    ElMessage.error('删除失败')
    return
  }
  ElMessage.success('已删除')
  await Promise.all([loadCategories(), loadBooks()])
}

function addToCart(book) {
  if (!requireLogin()) {
    return
  }
  if (!book.stock || book.stock <= 0) {
    ElMessage.warning('库存不足')
    return
  }
  const item = cart.value.find(entry => entry.book.id === book.id)
  if (item) {
    if (item.quantity >= book.stock) {
      ElMessage.warning('不能超过库存数量')
      return
    }
    item.quantity += 1
  } else {
    cart.value.push({ book, quantity: 1 })
  }
}

function changeQuantity(bookId, delta) {
  const item = cart.value.find(entry => entry.book.id === bookId)
  if (!item) {
    return
  }
  const next = item.quantity + delta
  if (next <= 0) {
    cart.value = cart.value.filter(entry => entry.book.id !== bookId)
    return
  }
  if (next > item.book.stock) {
    ElMessage.warning('不能超过库存数量')
    return
  }
  item.quantity = next
}

function openCheckout() {
  if (!requireLogin()) {
    return
  }
  checkoutDialog.value = true
}

async function submitOrder() {
  if (!checkoutForm.receiverName || !checkoutForm.receiverPhone || !checkoutForm.receiverAddress) {
    ElMessage.warning('请完善收货信息')
    return
  }
  const result = await api(`/api/orders?username=${encodeURIComponent(session.username)}`, {
    method: 'POST',
    body: {
      items: cart.value.map(item => ({ bookId: item.book.id, quantity: item.quantity })),
      ...checkoutForm
    }
  })
  if (!result.success) {
    ElMessage.error(result.message || '提交订单失败')
    return
  }
  ElMessage.success(`订单 #${result.orderId} 已创建`)
  checkoutDialog.value = false
  cart.value = []
  Object.assign(checkoutForm, { receiverName: '', receiverPhone: '', receiverAddress: '' })
  await Promise.all([loadBooks(), loadOrders()])
  currentView.value = 'orders'
}

async function openOrders() {
  if (!requireLogin()) {
    return
  }
  currentView.value = 'orders'
  await loadOrders()
}

async function loadOrders() {
  if (!isLoggedIn.value) {
    return
  }
  try {
    myOrders.value = await api(`/api/orders/my?username=${encodeURIComponent(session.username)}`)
    if (isAdmin.value) {
      adminOrders.value = await api('/api/orders?role=ADMIN')
    }
  } catch (error) {
    ElMessage.warning(`订单接口暂不可用：${readError(error)}`)
  }
}

async function payOrder(row) {
  const result = await api(`/api/orders/${row.id}/pay?username=${encodeURIComponent(session.username)}`, { method: 'POST' })
  result.success ? ElMessage.success('支付成功') : ElMessage.error(result.message || '支付失败')
  await loadOrders()
}

async function cancelOrder(row) {
  const result = await api(`/api/orders/${row.id}/cancel?username=${encodeURIComponent(session.username)}`, { method: 'POST' })
  result.success ? ElMessage.success('订单已取消') : ElMessage.error(result.message || '取消失败')
  await Promise.all([loadOrders(), loadBooks()])
}

async function updateOrderStatus(row, status) {
  const result = await api(`/api/orders/${row.id}/status?role=ADMIN`, {
    method: 'PUT',
    body: { status }
  })
  result.success ? ElMessage.success('订单状态已更新') : ElMessage.error(result.message || '更新失败')
  await Promise.all([loadOrders(), loadBooks()])
}

async function openReviewsHub() {
  if (!requireLogin()) {
    return
  }
  currentView.value = 'reviews'
  await loadMyReviews()
}

async function loadMyReviews() {
  if (!isLoggedIn.value) {
    return
  }
  try {
    myReviews.value = await api(`/api/reviews/my?username=${encodeURIComponent(session.username)}`)
  } catch (error) {
    ElMessage.warning(`评论接口暂不可用：${readError(error)}`)
  }
}

async function openBookReviews(book) {
  reviewDialog.book = book
  reviewDialog.visible = true
  await refreshBookReviews(book.id)
}

async function openOrderItemReview(item) {
  if (!requireLogin()) {
    return
  }
  reviewDialog.book = {
    id: item.bookId,
    title: item.bookTitle
  }
  reviewDialog.visible = true
  reviewForm.rating = 5
  reviewForm.content = ''
  await refreshBookReviews(item.bookId)
}

function canReviewOrder(order) {
  return !isAdmin.value && ['PAID', 'SHIPPED', 'COMPLETED'].includes(order.status)
}

async function refreshBookReviews(bookId) {
  const result = await api(`/api/reviews/book/${bookId}`)
  reviewDialog.averageRating = result.averageRating || 0
  reviewDialog.reviews = result.reviews || []
}

async function submitReview() {
  if (!requireLogin()) {
    return
  }
  if (!reviewDialog.book) {
    return
  }
  const result = await api(`/api/reviews?username=${encodeURIComponent(session.username)}`, {
    method: 'POST',
    body: {
      bookId: reviewDialog.book.id,
      rating: reviewForm.rating,
      content: reviewForm.content
    }
  })
  if (!result.success) {
    ElMessage.error(result.message || '评论失败')
    return
  }
  ElMessage.success('评论已发布')
  reviewForm.rating = 5
  reviewForm.content = ''
  await Promise.all([refreshBookReviews(reviewDialog.book.id), loadMyReviews()])
}

async function deleteReview(review) {
  const result = await api(`/api/reviews/${review.id}?username=${encodeURIComponent(session.username)}&role=${session.role || 'USER'}`, {
    method: 'DELETE'
  })
  result.success ? ElMessage.success('已删除评论') : ElMessage.error(result.message || '删除失败')
  await loadMyReviews()
}

async function openSupport() {
  if (!requireLogin()) {
    return
  }
  currentView.value = 'support'
  await refreshSupportCenter()
}

async function refreshSupportCenter() {
  await Promise.all([loadTickets(), loadFaqs()])
}

async function loadFaqs() {
  try {
    const params = new URLSearchParams()
    const keyword = faqSearch.value.trim()
    if (keyword) {
      params.set('keyword', keyword)
    }
    params.set('role', session.role || 'USER')
    faqs.value = await api(`/api/faqs?${params.toString()}`)
  } catch (error) {
    faqs.value = []
    ElMessage.warning(`知识库接口暂不可用：${readError(error)}`)
  }
}

async function loadTickets() {
  if (!isLoggedIn.value) {
    return
  }
  try {
    if (isAdmin.value) {
      adminTickets.value = await api('/api/support/tickets?role=ADMIN')
    } else {
      myTickets.value = await api(`/api/support/tickets/my?username=${encodeURIComponent(session.username)}`)
    }
  } catch (error) {
    ElMessage.warning(`客服接口暂不可用：${readError(error)}`)
  }
}

async function submitTicket() {
  if (!ticketForm.subject || !ticketForm.content) {
    ElMessage.warning('请填写问题主题和描述')
    return
  }
  const result = await api(`/api/support/tickets?username=${encodeURIComponent(session.username)}`, {
    method: 'POST',
    body: { ...ticketForm }
  })
  if (!result.success) {
    ElMessage.error(result.message || '提交失败')
    return
  }
  ElMessage.success('客服工单已创建')
  ticketDialog.value = false
  Object.assign(ticketForm, { subject: '', content: '' })
  await loadTickets()
}

async function openTicket(ticket) {
  const result = await api(
    `/api/support/tickets/${ticket.id}?username=${encodeURIComponent(session.username)}&role=${session.role || 'USER'}`
  )
  if (!result.success) {
    ElMessage.error(result.message || '无法打开工单')
    return
  }
  activeTicket.value = result.ticket
  activeMessages.value = result.messages || []
}

async function replyTicket() {
  if (!activeTicket.value || !replyContent.value.trim()) {
    ElMessage.warning('请输入回复内容')
    return
  }
  const result = await api(
    `/api/support/tickets/${activeTicket.value.id}/messages?username=${encodeURIComponent(session.username)}&role=${session.role || 'USER'}`,
    {
      method: 'POST',
      body: { content: replyContent.value }
    }
  )
  if (!result.success) {
    ElMessage.error(result.message || '发送失败')
    return
  }
  replyContent.value = ''
  await openTicket(activeTicket.value)
  await loadTickets()
}

async function updateTicketStatus(status) {
  if (!activeTicket.value) {
    return
  }
  const result = await api(`/api/support/tickets/${activeTicket.value.id}/status?role=ADMIN`, {
    method: 'PUT',
    body: { status }
  })
  result.success ? ElMessage.success('工单状态已更新') : ElMessage.error(result.message || '更新失败')
  await openTicket(activeTicket.value)
  await loadTickets()
}

function coverStyle(book) {
  const palette = ['#4285f4', '#ea4335', '#fbbc04', '#34a853']
  return { '--cover-color': palette[(book.id || 0) % palette.length] }
}

function money(value) {
  return Number(value || 0).toFixed(2)
}

function statusType(status) {
  return {
    PENDING: 'warning',
    PAID: 'primary',
    SHIPPED: 'success',
    COMPLETED: 'success',
    CANCELLED: 'info'
  }[status] || 'info'
}

function statusText(status) {
  return {
    PENDING: '待支付',
    PAID: '已支付',
    SHIPPED: '已发货',
    COMPLETED: '已完成',
    CANCELLED: '已取消'
  }[status] || status
}

function readError(error) {
  const message = error?.message || String(error)
  try {
    const parsed = JSON.parse(message)
    return parsed.message || message
  } catch {
    return message
  }
}
</script>
