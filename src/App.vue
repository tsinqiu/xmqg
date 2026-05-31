<script setup>
import { computed, onMounted, ref, watch } from 'vue'
import { aboutSections, dailyFacts, feeds, navItems, profile, socialLinks, timeline } from './data/site'

const view = ref('home')
const failedCovers = ref({})
const dailyQuote = ref('慢慢来，你已经在变好的路上了。')
const quoteFrom = ref('本地兜底')
const weather = ref({
  place: '江南大学附近',
  text: '天气加载中...',
  detail: '正在获取今日天气',
})
const feedKeys = Object.keys(feeds)

const allViews = computed(() => [...navItems.map((item) => item.id), ...feedKeys])
const currentFeed = computed(() => feeds[view.value])

const today = new Intl.DateTimeFormat('zh-CN', {
  year: 'numeric',
  month: 'long',
  day: 'numeric',
  weekday: 'long',
}).format(new Date())

function setView(nextView) {
  if (!allViews.value.includes(nextView)) return
  view.value = nextView

  if (nextView === 'home') {
    history.replaceState(null, '', window.location.pathname)
  } else {
    window.location.hash = nextView
  }

  window.scrollTo({ top: 0, behavior: 'smooth' })
}

function onSocialClick(item, event) {
  if (item.external) return
  event.preventDefault()
  setView(item.id)
}

function markCoverFailed(coverUrl) {
  failedCovers.value = { ...failedCovers.value, [coverUrl]: true }
}

function viewLabel(id) {
  return navItems.find((item) => item.id === id)?.label || feeds[id]?.label || '个人主页'
}

const weatherText = {
  0: '晴',
  1: '大部晴朗',
  2: '局部多云',
  3: '阴',
  45: '有雾',
  48: '霜雾',
  51: '小毛毛雨',
  53: '中等毛毛雨',
  55: '大毛毛雨',
  61: '小雨',
  63: '中雨',
  65: '大雨',
  71: '小雪',
  73: '中雪',
  75: '大雪',
  80: '阵雨',
  81: '中等阵雨',
  82: '强阵雨',
  95: '雷雨',
}

async function loadDailyQuote() {
  try {
    const response = await fetch('https://v1.hitokoto.cn/?c=d&encode=json')
    if (!response.ok) throw new Error('quote request failed')
    const data = await response.json()
    dailyQuote.value = data.hitokoto || dailyQuote.value
    quoteFrom.value = data.from ? `《${data.from}》` : '每日一句'
  } catch {
    const dayIndex = new Date().getDate() % dailyFacts.length
    dailyQuote.value = dailyFacts[dayIndex]
    quoteFrom.value = '本地兜底'
  }
}

function getCurrentPosition() {
  return new Promise((resolve) => {
    if (!navigator.geolocation) {
      resolve(null)
      return
    }

    navigator.geolocation.getCurrentPosition(
      (position) => resolve(position.coords),
      () => resolve(null),
      { enableHighAccuracy: false, timeout: 3500, maximumAge: 30 * 60 * 1000 },
    )
  })
}

async function loadWeather() {
  const coords = (await getCurrentPosition()) || { latitude: 31.49, longitude: 120.27 }
  const usingFallback = !('accuracy' in coords)
  const url = new URL('https://api.open-meteo.com/v1/forecast')

  url.search = new URLSearchParams({
    latitude: String(coords.latitude),
    longitude: String(coords.longitude),
    current: 'temperature_2m,weather_code,relative_humidity_2m,wind_speed_10m',
    daily: 'temperature_2m_max,temperature_2m_min',
    timezone: 'auto',
    forecast_days: '1',
  }).toString()

  try {
    const response = await fetch(url)
    if (!response.ok) throw new Error('weather request failed')
    const data = await response.json()
    const current = data.current || {}
    const daily = data.daily || {}
    const summary = weatherText[current.weather_code] || '天气未知'
    const temperature = Math.round(current.temperature_2m)
    const high = Math.round(daily.temperature_2m_max?.[0] ?? temperature)
    const low = Math.round(daily.temperature_2m_min?.[0] ?? temperature)

    weather.value = {
      place: usingFallback ? '江南大学附近' : '当前位置',
      text: `${summary} ${temperature}°C`,
      detail: `今日 ${low}°C - ${high}°C · 湿度 ${current.relative_humidity_2m ?? '--'}% · 风速 ${current.wind_speed_10m ?? '--'} km/h`,
    }
  } catch {
    weather.value = {
      place: '天气暂不可用',
      text: '保持好心情',
      detail: 'API 暂时没有响应，出门前记得看一眼天空。',
    }
  }
}

onMounted(() => {
  const hashView = window.location.hash.replace('#', '')
  if (allViews.value.includes(hashView)) {
    view.value = hashView
  } else if (hashView) {
    setView('home')
  }

  loadDailyQuote()
  loadWeather()
})

watch(
  view,
  (nextView) => {
    document.title = nextView === 'home' ? `${profile.name}的小田园` : `${viewLabel(nextView)} | ${profile.name}`
  },
  { immediate: true },
)
</script>

