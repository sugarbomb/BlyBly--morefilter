<script setup lang="ts">
import { Icon } from '@iconify/vue'
import { useI18n } from 'vue-i18n'

import Empty from '~/components/Empty.vue'
import Loading from '~/components/Loading.vue'
import Tooltip from '~/components/Tooltip.vue'
import type { TopBarLiveMomentResult } from '~/models/moment/topBarLiveMoment'
import type { TopBarMomentResult } from '~/models/moment/topBarMoment'
import api from '~/utils/api'
import { getCSRF, scrollToTop } from '~/utils/main'

type MomentType = 'video' | 'live' | 'article'
interface MomentTab { type: MomentType, name: any }
interface MomentCard {
  type: MomentType
  title: string
  author: string
  authorFace: string
  authorJumpUrl?: string
  pubTime?: string
  cover: string
  link: string
  rid?: number
}

const { t } = useI18n()

const moments = reactive<MomentCard[]>([])
const addedWatchLaterList = reactive<number[]>([])
const momentTabs = computed((): MomentTab[] => {
  return [
    {
      type: 'video',
      name: t('topbar.moments_dropdown.tabs.videos'),
    },
    {
      type: 'live',
      name: t('topbar.moments_dropdown.tabs.live'),
    },
    {
      type: 'article',
      name: t('topbar.moments_dropdown.tabs.articles'),
    },
  ]
},
)
const selectedMomentTab = ref<MomentTab>(momentTabs.value[0])
const isLoading = ref<boolean>(false)
const noMoreContent = ref<boolean>(false) // when noMoreContent is true, the user can't scroll down to load more content
const livePage = ref<number>(1)
const momentsWrap = ref()
const momentUpdateBaseline = ref<string>('')
const momentOffset = ref<string>('')
const newMomentsCount = ref<number>(0)

watch(() => selectedMomentTab.value.type, (newVal, oldVal) => {
  if (newVal === oldVal)
    return

  if (momentsWrap.value)
    scrollToTop(momentsWrap.value)

  initData()
}, { immediate: true })

onMounted(() => {
  if (momentsWrap.value) {
    momentsWrap.value.addEventListener('scroll', () => {
      if (
        momentsWrap.value.clientHeight + momentsWrap.value.scrollTop
        >= momentsWrap.value.scrollHeight - 20
        && moments.length > 0
        && !isLoading.value
      ) {
        getData()
      }
    })
  }
})

function onClickTab(tab: MomentTab) {
  // Prevent changing tab when loading, cuz it will cause a bug
  if (isLoading.value || tab.type === selectedMomentTab.value.type)
    return

  selectedMomentTab.value = tab
  initData()
}

async function initData() {
  moments.length = 0
  momentUpdateBaseline.value = ''
  momentOffset.value = ''
  newMomentsCount.value = 0
  livePage.value = 1
  noMoreContent.value = false

  getData()
}

function getData() {
  if (selectedMomentTab.value.type !== 'live')
    getTopBarMoments()
  else
    getTopBarLiveMoments()
}

function checkIfHasNewMomentsThenUpdateMoments() {
  if (selectedMomentTab.value.type === 'live')
    return

  api.moment.getTopBarMoments({
    type: selectedMomentTab.value.type,
    update_baseline: momentUpdateBaseline.value || undefined,
  })
    .then((res: TopBarMomentResult) => {
      if (res.code === 0) {
        const { has_more, items, update_baseline, update_num } = res.data

        if (!has_more) {
          noMoreContent.value = true
          return
        }
        if (update_num === 0)
          return

        for (let i = update_num - 1; i >= 0; i--) {
          moments.unshift({
            type: selectedMomentTab.value.type,
            title: items[i].title,
            author: items[i].author.name,
            authorFace: items[i].author.face,
            authorJumpUrl: items[i].author.jump_url,
            pubTime: items[i].pub_time,
            cover: items[i].cover,
            link: items[i].jump_url,
            rid: items[i].rid,
          })
        }

        newMomentsCount.value = update_num
        momentUpdateBaseline.value = update_baseline
        // newMomentsCount.value = update_num
        // setLastOffsetID('video', offset)
      }
    })
    .finally(() => isLoading.value = false)
}

