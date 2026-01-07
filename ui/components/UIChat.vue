<template>
    <div class="nrdb-ui-chat-container" :class="className">
        <div class="nrdb-ui-chat-toolbar">
            <div v-if="clientId" class="nrdb-ui-session-id">
                Session: {{ clientId.slice(0, 8) }}…
            </div>
            <button
                class="nrdb-ui-chat-reset"
                type="button"
                :disabled="!state.enabled && !messages.length"
                title="Start a new chat session"
                @click="startNewChat"
            >
                New chat
            </button>
        </div>
        <!-- Chat messages container -->
        <div class="nrdb-ui-chat-messages">
            <div
                v-for="(message, index) in messages"
                :key="index"
                class="nrdb-ui-message"
                :class="message.sent ? 'sent' : 'received'"
            >
                <div v-if="props.showAuthor && !message.sent" class="nrdb-ui-message-author">
                    {{ message.author }}
                </div>
                <div class="nrdb-ui-message-content" v-html="renderMarkdown(message.text)"/>
                <div class="nrdb-ui-message-time">
                    {{ message.time }}
                </div>
            </div>
            <div v-if="typing" class="nrdb-ui-message received">
                <div class="nrdb-ui-message-content nrdb-ui-message-typing">
                    <div class="nrdb-ui-message-typing-dot" />
                    <div class="nrdb-ui-message-typing-dot" />
                    <div class="nrdb-ui-message-typing-dot" />
                </div>
            </div>
        </div>

        <!-- Message input form -->
        <div class="nrdb-ui-chat-input">
            <div class="nrdb-ui-chat-input-container">
                <input
                    v-model="newMessage"
                    type="text"
                    :placeholder="props.inputPlaceholder === undefined ? 'Type a message...' : props.inputPlaceholder"
                    :disabled="!!inputDisabledReason"
                    :title="inputDisabledReason"
                    @keyup.enter="sendMessage"
                >
                <button
                    class="nrdb-ui-chat-send" :disabled="!!inputDisabledReason || !newMessage.trim()"
                    :title="inputDisabledReason"
                    @click="sendMessage"
                >
                    <AirplaneIcon />
                </button>
            </div>
        </div>
    </div>
</template>

<script>

import DOMPurify from 'dompurify'

import { marked } from 'marked'

import AirplaneIcon from './icons/AirplaneIcon.vue'

