# 第 14 章：CSS 與樣式處理（CSS Modules, Tailwind CSS）

## 概述

Next.js 提供了多種 CSS 解決方案，從傳統的全域 CSS 到 CSS Modules，再到現代的 Tailwind CSS。對於 Vue3 開發者來說，這些概念有些相似，但也有重要的差異。

## Vue3 vs Next.js 樣式處理

### Vue3 的做法

```vue
<template>
  <div class="container">
    <h1 class="title">Hello Vue</h1>
  </div>
</template>

<style scoped>
.container {
  padding: 20px;
}

.title {
  color: blue;
}
</style>
```

### Next.js 的做法

有多種選擇：

1. **CSS Modules**
2. **全域 CSS**
3. **CSS-in-JS（styled-jsx, styled-components）**
4. **Tailwind CSS**

## CSS Modules

### 基本用法

```css
/* styles/Home.module.css */
.container {
  padding: 20px;
  max-width: 1200px;
  margin: 0 auto;
}

.title {
  color: #0070f3;
  font-size: 2rem;
  margin-bottom: 1rem;
}

.button {
  background-color: #0070f3;
  color: white;
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.button:hover {
  background-color: #0051cc;
}
```

```javascript
// pages/index.js
import styles from '../styles/Home.module.css'

export default function Home() {
  return (
    <div className={styles.container}>
      <h1 className={styles.title}>Hello Next.js</h1>
      <button className={styles.button}>點擊</button>
    </div>
  )
}
```

**類似 Vue3 的 scoped：**
- 自動生成唯一的類別名稱
- 避免樣式衝突
- 組件級別的樣式隔離

### 組合多個類別

```javascript
import styles from './Button.module.css'

export default function Button({ variant, children }) {
  // 方法 1：字串模板
  return (
    <button className={`${styles.button} ${styles[variant]}`}>
      {children}
    </button>
  )
}
```

```javascript
// 方法 2：使用 classnames 套件
import classNames from 'classnames'
import styles from './Button.module.css'

export default function Button({ variant, isActive, children }) {
  return (
    <button
      className={classNames(
        styles.button,
        styles[variant],
        { [styles.active]: isActive }
      )}
    >
      {children}
    </button>
  )
}
```

### 完整的組件範例

```css
/* components/Card.module.css */
.card {
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  padding: 1.5rem;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  transition: transform 0.2s, box-shadow 0.2s;
}

.card:hover {
  transform: translateY(-4px);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.15);
}

.cardTitle {
  font-size: 1.5rem;
  font-weight: 600;
  margin-bottom: 0.5rem;
  color: #333;
}

.cardContent {
  color: #666;
  line-height: 1.6;
}

.cardFooter {
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px solid #e0e0e0;
  display: flex;
  justify-content: space-between;
  align-items: center;
}
```

```javascript
// components/Card.js
import styles from './Card.module.css'

export default function Card({ title, content, footer }) {
  return (
    <div className={styles.card}>
      <h2 className={styles.cardTitle}>{title}</h2>
      <div className={styles.cardContent}>{content}</div>
      {footer && (
        <div className={styles.cardFooter}>{footer}</div>
      )}
    </div>
  )
}
```

## 全域 CSS

### 設定全域樣式

```css
/* styles/globals.css */
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html,
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto',
    'Oxygen', 'Ubuntu', 'Cantarell', 'Fira Sans', 'Droid Sans',
    'Helvetica Neue', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

a {
  color: inherit;
  text-decoration: none;
}

a:hover {
  text-decoration: underline;
}

code {
  font-family: 'Courier New', Courier, monospace;
  background-color: #f4f4f4;
  padding: 0.2rem 0.4rem;
  border-radius: 3px;
}

/* 主題顏色變數 */
:root {
  --primary-color: #0070f3;
  --secondary-color: #ff4081;
  --background-color: #ffffff;
  --text-color: #333333;
  --border-color: #e0e0e0;
}

[data-theme='dark'] {
  --primary-color: #4da6ff;
  --secondary-color: #ff6b9d;
  --background-color: #1a1a1a;
  --text-color: #e0e0e0;
  --border-color: #333333;
}
```

```javascript
// pages/_app.js
import '../styles/globals.css'

export default function MyApp({ Component, pageProps }) {
  return <Component {...pageProps} />
}
```

**注意：** 全域 CSS 只能在 `_app.js` 中匯入！

## Tailwind CSS

### 安裝與設定

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

