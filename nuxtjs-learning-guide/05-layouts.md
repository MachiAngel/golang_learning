# Layouts 布局系統

## Layout 概念與用途

Layout（布局）是 Nuxt 3 提供的一個強大功能，用於定義頁面的共同結構。它允許您在多個頁面之間共享相同的 UI 元素，如導航列、側邊欄和頁腳。

### 為什麼需要 Layout？

```vue
<!-- ❌ 沒有 Layout：每個頁面都要重複相同的結構 -->
<!-- pages/index.vue -->
<template>
  <div>
    <TheHeader />
    <main>首頁內容</main>
    <TheFooter />
  </div>
</template>

<!-- pages/about.vue -->
<template>
  <div>
    <TheHeader />
    <main>關於頁面內容</main>
    <TheFooter />
  </div>
</template>

<!-- ✅ 使用 Layout：結構定義一次，到處使用 -->
<!-- layouts/default.vue -->
<template>
  <div>
    <TheHeader />
    <main>
      <slot /> <!-- 頁面內容插入這裡 -->
    </main>
    <TheFooter />
  </div>
</template>

<!-- pages/index.vue -->
<template>
  <div>首頁內容</div>
</template>

<!-- pages/about.vue -->
<template>
  <div>關於頁面內容</div>
</template>
```

### Layout 的主要用途

1. **共享 UI 元素**：導航列、頁腳、側邊欄等
2. **統一樣式**：確保網站的一致性
3. **減少重複代碼**：DRY（Don't Repeat Yourself）原則
4. **靈活切換**：不同頁面可使用不同的布局

## 預設 Layout (default.vue)

### 建立預設 Layout

```vue
<!-- layouts/default.vue -->
<template>
  <div class="app-layout">
    <!-- 導航列 -->
    <header class="site-header">
      <div class="container">
        <nav class="main-nav">
          <NuxtLink to="/" class="logo">
            🚀 我的網站
          </NuxtLink>

          <div class="nav-links">
            <NuxtLink to="/">首頁</NuxtLink>
            <NuxtLink to="/about">關於</NuxtLink>
            <NuxtLink to="/products">產品</NuxtLink>
            <NuxtLink to="/blog">部落格</NuxtLink>
            <NuxtLink to="/contact">聯絡我們</NuxtLink>
          </div>

          <div class="nav-actions">
            <button @click="toggleTheme" class="theme-toggle">
              {{ isDark ? '🌞' : '🌙' }}
            </button>
            <NuxtLink to="/login" class="btn-login">登入</NuxtLink>
          </div>
        </nav>
      </div>
    </header>

    <!-- 主要內容區域 -->
    <main class="main-content">
      <slot />
    </main>

    <!-- 頁腳 -->
    <footer class="site-footer">
      <div class="container">
        <div class="footer-content">
          <div class="footer-section">
            <h3>關於我們</h3>
            <p>我們致力於提供最好的服務</p>
          </div>

          <div class="footer-section">
            <h3>快速連結</h3>
            <ul>
              <li><NuxtLink to="/about">關於我們</NuxtLink></li>
              <li><NuxtLink to="/privacy">隱私政策</NuxtLink></li>
              <li><NuxtLink to="/terms">服務條款</NuxtLink></li>
            </ul>
          </div>

          <div class="footer-section">
            <h3>聯絡方式</h3>
            <p>Email: contact@example.com</p>
            <p>電話: 02-1234-5678</p>
          </div>

          <div class="footer-section">
            <h3>關注我們</h3>
            <div class="social-links">
              <a href="#" target="_blank">Facebook</a>
              <a href="#" target="_blank">Twitter</a>
              <a href="#" target="_blank">Instagram</a>
            </div>
          </div>
        </div>

        <div class="footer-bottom">
          <p>&copy; {{ currentYear }} 我的網站. All rights reserved.</p>
        </div>
      </div>
    </footer>
  </div>
</template>

<script setup lang="ts">
const isDark = ref(false)
const currentYear = new Date().getFullYear()

const toggleTheme = () => {
  isDark.value = !isDark.value
  // 實際應用中會切換全局主題
  document.documentElement.classList.toggle('dark', isDark.value)
}

// 監聽路由變化時滾動到頂部
const route = useRoute()
watch(() => route.path, () => {
  window.scrollTo(0, 0)
})
</script>

<style scoped>
.app-layout {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

/* Header 樣式 */
.site-header {
  background: white;
  border-bottom: 1px solid #e5e7eb;
  position: sticky;
  top: 0;
  z-index: 100;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 2rem;
}

.main-nav {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 0;
}

.logo {
  font-size: 1.5rem;
  font-weight: bold;
  text-decoration: none;
  color: #00DC82;
}

.nav-links {
  display: flex;
  gap: 2rem;
  flex: 1;
  justify-content: center;
}

.nav-links a {
  text-decoration: none;
  color: #374151;
  font-weight: 500;
  transition: color 0.3s;
  position: relative;
}

.nav-links a:hover {
  color: #00DC82;
}

.nav-links a.router-link-active::after {
  content: '';
  position: absolute;
  bottom: -1.25rem;
  left: 0;
  right: 0;
  height: 2px;
  background: #00DC82;
}

.nav-actions {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.theme-toggle {
  background: none;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  padding: 0.5rem;
  border-radius: 50%;
  transition: background 0.3s;
}

.theme-toggle:hover {
  background: #f3f4f6;
}

.btn-login {
  padding: 0.5rem 1.5rem;
  background: #00DC82;
  color: white;
  text-decoration: none;
  border-radius: 6px;
  font-weight: 500;
  transition: background 0.3s;
}

.btn-login:hover {
  background: #00b36b;
}

/* Main Content */
.main-content {
  flex: 1;
  padding: 2rem 0;
}

/* Footer 樣式 */
.site-footer {
  background: #111827;
  color: white;
  padding: 3rem 0 1rem;
  margin-top: auto;
}

.footer-content {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 2rem;
  margin-bottom: 2rem;
}

.footer-section h3 {
  margin-bottom: 1rem;
  color: #00DC82;
}

.footer-section p {
  color: #9ca3af;
  line-height: 1.6;
}

.footer-section ul {
  list-style: none;
  padding: 0;
}

.footer-section ul li {
  margin-bottom: 0.5rem;
}

.footer-section a {
  color: #9ca3af;
  text-decoration: none;
  transition: color 0.3s;
}

.footer-section a:hover {
  color: #00DC82;
}

.social-links {
  display: flex;
  gap: 1rem;
}

.footer-bottom {
  text-align: center;
  padding-top: 2rem;
  border-top: 1px solid #374151;
  color: #9ca3af;
}

/* 響應式設計 */
@media (max-width: 768px) {
  .main-nav {
    flex-direction: column;
    gap: 1rem;
  }

  .nav-links {
    flex-direction: column;
    gap: 0.5rem;
    text-align: center;
  }

  .footer-content {
    grid-template-columns: 1fr;
  }
}
</style>
```

