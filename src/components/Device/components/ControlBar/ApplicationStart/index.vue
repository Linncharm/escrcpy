<template>
  <el-dropdown
    :hide-on-click="false"
    :disabled="loading || floating"
    max-height="300px"
    @mouseenter="getAppData"
  >
    <slot :loading :trigger="handleTrigger" />

    <template #dropdown>
      <el-dropdown-menu>
        <el-dropdown-item
          v-for="item of options"
          :key="item.value"
          :command="item.value"
          :divided="item.divided"
          :icon="item.icon"
          @click="handleCommand(item)"
        >
          {{ $t(item.label) }}
        </el-dropdown-item>
      </el-dropdown-menu>
    </template>
  </el-dropdown>
</template>

<script>
import { openFloatControl } from '$/utils/device/index.js'

export default {
  props: {
    device: {
      type: Object,
      default: () => ({}),
    },
    floating: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      loading: false,
      appList: [],
    }
  },
  computed: {
    options() {
      const value = this.appList.map(item => ({
        ...item,
        label: item.name,
        value: item.packageName,
      }))

      value.unshift({
        label: this.$t('device.control.home'),
        value: '',
        icon: 'HomeFilled',
      })

      if (value[1]) {
        value[1].divided = true
      }

      return value
    },
  },
  created() {
    this.getAppData()
  },
  methods: {
    async getAppData() {
      const data = await window.scrcpy.getAppList(this.device.id)

      this.appList = data || []
    },
    handleTrigger() {
      if (!this.floating) {
        return false
      }
      const channel = 'startApp'

      window.electron.ipcRenderer.once(
        channel,
        (event, value, item) => {
          this.handleCommand(item)
        },
      )

      const options = toRaw(this.options)

      window.electron.ipcRenderer.send('open-system-menu', {
        channel,
        options,
      })
    },
    async handleCommand({ label, value }) {
      this.loading = true

      const title = `${this.$store.device.getLabel(this.device, 'synergy')}-${label}`

      const commands = this.$store.preference.scrcpyParameter(this.device.id, {
        excludes: ['--otg', '--mouse=aoa', '--keyboard=aoa'],
      })

      await window.scrcpy.startApp(this.device.id, { title, commands, packageName: value })
        .catch((e) => {
          console.error('mirror.commands', commands)
          console.error('mirror.error', e)
          if (e.message) {
            this.$message.warning(e.message)
          }
        })

      openFloatControl(toRaw(this.device))

      this.loading = false
    },
  },
}
</script>

<style></style>
