<template>
  <AppPaneLayout>
    <template #primary>
      <div
        class="sticky top-0 z-10 flex flex-shrink-0 p-4 overflow-x-auto space-x-2 bg-primary hide-scrollbar"
      >
        <div class="inline-flex flex-1 space-x-2">
          <input
            id="websocket-url"
            v-model="url"
            class="w-full px-4 py-2 border rounded bg-primaryLight border-divider text-secondaryDark"
            type="url"
            autocomplete="off"
            spellcheck="false"
            :class="{ error: !urlValid }"
            :placeholder="$t('websocket.url')"
            :disabled="connectionState"
            @keyup.enter="urlValid ? toggleConnection() : null"
          />
          <ButtonPrimary
            id="connect"
            :disabled="!urlValid"
            class="w-32"
            name="connect"
            :label="
              !connectionState ? $t('action.connect') : $t('action.disconnect')
            "
            :loading="connectingState"
            @click.native="toggleConnection"
          />
        </div>
      </div>
      <div
        class="sticky z-10 flex items-center justify-between pl-4 border-b bg-primary border-dividerLight top-upperPrimaryStickyFold"
      >
        <label class="font-semibold text-secondaryLight">
          {{ $t("websocket.protocols") }}
        </label>
        <div class="flex">
          <ButtonSecondary
            v-tippy="{ theme: 'tooltip' }"
            :title="$t('action.clear_all')"
            svg="trash-2"
            @click.native="clearContent"
          />
          <ButtonSecondary
            v-tippy="{ theme: 'tooltip' }"
            :title="$t('add.new')"
            svg="plus"
            @click.native="addProtocol"
          />
        </div>
      </div>
      <draggable
        v-model="protocols"
        animation="250"
        handle=".draggable-handle"
        draggable=".draggable-content"
        ghost-class="cursor-move"
        chosen-class="bg-primaryLight"
        drag-class="cursor-grabbing"
      >
        <div
          v-for="(protocol, index) of protocols"
          :key="`protocol-${index}`"
          class="flex border-b divide-x divide-dividerLight border-dividerLight draggable-content group"
        >
          <span>
            <ButtonSecondary
              svg="grip-vertical"
              class="cursor-auto text-primary hover:text-primary"
              :class="{
                'draggable-handle group-hover:text-secondaryLight !cursor-grab':
                  index !== protocols?.length - 1,
              }"
              tabindex="-1"
            />
          </span>
          <input
            v-model="protocol.value"
            class="flex flex-1 px-4 py-2 bg-transparent"
            :placeholder="$t('count.protocol', { count: index + 1 })"
            name="message"
            type="text"
            autocomplete="off"
            @change="
              updateProtocol(index, {
                value: $event.target.value,
                active: protocol.active,
              })
            "
          />
          <span>
            <ButtonSecondary
              v-tippy="{ theme: 'tooltip' }"
              :title="
                protocol.hasOwnProperty('active')
                  ? protocol.active
                    ? $t('action.turn_off')
                    : $t('action.turn_on')
                  : $t('action.turn_off')
              "
              :svg="
                protocol.hasOwnProperty('active')
                  ? protocol.active
                    ? 'check-circle'
                    : 'circle'
                  : 'check-circle'
              "
              color="green"
              @click.native="
                updateProtocol(index, {
                  value: protocol.value,
                  active: !protocol.active,
                })
              "
            />
          </span>
          <span>
            <ButtonSecondary
              v-tippy="{ theme: 'tooltip' }"
              :title="$t('action.remove')"
              svg="trash"
              color="red"
              @click.native="deleteProtocol({ index })"
            />
          </span>
        </div>
      </draggable>
      <div
        v-if="protocols.length === 0"
        class="flex flex-col items-center justify-center p-4 text-secondaryLight"
      >
        <img
          :src="`/images/states/${$colorMode.value}/add_category.svg`"
          loading="lazy"
          class="inline-flex flex-col object-contain object-center w-16 h-16 my-4"
          :alt="$t('empty.protocols')"
        />
        <span class="mb-4 text-center">{{ $t("empty.protocols") }}</span>
      </div>
    </template>
    <template #secondary>
      <RealtimeLog
        :title="$t('websocket.log')"
        :log="log"
        @delete="clearLogEntries()"
      />
    </template>
    <template #sidebar>
      <div class="flex items-center justify-between p-4">
        <label
          for="websocket-message"
          class="font-semibold text-secondaryLight"
          >{{ $t("websocket.communication") }}</label
        >
      </div>
      <div class="flex px-4 space-x-2">
        <input
          id="websocket-message"
          v-model="communication.input"
          name="message"
          type="text"
          autocomplete="off"
          :disabled="!connectionState"
          :placeholder="$t('websocket.message')"
          class="input"
          @keyup.enter="connectionState ? sendMessage() : null"
          @keyup.up="connectionState ? walkHistory('up') : null"
          @keyup.down="connectionState ? walkHistory('down') : null"
        />
        <ButtonPrimary
          id="send"
          name="send"
          :disabled="!connectionState"
          :label="$t('action.send')"
          @click.native="sendMessage"
        />
      </div>
    </template>
  </AppPaneLayout>