### 頁面如何使用預設 Layout

```vue
<!-- pages/index.vue -->
<template>
  <div class="home-page">
    <h1>歡迎來到首頁</h1>
    <p>這個頁面自動使用 default.vue 布局</p>
  </div>
</template>

<script setup lang="ts">
// 不需要任何特殊配置，預設就會使用 default.vue
</script>
```

## 自定義 Layout

### 建立管理後台 Layout

```vue
<!-- layouts/admin.vue -->
<template>
  <div class="admin-layout">
    <!-- 側邊欄 -->
    <aside class="sidebar">
      <div class="sidebar-header">
        <h2>🎛️ 管理後台</h2>
      </div>

      <nav class="sidebar-nav">
        <NuxtLink to="/admin" class="nav-item">
          <span class="icon">📊</span>
          儀表板
        </NuxtLink>

        <NuxtLink to="/admin/users" class="nav-item">
          <span class="icon">👥</span>
          用戶管理
        </NuxtLink>

        <NuxtLink to="/admin/products" class="nav-item">
          <span class="icon">📦</span>
          產品管理
        </NuxtLink>

        <NuxtLink to="/admin/orders" class="nav-item">
          <span class="icon">🛒</span>
          訂單管理
        </NuxtLink>

        <NuxtLink to="/admin/settings" class="nav-item">
          <span class="icon">⚙️</span>
          系統設定
        </NuxtLink>

        <div class="nav-divider"></div>

        <button @click="handleLogout" class="nav-item logout">
          <span class="icon">🚪</span>
          登出
        </button>
      </nav>
    </aside>

    <!-- 主要內容區 -->
    <div class="admin-main">
      <!-- 頂部導航列 -->
      <header class="admin-header">
        <div class="header-left">
          <button @click="toggleSidebar" class="menu-toggle">☰</button>
          <h1>{{ pageTitle }}</h1>
        </div>

        <div class="header-right">
          <div class="notifications">
            <button class="notification-btn">
              🔔
              <span v-if="unreadCount > 0" class="badge">
                {{ unreadCount }}
              </span>
            </button>
          </div>

          <div class="user-menu">
            <img :src="currentUser.avatar" :alt="currentUser.name" />
            <span>{{ currentUser.name }}</span>
          </div>
        </div>
      </header>

      <!-- 頁面內容 -->
      <main class="admin-content">
        <slot />
      </main>
    </div>
  </div>
</template>

<script setup lang="ts">
interface User {
  name: string
  avatar: string
  role: string
}

const router = useRouter()
const route = useRoute()

const currentUser = ref<User>({
  name: '管理員',
  avatar: 'https://via.placeholder.com/40',
  role: 'admin'
})

const unreadCount = ref(5)
const sidebarCollapsed = ref(false)

const pageTitle = computed(() => {
  const titleMap: Record<string, string> = {
    '/admin': '儀表板',
    '/admin/users': '用戶管理',
    '/admin/products': '產品管理',
    '/admin/orders': '訂單管理',
    '/admin/settings': '系統設定'
  }
  return titleMap[route.path] || '管理後台'
})

const toggleSidebar = () => {
  sidebarCollapsed.value = !sidebarCollapsed.value
}

const handleLogout = async () => {
  if (confirm('確定要登出嗎？')) {
    // 執行登出邏輯
    await navigateTo('/login')
  }
}
</script>

<style scoped>
.admin-layout {
  display: flex;
  min-height: 100vh;
  background: #f3f4f6;
}

/* 側邊欄 */
.sidebar {
  width: 250px;
  background: #1f2937;
  color: white;
  display: flex;
  flex-direction: column;
}

.sidebar-header {
  padding: 1.5rem;
  border-bottom: 1px solid #374151;
}

.sidebar-header h2 {
  margin: 0;
  font-size: 1.25rem;
}

.sidebar-nav {
  flex: 1;
  padding: 1rem 0;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.75rem 1.5rem;
  color: #d1d5db;
  text-decoration: none;
  transition: all 0.3s;
  cursor: pointer;
  border: none;
  background: none;
  width: 100%;
  text-align: left;
  font-size: 1rem;
}

.nav-item:hover {
  background: #374151;
  color: white;
}

.nav-item.router-link-active {
  background: #00DC82;
  color: white;
}

.nav-item .icon {
  font-size: 1.25rem;
}

.nav-divider {
  height: 1px;
  background: #374151;
  margin: 1rem 0;
}

.logout {
  color: #ef4444;
}

.logout:hover {
  background: #7f1d1d;
  color: white;
}

/* 主要內容區 */
.admin-main {
  flex: 1;
  display: flex;
  flex-direction: column;
}

/* 頂部導航 */
.admin-header {
  background: white;
  padding: 1rem 2rem;
  border-bottom: 1px solid #e5e7eb;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.menu-toggle {
  background: none;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  padding: 0.5rem;
  border-radius: 4px;
  transition: background 0.3s;
}

.menu-toggle:hover {
  background: #f3f4f6;
}

.admin-header h1 {
  margin: 0;
  font-size: 1.5rem;
  color: #111827;
}

.header-right {
  display: flex;
  align-items: center;
  gap: 1.5rem;
}

.notification-btn {
  position: relative;
  background: none;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  padding: 0.5rem;
  border-radius: 50%;
  transition: background 0.3s;
}

.notification-btn:hover {
  background: #f3f4f6;
}

.badge {
  position: absolute;
  top: 0;
  right: 0;
  background: #ef4444;
  color: white;
  font-size: 0.75rem;
  padding: 0.125rem 0.375rem;
  border-radius: 10px;
}

.user-menu {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.5rem 1rem;
  background: #f3f4f6;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.3s;
}

.user-menu:hover {
  background: #e5e7eb;
}

.user-menu img {
  width: 40px;
  height: 40px;
  border-radius: 50%;
}

/* 內容區 */
.admin-content {
  flex: 1;
  padding: 2rem;
  overflow-y: auto;
}

@media (max-width: 768px) {
  .sidebar {
    width: 200px;
  }

  .admin-header {
    padding: 1rem;
  }

  .admin-content {
    padding: 1rem;
  }
}
</style>
```

