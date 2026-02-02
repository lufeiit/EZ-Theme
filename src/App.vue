<template>
  <div>
    <!-- 静态布局容器，包含不需要过渡效果的菜单和按钮 -->
    <div class="static-layout" v-if="$route.meta.requiresAuth">
      <!-- 网站名称 -->
      <div class="site-logo">
        <img v-if="siteConfig.showLogo" src="/images/logo.png" alt="Logo" class="site-logo-img" />
        {{ siteConfig.siteName }}
      </div>
      
      <!-- 顶部导航栏 - 保持不变 -->
      <SlideTabsNav />
      
      <!-- 顶部工具栏：语言选择器、主题切换和用户头像 -->
      <div class="top-toolbar">
        <ThemeToggle />
        <LanguageSelector />
        <button 
          v-if="PROFILE_CONFIG.showGiftCardRedeem" 
          class="gift-btn" 
          @click="$router.push('/profile')"
        >
          <IconGift :size="20" />
        </button>
        <UserAvatar :username="username" :avatarUrl="avatarUrl" />
      </div>
    </div>

    <!-- 认证页面顶部工具栏，确保认证页面也有语言切换器 -->
    <div class="auth-toolbar" v-if="!$route.meta.requiresAuth && $route.path.includes('/auth')">
      <div class="top-toolbar">
        <ThemeToggle />
        <LanguageSelector />
      </div>
    </div>

    <!-- 路由视图只对内容部分应用过渡效果 -->
    <router-view v-slot="{ Component, route }">
      <transition 
        name="page-transition" 
        mode="out-in"
        appear
      >
        <keep-alive :include="cachedRoutes" :max="5">
          <component 
            :is="Component" 
            :key="route.path"
            :is-active="true"
          />
        </keep-alive>
      </transition>
    </router-view>
    
    <!-- 全局Toast通知 - 放在最外层，确保不受页面切换影响 -->
    <Toast />
    
    <!-- 返回顶部按钮 -->
    <BackToTop />
    
    <!-- 自定义鼠标右键菜单 -->
    <CustomContextMenu />
    
    <!-- 客服图标 -->
    <CustomerServiceIcon v-if="$route.path !== '/customer-service'" />
    
    <!-- Crisp嵌入组件（第二种客服系统方案） -->
    <CrispEmbed v-if="customerServiceConfig.embedMode === 'embed'" />
    
    <!-- 资源预加载组件 -->
    <ResourcePreloader />
    
    <!-- SVG图标定义 -->
    <IconDefinitions />
  </div>
</template>

<script>
import { onMounted, onUnmounted, ref, computed, provide, watch } from 'vue';
import { useStore } from 'vuex';
import { useTheme } from '@/composables/useTheme';
import { useRouter, useRoute } from 'vue-router';
import { SITE_CONFIG, PROFILE_CONFIG, CUSTOMER_SERVICE_CONFIG } from '@/utils/baseConfig';
import { checkAuthAndReloadMessages } from '@/utils/authUtils';
import { checkUserLoginStatus } from '@/api/auth';
import { handleRedirectPath } from '@/utils/redirectHandler';
import Toast from '@/components/common/Toast.vue';
import IconDefinitions from '@/components/icons/IconDefinitions.vue';
import SlideTabsNav from '@/components/common/SlideTabsNav.vue';
import ThemeToggle from '@/components/common/ThemeToggle.vue';
import LanguageSelector from '@/components/common/LanguageSelector.vue';
import UserAvatar from '@/components/common/UserAvatar.vue';
import BackToTop from '@/components/common/BackToTop.vue';
import CustomContextMenu from '@/components/common/CustomContextMenu.vue';
import CustomerServiceIcon from '@/components/common/CustomerServiceIcon.vue';
import CrispEmbed from '@/components/common/CrispEmbed.vue';
import ResourcePreloader from '@/components/common/ResourcePreloader.vue';
import { IconGift } from '@tabler/icons-vue';
import NProgress from 'nprogress';
import 'nprogress/nprogress.css';
import pageCache from '@/utils/pageCache';
// Chatwoot 客服系统集成工具
// 功能：自动加载聊天 Widget，同步用户信息（25+元数据）
// 文档：参考 CHATWOOT_INTEGRATION.md 和 QUICK_START_CHATWOOT.md
import { initChatwootWidget, setupChatwootSync } from '@/utils/chatwoot';
// 导入用户信息 API
import { getUserInfo } from '@/api/user';

NProgress.configure({ 
  showSpinner: true,   
  easing: 'ease',      
  speed: 400,          
  minimum: 0.2         
});

