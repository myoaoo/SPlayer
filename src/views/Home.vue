<!-- 首页 -->
<template>
  <div class="home">
    <!-- 欢迎 -->
    <div class="greetings">
      <n-h1 class="welcome">
        {{ getGreetings() }}
      </n-h1>
      <n-text depth="3">由此开启好心情 ~</n-text>
    </div>
    <!-- 专属推荐 -->
    <div class="rec-private">
      <n-grid class="rec-private" cols="1" x-gap="20" y-gap="20">
        <!-- 每日推荐 -->
        <n-gi>
          <SpecialCover
            :data="dailySongsCoverData"
            :showIcon="false"
            showDate
            @click="jumpPage('daily-songs')"
          />
        </n-gi>
        <!-- 喜欢的音乐 -->
        <n-gi>
          <SpecialCover :data="likeSongsCoverData" @click="jumpPage('like-songs')" />
        </n-gi>
      </n-grid>
      <PrivateFm class="rec-fm" />
    </div>
    <!-- 自定义网易云歌单 -->
    <div v-if="customPlaylistData.data.length" class="rec-public">
      <n-h3 class="title" prefix="bar">
        <n-text class="name">{{ customPlaylistData.name }}</n-text>
      </n-h3>
      <MainCover
        :data="customPlaylistData.data"
        :loadingNum="customPlaylistData.loadingNum"
        :columns="customPlaylistData.columns"
      />
    </div>

    <!-- 本地歌单 -->
    <div v-if="localPlaylistData.data.length" class="rec-public">
      <n-h3 class="title" prefix="bar">
        <n-text class="name">{{ localPlaylistData.name }}</n-text>
      </n-h3>
      <MainCover
        :data="localPlaylistData.data"
        :loadingNum="localPlaylistData.loadingNum"
        :columns="localPlaylistData.columns"
      />
    </div>

    <!-- 公共推荐 -->
    <div v-for="(item, index) in recommendData" :key="index" class="rec-public">
      <n-h3 class="title" prefix="bar" @click="item.to ? router.push(item.to) : null">
        <n-text class="name">{{ item.name }}</n-text>
        <n-icon v-if="item.to" class="more" depth="3">
          <SvgIcon icon="chevron-right" />
        </n-icon>
      </n-h3>
      <MainCover
        :data="item.data"
        :type="item.type"
        :loadingNum="item.loadingNum"
        :columns="item.columns"
      />
    </div>
  </div>
</template>

<script setup>
import { storeToRefs } from "pinia";
import { useRouter } from "vue-router";
import { getGreetings } from "@/utils/timeTools";
import {
  getDailyRec,
  getPersonalized,
  getRadarPlaylist,
  getTopArtists,
  getNewAlbum,
} from "@/api/recommend";
import { getPlayListDetail } from "@/api/playlist";
import { allMv } from "@/api/video";
import { getDjRecommend } from "@/api/dj";
import { siteData, siteSettings } from "@/stores";
import { getCacheData, checkPlatform } from "@/utils/helper";
import { isLogin } from "@/utils/auth";
import formatData from "@/utils/formatData";

const data = siteData();
const router = useRouter();
const settings = siteSettings();
const { showSider } = storeToRefs(settings);
const { userLikeData, dailySongsData } = storeToRefs(data);

// 每日推荐
const dailySongsCoverData = computed(() => {
  const dailySongsCover = {
    name: "每日推荐",
    desc: "根据你的音乐口味 · 每日更新",
  };
  if (isLogin() && dailySongsData.value.data?.length) {
    const randomIndex = Math.floor(Math.random() * dailySongsData.value.data.length);
    dailySongsCover.cover =
      dailySongsData.value.data[randomIndex]?.coverSize?.s || "/imgs/pic/like.jpg";
    return dailySongsCover;
  }
  dailySongsCover.cover = "/imgs/pic/cover-2.jpg";
  return dailySongsCover;
});

// 喜欢的音乐
const likeSongsCoverData = computed(() => {
  const likeSongsCover = {
    id: 1024,
    name: "喜欢的音乐",
    desc: "发现你独特的音乐品味",
  };
  if (isLogin() && userLikeData.value.playlists?.length) {
    likeSongsCover.cover = userLikeData.value.playlists[0]?.coverSize?.s || "/imgs/pic/like.jpg";
    return likeSongsCover;
  }
  likeSongsCover.cover = "/imgs/pic/like.jpg";
  return likeSongsCover;
});