### 建立空白 Layout

```vue
<!-- layouts/blank.vue -->
<template>
  <div class="blank-layout">
    <slot />
  </div>
</template>

<style scoped>
.blank-layout {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}
</style>
```

## 切換 Layout

### 使用 definePageMeta 指定 Layout

```vue
<!-- pages/admin/index.vue -->
<template>
  <div class="dashboard">
    <h1>管理儀表板</h1>

    <div class="stats-grid">
      <div class="stat-card">
        <h3>總用戶數</h3>
        <p class="stat-value">1,234</p>
      </div>

      <div class="stat-card">
        <h3>總訂單</h3>
        <p class="stat-value">567</p>
      </div>

      <div class="stat-card">
        <h3>總收入</h3>
        <p class="stat-value">$12,345</p>
      </div>

      <div class="stat-card">
        <h3>活躍用戶</h3>
        <p class="stat-value">89</p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
// 指定使用 admin layout
definePageMeta({
  layout: 'admin'
})
</script>

<style scoped>
.dashboard h1 {
  margin-bottom: 2rem;
  color: #111827;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1.5rem;
}

.stat-card {
  background: white;
  padding: 2rem;
  border-radius: 12px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.stat-card h3 {
  color: #6b7280;
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  margin-bottom: 0.5rem;
}

.stat-value {
  font-size: 2rem;
  font-weight: bold;
  color: #00DC82;
  margin: 0;
}
</style>
```

