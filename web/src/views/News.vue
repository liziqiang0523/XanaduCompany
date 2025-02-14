<template>
  <div class="news">
    <AwHeader class="news_header" ref="headRef"></AwHeader>
    <div class="box">
      <div class="news-banner">
        <div class="banner-title">
          <h2>新视野</h2>
          <h3>了解更多新闻</h3>
        </div>
        <el-autocomplete
          class="search-input"
          popper-class="my-autocomplete"
          highlight-first-item
          v-model="searchNews"
          clearable
          ref="autocomplete"
          @focus="autocompleteFlag = true"
          @blur="autocompleteFlag = false"
          @clear="searchHandle"
          :fetch-suggestions="querySearchAsync"
          placeholder="请输入新闻关键词"
          :trigger-on-focus="false"
        >
          <template #prefix>
            <el-icon>
              <Search />
            </el-icon>
          </template>
          <template #default="{ item }">
            <router-link :to="'/news/'+item.news_path" target="_blank">
              <div class="name" v-html="item.news_title"></div>
              <span class="desc" v-html="item.news_desc"></span>
            </router-link>
          </template>
        </el-autocomplete>
      </div>
      <div class="news-container">
        <div class="news-card">
          <el-card shadow="never" v-for="(item, index) in recomNews" :key="index">
            <router-link :to="`/news/${item.news_path}`">
              <div class="news-card-item">
                <img :src="item.cover_img" alt="" />
                <p class="item-mask">
                  <span>{{ item.news_title }}</span>
                </p>
              </div>
            </router-link>
          </el-card>
        </div>
        <div class="news-list">
          <el-tabs
            class="list-left"
            v-model="pageInfo.activeName"
            @tab-click="handleClick"
          >
            <el-tab-pane :label="newsTabs[0].name" :name="newsTabs[0].id">
              <news-list
                :items="newsItems.list"
                v-if="pageInfo.activeName === newsTabs[0].id"
              ></news-list>
            </el-tab-pane>
            <el-tab-pane :label="newsTabs[1].name" :name="newsTabs[1].id">
              <news-list
                :items="newsItems.list"
                v-if="pageInfo.activeName === newsTabs[1].id"
              ></news-list>
            </el-tab-pane>
            <el-tab-pane :label="newsTabs[2].name" :name="newsTabs[2].id">
              <news-list
                :items="newsItems.list"
                v-if="pageInfo.activeName === newsTabs[2].id"
              ></news-list>
            </el-tab-pane>
            <el-pagination
              class="pagination"
              background
              @current-change="handleCurrentChange"
              v-model:current-page="pageInfo.pagenum"
              v-model:page-size="pageInfo.pagesize"
              layout="prev, pager, next, jumper"
              :total="newsItems.total"
              :hide-on-single-page="singlePage"
              v-scroll-to="{
                element: '.news-container',
                duration: 300,
                easing: 'ease',
                offset: -40,
              }"
            >
            </el-pagination>
          </el-tabs>
          <div class="list-right">
            <div class="search-by-date">
              <p>按日期搜索：</p>
              <el-date-picker
                v-model="pageInfo.selectDate"
                type="month"
                placeholder="选择日期"
                value-format="yyyy-MM"
                @change="searchByDate(pageInfo.selectDate)"
              >
              </el-date-picker>
            </div>
            <hot-news></hot-news>
          </div>
        </div>
      </div>
    </div>
    <AwFooter></AwFooter>
  </div>
