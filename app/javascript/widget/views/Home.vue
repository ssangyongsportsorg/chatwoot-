<template>
  <div
    class="z-50 rounded-md w-full flex flex-1 flex-col"
    :class="{ 'pb-2': showArticles, 'justify-end': !showArticles }"
  >
    <div class="px-4 pt-4 w-full">
      <team-availability
        :available-agents="availableAgents"
        :has-conversation="!!conversationSize"
        :unread-count="unreadMessageCount"
        @start-conversation="startConversation"
      />

      <!-- 新增狀態區塊 -->
      <div class="mt-4 p-4 rounded-md bg-white dark:bg-slate-700 shadow-sm">
        <h2 class="text-lg font-bold">目前服務狀態</h2>
        <p v-if="status">{{ status.page.name }}: {{ status.page.status }}</p>
        <p v-else>加載中...</p>
        <button @click="showStatusPage" class="mt-2 px-4 py-2 bg-blue-500 text-white rounded-md">查看詳情</button>
      </div>
    </div>
    <div v-if="showArticles" class="px-4 py-2 w-full">
      <div class="p-4 rounded-md bg-white dark:bg-slate-700 shadow-sm w-full">
        <article-hero
          v-if="
            !articleUiFlags.isFetching &&
            !articleUiFlags.isError &&
            popularArticles.length
          "
          :articles="popularArticles"
          @view="openArticleInArticleViewer"
          @view-all="viewAllArticles"
        />
      </div>
    </div>
    <div v-if="articleUiFlags.isFetching" class="px-4 py-2 w-full">
      <div class="p-4 rounded-md bg-white dark:bg-slate-700 shadow-sm w-full">
        <article-card-skeleton-loader />
      </div>
    </div>

    <!-- 新增的 iframe 狀態網頁 -->
    <div v-if="showIframe" class="fixed inset-0 bg-gray-800 bg-opacity-75 flex items-center justify-center z-50">
      <div class="bg-white rounded-lg overflow-hidden shadow-xl transform transition-all max-w-3xl w-full">
        <iframe :src="statusPageUrl" class="w-full h-96"></iframe>
        <button @click="closeIframe" class="absolute top-2 right-2 text-white bg-red-500 rounded-full p-2">X</button>
      </div>
    </div>
  </div>
</template>

<script>
import TeamAvailability from 'widget/components/TeamAvailability.vue';
import ArticleHero from 'widget/components/ArticleHero.vue';
import ArticleCardSkeletonLoader from 'widget/components/ArticleCardSkeletonLoader.vue';

import { mapGetters } from 'vuex';
import darkModeMixin from 'widget/mixins/darkModeMixin';
import routerMixin from 'widget/mixins/routerMixin';
import configMixin from 'widget/mixins/configMixin';

export default {
  name: 'Home',
  components: {
    ArticleHero,
    TeamAvailability,
    ArticleCardSkeletonLoader,
  },
  mixins: [configMixin, routerMixin, darkModeMixin],
  props: {
    hasFetched: {
      type: Boolean,
      default: false,
    },
    isCampaignViewClicked: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      status: null,
      showIframe: false,
      statusPageUrl: 'https://status.ssangyongsports.eu.org/', // 替換為您的狀態頁 URL
    };
  },
  computed: {
    ...mapGetters({
      availableAgents: 'agent/availableAgents',
      activeCampaign: 'campaign/getActiveCampaign',
      conversationSize: 'conversation/getConversationSize',
      unreadMessageCount: 'conversation/getUnreadMessageCount',
      popularArticles: 'article/popularArticles',
      articleUiFlags: 'article/uiFlags',
    }),
    widgetLocale() {
      return this.$i18n.locale || 'en';
    },
    portal() {
      return window.chatwootWebChannel.portal;
    },
    showArticles() {
      return (
        this.portal &&
        !this.articleUiFlags.isFetching &&
        this.popularArticles.length
      );
    },
    defaultLocale() {
      const widgetLocale = this.widgetLocale;
      const { allowed_locales: allowedLocales, default_locale: defaultLocale } =
        this.portal.config;

      // IMPORTANT: Variation strict locale matching, Follow iso_639_1_code
      // If the exact match of a locale is available in the list of portal locales, return it
      // Else return the default locale. Eg: `es` will not work if `es_ES` is available in the list
      if (allowedLocales.includes(widgetLocale)) {
        return widgetLocale;
      }
      return defaultLocale;
    },
  },
  mounted() {
    if (this.portal && this.popularArticles.length === 0) {
      const locale = this.defaultLocale;
      this.$store.dispatch('article/fetch', {
        slug: this.portal.slug,
        locale,
      });
    }

    // 獲取狀態
    this.fetchStatus();
  },
  methods: {
    startConversation() {
      if (this.preChatFormEnabled && !this.conversationSize) {
        return this.replaceRoute('prechat-form');
      }
      return this.replaceRoute('messages');
    },
    openArticleInArticleViewer(link) {
      let linkToOpen = `${link}?show_plain_layout=true`;
      const isDark = this.prefersDarkMode;
      if (isDark) {
        linkToOpen = `${linkToOpen}&theme=dark`;
      }
      this.$router.push({
        name: 'article-viewer',
        params: { link: linkToOpen },
      });
    },
    viewAllArticles() {
      const locale = this.defaultLocale;
      const {
        portal: { slug },
      } = window.chatwootWebChannel;
      this.openArticleInArticleViewer(`https://abc.ssport.eu.org/hc/${slug}/${locale}`);
    },
    fetchStatus() {
      fetch('https://status.ssangyongsports.eu.org/summary.json') // 替換為您的狀態頁 API 地址
        .then(response => response.json())
        .then(data => {
          this.status = data;
        })
        .catch(error => {
          console.error('Error fetching status:', error);
        });
    },
    showStatusPage() {
      this.showIframe = true;
    },
    closeIframe() {
      this.showIframe = false;
    },
  },
};
</script>