### 登入頁面使用空白 Layout

```vue
<!-- pages/login.vue -->
<template>
  <div class="login-page">
    <div class="login-card">
      <h1>🔐 會員登入</h1>

      <form @submit.prevent="handleLogin">
        <div class="form-group">
          <label for="email">電子郵件</label>
          <input
            id="email"
            v-model="form.email"
            type="email"
            placeholder="example@email.com"
            required
          />
        </div>

        <div class="form-group">
          <label for="password">密碼</label>
          <input
            id="password"
            v-model="form.password"
            type="password"
            placeholder="••••••••"
            required
          />
        </div>

        <div class="form-options">
          <label class="checkbox">
            <input v-model="form.remember" type="checkbox" />
            記住我
          </label>

          <NuxtLink to="/forgot-password" class="forgot-link">
            忘記密碼？
          </NuxtLink>
        </div>

        <button type="submit" class="btn-submit" :disabled="isLoading">
          {{ isLoading ? '登入中...' : '登入' }}
        </button>
      </form>

      <p class="signup-link">
        還沒有帳號？
        <NuxtLink to="/register">立即註冊</NuxtLink>
      </p>
    </div>
  </div>
</template>

<script setup lang="ts">
// 使用空白 layout
definePageMeta({
  layout: 'blank'
})

const form = reactive({
  email: '',
  password: '',
  remember: false
})

const isLoading = ref(false)

const handleLogin = async () => {
  isLoading.value = true

  try {
    // 模擬登入 API 請求
    await new Promise(resolve => setTimeout(resolve, 1000))

    // 登入成功後導向
    await navigateTo('/admin')
  } catch (error) {
    console.error('登入失敗:', error)
    alert('登入失敗，請檢查您的帳號密碼')
  } finally {
    isLoading.value = false
  }
}

useHead({
  title: '會員登入'
})
</script>

<style scoped>
.login-page {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
}

.login-card {
  background: white;
  padding: 3rem;
  border-radius: 16px;
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
  width: 100%;
  max-width: 400px;
}

.login-card h1 {
  text-align: center;
  margin-bottom: 2rem;
  color: #111827;
}

.form-group {
  margin-bottom: 1.5rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.5rem;
  color: #374151;
  font-weight: 500;
}

.form-group input {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  font-size: 1rem;
  transition: border-color 0.3s;
}

.form-group input:focus {
  outline: none;
  border-color: #00DC82;
}

.form-options {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
}

.checkbox {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  cursor: pointer;
}

.forgot-link {
  color: #00DC82;
  text-decoration: none;
  font-size: 0.875rem;
}

.forgot-link:hover {
  text-decoration: underline;
}

.btn-submit {
  width: 100%;
  padding: 0.75rem;
  background: #00DC82;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.3s;
}

.btn-submit:hover:not(:disabled) {
  background: #00b36b;
}

.btn-submit:disabled {
  background: #9ca3af;
  cursor: not-allowed;
}

.signup-link {
  text-align: center;
  margin-top: 1.5rem;
  color: #6b7280;
}

.signup-link a {
  color: #00DC82;
  text-decoration: none;
  font-weight: 600;
}

.signup-link a:hover {
  text-decoration: underline;
}
</style>
```

### 動態切換 Layout