</template>

<script>
import { defineComponent } from "@nuxtjs/composition-api"
import debounce from "lodash/debounce"
import draggable from "vuedraggable"
import { logHoppRequestRunToAnalytics } from "~/helpers/fb/analytics"
import {
  setWSEndpoint,
  WSEndpoint$,
  WSProtocols$,
  setWSProtocols,
  addWSProtocol,
  deleteWSProtocol,
  updateWSProtocol,
  deleteAllWSProtocols,
  WSSocket$,
  setWSSocket,
  setWSConnectionState,
  setWSConnectingState,
  WSConnectionState$,
  WSConnectingState$,
  addWSLogLine,
  WSLog$,
  setWSLog,
} from "~/newstore/WebSocketSession"
import { useStream } from "~/helpers/utils/composables"

export default defineComponent({
  components: {
    draggable,
  },
  setup() {
    return {
      url: useStream(WSEndpoint$, "", setWSEndpoint),
      protocols: useStream(WSProtocols$, [], setWSProtocols),
      connectionState: useStream(
        WSConnectionState$,
        false,
        setWSConnectionState
      ),
      connectingState: useStream(
        WSConnectingState$,
        false,
        setWSConnectingState
      ),
      socket: useStream(WSSocket$, null, setWSSocket),
      log: useStream(WSLog$, [], setWSLog),
    }
  },
  data() {
    return {
      isUrlValid: true,
      communication: {
        input: "",
      },
      currentIndex: -1, // index of the message log array to put in input box
      activeProtocols: [],
    }
  },
  computed: {
    urlValid() {
      return this.isUrlValid
    },
  },
  watch: {
    url() {
      this.debouncer()
    },
    protocols: {
      handler(newVal) {
        this.activeProtocols = newVal
          .filter((item) =>
            Object.prototype.hasOwnProperty.call(item, "active")
              ? item.active === true
              : true
          )
          .map(({ value }) => value)
      },
      deep: true,
    },
  },
  created() {
    if (process.browser) {
      this.worker = this.$worker.createRejexWorker()
      this.worker.addEventListener("message", this.workerResponseHandler)
    }
  },
  destroyed() {
    this.worker.terminate()
  },
  methods: {
    clearContent() {
      deleteAllWSProtocols()
    },
    debouncer: debounce(function () {
      this.worker.postMessage({ type: "ws", url: this.url })
    }, 1000),
    workerResponseHandler({ data }) {
      if (data.url === this.url) this.isUrlValid = data.result
    },
    toggleConnection() {
      // If it is connecting:
      if (!this.connectionState) return this.connect()
      // Otherwise, it's disconnecting.
      else return this.disconnect()
    },
    clearLogEntries() {
      this.log = []
    },
    connect() {
      this.log = [
        {
          payload: this.$t("state.connecting_to", { name: this.url }),
          source: "info",
          event: "connecting",
          ts: Date.now(),
        },
      ]
      try {
        this.connectingState = true
        this.socket = new WebSocket(this.url, this.activeProtocols)
        this.socket.onopen = () => {
          this.connectingState = false
          this.connectionState = true
          this.log = [
            {
              payload: this.$t("state.connected_to", { name: this.url }),
              source: "info",
              event: "connected",
              ts: Date.now(),
            },
          ]
          this.$toast.success(this.$t("state.connected"))
        }
        this.socket.onerror = () => {
          this.handleError()
        }
        this.socket.onclose = () => {
          this.connectionState = false
          addWSLogLine({
            payload: this.$t("state.disconnected_from", { name: this.url }),
            source: "disconnected",
            event: "disconnected",
            ts: Date.now(),
          })
          this.$toast.error(this.$t("state.disconnected"))
        }
        this.socket.onmessage = ({ data }) => {
          addWSLogLine({
            payload: data,
            source: "server",
            ts: Date.now(),
          })
        }
      } catch (e) {
        this.handleError(e)
        this.$toast.error(this.$t("error.something_went_wrong"))
      }

      logHoppRequestRunToAnalytics({
        platform: "wss",
      })
    },
    disconnect() {
      if (this.socket) {
        this.socket.close()
        this.connectionState = false
        this.connectingState = false
      }
    },
    handleError(error) {
      this.disconnect()
      this.connectionState = false
      addWSLogLine({
        payload: this.$t("error.something_went_wrong"),
        source: "info",
        event: "error",
        ts: Date.now(),
      })
      if (error !== null)
        addWSLogLine({
          payload: error,
          source: "info",
          event: "error",
          ts: Date.now(),
        })
    },
    sendMessage() {
      const message = this.communication.input
      this.socket.send(message)
      addWSLogLine({
        payload: message,
        source: "client",
        ts: Date.now(),
      })
      this.communication.input = ""
    },
    walkHistory(direction) {
      const clientMessages = this.log.filter(
        ({ source }) => source === "client"
      )
      const length = clientMessages.length
      switch (direction) {
        case "up":
          if (length > 0 && this.currentIndex !== 0) {
            // does nothing if message log is empty or the currentIndex is 0 when up arrow is pressed
            if (this.currentIndex === -1) {
              this.currentIndex = length - 1
              this.communication.input =
                clientMessages[this.currentIndex].payload
            } else if (this.currentIndex === 0) {
              this.communication.input = clientMessages[0].payload
            } else if (this.currentIndex > 0) {
              this.currentIndex = this.currentIndex - 1
              this.communication.input =
                clientMessages[this.currentIndex].payload
            }
          }
          break
        case "down":
          if (length > 0 && this.currentIndex > -1) {
            if (this.currentIndex === length - 1) {
              this.currentIndex = -1
              this.communication.input = ""
            } else if (this.currentIndex < length - 1) {
              this.currentIndex = this.currentIndex + 1
              this.communication.input =
                clientMessages[this.currentIndex].payload
            }
          }
          break
      }
    },
    addProtocol() {
      addWSProtocol({ value: "", active: true })
    },
    deleteProtocol({ index }) {
      const oldProtocols = this.protocols.slice()
      deleteWSProtocol(index)
      this.$toast.success(this.$t("state.deleted"), {
        action: {
          text: this.$t("action.undo"),
          duration: 4000,
          onClick: (_, toastObject) => {
            this.protocols = oldProtocols
            toastObject.remove()
          },
        },
      })
    },
    updateProtocol(index, updated) {
      updateWSProtocol(index, updated)
    },
  },
})
</script>