export default {
  name: 'App',
  components: {
    Toast,
    IconDefinitions,
    SlideTabsNav,
    ThemeToggle,
    LanguageSelector,
    UserAvatar,
    BackToTop,
    CustomContextMenu,
    CustomerServiceIcon,
    CrispEmbed,
    ResourcePreloader,
    IconGift
  },
  setup() {
    const router = useRouter();
    const route = useRoute();
    const store = useStore();
    const { applyTheme } = useTheme();
    const siteConfig = ref(SITE_CONFIG);
    const cachedRoutes = computed(() => pageCache.getCachedRoutes());
    
    const customerServiceConfig = computed(() => CUSTOMER_SERVICE_CONFIG);
    
    router.beforeEach((to, from, next) => {
      if (to.meta.keepAlive && to.name) {
        pageCache.addRouteToCache(to.name);
      }
      
      if (from.name && from.meta.keepAlive === false) {
        pageCache.removeRouteFromCache(from.name);
      }
      
      NProgress.start();
      next();
    });
    
    router.afterEach(() => {
      NProgress.done();
    });
    
    const handleRedirectParam = () => {
      let redirectParam = null;
      
      const hashParts = window.location.hash.split('?');
      if (hashParts.length > 1) {
        const hashParams = new URLSearchParams(hashParts[1]);
        redirectParam = hashParams.get('redirect');
      }
      
      if (!redirectParam) {
        redirectParam = route.query.redirect;
      }
      
      if (redirectParam && typeof redirectParam === 'string') {
        const targetPath = handleRedirectPath(redirectParam);
        
        if (route.path !== targetPath) {
          router.replace(targetPath);
        }
      }
    };
    
    watch(() => route.fullPath, () => {
      handleRedirectParam();
    });
    
    const username = computed(() => store.getters.username);
    const avatarUrl = computed(() => store.getters.avatarUrl || '');
    
    const languageChangedSignal = ref(0);
    
    const onLanguageChanged = () => {
      languageChangedSignal.value++;
      
      setTimeout(() => {
        document.body.classList.add('language-transitioning');
        setTimeout(() => {
          document.body.classList.remove('language-transitioning');
        }, 300);
      }, 0);
    };
    
    const handleVisibilityChange = () => {
      if (!document.hidden) {
        checkAuthAndReloadMessages();
        
        checkUserLoginStatus().then(result => {
          if (result.isLoggedIn === false && result.message) {
            const { showToast } = require('@/composables/useToast').useToast();
            if (showToast) {
              showToast(result.message, 'warning');
            }
          }
        }).catch(err => {
          console.error('检查登录状态出错:', err);
        });
      }
    };
    
    provide('languageChangedSignal', languageChangedSignal);
    
    const clearCache = () => {
      pageCache.clearCache();
    };
    
    const removeCachedRoute = (routeName) => {
      pageCache.removeRouteFromCache(routeName);
    };
    
    provide('clearCache', clearCache);
    provide('removeCachedRoute', removeCachedRoute);
    
    onMounted(() => {
      window.addEventListener('languageChanged', onLanguageChanged);
      
      applyTheme(store.getters.currentTheme);
      
      checkAuthAndReloadMessages();
      
      document.addEventListener('visibilitychange', handleVisibilityChange);
      
      // 检查登录状态并加载用户信息
      checkUserLoginStatus().then(async result => {
        if (result.isLoggedIn === false && result.message) {
          const { showToast } = require('@/composables/useToast').useToast();
          if (showToast) {
            showToast(result.message, 'warning');
          }
        } else if (result.isLoggedIn === true) {
          // 用户已登录，加载用户信息到 store
          try {
            const userInfoResponse = await getUserInfo();
            if (userInfoResponse && userInfoResponse.data) {
              const userData = userInfoResponse.data;
              
              console.log('[App] 用户信息已加载:', userData.email);
              console.log('[App] plan_id:', userData.plan_id, 'plan:', userData.plan);
              console.log('[App] invite_user_id:', userData.invite_user_id || '(无推荐人)');
              
              // 获取订阅信息（包含在线设备数 alive_ip 和流量数据 u/d）
              try {
                const { getSubscribe } = await import('@/api/dashboard.js');
                const subResponse = await getSubscribe();
                if (subResponse && subResponse.data) {
                  const subData = subResponse.data;
                  // 在线设备数
                  if (subData.alive_ip !== undefined) {
                    userData.alive_ip = subData.alive_ip;
                  }
                  // 流量数据
                  if (subData.u !== undefined) userData.u = subData.u;
                  if (subData.d !== undefined) userData.d = subData.d;
                  if (subData.transfer_enable !== undefined) userData.transfer_enable = subData.transfer_enable;
                  
                  console.log('[App] 订阅数据已同步:', { 
                    alive_ip: userData.alive_ip, 
                    u: userData.u, 
                    d: userData.d,
                    transfer_enable: userData.transfer_enable 
                  });
                }
              } catch (subError) {
                console.warn('[App] 获取订阅信息失败:', subError.message);
              }
              
              // 获取邀请统计信息（包含邀请人数、累计佣金、确认中的佣金）
              try {
                const { getInviteData } = await import('@/api/invite.js');
                const inviteResponse = await getInviteData();
                if (inviteResponse && inviteResponse.data && inviteResponse.data.stat) {
                  const stat = inviteResponse.data.stat;
                  // stat[0]: 邀请人数
                  // stat[1]: 已确认佣金（单位：分）
                  // stat[2]: 确认中的佣金（单位：分）
                  userData.invite_stats = {
                    registered_users: stat[0] || 0,
                    total_commission: stat[1] || 0,
                    pending_commission: stat[2] || 0
                  };
                  console.log('[App] 邀请数据已同步:', userData.invite_stats);
                }
              } catch (inviteError) {
                console.warn('[App] 获取邀请信息失败:', inviteError.message);
              }
              
              // 尝试获取推荐人邮箱（通过 invite_user_id 查找）
              if (userData.invite_user_id) {
                try {
                  const { getInviteDetails } = await import('@/api/invite.js');
                  // 获取邀请详情，尝试从中找到 invite_user_id 对应的用户邮箱
                  const detailsResponse = await getInviteDetails(1, 100);
                  if (detailsResponse && detailsResponse.data) {
                    // 尝试查找当前用户的记录，其中可能包含推荐人信息
                    console.log('[App] 邀请详情 API 返回数据，无法直接获取推荐人邮箱');
                  }
                } catch (detailsError) {
                  console.warn('[App] 获取邀请详情失败:', detailsError.message);
                }
              }
              
              // 如果有 plan_id 但没有 plan 对象，尝试从套餐列表中匹配
              if (userData.plan_id && !userData.plan) {
                console.log('[App] 尝试获取套餐列表...');
                try {
                  const { fetchPlans } = await import('@/api/shop.js');
                  const plansResponse = await fetchPlans();
                  console.log('[App] 套餐列表响应:', plansResponse);
                  if (plansResponse && plansResponse.data) {
                    console.log('[App] 可用套餐:', plansResponse.data.map(p => ({ id: p.id, name: p.name, type: typeof p.id })));
                    // 查找匹配的套餐（兼容类型转换）
                    const matchedPlan = plansResponse.data.find(p => p.id == userData.plan_id); // 使用 == 而不是 === 允许类型转换
                    if (matchedPlan) {
                      userData.plan = matchedPlan;
                      console.log('[App] ✅ 已匹配套餐信息:', userData.plan.name);
                    } else {
                      // 套餐可能被隐藏（已下架但老用户仍在使用），创建一个临时 plan 对象显示 ID
                      userData.plan = { 
                        id: userData.plan_id, 
                        name: `套餐 #${userData.plan_id}`
                      };
                      console.warn('[App] ⚠️ 套餐已隐藏，使用 fallback 名称:', userData.plan.name);
                    }
                    
                    // 更新 store（会自动触发 Chatwoot 同步）
                    store.dispatch('setUser', userData);
                    console.log('[App] 🔄 已更新套餐信息到 store，watch 会自动同步到 Chatwoot');
                  }
                } catch (planError) {
                  console.error('[App] ❌ 获取套餐列表失败:', planError.message);
                }
              }
              
              store.dispatch('setUser', userData);
            }
          } catch (error) {
            console.error('[App] 加载用户信息失败:', error);
          }
        }
      }).catch(err => {
        console.error('检查登录状态出错:', err);
      });
      
      handleRedirectParam();
      
      // ==================== Chatwoot 客服系统集成 ====================
      // 初始化 Chatwoot Widget（聊天窗口）
      // 配置：src/config/index.js -> CUSTOMER_SERVICE_CONFIG.chatwoot
      // 功能：加载聊天 SDK，显示聊天图标
      initChatwootWidget();
      
      // 设置自动同步用户信息到 Chatwoot
      // 同步时机：用户登录时、定时刷新（默认5分钟）
      // 同步数据：25+用户元数据（余额、流量、订阅、账户状态等）
      // 客服可在 Chatwoot 对话界面右侧查看完整用户信息
      setupChatwootSync(store);
    });
    
    onUnmounted(() => {
      window.removeEventListener('languageChanged', onLanguageChanged);
      document.removeEventListener('visibilitychange', handleVisibilityChange);
    });
    
    return {
      username,
      avatarUrl,
      siteConfig,
      PROFILE_CONFIG,
      cachedRoutes,
      customerServiceConfig
    };
  }
};
</script>