</template>
<script lang="ts" setup>
import AwHeader from '@/components/public/Header.vue'
import AwFooter from '@/components/public/Footer.vue'
import HotNews from '@/components/HotNews.vue'
import NewsList from '@/components/NewsList.vue'
import { Search } from '@element-plus/icons-vue'
import { onBeforeMount, onMounted, ref } from 'vue'
import mainStore from '@/store'
import { onBeforeRouteLeave } from 'vue-router'
import { searchNewsList, getNewsList, getRecomNewsApi } from '@/apis/news'
type NewsItem = {
  id: string;
  name: string;
};
const headRef = ref()
const newsTabs = ref<NewsItem[]>([])
const newsItems = ref<Record<any, any>>({})
const pageInfo = ref<{
  activeName: string;
  pagenum: number;
  pagesize: number;
  selectDate: '';
}>({
  activeName: '1',
  // 当前页码
  pagenum: 1,
  // 当前每页显示多少条数据
  pagesize: 10,
  selectDate: ''
})
const singlePage = ref(false)
const recomNews = ref<Array<any>>([])
const selectDate = ref('')
const searchList = ref<Array<any>>([])
const timeout = ref()
const autocomplete = ref()
const autocompleteFlag = ref(false)
const hotNews = ref<Array<any>>([])
const searchNews = ref<any>()
onBeforeMount(() => {
  newsTabs.value = [
    {
      id: '1',
      name: '最新动态'
    },
    {
      id: '2',
      name: '典型案例'
    },
    {
      id: '3',
      name: '通知公告'
    }
  ]
  getNewsItems()
  getRecomNews()
})
onMounted(() => {
  mainStore.commit('setHeaderLogo', {
    headerLogoShow: false
  })
  mainStore.commit('setShadowActive', {
    headerShadowActive: false
  })
  mainStore.commit('setNavDarkActive', {
    navDarkActive: true
  })
  mainStore.commit('setHeaderShow', {
    headerShow: false
  })
})

async function querySearchAsync (queryString: string, cb: (list: Array<any>) => void) {
  searchList.value = []
  const { data: res } = await searchNewsList(queryString)
  searchList.value = res?.data?.list ?? []
  const newHtml = `<span style="color: #3370ff">${queryString}</span>`
  searchList.value.forEach((item) => {
    item.news_title = item.news_title.replace(queryString, newHtml)
    item.news_desc = item.news_desc.replace(queryString, newHtml)
    // item.news_desc = item.news_desc.replace(queryString, newHtml)
  })
  clearTimeout(timeout.value)
  timeout.value = setTimeout(() => {
    cb(searchList.value)
  }, 1000 * Math.random())
}
/**
 * 解决 clearable 搜索框后再次输入不显示下拉
 */
function searchHandle () {
  if (autocompleteFlag.value) {
    autocomplete.value.activated = true
  }
}

/**
 * 选项卡切换
 * @param tab
 * @param event
 */
function handleClick (tab: number | string, event: Event) {
  getNewsItems()
}

/**
 * 新闻列表页切换
 * @param val
 */
function handleCurrentChange (val: number) {
  getNewsItems()
}

async function getNewsItems () {
  const { data: res } = await getNewsList(pageInfo.value)
  if (res.status !== 200) {
    newsItems.value = {}
  } else {
    newsItems.value = res.data
    if (newsItems.value.total <= newsItems.value.limit) {
      singlePage.value = true
    }
  }
}

function searchByDate (data: any) {
  getNewsItems()
}

async function getRecomNews () {
  const { data: res } = await getRecomNewsApi()
  if (res.status !== 200) {
    hotNews.value = []
  } else {
    recomNews.value = res.data.list
  }
}

onBeforeRouteLeave((to, from, next) => {
  if (from.name === 'News') {
    mainStore.commit('setNavDarkActive', {
      navDarkActive: false
    })
    mainStore.commit('setHeaderLogo', {
      headerLogoShow: false
    })
  }
  next()
})
</script>
<style lang="less" scoped>
@hover_color: #3370ff;

* {
  margin: 0;
  padding: 0;
}

.news_header {
  background-color: rgba(255, 255, 255, 0.5);
  backdrop-filter: blur(10px);
  //border-bottom: 1px solid #eff0f1;
}

.box {
  padding-top: 60px;
  //background: url("../../assets/img/news/bg_02.jpg");
  //background-size: cover;
}

.news-banner {
  width: 100%;
  height: 280px;
  background: url("../assets/img/news/newsbanner.jpg") 50% no-repeat;
  background-size: cover;
  text-align: center;
  padding-top: 70px;

  .banner-title {
    padding-bottom: 30px;

    h2 {
      font-size: 40px;
      line-height: 60px;
      font-weight: 600;
    }

    h3 {
      color: #828282;
      margin-top: 5px;
      font-size: 100%;
      font-weight: 400;
      font-variant: normal;
    }
  }
  :deep(.el-autocomplete) {
    width: 46%;
    .el-input__wrapper {
      border-radius: 30px;
    }
    .el-input {
      border-radius: 30px !important;
    }
    .el-input__inner {
      height: 46px;
      line-height: 46px;
      border-radius: 30px;
      box-shadow: 0 2px 4px rgb(0 0 0 / 12%), 0 0 6px rgb(0 0 0 / 4%);
    }
    .el-input__icon {
      line-height: 46px;
      font-size: 16px;
    }
  }
}

