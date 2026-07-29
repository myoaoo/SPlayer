<template>
  <Provider>
    <!-- 主框架 -->
    <n-layout :class="['all-layout', { 'full-player': showFullPlayer }]">
      <!-- 导航栏 -->
      <n-layout-header bordered>
        <MainNav />
      </n-layout-header>
      <!-- 主内容 - 有侧边栏 -->
      <n-layout
        v-if="showSider"
        :class="{
          'body-layout': true,
          'player-bar': music.getPlaySongData?.id && showPlayBar,
        }"
        position="absolute"
        has-sider
      >
        <!-- 侧边栏 -->
        <n-layout-sider
          :collapsed="asideMenuCollapsed"
          :native-scrollbar="false"
          :collapsed-width="64"
          :width="240"
          class="main-sider"
          show-trigger="bar"
          collapse-mode="width"
          bordered
          @collapse="asideMenuCollapsed = true"
          @expand="asideMenuCollapsed = false"
        >
          <div class="sider-all">
            <Menu />
          </div>
        </n-layout-sider>
        <!-- 页面区 -->
        <n-layout :native-scrollbar="false" embedded>
          <MainLayout />
        </n-layout>
      </n-layout>
      <!-- 主内容 - 无侧边栏 -->
      <n-layout-content
        v-else
        :class="{
          'body-layout': true,
          'player-bar': music.getPlaySongData?.id && showPlayBar,
        }"
        :native-scrollbar="false"
        position="absolute"
        embedded
      >
        <MainLayout />
      </n-layout-content>
    </n-layout>
    <!-- 主播放器 -->
    <MainControl />
    <!-- 全屏播放器 -->
    <FullPlayer />
    <!-- 全局播放列表 -->
    <n-config-provider v-if="showFullPlayer" :theme="darkTheme">
      <Playlist />
    </n-config-provider>
    <Playlist v-else />
    <!-- 全局水印 -->
    <!-- <n-watermark
      :font-size="16"
      :line-height="16"
      :width="384"
      :height="384"
      :x-offset="12"
      :y-offset="60"
      :rotate="-15"
      content="开发中，敬请期待"
      cross
      fullscreen
    /> -->
  </Provider>
</template>

<script setup>
import { storeToRefs } from "pinia";
import { useRouter } from "vue-router";
import { darkTheme, NButton } from "naive-ui";
import { musicData, siteStatus, siteSettings } from "@/stores";
import { checkPlatform, setViewportHeight } from "@/utils/helper";
import { initPlayer } from "@/utils/Player";
import userSignIn from "@/utils/userSignIn";
import globalShortcut from "@/utils/globalShortcut";
import globalEvents from "@/utils/globalEvents";
import packageJson from "@/../package.json";

const router = useRouter();
const music = musicData();
const status = siteStatus();
const settings = siteSettings();
const { autoPlay, showSider, autoSignIn, autoCheckUpdates } = storeToRefs(settings);
const { showPlayBar, asideMenuCollapsed, showFullPlayer } = storeToRefs(status);

// 公告数据
const annShow =
  import.meta.env.RENDERER_VITE_ANN_TITLE && import.meta.env.RENDERER_VITE_ANN_CONTENT
    ? true
    : false;
const annType = import.meta.env.RENDERER_VITE_ANN_TYPE;
const annTitle = import.meta.env.RENDERER_VITE_ANN_TITLE;
const annContene = import.meta.env.RENDERER_VITE_ANN_CONTENT;
const annDuration = Number(import.meta.env.RENDERER_VITE_ANN_DURATION);

// PWA
if ("serviceWorker" in navigator) {
  // 更新完成提醒
  navigator.serviceWorker.addEventListener("controllerchange", () => {
    if (checkPlatform.electron()) {
      $notification.create({
        title: "🎉 有更新啦",
        content: "检测到软件内资源有更新，是否重新启动软件以应用更新？",
        meta: "当前版本 v " + (packageJson.version || "1.0.0"),
        action: () =>
          h(
            NButton,
            {
              text: true,
              type: "primary",
              onClick: () => {
                electron.ipcRenderer.send("window-relaunch");
              },
            },
            {
              default: () => "更新",
            },
          ),
        onAfterLeave: () => {
          $message.info("已取消本次更新，更新将在下次启动软件后生效", {
            duration: 6000,
          });
        },
      });
    } else {
      console.info("站点资源有更新，请刷新以应用更新");
      $message.info("站点资源有更新，请刷新以应用更新", {
        closable: true,
        duration: 0,
      });
    }
  });
}

// 自动检查更新
const checkUpdates = () => {
  if (!checkPlatform.electron()) return false;
  electron.ipcRenderer.send("check-updates");
};

// 显示公告
const showAnnouncements = () => {
  if (annShow) {
    $notification[annType]({
      content: annTitle,
      meta: annContene,
      duration: annDuration,
    });
  }
};

// 网络无法连接
const canNotConnect = (error) => {
  console.error("网络连接错误：", error.message);
  $dialog.destroyAll();
  $dialog.error({
    title: "网络连接错误",
    content: "网络连接错误，请检查您当前的网络状态",
    positiveText: "重试",
    negativeText: "知道了",
    onPositiveClick: () => {
      location.reload();
    },
  });
};

// 网页端键盘事件
const handleKeyUp = (event) => {
  globalShortcut(event, router);
};

onMounted(async () => {
  // 挂载方法
  window.$canNotConnect = canNotConnect;
  // 设置移动端视口高度
  if (!checkPlatform.electron()) {
    setViewportHeight();
  }
  // 主播放器
  await initPlayer(autoPlay.value);
  // 更改全局字体
  settings.changeSystemFonts();
  // 全局事件
  globalEvents(router);
  // 键盘监听
  if (!checkPlatform.electron()) {
    window.addEventListener("keyup", handleKeyUp);
  }
  // 自动签到
  if (autoSignIn.value) await userSignIn();
  // 检查更新
  if (autoCheckUpdates.value) checkUpdates();
  // 显示公告
  showAnnouncements();
});

onUnmounted(() => {
  if (!checkPlatform.electron()) window.removeEventListener("keyup", handleKeyUp);
});
</script>

<style lang="scss" scoped>
.all-layout {
  height: 100%;
  transition:
    transform 0.3s,
    opacity 0.3s;
  .n-layout-header {
    height: 60px;
    display: flex;
    flex-direction: row;
    align-items: center;
    -webkit-app-region: drag;
    padding-top: env(safe-area-inset-top);
    min-height: calc(60px + env(safe-area-inset-top));
  }
  .body-layout {
    top: calc(60px + env(safe-area-inset-top));
    transition: bottom 0.3s;
    .main-sider {
      :deep(.n-scrollbar-content) {
        height: 100%;
      }
      .sider-all {
        height: 100%;
      }
      @media (max-width: 900px) {
        display: none;
      }
    }
    &.player-bar {
      bottom: 80px;
    }
  }
  &.full-player {
    opacity: 0;
    transform: scale(0.9);
  }
}
</style>
