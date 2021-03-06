<template>
  <div class="queue">
    <div ref="container" class="queue-container">
      <container
        lock-axis="y"
        drag-handle-selector=".handle"
        :get-child-payload="getChildPayload"
        @drop="handleDrop"
      >
        <draggable v-for="anime in animes" :key="anime && anime.id">
          <queue-item
            v-if="anime != null && getItem(anime.id) != null"
            :key="anime.id"
            :anime="anime"
            :item="getItem(anime.id)"
            :set-provider="setProvider(anime.id)"
          />
        </draggable>
      </container>

      <transition name="fade">
        <div v-if="queue.length < 1" class="empty-message">
          Seems your queue is empty! <br />You can import shows from your list
          or add some by searching! <br />

          <c-button
            content="Import Watching from List"
            :icon="currentSvg"
            :click="importWatching"
          />
        </div>
      </transition>
    </div>

    <div
      class="sidebar"
      :class="{ small: isPlayerOpen, external: isExternalPlayer }"
    >
      <span class="fill" />

      <a
        href="https://github.com/BeeeQueue/yuna/blob/master/docs/local-files.md"
        class="local-files-help"
      >
        Want to play local files?
      </a>

      <c-button
        content="Import Watching from List"
        :icon="currentSvg"
        :click="importWatching"
      />

      <c-button
        content="Import Random from Planning"
        :icon="planningSvg"
        :click="importPlanning"
      />

      <c-button
        content="Import Random from Paused"
        :icon="pausedSvg"
        :click="importPaused"
      />

      <c-button
        content="Import Exported Queue"
        :click="importQueueFromBackup"
      />

      <c-button content="Export Queue" :click="exportQueue" />

      <c-button
        type="danger"
        confirm
        :icon="clearListSvg"
        content="Clear queue"
        :click="clearQueue"
      />
    </div>

    <manual-search-modal />
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator'
import { Container, Draggable } from 'vue-smooth-dnd'
import { remote, shell } from 'electron'
import { activeWindow, api } from 'electron-util'
import { existsSync, mkdirSync, readFileSync, writeFileSync } from 'fs'
import { resolve } from 'path'
import indexBy from 'lodash.keyby'
import { mdiClockOutline, mdiPause, mdiPlay, mdiPlaylistRemove } from '@mdi/js'

import CButton from '@/common/components/button.vue'
import QueueItem from './components/queue-item.vue'
import ManualSearchModal from './modals/manual-search/manual-search-modal.vue'

import {
  IMPORT_EXTERNAL_LINKS_QUERY,
  IMPORT_QUERY,
  QUEUE_QUERY,
} from './graphql/documents'
import {
  ImportExternalLinksExternalLinks,
  ImportExternalLinksQuery,
  ImportExternalLinksQueryVariables,
  ImportQuery,
  ImportQueryVariables,
  MediaListStatus,
  Provider,
  QueueAnime,
  QueueQuery,
  QueueVariables,
} from '@/graphql/types'

import { Query } from '@/decorators'
import { getPlayerData, sendErrorToast, sendToast } from '@/state/app'
import { getAnilistUsername } from '@/state/auth'
import {
  addToQueue,
  getQueue,
  setQueue,
  setQueueItemProvider,
  toggleQueueItemOpen,
} from '@/state/user'
import { QueueItem as IQueueItem } from '@/lib/user'
import { isNil, isNotNil, pick, prop, propEq, sortNumber } from '@/utils'

@Component({
  components: {
    ManualSearchModal,
    QueueItem,
    CButton,
    Container,
    Draggable,
  },
})
export default class Queue extends Vue {
  private defaultBackupPath = resolve(api.app.getPath('userData'), 'backups')
  private jsonFilter = { extensions: ['json'], name: '*' }

  @Query<Queue, QueueQuery, QueueVariables>({
    query: QUEUE_QUERY,
    variables() {
      return {
        ids: this.queue.map(prop<any, any>('id')).sort(sortNumber),
      }
    },
    update(data) {
      const items = indexBy(data.queue?.anime?.filter(isNotNil), anime =>
        anime.id.toString(),
      )

      return this.queue.map(item => items[item.id])
    },
  })
  public animes!: QueueAnime[]

  public $refs!: {
    container: HTMLDivElement
  }