```vue
<script setup lang="ts">
const route = useRoute()

// 根據條件動態設定 layout
definePageMeta({
  layout: false // 禁用自動 layout
})

// 在 template 中使用 NuxtLayout 組件動態切換
const currentLayout = computed(() => {
  if (route.path.startsWith('/admin')) {
    return 'admin'
  } else if (route.path === '/login') {
    return 'blank'
  }
  return 'default'
})
</script>

<template>
  <NuxtLayout :name="currentLayout">
    <slot />
  </NuxtLayout>
</template>
```

## 巢狀 Layout

巢狀 Layout 允許在一個 Layout 內部使用另一個 Layout。

### 建立巢狀 Layout 結構

```
layouts/
├── default.vue              # 基礎 layout
├── admin.vue                # 管理後台 layout
└── admin/
    └── settings.vue         # 設定頁面的子 layout
```

### 父 Layout

```vue
<!-- layouts/admin.vue -->
<template>
  <div class="admin-layout">
    <TheSidebar />

    <div class="admin-main">
      <TheAdminHeader />

      <main class="admin-content">
        <!-- 這裡可以是頁面內容，也可以是子 layout -->
        <slot />
      </main>
    </div>
  </div>
</template>
```

### 子 Layout

```vue
<!-- layouts/admin/settings.vue -->
<template>
  <div class="settings-layout">
    <!-- 設定頁面的側邊導航 -->
    <aside class="settings-sidebar">
      <h3>設定選項</h3>
      <nav>
        <NuxtLink to="/admin/settings/profile">個人資料</NuxtLink>
        <NuxtLink to="/admin/settings/security">安全設定</NuxtLink>
        <NuxtLink to="/admin/settings/notifications">通知設定</NuxtLink>
        <NuxtLink to="/admin/settings/billing">付款資訊</NuxtLink>
      </nav>
    </aside>

    <!-- 設定頁面的內容區 -->
    <div class="settings-content">
      <slot />
    </div>
  </div>
</template>

<style scoped>
.settings-layout {
  display: grid;
  grid-template-columns: 250px 1fr;
  gap: 2rem;
}

.settings-sidebar {
  background: white;
  padding: 1.5rem;
  border-radius: 12px;
  height: fit-content;
}

.settings-sidebar h3 {
  margin-bottom: 1rem;
  color: #111827;
}

.settings-sidebar nav {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.settings-sidebar a {
  padding: 0.75rem;
  color: #6b7280;
  text-decoration: none;
  border-radius: 6px;
  transition: all 0.3s;
}

.settings-sidebar a:hover,
.settings-sidebar a.router-link-active {
  background: #00DC82;
  color: white;
}

.settings-content {
  background: white;
  padding: 2rem;
  border-radius: 12px;
}
</style>
```

### 使用巢狀 Layout

```vue
<!-- pages/admin/settings/profile.vue -->
<template>
  <div class="profile-settings">
    <h2>個人資料設定</h2>

    <form @submit.prevent="saveProfile">
      <div class="form-group">
        <label>姓名</label>
        <input v-model="profile.name" type="text" />
      </div>

      <div class="form-group">
        <label>電子郵件</label>
        <input v-model="profile.email" type="email" />
      </div>

      <div class="form-group">
        <label>電話</label>
        <input v-model="profile.phone" type="tel" />
      </div>

      <button type="submit" class="btn-save">
        儲存變更
      </button>
    </form>
  </div>
</template>

<script setup lang="ts">
// 使用子 layout
definePageMeta({
  layout: 'admin/settings'
})

const profile = reactive({
  name: '張三',
  email: 'zhang@example.com',
  phone: '0912-345-678'
})

const saveProfile = () => {
  console.log('儲存個人資料:', profile)
  alert('個人資料已更新')
}
</script>

<style scoped>
.profile-settings h2 {
  margin-bottom: 2rem;
  color: #111827;
}

.form-group {
  margin-bottom: 1.5rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.5rem;
  color: #374151;
  font-weight: 500;
}

.form-group input {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #d1d5db;
  border-radius: 8px;
}

.btn-save {
  padding: 0.75rem 2rem;
  background: #00DC82;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: 600;
}

.btn-save:hover {
  background: #00b36b;
}
</style>
```

## App.vue 與 Layout 的關係

### App.vue 的角色

`app.vue` 是應用程式的根組件，Layout 在其內部運作。