<style lang="scss">
@use "sass:math";
@use "@/assets/styles/base/variables.scss" as *;
@use "@/assets/styles/base/reset.scss" as *;
@use "@/assets/styles/base/animations.scss" as *;
@use "@/assets/styles/base/scrollbar.scss" as *;


.page-transitioning {
  overflow: hidden;
}


.static-layout {
  position: fixed;
  width: 100%;
  top: 0;
  left: 0;
  z-index: 100;
}


.site-logo {
  position: fixed;
  top: 20px;  
  left: 25px;
  font-size: 20px;  
  font-weight: 700;
  color: var(--theme-color);
  z-index: 110;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  letter-spacing: -0.5px;
  background-color: rgba(255, 255, 255, 0.7);
  backdrop-filter: blur(8px);
  -webkit-backdrop-filter: blur(8px);
  padding: 6px 14px;
  border-radius: 10px;  
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 10px;
  
  .site-logo-img {
    height: 24px;
    width: 24px;
    border-radius: 6px;
    object-fit: cover;
  }
}


.dark-theme .site-logo {
  background-color: rgba(30, 30, 30, 0.7);
}


.top-toolbar {
  position: fixed;
  top: 20px;
  right: 25px;
  display: flex;
  gap: 12px;
  z-index: 110;
  
  .gift-btn {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 38px;
    height: 38px;
    border-radius: 50%;
    background-color: rgba(var(--theme-color-rgb), 0.1);
    border: 1px solid rgba(var(--theme-color-rgb), 0.3);
    color: var(--theme-color);
    cursor: pointer;
    transition: all 0.3s ease;
    
    &:hover {
      box-shadow: 0 0 0 3px rgba(var(--theme-color-rgb), 0.15);
      transform: translateY(-2px);
    }
  }
}


