<template>
  <div
    ref="toolbarRef"
    :class="[
      'editor-toolbar',
      className,
      { 'editor-toolbar-mobile': isMobile && !popup, 'editor-toolbar-popup': popup },
    ]"
    :style="isMobile ? { top: `${mobileView.top}px` } : {}"
    data-element="ui"
    @mousedown="triggerMouseDown"
    @mouseover="triggerMouseOver"
    @mousemove="triggerMouseMove"
    @contextmenu="triggerContextMenu"
  >
    <div class="editor-toolbar-content">
      <template v-if="engine">
        <AmGroup
          v-for="(group, index) in groups"
          :key="index"
          :engine="engine"
          :popup="popup"
          :items="group.items"
          :content="group.content"
          :icon="group.icon"
        />
      </template>
    </div>
  </div>
</template>
<script setup lang="ts" name="AmToolbar">
import { onMounted, onUnmounted, ref, reactive, toRefs } from 'vue'
import { merge, omit } from 'lodash'
import { isMobile } from '@aomao/engine'
import {
  CollapseItemProps,
  ToolbarDropdownProps,
  GroupDataProps,
  ToolbarCollapseGroupProps,
} from '../types'
import AmGroup from './group.vue'
import locales from '../locales'
import { getToolbarDefaultConfig } from '../config'
import { IPropItems, IPropEngine, IPropToolbarItem } from './IPropTypes'

interface IProp {
  engine: IPropEngine
  items: IPropItems[]
  className?: string
  popup?: boolean
  onLoad?: Function
}

const props = defineProps<IProp>()
const { className, popup, engine } = toRefs(props)

const groups = ref<GroupDataProps[]>([])

// TODO: support object and string type config
const update = () => {
  if (isMobile) calcMobileView()
  const dataGroup: GroupDataProps = { icon: '', items: [], content: '' }
  const data: GroupDataProps[] = []
  const defaultConfig = getToolbarDefaultConfig(engine.value)

  if (Array.isArray(props.items)) {
    props.items.forEach((group) => {
      if (!Array.isArray(group)) {
        dataGroup.icon = group.icon
        dataGroup.content = group.content
        group = [...group.items]
      }
      if (Array.isArray(group)) {
        group.forEach((item) => {
          let customItem: IPropToolbarItem | undefined
          if (typeof item === 'string') {
            const defaultItem = defaultConfig.find((config) =>
              item === 'collapse'
                ? config.type === item
                : config.type !== 'collapse' && config.name === item
            )
            if (defaultItem) customItem = defaultItem
          } else {
            const defaultItem = defaultConfig.find((config) =>
              item.type === 'collapse'
                ? config.type === item.type
                : config.type !== 'collapse' && config.name === item.name
            )
            // parse collapse item when it's string
            if (item.type === 'collapse') {
              const customCollapse: ToolbarCollapseGroupProps = {
                ...merge(omit({ ...defaultItem }, 'groups'), omit({ ...item }, 'groups')),
                groups: [],
              } as ToolbarCollapseGroupProps
              item.groups.forEach((group) => {
                const items: Array<Omit<CollapseItemProps, 'engine'>> = []
                group.items.forEach((cItem) => {
                  let targetItem = undefined
                  ;(defaultItem as ToolbarCollapseGroupProps).groups.some((g) =>
                    g.items.some((i) => {
                      const isEqual =
                        i.name === (typeof cItem === 'string' ? cItem : cItem.name)
                      if (isEqual) {
                        targetItem = { ...i, ...(typeof cItem === 'string' ? {} : cItem) }
                      }
                      return isEqual
                    })
                  )
                  if (targetItem) items.push(targetItem)
                  else if (typeof cItem === 'object') items.push(cItem)
                })
                if (items.length > 0) {
                  customCollapse.groups.push({ ...omit(group, 'items'), items })
                }
              })
              customItem = customCollapse.groups.length > 0 ? customCollapse : undefined
            } else if (item.type === 'dropdown') {
              customItem = defaultItem
                ? merge(defaultItem, omit({ ...item }, 'type', 'items'))
                : { ...item }
              ;(customItem as ToolbarDropdownProps).items = item.items
            } else {
              customItem = defaultItem
                ? merge(defaultItem, omit({ ...item }, 'type'))
                : { ...item }
            }
          }
          if (customItem && props.engine) {
            if (customItem.type === 'button') {
              if (customItem.onActive) customItem.active = customItem.onActive()
              else if (props.engine.command.queryEnabled(customItem.name))
                customItem.active = props.engine.command.queryState(customItem.name)
            } else if (customItem.type === 'dropdown') {
              if (customItem.onActive)
                customItem.values = customItem.onActive(customItem.items)
              else customItem.values = props.engine.command.queryState(customItem.name)
            }
            if (customItem.type !== 'collapse')
              customItem.disabled = customItem.onDisabled
                ? customItem.onDisabled()
                : !props.engine.command.queryEnabled(customItem.name)
            else {
              customItem.groups.forEach((group) =>
                group.items.forEach((item) => {
                  item.disabled = item.onDisabled
                    ? item.onDisabled()
                    : !props.engine?.command.queryEnabled(item.name)
                })
              )
              customItem.disabled = !customItem.groups.some((g) =>
                g.items.some((item) => !item.disabled)
              )
            }
            dataGroup.items.push(customItem as IPropToolbarItem)
          }
        })
      }
    })
  }

  if (dataGroup.items.length > 0) {
    data.push(dataGroup)
    groups.value = [...data]
  }
}