export default {
    name: 'UIChat',
    components: {
        AirplaneIcon
    },
    inject: ['$socket', '$dataTracker'],
    props: {
        /* do not remove entries from this - Dashboard's Layout Manager's will pass this data to your component */
        id: { type: String, required: true },
        props: { type: Object, default: () => ({}) },
        state: { type: Object, default: () => ({ enabled: false, visible: false }) }
    },
    setup (props) {
        // Configure marked options
        marked.setOptions({
            breaks: true, // Convert line breaks to <br>
            gfm: true, // Enable GitHub Flavored Markdown
            sanitize: true, // Don't HTML (be careful with user input)
            smartLists: true,
            smartypants: true
        })
    },
    data () {
        return {
            messages: [],
            newMessage: '',
            typing: false,
            sessionId: null,
            clientId: null
        }
    },
    computed: {
        inputDisabledReason () {
            if (!this.sessionId) {
                return 'Waiting for session to be established'
            }
            if (!this.state.enabled) {
                return 'Input disabled'
            }
            return ''
        }
    },
    created () {
        this.clientId = this.loadClientId()
        const persistedMessages = this.restoreMessages()
        if (persistedMessages.length) {
            this.messages = persistedMessages
        }

        if (this.$socket?.on) {
            this.$socket.on('connect', this.updateSessionId)
            this.$socket.on('disconnect', this.handleDisconnect)
        }
        this.updateSessionId()
        // setup our event handlers, and informs Node-RED that this widget has loaded
        this.$dataTracker(this.id, this.onInput, this.onLoad)
    },
    beforeUnmount () {
        if (this.$socket?.off) {
            this.$socket.off('connect', this.updateSessionId)
            this.$socket.off('disconnect', this.handleDisconnect)
        }
    },
    methods: {
        storageKey (suffix) {
            return `ui-chat:${this.id}:${suffix}`
        },
        generateClientId () {
            return typeof crypto !== 'undefined' && crypto.randomUUID ? crypto.randomUUID() : `ui-chat-${Date.now()}-${Math.random().toString(16).slice(2)}`
        },
        loadClientId () {
            try {
                const existing = window?.localStorage?.getItem(this.storageKey('clientId'))
                if (existing) {
                    return existing
                }
                const generated = this.generateClientId()
                window?.localStorage?.setItem(this.storageKey('clientId'), generated)
                return generated
            } catch (err) {
                console.warn('UIChat: Unable to access localStorage, session continuity will be disabled', err)
                return null
            }
        },
        persistClientId () {
            try {
                if (this.clientId) {
                    window?.localStorage?.setItem(this.storageKey('clientId'), this.clientId)
                }
            } catch (err) {
                console.warn('UIChat: Unable to persist client id', err)
            }
        },
        persistMessages () {
            try {
                window?.localStorage?.setItem(this.storageKey('history'), JSON.stringify(this.messages))
            } catch (err) {
                console.warn('UIChat: Unable to persist chat history', err)
            }
        },
        restoreMessages () {
            try {
                const raw = window?.localStorage?.getItem(this.storageKey('history'))
                if (!raw) {
                    return []
                }
                const parsed = JSON.parse(raw)
                if (!Array.isArray(parsed)) {
                    return []
                }
                return parsed.map(m => ({
                    ...m,
                    time: m.time || new Date(m.timestamp || Date.now()).toLocaleTimeString()
                }))
            } catch (err) {
                console.warn('UIChat: Unable to restore chat history', err)
                return []
            }
        },
        startNewChat () {
            const previousClientId = this.clientId
            this.clientId = this.generateClientId()
            this.messages = []
            this.typing = false
            this.persistClientId()
            this.persistMessages()
            if (this.sessionId) {
                this.send({
                    topic: 'user-session-reset',
                    previousClientId,
                    clientId: this.clientId,
                    _session: { id: this.sessionId, clientId: this.clientId }
                })
            }
        },
        updateSessionId () {
            this.sessionId = this.$socket?.id || null
            if (this.sessionId && this.clientId) {
                this.send({
                    topic: '_session_init',
                    clientId: this.clientId,
                    _session: { id: this.sessionId, clientId: this.clientId }
                })
            }
        },
        handleDisconnect () {
            this.sessionId = null
            this.typing = false
        },
        isForCurrentSession (msg) {
            const targetClient = msg?.clientId || msg?._session?.clientId
            // Prefer _session.id (Dashboard 2), but accept legacy sessionId/socketId from older payloads
            const targetSession = msg?._session?.id || msg?.sessionId || msg?.socketId
            if (!this.sessionId) {
                console.warn('UIChat: message received before session was established. This may indicate a timing issue in component initialization; the message will be ignored.')
                return targetClient ? targetClient === this.clientId : false
            }
            if (targetClient && this.clientId) {
                return targetClient === this.clientId
            }
            if (!targetSession) {
                const allowBroadcast = this.props?.allowBroadcast ?? false
                if (allowBroadcast) {
                    console.debug('UIChat: processing broadcast message without session targeting')
                    return true
                }
                console.warn('UIChat: dropping broadcast message without session targeting')
                return false
            }
            return targetSession === this.sessionId
        },
        renderMarkdown (text) {
            try {
                // Parse markdown and return HTML
                const html = marked.parse(text)
                return DOMPurify.sanitize(html)
            } catch (error) {
                console.error('Error parsing markdown:', error)
                // Fallback to plain text if markdown parsing fails
                return this.escapeHtml(text)
            }
        },
        escapeHtml (text) {
            // Basic HTML escaping for fallback
            const div = document.createElement('div')
            div.textContent = text
            return div.innerHTML
        },
        onInput (msg) {
            if (!this.isForCurrentSession(msg)) {
                return
            }
            if (msg.topic === '_typing') {
                this.typing = true
            } else if (msg?.payload) {
                // Handle incoming messages from Node-RED
                this.typing = false
                const timestamp = msg.timestamp || Date.now()
                this.messages.push({
                    author: msg.topic,
                    text: msg.payload,
                    time: new Date(timestamp).toLocaleTimeString(),
                    sent: false,
                    timestamp
                })
                this.persistMessages()
            }
            this.$nextTick(() => {
                this.scrollToBottom()
            })
        },
        onLoad (msg) {
            if (!this.isForCurrentSession(msg)) {
                return
            }
            // Handle initial load
            if (Array.isArray(msg?.payload)) {
                this.messages = msg.payload.map(m => ({
                    ...m,
                    time: new Date(m.timestamp || Date.now()).toLocaleTimeString()
                }))
                this.persistMessages()
                this.$nextTick(() => {
                    this.scrollToBottom()
                })
            }
        },
        send (msg) {
            this.$socket.emit('widget-action', this.id, msg)
        },
        sendMessage () {
            if (!this.newMessage.trim() || !this.state.enabled) return

            if (!this.sessionId) {
                console.warn('UIChat: cannot send message until session is established')
                return
            }

            // Create message object
            const timestamp = Date.now()
            const message = {
                author: 'user',
                text: this.newMessage,
                time: new Date(timestamp).toLocaleTimeString(),
                sent: true,
                timestamp
            }

            // Add to local messages
            this.messages.push(message)
            this.persistMessages()

            // Send to Node-RED
            const action = {
                topic: 'user-message',
                payload: this.newMessage,
                clientId: this.clientId,
                _session: { id: this.sessionId, clientId: this.clientId }
            }
            this.send(action)

            // Clear input
            this.newMessage = ''

            // Scroll to bottom
            this.$nextTick(() => {
                this.scrollToBottom()
            })
        },
        scrollToBottom () {
            const container = this.$el.querySelector('.nrdb-ui-chat-messages')
            if (container) {
                container.scrollTop = container.scrollHeight
            }
        }
    }
}
</script>