  public currentSvg = mdiPlay
  public planningSvg = mdiClockOutline
  public pausedSvg = mdiPause
  public clearListSvg = mdiPlaylistRemove

  public get isPlayerOpen() {
    return !!getPlayerData(this.$store)
  }

  public get isExternalPlayer() {
    return getPlayerData(this.$store)?.provider === Provider.Local
  }

  public get queue() {
    return getQueue(this.$store)
  }

  public set queue(value: IQueueItem[]) {
    setQueue(this.$store, value)
  }

  public getItem(id: number) {
    return this.queue.find(propEq('id', id)) as IQueueItem
  }

  public handleDrop({ removedIndex, addedIndex, payload }: any) {
    const newQueue = [...this.queue]

    newQueue.splice(removedIndex, 1)
    newQueue.splice(addedIndex, 0, payload)

    setQueue(this.$store, newQueue)
    this.$apollo.queries.animes.refresh()
  }

  public getChildPayload(index: number) {
    return this.queue[index]
  }

  public setProvider(id: number) {
    return (provider: Provider) => {
      setQueueItemProvider(this.$store, { id, provider })

      if (!this.getItem(id).open) {
        toggleQueueItemOpen(this.$store, id)
      }
    }
  }

  private async getExternalLinks(
    mediaId: number,
  ): Promise<ImportExternalLinksExternalLinks[]> {
    const variables: ImportExternalLinksQueryVariables = {
      mediaId,
    }

    const result = await this.$apollo.query<ImportExternalLinksQuery>({
      query: IMPORT_EXTERNAL_LINKS_QUERY,
      variables,
    })

    return (result.data.Media?.externalLinks ?? []).filter(isNotNil)
  }

  public async importFrom(
    statuses: MediaListStatus | MediaListStatus[],
    random: boolean = false,
  ) {
    if (!Array.isArray(statuses)) {
      statuses = [statuses]
    }

    const queueBefore = [...this.queue]

    const variables: ImportQueryVariables = {
      status: statuses[0],
      extraStatus: statuses[1],
      useExtraStatus: !isNil(statuses[1]),
    }
    const { data, errors } = await this.$apollo.query<ImportQuery>({
      query: IMPORT_QUERY,
      variables,
      fetchPolicy: 'network-only',
    })

    const entries = [...data.ListEntries, ...(data.ExtraListEntries || [])]

    if (entries.length < 1) {
      return sendErrorToast(
        this.$store,
        "Couldn't find any shows in that state!",
      )
    }

    if (errors || entries.length < 1) {
      return sendErrorToast(
        this.$store,
        "Couldn't find any shows in that state!",
      )
    }

    if (random) {
      const entriesNotInQueue = entries.filter(({ id }) => {
        return !this.queue.some(item => item.id === id)
      })

      if (entriesNotInQueue.length > 0) {
        const randomIdx = Math.floor(Math.random() * entriesNotInQueue.length)
        const { mediaId, status } = entriesNotInQueue[randomIdx]

        addToQueue(this.$store, {
          id: mediaId,
          externalLinks: await this.getExternalLinks(mediaId),
          listEntry: { status },
        })
      }
    } else {
      const currentQueue = this.queue.map(item => item.id)
      const newItems: number[] = []

      const firstTenEntries = entries.filter(entry => {
        const include = !newItems.includes(entry.mediaId)

        if (
          currentQueue.includes(entry.mediaId) ||
          !include ||
          newItems.length >= 10
        ) {
          return false
        }

        newItems.push(entry.mediaId)
        return true
      })

      await Promise.all(
        firstTenEntries.map(async ({ mediaId, status }) =>
          addToQueue(this.$store, {
            id: mediaId,
            externalLinks: await this.getExternalLinks(mediaId),
            listEntry: { status },
          }),
        ),
      )
    }

    const diff = this.queue.length - queueBefore.length

    if (diff < 1) {
      return sendToast(this.$store, {
        type: 'error',
        title: "Couldn't find any shows in that state!",
        message: "At least none that aren't already in the Queue",
      })
    }

    if (!random) {
      sendToast(this.$store, {
        type: 'success',
        title: `Imported ${diff} show${diff === 1 ? '' : 's'} into the Queue!`,
        message: '',
      })
    }

    setTimeout(() => {
      this.$refs.container.scrollTo({
        top: 100000,
        behavior: 'smooth',
      })
    }, 350)
  }

