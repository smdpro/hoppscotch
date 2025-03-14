<template>
  <AppPaneLayout>
    <template #primary>
      <div
        class="sticky top-0 z-10 flex flex-shrink-0 p-4 overflow-x-auto space-x-2 bg-primary hide-scrollbar"
      >
        <div class="inline-flex flex-1 space-x-2">
          <input
            id="mqtt-url"
            v-model="url"
            type="url"
            autocomplete="off"
            spellcheck="false"
            class="w-full px-4 py-2 border rounded bg-primaryLight border-divider text-secondaryDark"
            :placeholder="$t('mqtt.url')"
            :disabled="connectionState"
            @keyup.enter="validUrl ? toggleConnection() : null"
          />
          <ButtonPrimary
            id="connect"
            :disabled="!validUrl"
            class="w-32"
            :label="
              connectionState ? $t('action.disconnect') : $t('action.connect')
            "
            :loading="connectingState"
            @click.native="toggleConnection"
          />
        </div>
        <div class="flex space-x-4">
          <input
            id="mqtt-username"
            v-model="username"
            type="text"
            spellcheck="false"
            class="input"
            :placeholder="$t('authorization.username')"
          />
          <input
            id="mqtt-password"
            v-model="password"
            type="password"
            spellcheck="false"
            class="input"
            :placeholder="$t('authorization.password')"
          />
        </div>
      </div>
    </template>
    <template #secondary>
      <RealtimeLog
        :title="$t('mqtt.log')"
        :log="log"
        @delete="clearLogEntries()"
      />
    </template>
    <template #sidebar>
      <div class="flex items-center justify-between p-4">
        <label for="pub_topic" class="font-semibold text-secondaryLight">
          {{ $t("mqtt.topic") }}
        </label>
      </div>
      <div class="flex px-4">
        <input
          id="pub_topic"
          v-model="pub_topic"
          class="input"
          :placeholder="$t('mqtt.topic_name')"
          type="text"
          autocomplete="off"
          spellcheck="false"
        />
      </div>
      <div class="flex items-center justify-between p-4">
        <label for="mqtt-message" class="font-semibold text-secondaryLight">
          {{ $t("mqtt.communication") }}
        </label>
      </div>
      <div class="flex px-4 space-x-2">
        <input
          id="mqtt-message"
          v-model="msg"
          class="input"
          type="text"
          autocomplete="off"
          :placeholder="$t('mqtt.message')"
          spellcheck="false"
        />
        <ButtonPrimary
          id="publish"
          name="get"
          :disabled="!canpublish"
          :label="$t('mqtt.publish')"
          @click.native="publish"
        />
      </div>
      <div
        class="flex items-center justify-between p-4 mt-4 border-t border-dividerLight"
      >
        <label for="sub_topic" class="font-semibold text-secondaryLight">
          {{ $t("mqtt.topic") }}
        </label>
      </div>
      <div class="flex px-4 space-x-2">
        <input
          id="sub_topic"
          v-model="sub_topic"
          type="text"
          autocomplete="off"
          :placeholder="$t('mqtt.topic_name')"
          spellcheck="false"
          class="input"
        />
        <ButtonPrimary
          id="subscribe"
          name="get"
          :disabled="!cansubscribe"
          :label="
            subscriptionState ? $t('mqtt.unsubscribe') : $t('mqtt.subscribe')
          "
          reverse
          @click.native="toggleSubscription"
        />
      </div>
    </template>
  </AppPaneLayout>
</template>

<script>
import { defineComponent } from "@nuxtjs/composition-api"
import Paho from "paho-mqtt"
import debounce from "lodash/debounce"
import { logHoppRequestRunToAnalytics } from "~/helpers/fb/analytics"
import {
  MQTTEndpoint$,
  setMQTTEndpoint,
  MQTTConnectingState$,
  MQTTConnectionState$,
  setMQTTConnectingState,
  setMQTTConnectionState,
  MQTTSubscriptionState$,
  setMQTTSubscriptionState,
  MQTTSocket$,
  setMQTTSocket,
  MQTTLog$,
  setMQTTLog,
  addMQTTLogLine,
} from "~/newstore/MQTTSession"
import { useStream } from "~/helpers/utils/composables"

