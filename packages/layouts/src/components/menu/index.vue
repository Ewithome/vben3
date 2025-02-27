<script lang="ts" setup>
import { ref, h, onMounted, unref, nextTick, computed } from 'vue'
import { createNamespace, mapTree } from '@vben/utils'
import {
  RouteLocationNormalizedLoaded,
  RouterLink,
  useRouter,
} from 'vue-router'
import { useI18n } from '@vben/locale'
import { REDIRECT_NAME } from '@vben/constants'
import { renderIcon } from '@vben/vbencomponents'
import { context } from '../../../bridge'
import type { RouteMeta } from 'vue-router'
import { Menu } from '@vben/types'
const { Logo, useAppInject, useAppConfig, useMenuSetting } = context
import { getMenus, listenerRouteChange, emitter } from '@vben/router'
import FooterTrigger from '../trigger/FooterTrigger.vue'

const { getIsMobile } = useAppInject()

const {
  menu,
  isMixSidebar,
  getCollapsedShowTitle,
  sidebar,
  isSidebar,
  isTopMenu,
} = useAppConfig()
const { getTopMenuAlign, getShowFooterTrigger } = useMenuSetting()
const showSidebarLogo = computed(() => {
  return unref(isSidebar) || unref(isMixSidebar)
})
const props = defineProps({
  mode: {
    type: String,
    default: () => 'vertical',
  },
  split: {
    type: Boolean,
    default: () => false,
  },
})
const { bem } = createNamespace('layout-menu')
const { t } = useI18n()
const { currentRoute } = useRouter()

const menuRef = ref(null)
const options = ref<Menu[]>([])
const menuList = ref<Menu[]>([])
const activeKey = ref()

const getMenuCollapsed = computed(() => {
  if (unref(isMixSidebar)) return true
  return unref(sidebar).collapsed
})

// 定位菜单选择 与 当前路由匹配
const showOption = () => {
  nextTick(() => {
    if (!menuRef.value) return
    menuRef.value.Ref.showOption()
  })
}

// TODO 静态路由 待实现
onMounted(async () => {
  const menus = await getMenus()
  menuList.value = mapTree(menus, {
    conversion: (menu) => routerToMenu(menu),
  })
  options.value = mapTree(menus, {
    conversion: (menu) => routerToMenu(menu),
  })

  if (props.split) {
    //监听菜单改变事件以接受子路由
    emitter.on('menuChange', (p) => {
      activeKey.value = p.name
      options.value = p.options
    })
  }
  handleMenuChange()
})

listenerRouteChange((route) => {
  if (route.name === REDIRECT_NAME) return
  const currentActiveMenu = route.meta?.currentActiveMenu as string
  handleMenuChange(route)
  if (currentActiveMenu) {
    activeKey.value = currentActiveMenu
  }

  showOption()
}, false)

async function handleMenuChange(route?: RouteLocationNormalizedLoaded) {
  const currentMenu = route || unref(currentRoute)
  activeKey.value = currentMenu.name
  //分割菜单 独有逻辑
  if (menu.value.split) {
    if (!props.split) {
      options.value.forEach((v) => {
        delete v.children
      })
    }
    //递归获取顶层key
    activeKey.value = findTopKey(
      flatten(menuList.value),
      currentMenu.name as string,
    )
    // console.log(menuList.value, currentMenu.name)
    //切换tab更新子路由
    emitter.emit('menuChange', {
      name: currentMenu.name,
      options: menuList.value.find((v) => v.key == activeKey.value)?.children,
    })
  }
  showOption()
}

//递归通过子菜单key获取菜单顶层key
function flatten(arr, prefix = '') {
  let result = {}
  arr.forEach((item) => {
    let key = `${prefix}|${item.key}`
    if (Array.isArray(item.children)) {
      result = { ...result, ...flatten(item.children, key) }
    } else {
      result[key] = item
    }
  })
  return result
}
//筛选获取顶层key
function findTopKey(flattened, key) {
  return Object.keys(flattened)
    .find((k) => k.endsWith(key))
    .split('|')
    .filter((v) => v)[0]
}

// 路由格式化
const routerToMenu = (item: RouteRecordItem & RouteMeta) => {
  const { name, children, meta, icon } = item
  const title = t(meta.title as string)
  return {
    label: () => {
      if (children) {
        return title
      }
      return h(
        RouterLink,
        {
          to: {
            name,
          },
        },
        { default: () => title },
      )
    },
    key: name,
    icon: renderIcon(icon),
  }
}
const clickMenu = (key) => {
  if (isTopMenu && menu.value.split && !props.split) {
    //通过emit将子路由传递出去
    emitter.emit('menuChange', {
      name: activeKey.value,
      options: menuList.value.find((v) => v.key == key)?.children,
    })
  }
}
</script>

<template>
  <div :class="bem()">
    <logo
      :class="bem('logo')"
      v-if="showSidebarLogo || getIsMobile"
      :showTitle="getCollapsedShowTitle"
    />

    <VbenScrollbar :class="bem('scrollbar')">
      <VbenMenu
        class="w-full"
        :style="{
          justifyContent:
            getTopMenuAlign === 'center' ? 'center' : `flex-${getTopMenuAlign}`,
        }"
        v-model:value="activeKey"
        :options="options"
        :collapsed="getMenuCollapsed"
        :collapsed-width="48"
        :collapsed-icon-size="22"
        :indent="18"
        :root-indent="18"
        ref="menuRef"
        :mode="props.mode"
        :accordion="menu.accordion"
        @update:value="clickMenu"
      />
    </VbenScrollbar>
    <FooterTrigger v-if="getShowFooterTrigger" />
  </div>
</template>
<style>
div:has(> div[class='layout-menu']) {
  flex: 1;
}
</style>

<style lang="less" scoped>
.layout-menu {
  display: flex;
  flex-direction: column;
  height: 100%;

  &__logo {
    flex-shrink: 0;
  }

  &__scrollbar {
    flex: 1;
    flex-basis: auto;
  }
}
</style>