  public async importWatching() {
    return this.importFrom([MediaListStatus.Current, MediaListStatus.Repeating])
  }

  public async importPlanning() {
    return this.importFrom(MediaListStatus.Planning, true)
  }

  public async importPaused() {
    return this.importFrom(MediaListStatus.Paused, true)
  }

  public async exportQueue() {
    const exportFilePath = resolve(
      this.defaultBackupPath,
      `queue-${getAnilistUsername(this.$store)}-${Date.now()}.json`,
    )

    const data = this.animes.map(anime =>
      pick(anime, ['id', 'externalLinks', 'listEntry']),
    )

    if (!existsSync(this.defaultBackupPath)) {
      mkdirSync(this.defaultBackupPath)
    }

    const { filePath } = await remote.dialog.showSaveDialog(activeWindow(), {
      title: 'Export Queue...',
      buttonLabel: 'Export',
      defaultPath: exportFilePath,
      showsTagField: false,
      filters: [this.jsonFilter],
    })

    if (isNil(filePath)) return

    writeFileSync(filePath, JSON.stringify(data))

    sendToast(this.$store, {
      type: 'success',
      title: 'Exported Queue!',
      message: `Click this to see the file!`,
      timeout: 6000,
      click: () => shell.showItemInFolder(filePath!),
    })
  }

  public async importQueueFromBackup() {
    const { filePaths } = await remote.dialog.showOpenDialog({
      title: 'Import Backup...',
      buttonLabel: 'Import',
      defaultPath: this.defaultBackupPath,
      filters: [this.jsonFilter],
      properties: ['openFile'],
    })

    if (isNil(filePaths) || length < 1) return
    const openPath = filePaths[0]

    try {
      const data = JSON.parse(readFileSync(openPath).toString())

      if (typeof data !== 'object' || !Array.isArray(data)) {
        throw new Error('Could not parse backup file!')
      }

      if (data.some(({ id }) => typeof id !== 'number')) {
        throw new Error('This backup is too old and is not supported anymore.')
      }

      ;(data as any[]).forEach(item => addToQueue(this.$store, item))
    } catch (err) {
      sendErrorToast(this.$store, err.message)
    }
  }

  public clearQueue() {
    setQueue(this.$store, [])
  }
}
</script>

<style scoped lang="scss">
@import '../../colors';

.queue {
  display: flex;

  position: absolute;
  top: 80px;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.85);

  .queue-container {
    position: relative;
    padding: 15px 25px;
    overflow-y: auto;
    overflow-x: hidden;

    width: 100%;
    min-width: 800px;
    user-select: none;

    & > .smooth-dnd-container {
      position: relative;
      height: 100%;

      & > .smooth-dnd-draggable-wrapper {
        position: relative;
        overflow: visible;
      }
    }

    & > .empty-message {
      position: absolute;
      top: 0;
      left: 0;
      height: 100%;
      width: 100%;
      display: flex;
      justify-content: center;
      flex-direction: column;
      align-items: center;
      font-size: 1.25em;
      font-weight: 600;

      & > .button {
        margin-top: 10px;
        font-size: 0.85em;
      }
    }
  }

  .sidebar {
    width: 325px;
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    align-items: stretch;
    padding: 20px 25px;
    background: #202130;
    transition: margin-bottom 0.25s;

    & > .local-files-help {
      margin-bottom: 10px;
      font-weight: 700;
      text-decoration: none;
      color: color($highlightText, 400);
      transition: color 0.15s;

      &:hover {
        color: color($highlightText, 600);
      }
    }

    &.small {
      margin-bottom: 183px;
    }

    &.external {
      margin-bottom: 50px !important;
    }

    & > .button {
      margin: 8px 0;
      flex-shrink: 0;
    }

    & > .fill {
      height: 100%;
    }
  }
}

.route-enter-active,
.route-leave-active {
  transition: background 0.5s;

  & > .queue-container,
  & > .sidebar {
    transition: transform 0.5s;
  }

  & > .queue-container {
    overflow: visible;
  }
}

.route-enter,
.route-leave-to {
  background: none;

  & > .queue-container {
    transform: translateX(-100%);
  }

  & > .sidebar {
    transform: translateY(-125%);
  }
}
</style>
