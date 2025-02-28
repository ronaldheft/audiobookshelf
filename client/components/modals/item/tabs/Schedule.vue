<template>
  <div class="w-full h-full relative">
    <div id="scheduleWrapper" class="w-full overflow-y-auto px-2 py-4 md:px-6 md:py-6">
      <template v-if="!feedUrl">
        <widgets-alert type="warning" class="text-base mb-4">No RSS feed URL is set for this podcast</widgets-alert>
      </template>
      <template v-if="feedUrl || autoDownloadEpisodes">
        <div class="flex items-center justify-between mb-4">
          <p class="text-base md:text-xl font-semibold">Schedule Automatic Episode Downloads</p>
          <ui-checkbox v-model="enableAutoDownloadEpisodes" label="Enable" medium checkbox-bg="bg" label-class="pl-2 text-base md:text-lg" />
        </div>

        <div v-if="enableAutoDownloadEpisodes" class="flex items-center py-2">
          <ui-text-input ref="maxEpisodesInput" type="number" v-model="newMaxEpisodesToKeep" no-spinner :padding-x="1" text-center class="w-10 text-base" @change="updatedMaxEpisodesToKeep" />
          <ui-tooltip text="Value of 0 sets no max limit. After a new episode is auto-downloaded this will delete the oldest episode if you have more than X episodes. <br>This will only delete 1 episode per new download.">
            <p class="pl-4 text-base">
              Max episodes to keep
              <span class="material-icons icon-text text-sm">info_outlined</span>
            </p>
          </ui-tooltip>
        </div>

        <widgets-cron-expression-builder ref="cronExpressionBuilder" v-if="enableAutoDownloadEpisodes" v-model="cronExpression" />
      </template>
    </div>

    <div v-if="feedUrl || autoDownloadEpisodes" class="absolute bottom-0 left-0 w-full py-2 md:py-4 bg-bg border-t border-white border-opacity-5">
      <div class="flex items-center px-2 md:px-4">
        <div class="flex-grow" />
        <ui-btn @click="save" :disabled="!isUpdated" :color="isUpdated ? 'success' : 'primary'" class="mx-2">{{ isUpdated ? 'Save' : 'No update necessary' }}</ui-btn>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  props: {
    processing: Boolean,
    libraryItem: {
      type: Object,
      default: () => {}
    }
  },
  data() {
    return {
      enableAutoDownloadEpisodes: false,
      cronExpression: null,
      newMaxEpisodesToKeep: 0
    }
  },
  computed: {
    isProcessing: {
      get() {
        return this.processing
      },
      set(val) {
        this.$emit('update:processing', val)
      }
    },
    userIsAdminOrUp() {
      return this.$store.getters['user/getIsAdminOrUp']
    },
    media() {
      return this.libraryItem ? this.libraryItem.media || {} : {}
    },
    mediaMetadata() {
      return this.media.metadata || {}
    },
    libraryItemId() {
      return this.libraryItem ? this.libraryItem.id : null
    },
    feedUrl() {
      return this.mediaMetadata.feedUrl
    },
    autoDownloadEpisodes() {
      return !!this.media.autoDownloadEpisodes
    },
    autoDownloadSchedule() {
      return this.media.autoDownloadSchedule
    },
    maxEpisodesToKeep() {
      return this.media.maxEpisodesToKeep
    },
    isUpdated() {
      return this.autoDownloadSchedule !== this.cronExpression || this.autoDownloadEpisodes !== this.enableAutoDownloadEpisodes || this.maxEpisodesToKeep !== Number(this.newMaxEpisodesToKeep)
    }
  },
  methods: {
    updatedMaxEpisodesToKeep() {
      if (isNaN(this.newMaxEpisodesToKeep) || this.newMaxEpisodesToKeep < 0) {
        this.newMaxEpisodesToKeep = 0
      } else {
        this.newMaxEpisodesToKeep = Number(this.newMaxEpisodesToKeep)
      }
    },
    save() {
      // If custom expression input is focused then unfocus it instead of submitting
      if (this.$refs.cronExpressionBuilder && this.$refs.cronExpressionBuilder.checkBlurExpressionInput) {
        if (this.$refs.cronExpressionBuilder.checkBlurExpressionInput()) {
          return
        }
      }
      if (this.$refs.maxEpisodesInput && this.$refs.maxEpisodesInput.isFocused) {
        this.$refs.maxEpisodesInput.blur()
        return
      }

      const updatePayload = {
        autoDownloadEpisodes: this.enableAutoDownloadEpisodes
      }
      if (this.enableAutoDownloadEpisodes) {
        updatePayload.autoDownloadSchedule = this.cronExpression
      }
      if (this.newMaxEpisodesToKeep !== this.maxEpisodesToKeep) {
        updatePayload.maxEpisodesToKeep = this.newMaxEpisodesToKeep
      }

      this.updateDetails(updatePayload)
    },
    async updateDetails(updatePayload) {
      this.isProcessing = true
      var updateResult = await this.$axios.$patch(`/api/items/${this.libraryItemId}/media`, updatePayload).catch((error) => {
        console.error('Failed to update', error)
        return false
      })
      this.isProcessing = false
      if (updateResult) {
        if (updateResult.updated) {
          this.$toast.success('Item details updated')
          return true
        } else {
          this.$toast.info('No updates were necessary')
        }
      }
      return false
    },
    init() {
      this.enableAutoDownloadEpisodes = this.autoDownloadEpisodes
      this.cronExpression = this.autoDownloadSchedule
      this.newMaxEpisodesToKeep = this.maxEpisodesToKeep
    }
  },
  mounted() {
    this.init()
  }
}
</script>

<style scoped>
#scheduleWrapper {
  height: calc(100% - 80px);
  max-height: calc(100% - 80px);
}
</style>