```javascript
// tailwind.config.js
module.exports = {
  content: [
    './pages/**/*.{js,ts,jsx,tsx}',
    './components/**/*.{js,ts,jsx,tsx}',
  ],
  theme: {
    extend: {
      colors: {
        primary: '#0070f3',
        secondary: '#ff4081',
      },
      spacing: {
        '128': '32rem',
      },
    },
  },
  plugins: [],
}
```

```css
/* styles/globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### 基本使用

```javascript
export default function Home() {
  return (
    <div className="min-h-screen bg-gray-50">
      <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-12">
        <h1 className="text-4xl font-bold text-gray-900 mb-8">
          歡迎使用 Tailwind CSS
        </h1>

        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
          <div className="bg-white rounded-lg shadow-md p-6 hover:shadow-lg transition-shadow">
            <h2 className="text-xl font-semibold mb-2">卡片標題</h2>
            <p className="text-gray-600">卡片內容</p>
          </div>
        </div>
      </div>
    </div>
  )
}
```

### 響應式設計

```javascript
export default function ResponsiveLayout() {
  return (
    <div className="container mx-auto px-4">
      {/* 手機：1 列，平板：2 列，桌面：4 列 */}
      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
        <div className="bg-blue-500 text-white p-4 rounded">項目 1</div>
        <div className="bg-blue-500 text-white p-4 rounded">項目 2</div>
        <div className="bg-blue-500 text-white p-4 rounded">項目 3</div>
        <div className="bg-blue-500 text-white p-4 rounded">項目 4</div>
      </div>

      {/* 響應式文字大小 */}
      <h1 className="text-2xl sm:text-3xl md:text-4xl lg:text-5xl font-bold">
        響應式標題
      </h1>

      {/* 響應式間距 */}
      <div className="mt-4 sm:mt-6 md:mt-8 lg:mt-10">
        內容區塊
      </div>
    </div>
  )
}
```

### 實際應用案例

#### 案例 1：導覽列

```javascript
// components/Navbar.js
import Link from 'next/link'
import { useState } from 'react'

export default function Navbar() {
  const [mobileMenuOpen, setMobileMenuOpen] = useState(false)

  return (
    <nav className="bg-white shadow-lg">
      <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
        <div className="flex justify-between h-16">
          <div className="flex items-center">
            <Link href="/">
              <span className="text-2xl font-bold text-primary cursor-pointer">
                Logo
              </span>
            </Link>
          </div>

          {/* 桌面版選單 */}
          <div className="hidden md:flex items-center space-x-8">
            <Link href="/about">
              <span className="text-gray-700 hover:text-primary cursor-pointer transition-colors">
                關於我們
              </span>
            </Link>
            <Link href="/products">
              <span className="text-gray-700 hover:text-primary cursor-pointer transition-colors">
                產品
              </span>
            </Link>
            <Link href="/contact">
              <span className="text-gray-700 hover:text-primary cursor-pointer transition-colors">
                聯絡我們
              </span>
            </Link>
            <button className="bg-primary text-white px-4 py-2 rounded-lg hover:bg-blue-600 transition-colors">
              登入
            </button>
          </div>

          {/* 手機版選單按鈕 */}
          <div className="md:hidden flex items-center">
            <button
              onClick={() => setMobileMenuOpen(!mobileMenuOpen)}
              className="text-gray-700 hover:text-primary"
            >
              <svg className="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                {mobileMenuOpen ? (
                  <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12" />
                ) : (
                  <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M4 6h16M4 12h16M4 18h16" />
                )}
              </svg>
            </button>
          </div>
        </div>
      </div>

      {/* 手機版選單 */}
      {mobileMenuOpen && (
        <div className="md:hidden">
          <div className="px-2 pt-2 pb-3 space-y-1">
            <Link href="/about">
              <span className="block px-3 py-2 rounded-md text-gray-700 hover:bg-gray-100 cursor-pointer">
                關於我們
              </span>
            </Link>
            <Link href="/products">
              <span className="block px-3 py-2 rounded-md text-gray-700 hover:bg-gray-100 cursor-pointer">
                產品
              </span>
            </Link>
            <Link href="/contact">
              <span className="block px-3 py-2 rounded-md text-gray-700 hover:bg-gray-100 cursor-pointer">
                聯絡我們
              </span>
            </Link>
            <button className="w-full text-left px-3 py-2 rounded-md bg-primary text-white hover:bg-blue-600">
              登入
            </button>
          </div>
        </div>
      )}
    </nav>
  )
}
```

#### 案例 2：產品卡片

```javascript
// components/ProductCard.js
import Image from 'next/image'