// 自定义网易云歌单列表
const customPlaylistIds = [5471077812, 12654888896, 12584586769, 12510385889, 8855877957, 8725429117, 8725850622, 8185045795, 8184817453, 8173191976, 8173838382, 5471042889];
const customPlaylistData = ref({
  name: "我的歌单",
  loadingNum: customPlaylistIds.length,
  columns: showSider.value ? undefined : "2 s:3 m:4 l:5 xl:6",
  data: [],
});

// 本地歌单数据
const localPlaylistData = ref({
  name: "本地歌单",
  loadingNum: 4,
  columns: showSider.value ? undefined : "2 s:3 m:4 l:5 xl:6",
  data: [],
});

// 个性化推荐数据
const recommendData = ref({
  playlist: {
    name: isLogin() ? "专属歌单" : "推荐歌单",
    loadingNum: 12,
    columns: showSider.value ? undefined : "2 s:3 m:4 l:5 xl:6",
    data: [],
    to: "/discover/playlists",
  },
  radar: {
    name: "雷达歌单",
    loadingNum: 6,
    columns: showSider.value ? undefined : "2 s:3 m:4 l:5 xl:6",
    data: [],
    to: "/discover/playlists",
  },
  artist: {
    name: "歌手推荐",
    type: "artist",
    columns: showSider.value ? "3 s:4 m:5 l:6" : "3 sm:4 m:5 l:6",
    loadingNum: 6,
    data: [],
    to: "/discover/artists",
  },
  mv: {
    name: "推荐 MV",
    type: "mv",
    columns: "2 s:2 m:3 l:4 xl:5",
    loadingNum: 12,
    data: [],
  },
  dj: {
    name: "推荐播客",
    type: "dj",
    loadingNum: 6,
    data: [],
    to: "/dj-hot",
  },
  album: {
    name: "新碟上架",
    type: "album",
    loadingNum: 12,
    data: [],
    to: "/discover/new",
  },
});

// 获取自定义网易云歌单
const getCustomPlaylists = async () => {
  try {
    // 添加加载缓存，避免重复请求
    const cacheKey = `custom_playlists_cache_${customPlaylistIds.join("_")}`;
    const cacheTime = 10 * 60 * 1000; // 10分钟缓存
    const cached = localStorage.getItem(cacheKey);
    const now = Date.now();
    
    if (cached) {
      const { data: cachedData, timestamp } = JSON.parse(cached);
      if (now - timestamp < cacheTime) {
        customPlaylistData.value.data = cachedData;
        return;
      }
    }
    
    const customRes = await Promise.allSettled(
      customPlaylistIds.map((id) => getPlayListDetail(id)),
    );
    const result = customRes
      .filter((res) => res.status === "fulfilled" && res.value.playlist)
      .map((res) => {
        const formatted = formatData(res.value.playlist);
        return formatted && formatted.length > 0 ? formatted[0] : null;
      })
      .filter((item) => item !== null);
    
    customPlaylistData.value.data = result;
    
    // 缓存新数据
    localStorage.setItem(cacheKey, JSON.stringify({ data: result, timestamp: now }));
  } catch (error) {
    console.error("获取自定义歌单失败：", error);
  }
};

// 获取本地歌单
const getLocalPlaylists = async () => {
  try {
    const cacheKey = "local_playlist_cache";
    const cacheTime = 30 * 60 * 1000; // 30分钟缓存
    const now = Date.now();
    
    // 先从缓存加载，保证快速显示
    const cached = localStorage.getItem(cacheKey);
    if (cached) {
      const { data: cachedData, timestamp } = JSON.parse(cached);
      // 即使过期也先用缓存显示，然后后台更新
      if (cachedData && Object.keys(cachedData).length > 0) {
        const playlists = formatLocalPlaylists(cachedData);
        localPlaylistData.value.data = playlists;
      }
      
      // 如果缓存还有效，直接返回
      if (now - timestamp < cacheTime) {
        return;
      }
    }
    
    // 生产环境使用完整URL，开发环境使用代理
    const playlistUrl = checkPlatform.electron() && import.meta.env.DEV
      ? "/local-playlist/music6.json"
      : "https://code.oaoo.top/music6.json";
    
    const response = await fetch(playlistUrl);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    const data = await response.json();
    
    // 缓存新数据
    localStorage.setItem(cacheKey, JSON.stringify({ data, timestamp: now }));
    
    const playlists = formatLocalPlaylists(data);
    localPlaylistData.value.data = playlists;
  } catch (error) {
    console.error("获取本地歌单失败：", error);
  }
};

