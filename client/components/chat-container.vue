<template>
  <div class="chat-container">
    <div id="messages" class="chat-thread">
      <infinite-loading
        v-if="thread"
        :identifier="infiniteIdentifier"
        direction="top"
        @infinite="loadMoreItems"
      >
        <div slot="no-more" class="no-more-messages">No more messages.</div>
      </infinite-loading>
      <div
        v-for="item in items"
        :key="item.item_id"
        :class="['chat-thread-item', item.isFromUser ? 'user' : 'friend']"
      >
        <img
          v-if="item.user && !item.isFromUser"
          :src="item.user.profile_pic_url"
          class="chat-profile-thumbnail"
        >
        <Message v-if="item.item_type === 'text'" :item="item"/>
        <ReelShare v-else-if="item.item_type === 'reel_share'" :item="item" :friend="item.user"/>
        <MediaShare v-else-if="item.item_type === 'media_share'" :item="item"/>
        <Audio v-else-if="item.item_type === 'audio'" :item="item"/>
        <VoiceMedia v-else-if="item.item_type === 'voice_media'" :item="item"/>
        <Media v-else-if="item.item_type === 'media'" :item="item"/>
        <AnimatedMedia v-else-if="item.item_type === 'animated_media'" :item="item"/>
        <Like v-else-if="item.item_type === 'like'"/>
        <StoryShare v-else-if="item.item_type === 'story_share'" :item="item" :friend="item.user"/>
      </div>
    </div>
    <ChatForm v-if="threadId" :thread-id="threadId" @refetch="getThread"/>
  </div>
</template>

<script>
import InfiniteLoading from 'vue-infinite-loading'
import Message from './chat-items/message'
import ReelShare from './chat-items/reel-share'
import MediaShare from './chat-items/media-share'
import Like from './chat-items/like'
import VoiceMedia from './chat-items/voice-media'
import Media from './chat-items/media'
import AnimatedMedia from './chat-items/animated-media'
import StoryShare from './chat-items/story-share'
import ChatForm from './chat-form'
import axios from 'axios'

export default {
  components: {
    Message,
    ReelShare,
    MediaShare,
    ChatForm,
    VoiceMedia,
    Media,
    Like,
    AnimatedMedia,
    StoryShare,
    InfiniteLoading,
  },
  props: {
    threadId: {
      type: String,
    },
    user: {
      type: Object,
    },
  },
  data() {
    return {
      message: '',
      thread: null,
      moreAvailable: false,
      cursor: null,
      infiniteIdentifier: +new Date(),
    }
  },
  computed: {
    items() {
      return this.thread
        ? this.thread.items
            .filter(el => !['action_log', 'placeholder', 'raven_media', 'video_call_event'].includes(el.item_type))
            .sort((a, b) => a.timestamp - b.timestamp)
            .map(item => {
              const isFromUser = item.user_id === this.user.pk
              return {
                ...item,
                isFromUser,
                user: isFromUser ? this.user : this.thread.users.find(user => item.user_id === user.pk),
              }
            })
        : []
    },
  },
  watch: {
    threadId(newId) {
      if (newId) {
        this.getThread(newId)
      }
    },
  },
  methods: {
    getThread(id = this.threadId) {
      axios.get(`/api/thread/${id}`).then(({ data: { thread, moreAvailable, cursor } }) => {
        this.thread = thread
        this.moreAvailable = moreAvailable
        this.cursor = cursor
        this.$nextTick(() => {
          this.scrollToBottom()
          this.infiniteIdentifier = +new Date()
        })
      })
    },
    scrollToBottom() {
      const container = this.$el.querySelector('#messages')
      if (container) {
        container.scrollTop = container.scrollHeight
      }
    },
    loadMoreItems($state) {
      axios
        .get(`/api/thread/${this.threadId}?cursor=${this.cursor}`)
        .then(({ data: { thread, moreAvailable, cursor } }) => {
          this.thread = {
            ...thread,
            items: [...this.thread.items, ...thread.items],
          }
          this.moreAvailable = moreAvailable
          this.cursor = cursor

          if (moreAvailable) $state.loaded()
          else $state.complete()
        })
    },
  },
}
</script>

<style>
.chat-container {
  flex-grow: 1;
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
}

.chat-thread {
  overflow: auto;
  display: flex;
  flex-direction: column;
  padding-right: 5px;
}

.chat-thread-item {
  margin: 7px 5px;
  max-width: 400px;
  display: flex;
  min-height: min-content;
  align-items: flex-start;
}

.chat-thread-item.friend {
  align-self: flex-start;
  align-items: flex-start;
}

.chat-thread-item.friend .chat-message {
  background-color: lightgray;
  color: black;
}

.chat-thread-item.user {
  align-self: flex-end;
  align-items: flex-end;
}

.chat-thread-item.user .chat-message {
  background-color: blue;
  color: white;
}

.chat-profile-thumbnail {
  min-width: 40px;
  height: 40px;
  border-radius: 100%;
  margin-right: 10px;
}

.chat-message {
  padding: 5px 15px;
  border-radius: 20px;
  line-height: 26px;
}

.message-preface {
  color: grey;
  margin-bottom: 3px;
}

.no-more-messages {
  color: grey;
}
</style>