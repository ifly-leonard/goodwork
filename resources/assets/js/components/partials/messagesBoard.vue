<template>
<div v-if="activeTab === 'messages'" class="w-full">
  <div class="flex flex-col bg-white mx-auto mt-4 mb-8 shadow rounded">

    <div class="text-gray-600 bg-white shadow p-4 text-xl flex flex-row items-center z-10">
      <div class="pr-2">
        {{ 'Currently in room' | localize }}:
      </div>
      <template v-for="user in users">
        <img :src="generateUrl(user.avatar)" alt="" class="w-8 h-8 rounded-full mr-2" :title="user.name">
      </template>
    </div>

    <div id="message-box" class="h-50-vh overflow-y-auto">
      <div class="">
        <message v-for="(message, index) in messages" :key="index" :message="message" :user="user" :index="index" @deleted="deleteMessage" :last="messages.length === (index + 1)"></message>
      </div>
      <div v-if="messages.length === 0" class="flex flex-col justify-center items-center">
        <div class="text-gray-600 text-lg text-center py-8">
          Start communicating with your team member
        </div>
        <img src="/image/work_chat.svg" alt="direct message" class="w-96">
      </div>
    </div>

    <div class="relative bg-gray-200">
      <div class="static text-center p-8">
        <div class="relative">
          <user-suggestion-box
            :users="members"
            :name="name"
            :suggestionShown="suggestionShown"
            :suggestionSelected="suggestionSelected"
            :suggestionHighlightIndex="suggestionHighlightIndex"
            :suggestionHighlightDirection="suggestionHighlightDirection"
            :suggestionHighlightDirectionInverted="true"
            @selected="userSelected"
            class="absolute mb-2 w-full bottom-0"></user-suggestion-box>
        </div>
        <textarea class="static textarea resize-none rounded w-full p-4 text-gray-800"
          id="send-message"
          :style="{height: messageTextareaHeight}"
          ref="messageTextarea"
          :placeholder="'write your message here' | localize"
          rows=1
          v-model="message"
          @keyup="checkForMention($event)"
          @keydown="mentionHighlightMove($event)"
          @keydown.enter.prevent="sendMessage($event)"
          @focus="clearTitleNotification()"></textarea>
      </div>
    </div>
  </div>
</div>
</template>

<script>
import { mapState, mapActions } from 'vuex'
import userSuggestionBox from './userSuggestionBox'
import message from './message'