// 格式化本地歌单数据
const formatLocalPlaylists = (data) => {
  return Object.keys(data).map((name) => ({
    id: `local-${name}`,
    name: name,
    cover: data[name].cover,
    coverSize: {
      s: data[name].cover,
      m: data[name].cover,
      l: data[name].cover,
      xl: data[name].cover,
    },
    count: data[name].songs.length,
    tracks: data[name].songs.map((songUrl, index) => ({
      id: `local-song-${name}-${index}`,
      name: songUrl.split("/").pop().replace(/\.[^/.]+$/, "").replace(/_/g, " "),
      path: songUrl,
      isLocal: true,
      cover: data[name].cover,
      coverSize: {
        s: data[name].cover,
        m: data[name].cover,
        l: data[name].cover,
        xl: data[name].cover,
      },
      artists: [{ name: "本地音乐" }],
      album: { name: name },
    })),
  }));
};

// 获取个性化推荐数据
const getRecommendData = async () => {
  try {
    const [playlistRes, radarRes, artistRes, mvRes, djRes, albumRes] = await Promise.allSettled([
      // 歌单
      isLogin()
        ? getCacheData("recPl-P", 5, getDailyRec, "resource")
        : getCacheData("recPl", 5, getPersonalized),
      // 雷达歌单
      getCacheData("recRadar", 30, getRadarPlaylist),
      // 歌手
      getCacheData("recAr", 5, getTopArtists),
      // MV
      getCacheData("recMv", 5, allMv),
      // 电台
      getCacheData("recDj", 5, getDjRecommend),
      // 专辑
      getCacheData("recAl", 5, getNewAlbum),
    ]);
    // 检查请求状态
    playlistRes.status === "fulfilled" &&
      (recommendData.value.playlist.data = formatData(
        isLogin()
          ? playlistRes.value.recommend.filter((playlist) => {
              return !playlist.name.includes("私人雷达");
            })
          : playlistRes.value.result,
      ));
    radarRes.status === "fulfilled" &&
      (recommendData.value.radar.data = formatData(radarRes.value));
    artistRes.status === "fulfilled" &&
      (recommendData.value.artist.data = formatData(artistRes.value.artists, "artist"));
    mvRes.status === "fulfilled" &&
      (recommendData.value.mv.data = formatData(mvRes.value.data, "mv"));
    djRes.status === "fulfilled" &&
      (recommendData.value.dj.data = formatData(djRes.value.djRadios, "dj"));
    albumRes.status === "fulfilled" &&
      (recommendData.value.album.data = formatData(albumRes.value.albums, "album"));
    // 检查是否有任何请求失败
    const anyRejected = [playlistRes, radarRes, artistRes, mvRes, albumRes].some(
      (res) => res.status === "rejected",
    );
    if (anyRejected) {
      throw new Error("一个或多个请求失败");
    }
  } catch (error) {
    $message.error("个性化推荐获取失败");
    console.error("个性化推荐获取失败：", error);
  }
};

// 页面跳转
const jumpPage = (key, id) => {
  switch (key) {
    case "playlist":
      router.push({
        path: "/playlist",
        query: { id },
      });
      break;
    case "like-songs":
      router.push("/like-songs");
      break;
    case "daily-songs":
      router.push("/daily-songs");
      break;
    default:
      break;
  }
};

onBeforeMount(() => {
  // 每日推荐
  data.setDailySongsData();
  // 获取自定义歌单
  getCustomPlaylists();
  // 获取本地歌单
  getLocalPlaylists();
  // 个性化推荐
  getRecommendData();
});
</script>

<style lang="scss" scoped>
.home {
  max-width: 1400px;
  margin: 0 auto;
  .greetings {
    margin-bottom: 20px;
    .welcome {
      margin: 0;
      font-weight: bold;
    }
  }
  .title {
    position: relative;
    display: flex;
    flex-direction: row;
    align-items: center;
    margin-left: 2px;
    margin-top: 28px;
    padding-left: 16px;
    cursor: pointer;
    .more {
      font-size: 26px;
      opacity: 0;
      transform: translateX(4px);
      transition:
        opacity 0.3s,
        transform 0.3s;
    }
    &:hover {
      .more {
        opacity: 1;
        transform: translateX(0);
      }
    }
  }
  .rec-private {
    display: flex;
    flex-direction: row;
    align-items: flex-start;
    .rec-fm {
      width: 100%;
      height: 220px;
      margin-left: 20px;
      max-width: calc(50% - 10px);
    }
    @media (max-width: 700px) {
      flex-direction: column;
      .rec-fm {
        margin-left: 0;
        margin-top: 20px;
        max-width: 100%;
      }
    }
  }
}
</style>
