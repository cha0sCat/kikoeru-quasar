<template>
  <q-page padding>
    <div class="fit row wrap justify-between items-start q-px-sm">
      <div class="col-lg-3 col-sm-12 col-xs-12">
          <q-btn-toggle
            v-model="mode"
            spread
            no-caps
            rounded
            toggle-color="primary"
            color="white"
            class="text-bold"
            text-color="black"
            :options="[
              {label: '我的评价', value: 'review'},
              {label: '我的进度', value: 'progress'},
              {label: '分类整理', value: 'folder'}
            ]"
          />
      </div>
      <div class="col-auto gt-sm">
        <q-select dense rounded outlined v-model="sortBy" :options="sortOptions" bg-color="white" />
      </div>
    </div>
    <div class="q-pt-md q-px-sm">
      <q-btn-toggle
        v-if="mode === 'progress'"
        v-model="progressFilter"
        toggle-color="primary"
        color="white"
        text-color="black"
        rounded
        :options="[
          {label: '想听', value: 'marked'},
          {label: '在听', value: 'listening'},
          {label: '听过', value: 'listened'},
          {label: '搁置', value: 'postponed'}
        ]"
      />
    </div>

    <div class="q-pt-md">
      <div class="q-px-sm q-py-md">
        <q-infinite-scroll @load="onLoad" :offset="500" :disable="stopLoad" v-if="mode !=='folder'">
          <q-list bordered separator class="shadow-2" v-if="works.length">
             <FavListItem v-for="work in works" :key="work.id" :workid="work.id" :metadata="work" @reset="reset()" :mode="mode"></FavListItem> 
          </q-list>
          <template v-slot:loading>
            <div class="row justify-center q-my-md">
              <q-spinner-dots color="primary" size="40px" />
            </div>
          </template>
        </q-infinite-scroll>

        <div v-else class="row justify-center text-grey">尚未实现，敬请期待</div>
      </div>
    </div>
  </q-page>
</template>

<script>
import FavListItem from 'components/FavListItem'

export default {
  name: 'Favourites',

  components: {
    FavListItem
  },

  data() {
    return {
      mode: 'review',
      progressFilter: 'marked',
      works: [],
      stopLoad: false,
      pagination: {},
      sortBy: {
          label: '按照标记时间排序',
          order: 'updated_at',
          sort: 'desc'
        },
      sortOptions: [
        {
          label: '按照标记时间排序',
          order: 'updated_at',
          sort: 'desc'
        },
        {
          label: '按照评价排序',
          order: 'userRating',
          sort: 'desc'
        },
        {
          label: '按照新作优先顺序',
          order: 'release',
          sort: 'desc'
        },
        {
          label: '按照旧作优先顺序',
          order: 'release',
          sort: 'asc'
        },
        {
          label: '按照评论多到少的顺序',
          order: 'review_count',
          sort: 'desc'
        },
        {
          label: '按照售出数量多到少的顺序',
          order: 'dl_count',
          sort: 'desc'
        },
        {
          label: '按照全年龄新作优先的顺序',
          order: 'nsfw',
          sort: 'asc'
        },
        {
          label: '按照18禁新作优先的顺序',
          order: 'nsfw',
          sort: 'desc'
        }
      ]
    }
  },

  mounted() {
    if (localStorage.sortByFavourites) {
      try {
        this.sortBy = JSON.parse(localStorage.sortByFavourites);
      } catch {
        localStorage.removeItem('sortByFavourites');
      }
    }
  },

  watch: {
    sortBy(newSortOptionSetting) {
      localStorage.sortByFavourites = JSON.stringify(newSortOptionSetting);
      this.reset();
    },
    mode() {
      this.reset();
    },
    progressFilter() {
      this.reset();
    }
  },

  methods: {
    onLoad (index, done) {
      this.requestWorksQueue()
        .then(() => done())
    },

    reset () {
      this.stopLoad = true
      this.pagination = {}
      this.requestWorksQueue()
        .then(() => {
          this.stopLoad = false
        })
    },

    requestWorksQueue () {
      const params = {
        order: this.sortBy.order,
        sort: this.sortBy.sort,
        page: this.pagination.currentPage + 1 || 1
      }

      if (this.mode === 'progress') {
        params.filter = this.progressFilter;
      }

      return this.$axios.get('/api/favourites', { params })
        .then((response) => {                  
          const works = response.data.works
          this.works = (params.page === 1) ? works.concat() : this.works.concat(works)
          this.pagination = response.data.pagination

          if (this.works.length >= this.pagination.totalCount) {
            this.stopLoad = true
          }
        })
        .catch((error) => {
          if (error.response) {
            // 请求已发出，但服务器响应的状态码不在 2xx 范围内
            if (error.response.status !== 401) {
              this.showErrNotif(error.response.data.error || `${error.response.status} ${error.response.statusText}`)
            }
          } else {
            this.showErrNotif(error.message || error)
          }
          this.stopLoad = true
        })
    },

    showErrNotif (message) {
      this.$q.notify({
        message,
        color: 'negative',
        icon: 'bug_report'
      })
    },
  }
}
</script>