function getTopBarMoments() {
  if (isLoading.value)
    return
  if (noMoreContent.value)
    return

  isLoading.value = true
  api.moment.getTopBarMoments({
    type: selectedMomentTab.value.type,
    update_baseline: momentUpdateBaseline.value || undefined,
    offset: momentOffset.value || undefined,
  })
    .then((res: TopBarMomentResult) => {
      if (res.code === 0) {
        const { has_more, items, offset, update_baseline, update_num } = res.data

        if (!has_more) {
          noMoreContent.value = true
          return
        }

        newMomentsCount.value = update_num
        momentUpdateBaseline.value = update_baseline
        momentOffset.value = offset

        // set this lastest offset id, which will clear the new moment's marker point
        // after you watch these moments.

        // setLastOffsetID('video', offset)

        moments.push(
          ...items.map(item => ({
            type: selectedMomentTab.value.type,
            title: item.title,
            author: item.author.name,
            authorFace: item.author.face,
            authorJumpUrl: item.author.jump_url,
            pubTime: item.pub_time,
            cover: item.cover,
            link: item.jump_url,
            rid: item.rid,
          }),
          ),
        )
      }
    })
    .finally(() => isLoading.value = false)
}

function isNewMoment(index: number) {
  return index < newMomentsCount.value
}

function getTopBarLiveMoments() {
  if (isLoading.value)
    return
  if (noMoreContent.value)
    return

  isLoading.value = true
  const pageSize = 10
  api.moment.getTopBarLiveMoments({
    page: livePage.value,
    pagesize: pageSize,
  })
    .then((res: TopBarLiveMomentResult) => {
      if (res.code === 0) {
        const { list } = res.data

        // if the length of this list is less then the pageSize, it means that it have no more contents
        if (list.length < pageSize) {
          noMoreContent.value = true
        }

        // if the length of this list is equal to the pageSize, this means that it may have the next page.
        if (list.length === pageSize)
          livePage.value++

        moments.push(
          ...list.map(item => ({
            type: selectedMomentTab.value.type,
            title: item.title,
            author: item.uname,
            authorFace: item.face,
            cover: item.pic,
            link: item.link,
          }),
          ),
        )
      }
    })
    .finally(() => isLoading.value = false)
}

function toggleWatchLater(aid: number) {
  const isInWatchLater = addedWatchLaterList.includes(aid)

  if (!isInWatchLater) {
    api.watchlater.saveToWatchLater({
      aid,
      csrf: getCSRF(),
    })
      .then((res) => {
        if (res.code === 0)
          addedWatchLaterList.push(aid)
      })
  }
  else {
    api.watchlater.removeFromWatchLater({
      aid,
      csrf: getCSRF(),
    })
      .then((res) => {
        if (res.code === 0) {
          addedWatchLaterList.length = 0
          Object.assign(addedWatchLaterList, addedWatchLaterList.filter(item => item !== aid))
        }
      })
  }
}

defineExpose({
  checkIfHasNewMomentsThenUpdateMoments,
})
</script>

