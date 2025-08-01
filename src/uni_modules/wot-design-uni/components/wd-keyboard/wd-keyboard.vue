<template>
  <wd-popup
    v-model="show"
    position="bottom"
    :z-index="zIndex"
    :safe-area-inset-bottom="safeAreaInsetBottom"
    :modal-style="modal ? '' : 'opacity: 0;'"
    :modal="hideOnClickOutside"
    :lockScroll="lockScroll"
    :root-portal="rootPortal"
    @click-modal="handleClose"
  >
    <view :class="`wd-keyboard ${customClass}`" :style="customStyle">
      <view class="wd-keyboard__header" v-if="showHeader">
        <slot name="title" v-if="showTitle">
          <text class="wd-keyboard__title">{{ title }}</text>
        </slot>
        <view class="wd-keyboard__close" hover-class="wd-keyboard__close--hover" v-if="showClose" @click="handleClose">
          <text>{{ closeText }}</text>
        </view>
      </view>
      <template v-if="mode !== 'car'">
        <view class="wd-keyboard__body">
          <view class="wd-keyboard__keys">
            <wd-key v-for="key in keys" :key="key.text" :text="key.text" :type="key.type" :wider="key.wider" @press="handlePress"></wd-key>
          </view>
          <view class="wd-keyboard__sidebar" v-if="mode === 'custom'">
            <wd-key v-if="showDeleteKey" large :text="deleteText" type="delete" @press="handlePress"></wd-key>
            <wd-key large :text="closeText" type="close" :loading="closeButtonLoading" @press="handlePress"></wd-key>
          </view>
        </view>
      </template>
      <template v-if="mode === 'car'">
        <view class="wd-keyboard-car__body">
          <view class="wd-keyboard-car__keys">
            <wd-key v-for="key in keys" :key="key.text" :text="key.text" :type="key.type" :wider="key.wider" @press="handlePress"></wd-key>
          </view>
        </view>
      </template>
    </view>
  </wd-popup>
</template>
<script lang="ts">
export default {
  name: 'wd-keyboard',
  options: {
    virtualHost: true,
    addGlobalClass: true,
    styleIsolation: 'shared'
  }
}
</script>

<script lang="ts" setup>
import { computed, ref, watch, useSlots } from 'vue'
import wdPopup from '../wd-popup/wd-popup.vue'
import WdKey from './key/index.vue'
import { keyboardProps, type Key } from './types'
import type { NumberKeyType } from './key/types'
import { CAR_KEYBOARD_AREAS, CAR_KEYBOARD_KEYS } from './constants'

const props = defineProps(keyboardProps)
const emit = defineEmits(['update:visible', 'input', 'close', 'delete', 'update:modelValue'])
const slots = useSlots()

const show = ref(props.visible)
watch(
  () => props.visible,
  (newValue) => {
    show.value = newValue
  }
)

const carKeyboardLang = ref('zh')
const keys = computed(() => (props.mode !== 'car' ? (props.mode === 'custom' ? genCustomKeys() : genDefaultKeys()) : genCarKeys()))

const showClose = computed(() => {
  return props.closeText && (props.mode === 'default' || props.mode === 'car')
})

const showTitle = computed(() => {
  return !!props.title || !!slots.title
})

const showHeader = computed(() => {
  return showTitle.value || showClose.value
})

/**
 * 随机打乱数组的顺序
 * @param arr 要打乱顺序的数组
 * @returns 打乱顺序后的数组
 */
function shuffleArray<T>(arr: T[]): T[] {
  const newArr = [...arr]
  for (let i = newArr.length - 1; i > 0; i--) {
    // 生成一个随机索引 j，范围是 [0, i]
    const j = Math.floor(Math.random() * (i + 1))

    // 交换索引 i 和 j 处的元素
    ;[newArr[i], newArr[j]] = [newArr[j], newArr[i]]
  }
  return newArr
}
function genBasicKeys(): Key[] {
  const keys = Array.from({ length: 9 }, (_, i) => ({ text: i + 1 }))

  // 如果需要随机顺序，则调用 shuffleArray 方法打乱数组顺序
  return props.randomKeyOrder ? shuffleArray(keys) : keys
}

function genDefaultKeys(): Key[] {
  return [
    ...genBasicKeys(),
    { text: props.extraKey as string, type: 'extra' },
    { text: 0 },
    {
      // 根据条件是否显示删除键的文本和类型
      text: props.showDeleteKey ? props.deleteText : '',
      type: props.showDeleteKey ? 'delete' : ''
    }
  ]
}

function genCustomKeys(): Key[] {
  const keys = genBasicKeys()
  const extraKeys = Array.isArray(props.extraKey) ? props.extraKey : [props.extraKey]
  if (extraKeys.length === 1) {
    // 如果只有一个额外按键，则添加一个宽度较大的数字0和一个额外按键
    keys.push({ text: 0, wider: true }, { text: extraKeys[0], type: 'extra' })
  } else if (extraKeys.length === 2) {
    // 如果有两个额外按键，则添加两个额外按键和一个数字0
    keys.push({ text: extraKeys[0], type: 'extra' }, { text: 0 }, { text: extraKeys[1], type: 'extra' })
  }

  return keys
}

// 生成车牌键盘
function genCarKeys(): Array<Key> {
  const [keys, remainKeys] = splitCarKeys()
  return [
    ...keys,
    { text: carKeyboardLang.value === 'zh' ? 'ABC' : '返回', type: 'extra', wider: true },
    ...remainKeys,
    { text: props.deleteText, type: 'delete', wider: true }
  ]
}

function splitCarKeys(): Array<Array<Key>> {
  const keys = carKeyboardLang.value === 'zh' ? CAR_KEYBOARD_AREAS.map((key) => ({ text: key })) : CAR_KEYBOARD_KEYS.map((key) => ({ text: key }))
  return [keys.slice(0, 30), keys.slice(30)]
}

const handleClose = () => {
  emit('close')
  emit('update:visible', false)
}

const handlePress = (text: string, type: NumberKeyType) => {
  if (type === 'extra') {
    if (text === '') {
      return handleClose()
    } else if (text === 'ABC' || text === '返回') {
      carKeyboardLang.value = carKeyboardLang.value === 'zh' ? 'en' : 'zh'
      return
    }
  }

  const value = props.modelValue
  if (type === 'delete') {
    emit('delete')
    emit('update:modelValue', value.slice(0, value.length - 1))
  } else if (type === 'close') {
    handleClose()
  } else if (value.length < +props.maxlength) {
    emit('input', text)
    emit('update:modelValue', value + text)
  }
}
</script>

<style lang="scss">
@import './index.scss';
</style>