export default function ProductCard({ product }) {
  return (
    <div className="bg-white rounded-xl shadow-md overflow-hidden hover:shadow-xl transition-shadow duration-300">
      <div className="relative h-48 w-full">
        <Image
          src={product.image}
          alt={product.name}
          layout="fill"
          objectFit="cover"
          className="hover:scale-105 transition-transform duration-300"
        />
        {product.isNew && (
          <span className="absolute top-2 right-2 bg-red-500 text-white text-xs font-bold px-2 py-1 rounded">
            新品
          </span>
        )}
      </div>

      <div className="p-6">
        <h3 className="text-xl font-semibold text-gray-900 mb-2">
          {product.name}
        </h3>

        <p className="text-gray-600 text-sm mb-4 line-clamp-2">
          {product.description}
        </p>

        <div className="flex items-center justify-between mb-4">
          <div className="flex items-center">
            <span className="text-2xl font-bold text-primary">
              NT$ {product.price.toLocaleString()}
            </span>
            {product.originalPrice && (
              <span className="ml-2 text-sm text-gray-400 line-through">
                NT$ {product.originalPrice.toLocaleString()}
              </span>
            )}
          </div>

          {product.discount && (
            <span className="bg-yellow-100 text-yellow-800 text-xs font-semibold px-2 py-1 rounded">
              -{product.discount}%
            </span>
          )}
        </div>

        <div className="flex items-center justify-between">
          <div className="flex items-center text-sm text-gray-500">
            <svg className="h-4 w-4 text-yellow-400 mr-1" fill="currentColor" viewBox="0 0 20 20">
              <path d="M9.049 2.927c.3-.921 1.603-.921 1.902 0l1.07 3.292a1 1 0 00.95.69h3.462c.969 0 1.371 1.24.588 1.81l-2.8 2.034a1 1 0 00-.364 1.118l1.07 3.292c.3.921-.755 1.688-1.54 1.118l-2.8-2.034a1 1 0 00-1.175 0l-2.8 2.034c-.784.57-1.838-.197-1.539-1.118l1.07-3.292a1 1 0 00-.364-1.118L2.98 8.72c-.783-.57-.38-1.81.588-1.81h3.461a1 1 0 00.951-.69l1.07-3.292z" />
            </svg>
            <span>{product.rating}</span>
            <span className="ml-1">({product.reviews})</span>
          </div>

          <button className="bg-primary text-white px-4 py-2 rounded-lg text-sm font-semibold hover:bg-blue-600 transition-colors">
            加入購物車
          </button>
        </div>
      </div>
    </div>
  )
}
```

#### 案例 3：表單

```javascript
// components/ContactForm.js
import { useState } from 'react'