<style scoped>
/* CSS is auto scoped, but using named classes is still recommended */
@import "../stylesheets/ui-chat.css";

/* Additional styles for markdown content */
.nrdb-ui-message-content :deep(p) {
    margin: 0.5em 0;
}

.nrdb-ui-message-content :deep(p:first-child) {
    margin-top: 0;
}

.nrdb-ui-message-content :deep(p:last-child) {
    margin-bottom: 0;
}

.nrdb-ui-message-content :deep(code) {
    background-color: rgba(0, 0, 0, 0.1);
    padding: 0.2em 0.4em;
    border-radius: 3px;
    font-family: monospace;
}

.nrdb-ui-message-content :deep(pre) {
    background-color: rgba(0, 0, 0, 0.1);
    padding: 1em;
    border-radius: 5px;
    overflow-x: auto;
    white-space: pre-wrap;
}

.nrdb-ui-message-content :deep(blockquote) {
    border-left: 4px solid #ddd;
    margin: 0.5em 0;
    padding-left: 1em;
    color: #666;
}

.nrdb-ui-message-content :deep(ul),
.nrdb-ui-message-content :deep(ol) {
    margin: 0.5em 0;
    padding-left: 1.5em;
}

.nrdb-ui-message-content :deep(strong) {
    font-weight: bold;
}

.nrdb-ui-message-content :deep(em) {
    font-style: italic;
}

.nrdb-ui-message-content :deep(a) {
    color: #007bff;
    text-decoration: underline;
}

.nrdb-ui-message-content :deep(h1),
.nrdb-ui-message-content :deep(h2),
.nrdb-ui-message-content :deep(h3),
.nrdb-ui-message-content :deep(h4),
.nrdb-ui-message-content :deep(h5),
.nrdb-ui-message-content :deep(h6) {
    margin: 0.5em 0;
    font-weight: bold;
}
</style>
