<template>
  <span v-if="!state.isGuest">
    <tiny-popover
      trigger="hover"
      :open-delay="1000"
      popper-class="toolbar-right-popover"
      append-to-body
      :content="statusMessageMap[state.status]?.nextOptName"
    >
      <template #reference>
        <span class="icon">
          <svg-icon :name="iconName" @click="lockOrUnlock"></svg-icon>
        </span>
      </template>
    </tiny-popover>
  </span>
</template>

<script>
import { computed, reactive } from 'vue'
import { useCanvas, useLayout, useBlock, useNotify, useEditorInfo } from '@opentiny/tiny-engine-controller'
import { constants } from '@opentiny/tiny-engine-utils'
import { Popover } from '@opentiny/vue'
import { requestBlockPage } from './http'

export const api = {}

const { COMPONENT_NAME, PAGE_STATUS } = constants

const componentType = {
  block: '区块',
  page: '页面'
}

const statusMessageMap = {
  [PAGE_STATUS.Release]: {
    message: (type) => `${componentType[type]}解锁成功！`,
    currentOptName: '解锁',
    nextOptName: '锁定'
  },
  [PAGE_STATUS.Lock]: {
    message: (type) => `${componentType[type]}解锁成功！`,
    currentOptName: '解锁',
    nextOptName: '解锁'
  },
  [PAGE_STATUS.Occupy]: {
    message: (type) => `${componentType[type]}锁定成功`,
    currentOptName: '锁定',
    nextOptName: '解锁'
  }
}

export default {
  components: {
    TinyPopover: Popover
  },
  setup() {
    const { pageState } = useCanvas()
    const { layoutState } = useLayout()
    const { getCurrentBlock } = useBlock()

    const state = reactive({
      status: computed(() => layoutState.pageStatus.state),
      isGuest: computed(() => layoutState.pageStatus.state === PAGE_STATUS.Guest),
      type: '',
      disabled: false
    })

    const iconName = computed(() => (state.status === PAGE_STATUS.Occupy ? 'unlocked' : 'locked'))

    const lockPage = (id, type, newState) => {
      requestBlockPage(`id=${id}&state=${newState}&type=${type}`)
        .then((data) => {
          if (data) {
            useNotify({
              type: 'success',
              message: statusMessageMap[newState].message(type)
            })
            layoutState.pageStatus.state = newState
          }
        })
        .catch((error) => {
          useNotify({
            type: 'error',
            title: `页面${statusMessageMap[newState].currentOptName}失败`,
            message: JSON.stringify(error?.message || error)
          })
        })
        .finally(() => {
          state.disabled = false
        })
    }

    const hasPermission = () => {
      if (state.disabled) {
        return false
      }

      const isAdmin = useEditorInfo().isAdmin()

      if (state.status === PAGE_STATUS.Lock) {
        if (!isAdmin) {
          const pageInfo = layoutState.pageStatus?.data

          useNotify({
            type: 'warning',
            message: `当前页面被 ${pageInfo?.username || ''} ${pageInfo?.resetPasswordToken || ''} 锁定，请联系解锁`
          })
        }

        return isAdmin
      }

      if (![PAGE_STATUS.Occupy, PAGE_STATUS.Release].includes(state.status)) {
        return false
      }

      return true
    }

    const lockOrUnlock = () => {
      if (!hasPermission()) {
        return
      }

      const optType =
        state.status === PAGE_STATUS.Occupy || state.status === PAGE_STATUS.Lock
          ? PAGE_STATUS.Release
          : PAGE_STATUS.Occupy

      pageState.currentSchema = {}
      pageState.properties = null

      state.disabled = true

      if (pageState.pageSchema.componentName !== COMPONENT_NAME.Block) {
        lockPage(pageState.currentPage.id, 'page', optType)
      } else {
        lockPage(getCurrentBlock().id, 'block', optType)
      }
    }

    return {
      state,
      lockOrUnlock,
      PAGE_STATUS,
      statusMessageMap,
      iconName
    }
  }
}
</script>