export default function ContactForm() {
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    subject: '',
    message: ''
  })

  const [errors, setErrors] = useState({})

  const handleSubmit = (e) => {
    e.preventDefault()
    // 表單驗證和提交邏輯
  }

  return (
    <form onSubmit={handleSubmit} className="max-w-2xl mx-auto bg-white rounded-lg shadow-md p-8">
      <h2 className="text-3xl font-bold text-gray-900 mb-6">聯絡我們</h2>

      <div className="mb-6">
        <label htmlFor="name" className="block text-sm font-medium text-gray-700 mb-2">
          姓名 <span className="text-red-500">*</span>
        </label>
        <input
          type="text"
          id="name"
          className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-primary focus:border-transparent outline-none transition"
          value={formData.name}
          onChange={(e) => setFormData({ ...formData, name: e.target.value })}
        />
        {errors.name && (
          <p className="mt-1 text-sm text-red-500">{errors.name}</p>
        )}
      </div>

      <div className="mb-6">
        <label htmlFor="email" className="block text-sm font-medium text-gray-700 mb-2">
          Email <span className="text-red-500">*</span>
        </label>
        <input
          type="email"
          id="email"
          className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-primary focus:border-transparent outline-none transition"
          value={formData.email}
          onChange={(e) => setFormData({ ...formData, email: e.target.value })}
        />
        {errors.email && (
          <p className="mt-1 text-sm text-red-500">{errors.email}</p>
        )}
      </div>

      <div className="mb-6">
        <label htmlFor="subject" className="block text-sm font-medium text-gray-700 mb-2">
          主旨
        </label>
        <select
          id="subject"
          className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-primary focus:border-transparent outline-none transition"
          value={formData.subject}
          onChange={(e) => setFormData({ ...formData, subject: e.target.value })}
        >
          <option value="">請選擇主旨</option>
          <option value="general">一般詢問</option>
          <option value="support">技術支援</option>
          <option value="sales">銷售諮詢</option>
        </select>
      </div>

      <div className="mb-6">
        <label htmlFor="message" className="block text-sm font-medium text-gray-700 mb-2">
          訊息 <span className="text-red-500">*</span>
        </label>
        <textarea
          id="message"
          rows={5}
          className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-primary focus:border-transparent outline-none transition resize-none"
          value={formData.message}
          onChange={(e) => setFormData({ ...formData, message: e.target.value })}
        />
        {errors.message && (
          <p className="mt-1 text-sm text-red-500">{errors.message}</p>
        )}
      </div>

      <button
        type="submit"
        className="w-full bg-primary text-white py-3 rounded-lg font-semibold hover:bg-blue-600 transition-colors focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-primary"
      >
        送出訊息
      </button>
    </form>
  )
}
```

### Tailwind 自訂組件

```css
/* styles/globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer components {
  .btn {
    @apply px-4 py-2 rounded-lg font-semibold transition-colors focus:outline-none focus:ring-2 focus:ring-offset-2;
  }

  .btn-primary {
    @apply btn bg-primary text-white hover:bg-blue-600 focus:ring-primary;
  }

  .btn-secondary {
    @apply btn bg-gray-200 text-gray-800 hover:bg-gray-300 focus:ring-gray-400;
  }

  .btn-danger {
    @apply btn bg-red-500 text-white hover:bg-red-600 focus:ring-red-500;
  }

  .card {
    @apply bg-white rounded-lg shadow-md p-6;
  }

  .input {
    @apply w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-primary focus:border-transparent outline-none transition;
  }
}
```

```javascript
// 使用自訂組件
export default function Example() {
  return (
    <div className="card">
      <input type="text" className="input mb-4" placeholder="輸入文字" />
      <div className="flex space-x-4">
        <button className="btn-primary">主要按鈕</button>
        <button className="btn-secondary">次要按鈕</button>
        <button className="btn-danger">危險按鈕</button>
      </div>
    </div>
  )
}
```

## CSS-in-JS (styled-jsx)

Next.js 內建 styled-jsx：

```javascript
export default function StyledComponent() {
  return (
    <div className="container">
      <h1 className="title">Hello</h1>
      <p className="description">這是 styled-jsx 的範例</p>

      <style jsx>{`
        .container {
          padding: 20px;
          max-width: 800px;
          margin: 0 auto;
        }

        .title {
          color: #0070f3;
          font-size: 2rem;
        }

        .description {
          color: #666;
          line-height: 1.6;
        }
      `}</style>
    </div>
  )
}
```

## 樣式方案比較

| 方案 | 優點 | 缺點 | 適用場景 |
|------|------|------|----------|
| **CSS Modules** | 作用域隔離、類似 Vue scoped | 需要匯入樣式檔案 | 中大型專案 |
| **Tailwind CSS** | 快速開發、一致性高、檔案小 | 學習曲線、HTML 較複雜 | 所有專案（推薦） |
| **全域 CSS** | 簡單直接 | 容易衝突 | 基礎樣式、重置樣式 |
| **styled-jsx** | 組件內樣式、動態樣式 | 語法較特殊 | 需要動態樣式時 |

## 最佳實踐

### 1. 推薦：Tailwind CSS + CSS Modules

```javascript
// 使用 Tailwind 處理大部分樣式
export default function Component() {
  return (
    <div className="container mx-auto px-4">
      <h1 className="text-3xl font-bold mb-4">標題</h1>
    </div>
  )
}

// 使用 CSS Modules 處理複雜的動畫或特殊效果
import styles from './Component.module.css'

export default function ComplexComponent() {
  return (
    <div className={styles.animatedBox}>
      內容
    </div>
  )
}
```

### 2. 建立設計系統

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        brand: {
          50: '#e6f2ff',
          100: '#b3d9ff',
          500: '#0070f3',
          900: '#003d80',
        },
      },
      fontFamily: {
        sans: ['Inter', 'sans-serif'],
        heading: ['Poppins', 'sans-serif'],
      },
      spacing: {
        '72': '18rem',
        '84': '21rem',
        '96': '24rem',
      },
    },
  },
}
```

### 3. 響應式設計優先

```javascript
// 從手機優先設計
<div className="
  text-sm sm:text-base md:text-lg lg:text-xl
  p-4 sm:p-6 md:p-8
  grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3
">
  內容
</div>
```

## 小結

Next.js 提供了豐富的樣式解決方案，對於 Vue3 開發者來說：
- CSS Modules 類似 Vue 的 scoped styles
- Tailwind CSS 是現代化的最佳選擇
- 可以根據專案需求混合使用多種方案

**推薦組合：** Tailwind CSS（主要） + CSS Modules（特殊場景）