<template>
  <div class="site-shell">
    <header class="topbar">
      <button class="brand" type="button" @click="setView('home')">
        <span class="brand-mark" aria-hidden="true">晴</span>
        <span>
          <strong>{{ profile.name }}</strong>
          <small>Personal field notes</small>
        </span>
      </button>

      <nav class="nav" aria-label="主导航">
        <button
          v-for="item in navItems"
          :key="item.id"
          :class="['nav-link', { active: view === item.id }]"
          type="button"
          @click="setView(item.id)"
        >
          {{ item.label }}
        </button>
      </nav>
    </header>

    <main>
      <section v-if="view === 'home'" class="home-grid" aria-labelledby="home-title">
        <div class="hero-panel">
          <div class="hero-copy">
            <p class="quiet-line">你好，欢迎来到我的主页</p>
            <h1 id="home-title">{{ profile.name }}</h1>
            <p class="tagline">{{ profile.tagline }}</p>
            <p class="bio">{{ profile.bio }}</p>

            <div class="hero-actions" aria-label="站内页面入口">
              <button type="button" @click="setView('about')">了解我</button>
              <button type="button" @click="setView('timeline')">看轨迹</button>
            </div>
          </div>

          <figure class="portrait">
            <img :src="profile.avatarUrl" :alt="`${profile.name} 的主页照片`" />
            <figcaption>{{ profile.location }}</figcaption>
          </figure>
        </div>

        <aside class="side-stack" aria-label="今日信息">
          <article class="note-panel">
            <span>今天</span>
            <strong>{{ today }}</strong>
          </article>
          <article class="note-panel">
            <span>今日天气 · {{ weather.place }}</span>
            <strong>{{ weather.text }}</strong>
            <p>{{ weather.detail }}</p>
          </article>
          <article class="note-panel">
            <span>每日一句 · {{ quoteFrom }}</span>
            <p>{{ dailyQuote }}</p>
          </article>
        </aside>

        <section class="content-index" aria-labelledby="content-title">
          <div class="section-heading">
            <p>Content index</p>
            <h2 id="content-title">西门的生活记录本</h2>
            <span>文章、帖子、视频和长期笔记都收在这里，像一张慢慢长出来的内容地图。</span>
          </div>

          <div class="link-grid">
            <a
              v-for="item in socialLinks"
              :key="item.id"
              class="social-card"
              :href="item.href"
              :style="{ '--accent': item.color }"
              :target="item.external ? '_blank' : undefined"
              :rel="item.external ? 'noopener noreferrer' : undefined"
              @click="onSocialClick(item, $event)"
            >
              <span class="social-line" aria-hidden="true"></span>
              <span>
                <strong>{{ item.label }}</strong>
                <small>{{ item.detail }}</small>
              </span>
              <span class="arrow" aria-hidden="true">{{ item.external ? '↗' : '→' }}</span>
            </a>
          </div>
        </section>
      </section>

      <section v-else-if="view === 'about'" class="page-card readable-page" aria-labelledby="about-title">
        <div class="section-heading">
          <p>Profile</p>
          <h1 id="about-title">关于我</h1>
        </div>

        <article v-for="section in aboutSections" :key="section.title" class="prose-section">
          <h2>{{ section.title }}</h2>
          <p v-for="paragraph in section.paragraphs" :key="paragraph">{{ paragraph }}</p>
        </article>
      </section>

      <section v-else-if="view === 'timeline'" class="page-card" aria-labelledby="timeline-title">
        <div class="section-heading">
          <p>Timeline</p>
          <h1 id="timeline-title">生活轨迹</h1>
        </div>

        <ol class="timeline">
          <li v-for="item in timeline" :key="`${item.date}-${item.text}`">
            <time>{{ item.date }}</time>
            <p>{{ item.text }}</p>
          </li>
        </ol>
      </section>

      <section v-else-if="currentFeed" class="page-card" aria-labelledby="feed-title">
        <div class="feed-top">
          <div class="section-heading">
            <p>{{ currentFeed.platform }}</p>
            <h1 id="feed-title">{{ currentFeed.label }}</h1>
            <span>{{ currentFeed.description }}</span>
          </div>

          <div class="feed-tabs" aria-label="内容平台">
            <button
              v-for="key in feedKeys"
              :key="key"
              :class="{ active: view === key }"
              type="button"
              @click="setView(key)"
            >
              {{ feeds[key].platform }}
            </button>
          </div>
        </div>

        <div :class="['feed-grid', { 'feed-grid-compact': currentFeed.platform !== 'BILI' }]">
          <article
            v-for="entry in currentFeed.entries"
            :key="entry.url"
            :class="['feed-card', { 'feed-card-compact': currentFeed.platform !== 'BILI' || !entry.coverUrl }]"
          >
            <a :href="entry.url" target="_blank" rel="noopener noreferrer" :aria-label="`打开 ${entry.title}`">
              <div class="feed-cover">
                <img
                  v-if="currentFeed.platform === 'BILI' && entry.coverUrl && !failedCovers[entry.coverUrl]"
                  :src="entry.coverUrl"
                  :alt="`${entry.title} 封面`"
                  referrerpolicy="no-referrer"
                  @error="markCoverFailed(entry.coverUrl)"
                />
                <span v-else>{{ currentFeed.platform }}</span>
              </div>
              <div class="feed-body">
                <div class="meta-row">
                  <span>{{ entry.date }}</span>
                  <span v-if="entry.stats">{{ entry.stats }}</span>
                </div>
                <h2>{{ entry.title }}</h2>
                <p>{{ entry.summary }}</p>
              </div>
            </a>
          </article>
        </div>
      </section>
    </main>

    <footer class="site-footer">
      <span>© {{ new Date().getFullYear() }} {{ profile.name }}</span>
      <span>认真生活，慢慢记录。</span>
    </footer>
  </div>
</template>
