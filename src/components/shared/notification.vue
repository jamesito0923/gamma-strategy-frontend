<template>
  <div
    class="notification"
    :class="{'visible': visible}"
  >
    <div class="icon">
      <cv-inline-loading
        v-if="type==='loading'"
        loading-text=""
        state="loading"
      />
      <CheckmarkFilled20
        v-if="type==='success'"
        class="checkmark"
      />
    </div>
    <div class="content">
      <div class="header">
        {{ header }}
      </div>
      <div class="sub-header">
        {{ subHeader }}
      </div>
    </div>
    <div
      class="close"
      @click="close()"
    >
      <Close20 />
    </div>
  </div>
</template>

<script>
import Close20 from '@carbon/icons-vue/es/close/20'
import CheckmarkFilled20 from '@carbon/icons-vue/es/checkmark--filled/20'
export default {
  name: 'Notification',
  components: {
		Close20,
		CheckmarkFilled20
  },
	props: {
		visible: {
			type: Boolean,
			default: false
		},
		type: {
			type: String,
			default: 'loading',
		},
		header: {
			type: String,
			default: 'Loading notification',
		},
		subHeader: {
			type: String,
			default: 'Roius abnta mod tempor'
		}
	},
  computed: {
  },
  methods: {
		close() {
			this.visible = false
		}
  }
}
</script>
<style lang="scss" scoped>
@import "../../assets/css/main-theme.scss";
</style>