<template>
  <div
    ref="momentsWrap"
    style="backdrop-filter: var(--bew-filter-glass-1);" h="[calc(100vh-100px)]" max-h-500px
    important-overflow-y-overlay
    bg="$bew-elevated"
    w="380px"
    rounded="$bew-radius"
    pos="relative"
    shadow="[var(--bew-shadow-edge-glow-1),var(--bew-shadow-3)]"
    border="1 $bew-border-color"
  >
    <!-- top bar -->
    <header
      style="backdrop-filter: var(--bew-filter-glass-1);"
      flex="~ justify-between items-center"
      p="y-4 x-6"
      pos="sticky top-0 left-0"
      w="full"
      z="1"
      bg="$bew-elevated"
    >
      <div flex="~">
        <div
          v-for="tab in momentTabs"
          :key="tab.type"
          m="r-4"
          transition="all duration-300"
          class="tab"
          :class="tab.type === selectedMomentTab.type ? 'tab-selected' : ''"
          cursor="pointer"
          @click="onClickTab(tab)"
        >
          {{ tab.name }}
        </div>
      </div>
      <ALink
        href="https://t.bilibili.com/"
        type="topBar"
        flex="~ items-center"
      >
        <span text="sm">{{ $t('common.view_all') }}</span>
      </ALink>
    </header>

    <!-- moments wrapper -->
    <main rounded="$bew-radius" overflow-hidden p="x-4">
      <!-- loading -->
      <Loading
        v-if="isLoading && moments.length === 0"
        h="full"
        flex="~"
        items="center"
      />

      <!-- empty -->
      <Empty
        v-if="!isLoading && moments.length === 0"
        pos="absolute top-0 left-0"
        bg="$bew-content"
        z="0" w="full" h="full"
        flex="~ items-center"
        rounded="$bew-radius-half"
      />

      <!-- moments -->
      <TransitionGroup name="list">
        <ALink
          v-for="(moment, index) in moments"
          :key="index"
          :href="moment.link"
          type="topBar"
          flex="~ justify-between"
          m="b-2" p="2"
          rounded="$bew-radius"
          hover:bg="$bew-fill-2"
          duration-300
          pos="relative"
        >
          <!-- new moment dot -->
          <div
            v-if="isNewMoment(index)"
            rounded="full"
            w="8px"
            h="8px"
            m="-2"
            bg="$bew-theme-color"
            pos="absolute -top-12px -left-12px"
            style="box-shadow: 0 0 4px var(--bew-theme-color)"
          />
          <ALink
            :href="moment.authorJumpUrl"
            type="topBar"
            rounded="1/2"
            w="40px" h="40px" m="r-4"
            bg="$bew-skeleton"
            shrink-0
          >
            <img
              :src="`${moment.authorFace}@50w_50h_1c`"
              rounded="1/2"
              w="40px" h="40px"
            >
          </ALink>

          <div flex="~" justify="between" w="full">
            <div>
              <!-- <span v-if="selectedTab !== 1">{{ `${moment.name} ${t('topbar.moments_dropdown.uploaded')}` }}</span> -->
              <!-- <span v-else>{{ `${moment.name} ${t('topbar.moments_dropdown.now_streaming')}` }}</span> -->

              <ALink
                :href="moment.authorJumpUrl"
                type="topBar"
                font-bold
              >
                {{ moment.author }}
              </ALink>
              <div overflow-hidden text-ellipsis break-anywhere>
                {{ moment.title }}
              </div>
              <div
                text="$bew-text-2 sm"
                m="y-2"
              >
                <!-- publish time -->
                <div v-if="selectedMomentTab.type !== 'live'">
                  {{ moment.pubTime }}
                </div>

                <!-- Live -->
                <div
                  v-else
                  text="$bew-theme-color"
                  font="bold"
                  flex="~"
                  items="center"
                >
                  <div i-fluent:live-24-filled m="r-2" />
                  {{ $t('topbar.moments_dropdown.live_status') }}
                </div>
              </div>
            </div>
            <div
              class="group"
              flex="~ items-center justify-center" w="82px"
              h="46px" m="l-4" shrink-0
              rounded="$bew-radius-half"
              bg="$bew-skeleton"
            >
              <img
                :src="`${moment.cover}@128w_72h_1c`"
                w="82px" h="46px"
                rounded="$bew-radius-half"
              >
              <div
                opacity-0 group-hover:opacity-100
                pos="absolute" duration-300 bg="black opacity-60"
                rounded="$bew-radius-half" p-1
                z-1 color-white
                @click.prevent="toggleWatchLater(moment.rid || 0)"
              >
                <Tooltip v-if="!addedWatchLaterList.includes(moment.rid || 0)" :content="$t('common.save_to_watch_later')" placement="bottom" type="dark">
                  <div i-mingcute:carplay-line />
                </Tooltip>
                <Tooltip v-else :content="$t('common.added')" placement="bottom" type="dark">
                  <Icon icon="line-md:confirm" />
                </Tooltip>
              </div>
            </div>
          </div>
        </ALink>
      </TransitionGroup>

      <!-- loading -->
      <Transition name="fade">
        <Loading v-if="isLoading && moments.length !== 0" m="-t-4" />
      </Transition>
    </main>
  </div>
</template>

<style lang="scss" scoped>
.tab {
  --uno: "relative text-$bew-text-2";

  &::after {
    --uno: "absolute bottom-0 left-0 w-full h-12px bg-$bew-theme-color opacity-0 transform scale-x-0 -z-1";
    --uno: "transition-all duration-300";
    content: "";
  }
}

.tab-selected {
  --uno: "font-bold text-$bew-text-1";

  &::after {
    --uno: "scale-x-80 opacity-40";
  }
}
</style>