@media (max-width: 768px) {
  .site-logo {
    top: 12px;  
    left: 20px;
    font-size: 20px;  
    padding: 5px 10px;
    border-radius: 8px;
  }
  
  .top-toolbar {
    top: 12px;  
    right: 20px;
    gap: 10px;
  }
  
  
  main, .main-content, .content-container {
    padding-bottom: 70px !important;
    margin-bottom: 10px !important;
  }
}


.page-transition-enter-active,
.page-transition-leave-active {
  transition: opacity 0.3s ease;
}

.page-transition-enter-from {
  opacity: 0;
}

.page-transition-leave-to {
  opacity: 0;
}


.language-transitioning .language-transition-item {
  animation: language-fade 0.3s ease-out;
}

@keyframes language-fade {
  0% {
    opacity: 0.2;
  }
  100% {
    opacity: 1;
  }
}


.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}


::-webkit-scrollbar {
  width: 6px;
  height: 6px;
}

::-webkit-scrollbar-track {
  background-color: var(--input-bg-color, rgba(0, 0, 0, 0.05));
  border-radius: 3px;
}

::-webkit-scrollbar-thumb {
  background-color: var(--theme-color);
  border-radius: 3px;
  opacity: 0.7;
  transition: background-color 0.3s ease;
}

::-webkit-scrollbar-thumb:hover {
  background-color: var(--theme-hover-color, rgba(var(--theme-color-rgb), 0.8));
}

::-webkit-scrollbar-corner {
  background-color: transparent;
}


* {
  scrollbar-width: thin;
  scrollbar-color: var(--theme-color) var(--input-bg-color, rgba(0, 0, 0, 0.05));
}


html {
  scroll-behavior: smooth;
}


.auth-toolbar {
  position: fixed;
  top: 0;
  right: 0;
  z-index: 100;
  
  .top-toolbar {
    position: fixed;
    top: 20px;
    right: 25px;
    display: flex;
    gap: 12px;
    z-index: 110;
  }
}


.eztheme-btn {
  text-decoration: none !important;
  border-bottom: none !important;
  background-image: none !important;
  background-repeat: no-repeat !important;
  background-position: initial !important;
  background-size: initial !important;
  
  &:hover, &:active, &:focus, &:visited {
    text-decoration: none !important;
    border-bottom: none !important;
  }
  
  &::after, &::before {
    display: none !important;
    content: none !important;
  }
}


#nprogress {
  pointer-events: none;
  
  .bar {
    background: var(--theme-color);
    position: fixed;
    z-index: 1031;
    top: 0;
    left: 0;
    width: 100%;
    height: 2px;
    box-shadow: 0 0 10px var(--theme-color), 0 0 5px var(--theme-color);
  }
  
  
  .spinner {
    display: block;
    position: fixed;
    z-index: 1031;
    top: 10px;  
    left: 10px; 
    
    .spinner-icon {
      width: 18px;
      height: 18px;
      box-sizing: border-box;
      border: solid 2px transparent;
      border-top-color: var(--theme-color);
      border-left-color: var(--theme-color);
      border-radius: 50%;
      animation: nprogress-spinner 400ms linear infinite;
    }
  }
}

@keyframes nprogress-spinner {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}


.nprogress-custom-parent {
  overflow: hidden;
  position: relative;
}

.nprogress-custom-parent #nprogress .bar {
  position: absolute;
}
</style> 