```vue
<!-- app.vue -->
<template>
  <div id="app">
    <!-- 全局組件（在所有 layout 之外） -->
    <NuxtLoadingIndicator />

    <!-- Layout 和頁面會在這裡渲染 -->
    <NuxtPage />

    <!-- 全局彈窗、Toast 等 -->
    <GlobalNotifications />
  </div>
</template>

<script setup lang="ts">
// 全局狀態初始化
const userStore = useUserStore()

// 全局錯誤處理
onErrorCaptured((error) => {
  console.error('全局錯誤:', error)
  return false
})

// 全局 head 設定
useHead({
  titleTemplate: '%s - 我的網站',
  htmlAttrs: { lang: 'zh-TW' },
  meta: [
    { charset: 'utf-8' },
    { name: 'viewport', content: 'width=device-width, initial-scale=1' }
  ]
})
</script>

<style>
/* 全局樣式 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  line-height: 1.6;
  color: #333;
}

#app {
  min-height: 100vh;
}
</style>
```

### 渲染層級

```
app.vue
  └─ NuxtPage
      └─ Layout (default.vue / admin.vue / blank.vue)
          └─ 頁面組件 (pages/index.vue, etc.)
```

## 完整範例（包含 header, footer）

### 完整的電商網站布局系統

```
layouts/
├── default.vue         # 一般頁面 layout
├── checkout.vue        # 結帳頁面 layout
└── account.vue         # 會員中心 layout

components/
├── layout/
│   ├── TheHeader.vue
│   ├── TheFooter.vue
│   ├── TheSidebar.vue
│   └── TheCheckoutSteps.vue
```

### Header 組件

```vue
<!-- components/layout/TheHeader.vue -->
<template>
  <header class="site-header">
    <!-- 頂部公告欄 -->
    <div class="top-bar">
      <div class="container">
        <p>🎉 全站商品免運費！結帳輸入優惠碼 FREESHIP</p>
      </div>
    </div>

    <!-- 主導航列 -->
    <div class="main-header">
      <div class="container">
        <NuxtLink to="/" class="logo">
          🛍️ ShopHub
        </NuxtLink>

        <!-- 搜尋列 -->
        <div class="search-bar">
          <input
            v-model="searchQuery"
            @keyup.enter="handleSearch"
            type="search"
            placeholder="搜尋商品..."
          />
          <button @click="handleSearch">🔍</button>
        </div>

        <!-- 右側功能 -->
        <div class="header-actions">
          <NuxtLink to="/wishlist" class="action-btn">
            ❤️
            <span v-if="wishlistCount > 0" class="badge">
              {{ wishlistCount }}
            </span>
          </NuxtLink>

          <NuxtLink to="/cart" class="action-btn">
            🛒
            <span v-if="cartCount > 0" class="badge">
              {{ cartCount }}
            </span>
          </NuxtLink>

          <NuxtLink v-if="!isLoggedIn" to="/login" class="action-btn">
            👤 登入
          </NuxtLink>

          <div v-else class="user-dropdown">
            <button class="user-btn">
              👤 {{ userName }}
            </button>
            <!-- 下拉選單可以在這裡實作 -->
          </div>
        </div>
      </div>
    </div>

    <!-- 分類導航 -->
    <nav class="category-nav">
      <div class="container">
        <NuxtLink to="/products">所有商品</NuxtLink>
        <NuxtLink to="/products/electronics">電子產品</NuxtLink>
        <NuxtLink to="/products/fashion">時尚服飾</NuxtLink>
        <NuxtLink to="/products/home">居家生活</NuxtLink>
        <NuxtLink to="/products/sports">運動戶外</NuxtLink>
        <NuxtLink to="/sale" class="sale-link">🔥 特價專區</NuxtLink>
      </div>
    </nav>
  </header>
</template>

<script setup lang="ts">
const router = useRouter()
const searchQuery = ref('')

// 假設有這些狀態
const isLoggedIn = ref(false)
const userName = ref('張三')
const cartCount = ref(3)
const wishlistCount = ref(5)

const handleSearch = () => {
  if (searchQuery.value.trim()) {
    router.push(`/search?q=${searchQuery.value}`)
  }
}
</script>

<style scoped>
.top-bar {
  background: #00DC82;
  color: white;
  padding: 0.5rem 0;
  text-align: center;
  font-size: 0.875rem;
}

.main-header {
  background: white;
  padding: 1rem 0;
  border-bottom: 1px solid #e5e7eb;
}

.main-header .container {
  display: flex;
  align-items: center;
  gap: 2rem;
}

.logo {
  font-size: 1.5rem;
  font-weight: bold;
  text-decoration: none;
  color: #00DC82;
  white-space: nowrap;
}

.search-bar {
  flex: 1;
  display: flex;
  max-width: 600px;
}

.search-bar input {
  flex: 1;
  padding: 0.75rem 1rem;
  border: 2px solid #e5e7eb;
  border-right: none;
  border-radius: 8px 0 0 8px;
  font-size: 1rem;
}

.search-bar button {
  padding: 0.75rem 1.5rem;
  background: #00DC82;
  color: white;
  border: none;
  border-radius: 0 8px 8px 0;
  cursor: pointer;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.action-btn {
  position: relative;
  padding: 0.5rem 1rem;
  text-decoration: none;
  color: #374151;
  background: #f9fafb;
  border-radius: 8px;
  transition: background 0.3s;
}

.action-btn:hover {
  background: #e5e7eb;
}

.badge {
  position: absolute;
  top: -5px;
  right: -5px;
  background: #ef4444;
  color: white;
  font-size: 0.75rem;
  padding: 0.125rem 0.5rem;
  border-radius: 10px;
}

.category-nav {
  background: #f9fafb;
  border-bottom: 1px solid #e5e7eb;
}

.category-nav .container {
  display: flex;
  gap: 2rem;
  padding: 1rem 0;
}

.category-nav a {
  text-decoration: none;
  color: #374151;
  font-weight: 500;
  transition: color 0.3s;
}

.category-nav a:hover {
  color: #00DC82;
}

.sale-link {
  color: #ef4444 !important;
  font-weight: 700;
}

.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 2rem;
}
</style>
```

