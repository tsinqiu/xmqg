<script setup>
import { computed, onMounted, ref, watch } from 'vue'
import { aboutSections, dailyFacts, feeds, navItems, profile, socialLinks, timeline } from './data/site'

const view = ref('home')
const failedCovers = ref({})
const feedKeys = Object.keys(feeds)

const allViews = computed(() => [...navItems.map((item) => item.id), ...feedKeys])
const currentFeed = computed(() => feeds[view.value])

const today = new Intl.DateTimeFormat('zh-CN', {
  year: 'numeric',
  month: 'long',
  day: 'numeric',
  weekday: 'long',
}).format(new Date())

const todayFact = computed(() => {
  const dayIndex = new Date().getDate() % dailyFacts.length
  return dailyFacts[dayIndex]
})

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

onMounted(() => {
  const hashView = window.location.hash.replace('#', '')
  if (allViews.value.includes(hashView)) {
    view.value = hashView
  } else if (hashView) {
    setView('home')
  }
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
            <span>你知道吗</span>
            <p>{{ todayFact }}</p>
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