export default defineComponent({
  setup() {
    return {
      url: useStream(MQTTEndpoint$, "", setMQTTEndpoint),
      connectionState: useStream(
        MQTTConnectionState$,
        false,
        setMQTTConnectionState
      ),
      connectingState: useStream(
        MQTTConnectingState$,
        false,
        setMQTTConnectingState
      ),
      subscriptionState: useStream(
        MQTTSubscriptionState$,
        false,
        setMQTTSubscriptionState
      ),
      log: useStream(MQTTLog$, null, setMQTTLog),
      client: useStream(MQTTSocket$, null, setMQTTSocket),
    }
  },
  data() {
    return {
      isUrlValid: true,
      pub_topic: "",
      sub_topic: "",
      msg: "",
      manualDisconnect: false,
      username: "",
      password: "",
    }
  },
  computed: {
    validUrl() {
      return this.isUrlValid
    },
    canpublish() {
      return this.pub_topic !== "" && this.msg !== "" && this.connectionState
    },
    cansubscribe() {
      return this.sub_topic !== "" && this.connectionState
    },
  },
  watch: {
    url() {
      this.debouncer()
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
    debouncer: debounce(function () {
      this.worker.postMessage({ type: "ws", url: this.url })
    }, 1000),
    workerResponseHandler({ data }) {
      if (data.url === this.url) this.isUrlValid = data.result
    },
    connect() {
      this.connectingState = true
      this.log = [
        {
          payload: this.$t("state.connecting_to", { name: this.url }),
          source: "info",
          event: "connecting",
          ts: Date.now(),
        },
      ]
      const parseUrl = new URL(this.url)
      this.client = new Paho.Client(
        `${parseUrl.hostname}${
          parseUrl.pathname !== "/" ? parseUrl.pathname : ""
        }`,
        parseUrl.port !== "" ? Number(parseUrl.port) : 8081,
        "hoppscotch"
      )
      const connectOptions = {
        onSuccess: this.onConnectionSuccess,
        onFailure: this.onConnectionFailure,
        useSSL: parseUrl.protocol !== "ws:",
      }
      if (this.username !== "") {
        connectOptions.userName = this.username
      }
      if (this.password !== "") {
        connectOptions.password = this.password
      }
      this.client.connect(connectOptions)
      this.client.onConnectionLost = this.onConnectionLost
      this.client.onMessageArrived = this.onMessageArrived

      logHoppRequestRunToAnalytics({
        platform: "mqtt",
      })
    },
    onConnectionFailure() {
      this.connectingState = false
      this.connectionState = false
      addMQTTLogLine({
        payload: this.$t("error.something_went_wrong"),
        source: "info",
        event: "error",
        ts: Date.now(),
      })
    },
    onConnectionSuccess() {
      this.connectingState = false
      this.connectionState = true
      addMQTTLogLine({
        payload: this.$t("state.connected_to", { name: this.url }),
        source: "info",
        event: "connected",
        ts: Date.now(),
      })
      this.$toast.success(this.$t("state.connected"))
    },
    onMessageArrived({ payloadString, destinationName }) {
      addMQTTLogLine({
        payload: `Message: ${payloadString} arrived on topic: ${destinationName}`,
        source: "info",
        event: "info",
        ts: Date.now(),
      })
    },
    toggleConnection() {
      if (this.connectionState) {
        this.disconnect()
      } else {
        this.connect()
      }
    },
    disconnect() {
      this.manualDisconnect = true
      this.client.disconnect()
      addMQTTLogLine({
        payload: this.$t("state.disconnected_from", { name: this.url }),
        source: "disconnected",
        event: "disconnected",
        ts: Date.now(),
      })
    },
    onConnectionLost() {
      this.connectingState = false
      this.connectionState = false
      if (this.manualDisconnect) {
        this.$toast.error(this.$t("state.disconnected"))
      } else {
        this.$toast.error(this.$t("error.something_went_wrong"))
      }
      this.manualDisconnect = false
      this.subscriptionState = false
    },
    publish() {
      try {
        this.client.publish(this.pub_topic, this.msg, 0, false)
        addMQTTLogLine({
          payload: `Published message: ${this.msg} to topic: ${this.pub_topic}`,
          ts: Date.now(),
          source: "info",
          event: "info",
        })
      } catch (e) {
        addMQTTLogLine({
          payload:
            this.$t("error.something_went_wrong") +
            `while publishing msg: ${this.msg} to topic:  ${this.pub_topic}`,
          source: "info",
          event: "error",
          ts: Date.now(),
        })
      }
    },
    toggleSubscription() {
      if (this.subscriptionState) {
        this.unsubscribe()
      } else {
        this.subscribe()
      }
    },
    subscribe() {
      try {
        this.client.subscribe(this.sub_topic, {
          onSuccess: this.usubSuccess,
          onFailure: this.usubFailure,
        })
      } catch (e) {
        addMQTTLogLine({
          payload:
            this.$t("error.something_went_wrong") +
            `while subscribing to topic:  ${this.sub_topic}`,
          source: "info",
          event: "error",
          ts: Date.now(),
        })
      }
    },
    usubSuccess() {
      this.subscriptionState = !this.subscriptionState
      addMQTTLogLine({
        payload:
          `Successfully ` +
          (this.subscriptionState ? "subscribed" : "unsubscribed") +
          ` to topic: ${this.sub_topic}`,
        source: "info",
        event: "info",
        ts: Date.now(),
      })
    },
    usubFailure() {
      addMQTTLogLine({
        payload:
          `Failed to ` +
          (this.subscriptionState ? "unsubscribe" : "subscribe") +
          ` to topic: ${this.sub_topic}`,
        source: "info",
        color: "error",
        ts: Date.now(),
      })
    },
    unsubscribe() {
      this.client.unsubscribe(this.sub_topic, {
        onSuccess: this.usubSuccess,
        onFailure: this.usubFailure,
      })
    },
    clearLogEntries() {
      this.log = []
    },
  },
})
</script>