.news-container {
  max-width: 1200px;
  //background: #d3dce6;
  min-height: 580px;
  margin: 0 auto;

  .news-card {
    padding-top: 40px;
    margin: 0 auto 50px auto;
    display: flex;
    justify-content: space-evenly;

    //background-color: #d3dce6;
    .el-card {
      width: 280px;
      height: 160px;
      overflow: hidden;
      color: #ffffff;
    }

    .news-card-item {
      width: 280px;
      height: 160px;
      position: relative;
      cursor: pointer;

      .item-mask {
        position: absolute;
        background: linear-gradient(transparent, #030822);
        left: 0;
        bottom: 0;
        width: 100%;
        padding: 50px 20px;
        height: 97px;
        box-sizing: border-box;
      }

      span {
        color: #fff;
        text-align: left;
        overflow: hidden;
        text-overflow: ellipsis;
        display: -webkit-box;
        -webkit-box-orient: vertical;
        -webkit-line-clamp: 2;
      }
    }

    .news-card-item:hover img {
      transform: scale(1.02);
    }

    :deep(.el-card__body) {
      padding: 0;
    }

    img {
      width: 100%;
      height: 100%;
      object-fit: cover;
      transition: all 0.4s ease-in-out;
    }
  }

  .news-list {
    position: relative;
    display: flex;
    justify-content: space-between;
  }

  .news-list:after {
    content: "";
    position: absolute;
    left: 0;
    top: 43px;
    width: 100%;
    height: 2px;
    background-color: #e4e7ed;
    z-index: 1;
  }

  :deep(.el-tabs__header) {
    height: 60px;
  }

  :deep(.el-tabs__active-bar) {
    bottom: 5px;
    //height: 3px;
  }

  :deep(.el-tabs__item) {
    height: 50px;
    line-height: 50px;
    font-weight: 600;
  }

  .list-left {
    width: 860px;

    :deep(.el-tabs__content) {
      height: auto;
      //background-color: #d2d3d4;
    }
  }

  .el-pagination {
    display: flex;
    justify-content: center;
    margin-top: 30px;
    margin-bottom: 50px;
  }

  .list-right {
    margin-left: 50px;

    .search-by-date {
      //padding: 6px 0;
      display: flex;
      height: 50px;
      align-items: center;
      justify-content: flex-end;
      font-size: 14px;
      margin-bottom: 25px;

      p {
        white-space: nowrap;
        position: relative;
        //right: -30px;
      }

      .el-date-editor.el-input {
        width: 160px;
      }

      :deep(.el-input__inner) {
        width: 160px;
        height: 30px;
        line-height: 30px;
      }

      :deep(.el-input__prefix),
      :deep(.el-input__suffix) {
        top: -4px;
      }

      ///deep/ .el-input__suffix{
      //  right: 50px;
      //}
    }

    :deep(.el-card__body) {
      padding-top: 0;
    }
  }
}

:deep(.el-tabs__nav-wrap::after) {
  content: none;
}

.box-card {
  width: 480px;
}
</style>

<style lang="less">
@hover_color: #3370ff;

.my-autocomplete {
  width: 46%;
  left: 50% !important;
  transform: translateX(-50%);
  li {
    line-height: normal;
    padding: 7px;

    .name {
      text-overflow: ellipsis;
      overflow: hidden;
    }

    .desc {
      font-size: 12px;
      color: #b4b4b4;
    }

    &.highlighted {
      background: #edf6ff !important;
    }

    .highlighted .addr {
      color: #ddd;
    }
  }

  a {
    color: rgba(0, 0, 0, 1);
    //transition: color .3s;
    display: block;
    text-overflow: ellipsis;
    overflow: hidden;
  }

  a:hover {
    color: @hover_color;
  }
}

.el-autocomplete-suggestion li:hover {
  background-color: #edf6ff !important;
}
</style>