export default {
  components: {message, userSuggestionBox},
  props: ['resource', 'resourceType', 'activeTab'],

  data: () => ({
    messages: [],
    nextPageUrl: null,
    message: '',
    messageTextareaHeight: 'auto',
    title: '',
    unreadMessage: 0,
    name: '',
    mentionStarted: false,
    startIndex: 0,
    suggestionHighlightDirection: 1,
    suggestionHighlightIndex: 0,
    suggestionSelected: false,
    suggestionShown: false,
    mentions: [],
    user: user,
    users: []
  }),

  created () {
    EventBus.$on('clear-title-notification', this.clearTitleNotification)
  },

  beforeDestroy () {
    EventBus.$off('clear-title-notification', this.clearTitleNotification)
  },

  mounted () {
    this.getMessages()
    this.title = document.title
    this.listen()
    let messageBoxElement = document.getElementById('message-box')
    if (messageBoxElement) {
      messageBoxElement.scrollTop = messageBoxElement.scrollHeight
    }
  },

  updated () {
    let messageBoxElement = document.getElementById('message-box')
    if (messageBoxElement) {
      messageBoxElement.scrollTop = messageBoxElement.scrollHeight
    }
  },

computed: {
    ...mapState({
      members: state => state.members
    })
  },

  watch: {
    message (newVal) {
      // increase the height of textarea based on text present there
      this.messageTextareaHeight = newVal ? `${this.$refs.messageTextarea.scrollHeight}px` : 'auto'
    },
    activeTab: function () {
      this.getMessages()
    }
  },

  methods: {
    ...mapActions([
      'showNotification',
    ]),
    getMessages () {
      if (this.activeTab === 'messages' && this.messages.length < 1) {
        axios.get('/messages', {
          params: {
            resource_type: this.resourceType,
            resource_id: this.resource.id
          }
        })
        .then((response) => {
          this.messages = response.data.messages.data.reverse()
          this.nextPageUrl = response.data.messages.next_page_url
        })
        .catch((error) => {
          console.log(error)
        })
      }
    },
    sendMessage (e) {
      if (e.shiftKey) {
        this.message = this.message + '\n'
      } else if (this.mentionStarted) {
        this.suggestionSelected = true
      } else if (this.message.length > 0) {
        var msg = this.message
        this.message = ''
        axios.post('/messages', {
          message: msg,
          group_type: this.resourceType,
          group_id: this.resource.id,
          mentions: this.mentions
        })
          .then((response) => {
            if (response.data.status === 'success') {
              response.data.message.user = user
              this.pushMessage(response.data.message)
            }
          })
          .catch((error) => {
            this.showNotification({type: error.response.data.status, message: error.response.data.message})
          })
      }
    },
    listen () {
      Echo.join(this.resourceType + '.' + this.resource.id)
        .here(users => {
          this.users = users
        })
        .joining(user => {
          this.users.push(user)
          this.pushSystemMessage(`${user.name} has joined`)
          if ((document.activeElement !== document.getElementById('send-message')) || (!document.hasFocus())) {
            this.unreadMessage += 1
            document.title = '(' + this.unreadMessage + ') ' + this.title
          }
        })
        .leaving(user => {
          this.users = this.users.filter(u => u.username !== user.username)
          this.pushSystemMessage(`${user.name} has left`)
          if ((document.activeElement !== document.getElementById('send-message')) || (!document.hasFocus())) {
            this.unreadMessage += 1
            document.title = '(' + this.unreadMessage + ') ' + this.title
          }
        })
        .listen('MessageCreated', event => {
          event.message.user = event.user
          this.pushMessage(event.message)
          if ((document.activeElement !== document.getElementById('send-message')) || (!document.hasFocus())) {
            this.unreadMessage += 1
            document.title = '(' + this.unreadMessage + ') ' + this.title
          }
        })
    },
    clearTitleNotification () {
      document.title = this.title
      this.unreadMessage = 0
    },
    deleteMessage (index) {
      this.messages.splice(index, 1)
    },
    pushMessage (message) {
      this.messages.push(message)
      EventBus.$emit('messagePushed')
    },
    pushSystemMessage (body) {
      this.pushMessage({
        body,
        system: true
      })
    },
    checkForMention (e) {
      if (e.key === "@") {
        this.suggestionShown = true
        this.mentionStarted = true
        this.startIndex = document.getElementById('send-message').selectionStart
      } else if (e.keyCode === 32 || e.keyCode === 27 || (e.keyCode === 8 && document.getElementById('send-message').selectionStart < this.startIndex)) {
        this.mentionStarted = false
        this.suggestionShown = false
        this.suggestionHighlightIndex = 0
        this.name = ''
      } else if (this.mentionStarted) {
        this.name = e.target.value.substring(this.startIndex)
      }
    },
    mentionHighlightMove (e) {
      if (this.mentionStarted && (e.keyCode === 38 || e.keyCode === 40)) {
        if (e.keyCode === 38) {
          this.suggestionHighlightIndex -= 1
          this.suggestionHighlightDirection = -1
        } else if (e.keyCode === 40) {
          this.suggestionHighlightIndex += 1
          this.suggestionHighlightDirection = 1
        }
        e.preventDefault()
      }
    },
    userSelected (text) {
      this.message = this.message.substring(0, this.startIndex) + text
      if (this.mentions.every(mention => mention !== text)) {
        this.mentions.push(text)
      }
      this.mentionStarted = false
      this.suggestionShown = false
      this.suggestionSelected = false
      this.suggestionHighlightIndex = 0
      this.name = ''
      document.getElementById('send-message').focus()
    }
  }
}
</script>