// mobile
const toolbarRef = ref<HTMLDivElement | null>(null)
const calcTimeoutRef = ref<NodeJS.Timeout | null>(null)
const mobileView = reactive({ top: 0 })

// track mobile view change
const calcMobileView = () => {
  if (!props.engine?.isFocus() || props.engine?.readonly) return

  if (calcTimeoutRef.value) clearTimeout(calcTimeoutRef.value as NodeJS.Timeout)
  calcTimeoutRef.value = setTimeout(() => {
    const rect = toolbarRef.value?.getBoundingClientRect()
    const height = rect?.height || 0
    mobileView.top =
      global.Math.max(document.body.scrollTop, document.documentElement.scrollTop) +
      (window.visualViewport.height || 0) -
      height
  }, 100)
}

let scrollTimer: NodeJS.Timeout

const hideMobileToolbar = () => {
  mobileView.top = -120
  clearTimeout(scrollTimer)
  scrollTimer = setTimeout(() => {
    calcMobileView()
  }, 200)
}

const handleReadonly = () => {
  if (props.engine?.readonly) {
    hideMobileToolbar()
  } else {
    calcMobileView()
  }
}

let updateTimer: NodeJS.Timeout

const updateByTimeout = () => {
  clearTimeout(updateTimer)
  updateTimer = setTimeout(() => {
    update()
  }, 100)
}

onMounted(() => {
  if (!props.engine) return
  props.engine.language.add(locales)
  props.engine.on('select', updateByTimeout)
  props.engine.on('change', updateByTimeout)
  props.engine.on('blur', updateByTimeout)
  props.engine.on('focus', updateByTimeout)
  if (isMobile) {
    props.engine.on('readonly', handleReadonly)
    props.engine.on('blur', hideMobileToolbar)
    document.addEventListener('scroll', hideMobileToolbar)
    visualViewport.addEventListener('resize', calcMobileView)
    visualViewport.addEventListener('scroll', calcMobileView)
  } else {
    props.engine.on('readonly', updateByTimeout)
  }
  updateByTimeout()
})

function preventDefault(event: MouseEvent) {
  event.preventDefault()
}
function triggerMouseDown(event: MouseEvent) {
  event.preventDefault()
}
function triggerMouseOver(event: MouseEvent) {
  preventDefault(event)
}
function triggerMouseMove(event: MouseEvent) {
  preventDefault(event)
}
function triggerContextMenu(event: MouseEvent) {
  preventDefault(event)
}

onUnmounted(() => {
  if (!props.engine) return
  props.engine.off('select', updateByTimeout)
  props.engine.off('change', updateByTimeout)
  props.engine.off('readonly', updateByTimeout)
  props.engine.off('blur', updateByTimeout)
  props.engine.off('focus', updateByTimeout)
  if (isMobile) {
    props.engine.off('readonly', handleReadonly)
    props.engine.off('blur', hideMobileToolbar)
    document.removeEventListener('scroll', hideMobileToolbar)
    visualViewport.removeEventListener('resize', calcMobileView)
    visualViewport.removeEventListener('scroll', calcMobileView)
  } else {
    props.engine.off('readonly', updateByTimeout)
  }
})
</script>
<style scoped>
.ant-tooltip .toolbar-tooltip-title {
  font-size: 12px;
  text-align: center;
}

.ant-tooltip .toolbar-tooltip-hotkey {
  font-size: 12px;
  color: rgba(255, 255, 255, 0.85);
  text-align: center;
}

.editor-toolbar {
  position: relative;
  width: 100%;
  padding: 0;
  z-index: 200;
  border-top: 1px solid rgba(0, 0, 0, 0.05);
  border-bottom: 1px solid rgba(0, 0, 0, 0.05);
  user-select: none;
}

.editor-toolbar .editor-toolbar-content {
  position: relative;
  flex-direction: row;
  background: transparent;
  text-align: center;
  width: 100%;
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
}

.editor-toolbar.editor-toolbar-mobile,
.editor-toolbar.editor-toolbar-popover {
  position: absolute;
  left: 0;
  box-shadow: none;
}

.editor-toolbar.editor-toolbar-popup {
  position: initial;
  box-shadow: none;
  top: 0;
  left: 0;
  border: 0 none;
}

.editor-toolbar-mobile .editor-toolbar-content {
  text-align: left;
  padding: 0 12px;
}

.editor-toolbar-mobile .editor-toolbar-group,
.editor-toolbar-popup .editor-toolbar-group {
  border: 0 none;
  padding: 0;
}

.editor-toolbar-popup .editor-toolbar-content {
  text-align: center;
  padding: 0;
}

.editor-toolbar-popover .editor-toolbar {
  position: relative;
  box-shadow: none;
  border: 0 none;
  left: 0;
  top: 0;
  display: flex;
}

.editor-toolbar-popover {
  border-radius: 3px;
  background: transparent;
}

.editor-toolbar-popover .ant-popover-inner {
  border-radius: 3px;
}

.editor-toolbar-popover .ant-popover-inner-content {
  padding: 2px;
}

.am-engine-mobile {
  margin-bottom: 40px;
}
</style>