### Footer 組件

```vue
<!-- components/layout/TheFooter.vue -->
<template>
  <footer class="site-footer">
    <div class="container">
      <!-- 主要內容區 -->
      <div class="footer-main">
        <div class="footer-section">
          <h3>關於 ShopHub</h3>
          <p>我們致力於提供最優質的購物體驗</p>
          <div class="social-links">
            <a href="#" aria-label="Facebook">📘</a>
            <a href="#" aria-label="Instagram">📷</a>
            <a href="#" aria-label="Twitter">🐦</a>
            <a href="#" aria-label="YouTube">📹</a>
          </div>
        </div>

        <div class="footer-section">
          <h3>購物指南</h3>
          <ul>
            <li><NuxtLink to="/how-to-order">如何下單</NuxtLink></li>
            <li><NuxtLink to="/payment">付款方式</NuxtLink></li>
            <li><NuxtLink to="/shipping">運送方式</NuxtLink></li>
            <li><NuxtLink to="/returns">退換貨政策</NuxtLink></li>
          </ul>
        </div>

        <div class="footer-section">
          <h3>客戶服務</h3>
          <ul>
            <li><NuxtLink to="/faq">常見問題</NuxtLink></li>
            <li><NuxtLink to="/contact">聯絡我們</NuxtLink></li>
            <li><NuxtLink to="/track-order">訂單查詢</NuxtLink></li>
            <li><NuxtLink to="/size-guide">尺寸指南</NuxtLink></li>
          </ul>
        </div>

        <div class="footer-section">
          <h3>會員專區</h3>
          <ul>
            <li><NuxtLink to="/account">我的帳戶</NuxtLink></li>
            <li><NuxtLink to="/account/orders">訂單記錄</NuxtLink></li>
            <li><NuxtLink to="/account/wishlist">願望清單</NuxtLink></li>
            <li><NuxtLink to="/account/points">紅利點數</NuxtLink></li>
          </ul>
        </div>

        <div class="footer-section">
          <h3>訂閱電子報</h3>
          <p>訂閱以獲取最新優惠資訊</p>
          <form @submit.prevent="handleSubscribe" class="newsletter-form">
            <input
              v-model="email"
              type="email"
              placeholder="輸入您的電子郵件"
              required
            />
            <button type="submit">訂閱</button>
          </form>
        </div>
      </div>

      <!-- 底部資訊 -->
      <div class="footer-bottom">
        <div class="payment-methods">
          <span>支援付款方式：</span>
          💳 信用卡 | 🏦 ATM轉帳 | 📱 行動支付
        </div>

        <div class="footer-legal">
          <NuxtLink to="/terms">服務條款</NuxtLink>
          <span>|</span>
          <NuxtLink to="/privacy">隱私權政策</NuxtLink>
          <span>|</span>
          <NuxtLink to="/cookies">Cookie 政策</NuxtLink>
        </div>

        <div class="copyright">
          <p>&copy; {{ currentYear }} ShopHub. All rights reserved.</p>
        </div>
      </div>
    </div>
  </footer>
</template>

<script setup lang="ts">
const email = ref('')
const currentYear = new Date().getFullYear()

const handleSubscribe = () => {
  console.log('訂閱電子報:', email.value)
  alert('訂閱成功！感謝您的訂閱')
  email.value = ''
}
</script>

<style scoped>
.site-footer {
  background: #111827;
  color: white;
  padding: 3rem 0 1rem;
  margin-top: 4rem;
}

.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 2rem;
}

.footer-main {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 2rem;
  margin-bottom: 3rem;
}

.footer-section h3 {
  color: #00DC82;
  margin-bottom: 1rem;
  font-size: 1.125rem;
}

.footer-section p {
  color: #9ca3af;
  line-height: 1.6;
  margin-bottom: 1rem;
}

.footer-section ul {
  list-style: none;
  padding: 0;
}

.footer-section li {
  margin-bottom: 0.75rem;
}

.footer-section a {
  color: #d1d5db;
  text-decoration: none;
  transition: color 0.3s;
}

.footer-section a:hover {
  color: #00DC82;
}

.social-links {
  display: flex;
  gap: 1rem;
  margin-top: 1rem;
}

.social-links a {
  font-size: 1.5rem;
  transition: transform 0.3s;
}

.social-links a:hover {
  transform: scale(1.2);
}

.newsletter-form {
  display: flex;
  gap: 0.5rem;
  margin-top: 1rem;
}

.newsletter-form input {
  flex: 1;
  padding: 0.75rem;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
}

.newsletter-form button {
  padding: 0.75rem 1.5rem;
  background: #00DC82;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 600;
  transition: background 0.3s;
}

.newsletter-form button:hover {
  background: #00b36b;
}

.footer-bottom {
  border-top: 1px solid #374151;
  padding-top: 2rem;
  text-align: center;
}

.payment-methods,
.footer-legal {
  margin-bottom: 1rem;
  color: #9ca3af;
}

.footer-legal {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 1rem;
}

.copyright {
  color: #6b7280;
  font-size: 0.875rem;
}
</style>
```

### 使用完整的 Layout

```vue
<!-- layouts/default.vue -->
<template>
  <div class="app-layout">
    <TheHeader />

    <main class="main-content">
      <slot />
    </main>

    <TheFooter />
  </div>
</template>

<style scoped>
.app-layout {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

.main-content {
  flex: 1;
}
</style>
```

## 最佳實踐建議

### 1. Layout 命名規範

```
✅ 推薦
layouts/default.vue
layouts/admin.vue
layouts/checkout.vue

❌ 不推薦
layouts/Layout1.vue
layouts/myLayout.vue
```

### 2. 共享組件提取

```vue
<!-- ✅ 將重複的 UI 元素提取為組件 -->
<template>
  <div>
    <TheHeader />
    <slot />
    <TheFooter />
  </div>
</template>

<!-- ❌ 在 layout 中直接寫大量 HTML -->
```

### 3. 響應式設計

```vue
<style scoped>
/* 確保 layout 在不同裝置上都能正常顯示 */
@media (max-width: 768px) {
  .sidebar {
    display: none;
  }

  .main-content {
    padding: 1rem;
  }
}
</style>
```

### 4. 效能優化

```vue
<script setup lang="ts">
// 避免在 layout 中進行重複的 API 請求
// 使用全局狀態管理
const userStore = useUserStore()

// 只在需要時才載入資料
const { data: user } = await useAsyncData('user', () => {
  return userStore.fetchUser()
}, {
  lazy: true,
  server: false
})
</script>
```

## 總結

本章節完整介紹了 Nuxt 3 的 Layouts 布局系統：

- ✅ Layout 的概念與用途
- ✅ 預設 Layout (default.vue) 的使用
- ✅ 自定義 Layout 的建立
- ✅ 如何切換不同的 Layout
- ✅ 巢狀 Layout 的實作
- ✅ App.vue 與 Layout 的關係
- ✅ 包含完整 Header/Footer 的實際範例

通過掌握 Layouts 系統，您可以建立更加模組化、可維護的 Nuxt 3 應用程式。

## 下一步學習

建議繼續學習以下主題：
- 中介軟體（Middleware）
- 資料取得（Data Fetching）
- 狀態管理（State Management）
- API 路由（Server Routes）
