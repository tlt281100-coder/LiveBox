<template>
  <div class="app-root">
    <!-- 首页 -->
    <section v-if="view === 'home'" class="home-view">
      <div class="home-inner">
        <div class="brand"></div>
        <h1>LiveBox</h1>
        <div class="enter-box">
          <input
            v-model="roomUrl"
            type="text"
            autocomplete="off"
            spellcheck="false"
            placeholder="粘贴直播间地址，如 https://live.douyin.com/123456"
            @keydown.enter.prevent="enterLive"
          >
          <button class="btn-primary" type="button" :disabled="loading" @click="enterLive">
            {{ loading ? '连接中…' : '进入直播' }}
          </button>
        </div>
        <p v-if="homeError" class="error">{{ homeError }}</p>
      </div>
    </section>

    <!-- 直播页 -->
    <section v-else class="live-view" :class="{ 'hide-comments': !commentsVisible }">
      <header class="topbar">
        <button class="icon-btn" type="button" title="返回首页" @click="goHome">←</button>
        <span v-if="isLive" class="live-pill"><i class="dot"></i>直播中</span>
        <span v-else class="live-pill off">未开播</span>
        <span class="count">在线 <b>{{ onlineText }}</b> 人</span>
        <button class="text-btn" type="button" @click="commentsVisible = !commentsVisible">
          {{ commentsVisible ? '收起评论' : '展开评论' }}
        </button>
        <select v-if="qualityList.length > 1" v-model="currentQuality" class="quality-select" title="切换清晰度" @change="onQualityChange">
          <option v-for="q in qualityList" :key="q.key" :value="q.key">{{ q.label }}</option>
        </select>
        <button class="icon-btn" type="button" title="设置" @click.stop="settingsOpen = !settingsOpen">⚙</button>
      </header>

      <div v-if="settingsOpen" class="settings-pop">
        <div class="pop-title">设置</div>
        <div class="pop-group">
          <span>消息推送类型</span>
          <label class="pop-option"><input v-model="msgTypes.chat" type="checkbox">聊天评论<span class="msg-dot dot-chat"></span></label>
          <label class="pop-option"><input v-model="msgTypes.gift" type="checkbox">礼物<span class="msg-dot dot-gift"></span></label>
          <label class="pop-option"><input v-model="msgTypes.like" type="checkbox">点赞<span class="msg-dot dot-like"></span></label>
          <label class="pop-option"><input v-model="msgTypes.follow" type="checkbox">关注<span class="msg-dot dot-follow"></span></label>
          <label class="pop-option"><input v-model="msgTypes.enter" type="checkbox">进场<span class="msg-dot dot-enter"></span></label>
        </div>
      </div>

      <div class="live-body">
        <div class="stage">
          <div id="dplayer-container" class="player"></div>
          <div v-if="anchorName || anchorAvatar" class="anchor-card">
            <img v-if="anchorAvatar" :src="anchorAvatar" class="avatar" alt="头像">
            <div v-else class="avatar placeholder"></div>
            <div class="anchor-info">
              <div class="anchor-row">
                <span class="anchor-name">{{ anchorName }}</span>
                <span v-if="fansText" class="fans">粉丝 {{ fansText }}</span>
              </div>
              <div class="anchor-desc">{{ roomDesc }}</div>
            </div>
          </div>
          <div v-if="!isLive || playerError" class="overlay">
            {{ playerError || (roomTitle || '直播间') }}
            <span v-if="!isLive" class="over-sub">该主播当前未开播或直播已结束</span>
          </div>
          <span v-if="isLive && currentQualityLabel" class="stage-q">{{ currentQualityLabel }}</span>
        </div>
        <aside class="comments">
          <div class="comments-head">实时消息<span class="badge-n">{{ totalMessages }} 条</span><button class="close-comments" type="button" title="收起评论" @click="commentsVisible = false">×</button></div>
          <div ref="msgListEl" class="comment-list" @scroll="onMsgScroll">
            <div v-for="m in messages" :key="m.id" class="msg" :class="'kind-' + m.type">
              <span class="name">{{ m.name }}：</span><span class="text">{{ m.text }}</span>
            </div>
          </div>
        </aside>
      </div>
    </section>

    <transition name="toast-fade">
      <div v-if="toastVisible" class="toast">{{ toastText }}</div>
    </transition>
  </div>
</template>

<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, reactive, ref } from 'vue'
import { invoke } from '@tauri-apps/api/tauri'
import DPlayer from 'dplayer'
import Hls from 'hls.js'
import Flv from 'flv.js'
import pako from 'pako'
import SocketCli from '@/utils/RustSocket'
import { douyin } from '@/proto/dy.js'

interface QualityOption {
    key: string
    label: string
    url: string
}

interface FeedMessage {
    id: number
    type: string
    name: string
    text: string
}

const QUALITY_LABELS: Record<string, string> = {
    FULL_HD1: '蓝光 / 超清',
    HD1: '高清',
    SD1: '标清',
    SD2: '流畅',
}
const KNOWN_QUALITY_ORDER = ['FULL_HD1', 'HD1', 'SD1', 'SD2']

const view = ref<'home' | 'live'>('home')
const roomUrl = ref('')
const loading = ref(false)
const homeError = ref('')

const roomTitle = ref('')
const anchorName = ref('')
const anchorAvatar = ref('')
const roomDesc = ref('')
const fansText = ref('')
const isLive = ref(false)
const onlineText = ref('0')
const playerError = ref('')

const qualityList = ref<QualityOption[]>([])
const currentQuality = ref('')
const currentQualityLabel = computed(() => {
    const q = qualityList.value.find((item) => item.key === currentQuality.value)
    return q ? q.label : ''
})

const commentsVisible = ref(true)
const settingsOpen = ref(false)
const msgTypes = reactive({
    chat: true,
    gift: false,
    like: false,
    follow: false,
    enter: false,
})

const messages = ref<FeedMessage[]>([])
const totalMessages = ref(0)
const msgListEl = ref<HTMLElement | null>(null)
const autoScroll = ref(true)

const toastVisible = ref(false)
const toastText = ref('')
let toastTimer: number | undefined

let player: any = null
let socketClient: any = null
let feedId = 0
let currentPayload: any = null
let liveRoomId = ''
let liveUniqueId = ''
let liveTtwid = ''

try {
    roomUrl.value = localStorage.getItem('livebox-url') || ''
} catch (e) {
    roomUrl.value = ''
}

function fmtCount(n: number): string {
    if (n >= 10000) {
        return (n / 10000).toFixed(1).replace(/\.0$/, '') + '万'
    }
    return n.toLocaleString('zh-CN')
}

function parseCountText(value: any): number | null {
    if (typeof value === 'number') return value
    if (typeof value !== 'string') return null
    const text = value.trim().replace(/,/g, '')
    if (!text) return null
    const match = /^([0-9]+(?:\.[0-9]+)?)\s*([亿万wWkK]?)/.exec(text)
    if (!match) return null
    const num = parseFloat(match[1])
    const unit = match[2]
    const mult: Record<string, number> = {
        亿: 100000000,
        万: 10000,
        w: 10000,
        W: 10000,
        千: 1000,
        k: 1000,
        K: 1000,
    }
    if (unit === '') return Math.round(num)
    return Math.round(num * (mult[unit] || 1))
}

function setOnlineNumber(value: any) {
    const num = parseCountText(value)
    if (num !== null) {
        onlineText.value = fmtCount(num)
    } else if (typeof value === 'string' && value) {
        onlineText.value = value.replace('w', '万')
    }
}

function fmtFans(value: any): string {
    const num = parseCountText(value)
    if (num === null) {
        return String(value || '').replace('w', '万')
    }
    return fmtCount(num)
}

function toast(text: string) {
    toastText.value = text
    toastVisible.value = true
    window.clearTimeout(toastTimer)
    toastTimer = window.setTimeout(() => {
        toastVisible.value = false
    }, 1500)
}

function toHttps(url: string): string {
    return String(url).replace(/^http:\/\//i, 'https://')
}

function buildQualityOptions(payload: any) {
    const stream = payload.stream_url || {}
    const flvMap: Record<string, string> = stream.flv_pull_url || {}
    const hlsMap: Record<string, string> = stream.hls_pull_url_map || {}
    let map: Record<string, string> = flvMap
    let fallback: Record<string, string> = hlsMap
    if (!Object.keys(map).length) {
        map = hlsMap
        fallback = {}
    }
    const unknown = Object.keys(map).filter((k) => KNOWN_QUALITY_ORDER.indexOf(k) < 0)
    const ordered = KNOWN_QUALITY_ORDER.filter((k) => k in map).concat(unknown)
    const options: QualityOption[] = ordered.map((key) => {
        const url = map[key] || fallback[key] || ''
        return {
            key,
            label: QUALITY_LABELS[key] || key,
            url: url ? toHttps(url) : '',
        }
    })
    qualityList.value = options
    const preferred = stream.default_resolution
    currentQuality.value = preferred && options.some((o) => o.key === preferred)
        ? preferred
        : (options[0] ? options[0].key : '')
}

function qualityUrl(key: string): string {
    const q = qualityList.value.find((item) => item.key === key)
    return q ? q.url : ''
}

function destroyPlayer() {
    try {
        if (player) {
            player.destroy()
        }
    } catch (e) {
        console.error('destroy player error', e)
    }
    player = null
}

function createPlayer(url: string) {
    destroyPlayer()
    playerError.value = ''
    const container = document.getElementById('dplayer-container') as HTMLElement
    if (!container) {
        playerError.value = '播放器初始化失败'
        return
    }
    if (!url) {
        playerError.value = '该直播间暂无可播放的视频流'
        return
    }
    const DPlayerCtor: any = DPlayer
    if (/\.m3u8/i.test(url)) {
        player = new DPlayerCtor({
            container,
            autoplay: true,
            live: true,
            lang: 'zh-cn',
            video: {
                url: '',
                type: 'customHls',
                customType: {
                    customHls: (video: HTMLVideoElement) => {
                        const hls = new Hls()
                        hls.loadSource(url)
                        hls.attachMedia(video)
                    },
                },
            },
        })
    } else if (/\.flv/i.test(url)) {
        player = new DPlayerCtor({
            container,
            autoplay: true,
            live: true,
            lang: 'zh-cn',
            video: {
                url: '',
                type: 'customFlv',
                customType: {
                    customFlv: (video: HTMLVideoElement) => {
                        const flv = Flv.createPlayer({ type: 'flv', url })
                        flv.attachMediaElement(video)
                        flv.load()
                    },
                },
            },
        })
    } else {
        player = new DPlayerCtor({
            container,
            autoplay: true,
            live: true,
            lang: 'zh-cn',
            video: { url, type: 'auto' },
        })
    }
}

async function onQualityChange() {
    await nextTick()
    createPlayer(qualityUrl(currentQuality.value))
    toast('已切换为「' + currentQualityLabel.value + '」')
}

function onMsgScroll() {
    const el = msgListEl.value
    if (!el) return
    autoScroll.value = el.scrollHeight - el.scrollTop - el.clientHeight < 48
}

function pushMessage(type: string, name: string, text: string) {
    messages.value.push({
        id: ++feedId,
        type,
        name: name || '游客',
        text,
    })
    totalMessages.value += 1
    if (messages.value.length > 300) {
        messages.value.splice(0, messages.value.length - 300)
    }
    nextTick(() => {
        if (autoScroll.value) {
            const el = msgListEl.value
            if (el) el.scrollTop = el.scrollHeight
        }
    })
}

function resetLiveState() {
    destroyPlayer()
    if (socketClient) {
        try {
            socketClient.disconnect()
        } catch (e) {
            console.error('socket disconnect error', e)
        }
        socketClient = null
    }
    messages.value = []
    totalMessages.value = 0
    qualityList.value = []
    currentQuality.value = ''
    playerError.value = ''
    settingsOpen.value = false
    commentsVisible.value = true
    isLive.value = false
    anchorName.value = ''
    anchorAvatar.value = ''
    roomDesc.value = ''
    fansText.value = ''
}

function goHome() {
    resetLiveState()
    view.value = 'home'
    loading.value = false
    homeError.value = ''
}

async function enterLive() {
    const url = roomUrl.value.trim()
    homeError.value = ''
    if (!url) {
        homeError.value = '请先粘贴抖音直播间地址'
        return
    }
    if (!/^https?:\/\//i.test(url)) {
        homeError.value = '地址看起来不像链接，请粘贴完整网址'
        return
    }
    try {
        localStorage.setItem('livebox-url', url)
    } catch (e) {
        // ignore
    }
    loading.value = true
    resetLiveState()
    try {
        const roomResult: any = await invoke('get_live_html', { url })
        let payload: any
        try {
            payload = JSON.parse(roomResult.room_info || '{}')
        } catch (e) {
            homeError.value = '直播间数据解析失败，可能已停播或链接失效'
            loading.value = false
            return
        }
        currentPayload = payload
        liveUniqueId = roomResult.unique_id || ''
        liveTtwid = roomResult.ttwid || ''
        roomTitle.value = payload.title || ''
        const owner = payload.owner || {}
        anchorName.value = owner.nickname || payload.nickname || ''
        const avatarList = (owner.avatar_thumb && owner.avatar_thumb.url_list) || (payload.avatar_thumb && payload.avatar_thumb.url_list) || []
        anchorAvatar.value = avatarList[0] || ''
        roomDesc.value = payload.title || owner.signature || ''
        const fansRaw = (owner.follow_info && owner.follow_info.follower_count_str)
            || owner.follower_count_str
            || (owner.follow_info && owner.follow_info.follower_count)
            || payload.follower_count_str
        fansText.value = fmtFans(fansRaw)
        liveRoomId = payload.id_str || payload.room_id_str || ''
        const rawStatus = Number(payload.status)
        isLive.value = rawStatus === 2 && !!liveRoomId
        const viewStats = payload.room_view_stats || {}
        if (typeof viewStats.display_value === 'number' && viewStats.display_value > 0) {
            setOnlineNumber(viewStats.display_value)
        } else {
            setOnlineNumber(payload.user_count_str || '0')
        }
        if (!isLive.value) {
            playerError.value = '该主播当前未开播或直播已结束'
            view.value = 'live'
            loading.value = false
            return
        }
        buildQualityOptions(payload)
        view.value = 'live'
        await nextTick()
        await nextTick()
        createPlayer(qualityUrl(currentQuality.value))
        connectSocket()
        loading.value = false
    } catch (e) {
        console.error('enter live error', e)
        homeError.value = '获取直播间失败，请检查地址是否正确，稍后重试'
        loading.value = false
        resetLiveState()
        view.value = 'home'
    }
}

function connectSocket() {
    if (!liveRoomId) return
    try {
        const sign = window.creatSignature(liveRoomId, liveUniqueId)
        const url = 'wss://webcast5-ws-web-lf.douyin.com/webcast/im/push/v2/'
            + '?room_id=' + encodeURIComponent(liveRoomId)
            + '&compress=gzip&version_code=180800&webcast_sdk_version=1.0.14-beta.0'
            + '&live_id=1&did_rule=3&user_unique_id=' + encodeURIComponent(liveUniqueId)
            + '&identity=audience&signature=' + encodeURIComponent(sign)
            + '&aid=6383&device_platform=web&browser_language=zh-CN&browser_platform=Win32'
            + '&browser_name=Mozilla&browser_version=5.0+%28Windows+NT+10.0%3B+Win64%3B+x64%29+AppleWebKit%2F537.36+%28KHTML%2C+like+Gecko%29+Chrome%2F126.0.0.0+Safari%2F537.36+Edg%2F126.0.0.0'
        const options: any = {
            headers: {
                cookie: 'ttwid=' + liveTtwid,
                'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36 Edg/126.0.0.0',
            },
        }
        const dy: any = douyin
        const pingMsg = dy.PushFrame.encode({ payloadType: 'hb' }).finish()
        socketClient = new SocketCli(url, options, onSocketMessage, pingMsg)
    } catch (e) {
        console.error('socket connect error', e)
        playerError.value = '实时消息连接失败'
    }
}

function onSocketMessage(msg: any) {
    const dy: any = douyin
    try {
        const frame = dy.PushFrame.decode(msg.data)
        if (frame.needAck) {
            const ack = dy.PushFrame.encode({
                payloadType: 'ack',
                logId: frame.logId,
            }).finish()
            if (socketClient) {
                socketClient.send(ack)
            }
        }
        if (!frame.payload || !frame.payload.length) return
        const gzipData = pako.inflate(frame.payload)
        const response = dy.Response.decode(gzipData)
        handleMessages(response.messagesList || [])
    } catch (e) {
        console.error('socket decode error', e)
    }
}

function handleMessages(list: any[]) {
    for (const msg of list) {
        try {
            switch (msg.method) {
                case 'WebcastRoomUserSeqMessage':
                    handleRoomStats(msg.payload)
                    break
                case 'WebcastChatMessage':
                    if (msgTypes.chat) handleChat(msg.payload)
                    break
                case 'WebcastGiftMessage':
                    if (msgTypes.gift) handleGift(msg.payload)
                    break
                case 'WebcastLikeMessage':
                    if (msgTypes.like) handleLike(msg.payload)
                    break
                case 'WebcastSocialMessage':
                    if (msgTypes.follow) handleFollow(msg.payload)
                    break
                case 'WebcastMemberMessage':
                    if (msgTypes.enter) handleEnter(msg.payload)
                    break
                default:
                    break
            }
        } catch (e) {
            console.error('handle message error', msg.method, e)
        }
    }
}

function handleRoomStats(data: any) {
    const dy: any = douyin
    const stats = dy.RoomUserSeqMessage.decode(data)
    if (stats.onlineUserForAnchor !== undefined && stats.onlineUserForAnchor !== '' && stats.onlineUserForAnchor !== 0) {
        setOnlineNumber(stats.onlineUserForAnchor)
    }
}

function handleChat(data: any) {
    const dy: any = douyin
    const chat = dy.ChatMessage.decode(data)
    pushMessage('chat', chat.user.nickName, chat.content || '')
}

function handleGift(data: any) {
    const dy: any = douyin
    const gift = dy.GiftMessage.decode(data)
    const name = gift.gift ? gift.gift.name : '礼物'
    const count = gift.repeatCount || 1
    pushMessage('gift', gift.user.nickName, '送出' + name + ' ×' + count)
}

function handleLike(data: any) {
    const dy: any = douyin
    const like = dy.LikeMessage.decode(data)
    pushMessage('like', like.user.nickName, '为主播点赞了')
}

function handleFollow(data: any) {
    const dy: any = douyin
    const social = dy.SocialMessage.decode(data)
    pushMessage('follow', social.user.nickName, '关注了主播')
    if (social.followCount) {
        fansText.value = fmtFans(social.followCount)
    }
}

function handleEnter(data: any) {
    const dy: any = douyin
    const member = dy.MemberMessage.decode(data)
    pushMessage('enter', member.user.nickName, '来了')
}

function onDocumentClick(event: MouseEvent) {
    const target = event.target as HTMLElement
    if (settingsOpen.value && !target.closest('.settings-pop') && !target.closest('.icon-btn')) {
        settingsOpen.value = false
    }
}

onMounted(() => {
    document.addEventListener('click', onDocumentClick)
})

onBeforeUnmount(() => {
    document.removeEventListener('click', onDocumentClick)
    resetLiveState()
})
</script>

<style scoped>
    .app-root { width: 100%; height: 100%; }
    .home-view {
        width: 100%;
        height: 100%;
        display: flex;
        align-items: center;
        justify-content: center;
        background:
            radial-gradient(1000px 500px at 20% -10%, rgba(255, 44, 85, .10), transparent 60%),
            radial-gradient(800px 400px at 90% 110%, rgba(99, 226, 255, .06), transparent 60%),
            #0e1013;
    }
    .home-inner { width: min(620px, 92vw); text-align: center; }
    .brand {
        display: inline-flex;
        align-items: center;
        justify-content: center;
        width: 72px;
        height: 72px;
        border-radius: 22px;
        margin-bottom: 22px;
        background: linear-gradient(145deg, #ff2c55, #c81e40);
        box-shadow: 0 16px 40px rgba(255, 44, 85, .25);
    }
    .brand::after {
        content: "";
        border-left: 22px solid #fff;
        border-top: 13px solid transparent;
        border-bottom: 13px solid transparent;
        margin-left: 5px;
    }
    .home-inner h1 { font-size: 32px; }
    .enter-box {
        display: flex;
        gap: 10px;
        margin-top: 10px;
        padding: 8px;
        background: #161a20;
        border: 1px solid #262c36;
        border-radius: 16px;
    }
    .enter-box:focus-within {
        border-color: rgba(255, 44, 85, .55);
        box-shadow: 0 0 0 3px rgba(255, 44, 85, .12);
    }
    .enter-box input {
        flex: 1;
        min-width: 0;
        background: transparent;
        border: 0;
        outline: none;
        padding: 12px 10px 12px 14px;
        font-size: 16px;
        color: #eef1f6;
    }
    .enter-box input::placeholder { color: #5d6673; }
    .btn-primary {
        background: #ff2c55;
        color: #fff;
        font-weight: 600;
        padding: 12px 26px;
        border-radius: 10px;
        white-space: nowrap;
    }
    .btn-primary:hover { filter: brightness(1.1); }
    .btn-primary:disabled { opacity: .6; cursor: default; }
    .error {
        margin-top: 16px;
        color: #ffb3bf;
        background: rgba(255, 44, 85, .08);
        padding: 10px 14px;
        border-radius: 10px;
    }

    .live-view {
        position: relative;
        width: 100%;
        height: 100%;
        display: flex;
        flex-direction: column;
        background: #0e1013;
    }
    .topbar {
        position: relative;
        z-index: 40;
        height: 58px;
        flex: none;
        display: flex;
        align-items: center;
        gap: 12px;
        padding: 0 16px;
        border-bottom: 1px solid #262c36;
        background: #0e1013;
    }
    .icon-btn {
        width: 38px;
        height: 38px;
        flex: none;
        display: inline-flex;
        align-items: center;
        justify-content: center;
        border-radius: 10px;
        background: #161a20;
        color: #eef1f6;
        font-size: 18px;
    }
    .icon-btn:hover { background: #1d222b; }
    .live-pill {
        margin-left: auto;
        display: inline-flex;
        align-items: center;
        gap: 7px;
        background: rgba(255, 44, 85, .12);
        color: #ff8099;
        padding: 7px 13px;
        border-radius: 999px;
        font-size: 13px;
        font-weight: 600;
        white-space: nowrap;
    }
    .live-pill.off { background: rgba(154, 164, 178, .12); color: #9aa4b2; }
    .dot { width: 8px; height: 8px; border-radius: 50%; background: #ff2c55; animation: pulse 1.6s infinite; }
    @keyframes pulse { 50% { opacity: .35; } }
    .count {
        display: inline-flex;
        align-items: center;
        gap: 6px;
        background: #161a20;
        border: 1px solid #262c36;
        padding: 7px 13px;
        border-radius: 999px;
        font-size: 13px;
        white-space: nowrap;
    }
    .count b { font-variant-numeric: tabular-nums; }
    .text-btn {
        background: #161a20;
        border: 1px solid #262c36;
        padding: 8px 13px;
        border-radius: 10px;
        font-size: 13px;
        white-space: nowrap;
    }
    .text-btn:hover { background: #1d222b; }
    .quality-select {
        background: #161a20;
        border: 1px solid #262c36;
        color: #eef1f6;
        padding: 8px 12px;
        border-radius: 10px;
        font-size: 13px;
        outline: none;
    }
    .settings-pop {
        position: absolute;
        top: 52px;
        right: 14px;
        width: 250px;
        background: #1d222b;
        border: 1px solid #262c36;
        border-radius: 14px;
        box-shadow: 0 18px 46px rgba(0, 0, 0, .5);
        padding: 16px;
    }
    .pop-title { font-size: 14px; font-weight: 700; margin-bottom: 12px; }
    .pop-group > span { display: block; color: #9aa4b2; font-size: 12px; margin-bottom: 8px; }
    .pop-option {
        display: flex;
        align-items: center;
        gap: 9px;
        padding: 7px 0;
        font-size: 13.5px;
        cursor: pointer;
    }
    .pop-option input { accent-color: #ff2c55; width: 15px; height: 15px; }
    .msg-dot { width: 7px; height: 7px; border-radius: 50%; margin-left: auto; }
    .dot-chat { background: #63e2ff; }
    .dot-gift { background: #ffc53d; }
    .dot-like { background: #ff2c55; }
    .dot-follow { background: #2ee59d; }
    .dot-enter { background: #a78bfa; }

    .live-body {
        flex: 1;
        min-height: 0;
        display: flex;
        gap: 12px;
        padding: 12px 14px 14px;
    }
    .stage {
        position: relative;
        flex: 1;
        min-width: 0;
        border-radius: 14px;
        overflow: hidden;
        background: #000;
        border: 1px solid #262c36;
    }
    .player {
        position: absolute;
        inset: 0;
        width: 100%;
        height: 100%;
    }
    .anchor-card {
        position: absolute;
        top: 14px;
        left: 14px;
        z-index: 7;
        display: flex;
        align-items: center;
        gap: 11px;
        max-width: min(62%, 460px);
        padding: 8px 14px 8px 8px;
        border-radius: 14px;
        background: linear-gradient(90deg, rgba(0, 0, 0, .62), rgba(0, 0, 0, .18));
        backdrop-filter: blur(4px);
        pointer-events: none;
    }
    .avatar {
        width: 46px;
        height: 46px;
        flex: none;
        border-radius: 50%;
        object-fit: cover;
        border: 2px solid rgba(255, 255, 255, .28);
    }
    .avatar.placeholder { background: rgba(255, 255, 255, .12); }
    .anchor-info { min-width: 0; }
    .anchor-row { display: flex; align-items: center; gap: 9px; }
    .anchor-name {
        font-size: 15px;
        font-weight: 700;
        color: #fff;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
    .fans {
        flex: none;
        background: rgba(0, 0, 0, .45);
        color: #ffd9df;
        font-size: 11px;
        padding: 3px 8px;
        border-radius: 999px;
        white-space: nowrap;
    }
    .anchor-desc {
        margin-top: 4px;
        font-size: 12px;
        color: rgba(255, 255, 255, .82);
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
    .overlay {
        position: absolute;
        inset: 0;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        gap: 10px;
        background: rgba(10, 12, 15, .86);
        color: #eef1f6;
        font-size: 18px;
        font-weight: 600;
        z-index: 5;
    }
    .over-sub { font-size: 13px; color: #9aa4b2; font-weight: 400; }
    .stage-q {
        position: absolute;
        right: 12px;
        top: 12px;
        z-index: 6;
        background: rgba(0, 0, 0, .55);
        color: #fff;
        font-size: 12px;
        padding: 5px 10px;
        border-radius: 8px;
        pointer-events: none;
    }

    .comments {
        width: 340px;
        min-width: 0;
        flex: none;
        display: flex;
        flex-direction: column;
        background: #161a20;
        border: 1px solid #262c36;
        border-radius: 14px;
        overflow: hidden;
        transition: width .28s ease, opacity .22s ease, border-color .28s ease;
    }
    .live-view.hide-comments .comments { width: 0; opacity: 0; border-color: transparent; }
    .comments-head {
        flex: none;
        display: flex;
        align-items: center;
        gap: 8px;
        padding: 12px 14px;
        border-bottom: 1px solid #262c36;
        font-size: 14px;
        font-weight: 600;
        white-space: nowrap;
    }
    .comments-head::before {
        content: "";
        width: 8px;
        height: 8px;
        border-radius: 3px;
        background: #63e2ff;
    }
    .badge-n { color: #9aa4b2; font-size: 12px; font-weight: 400; }
    .close-comments { margin-left: auto; color: #9aa4b2; font-size: 18px; line-height: 1; }
    .close-comments:hover { color: #eef1f6; }
    .comment-list {
        flex: 1;
        min-height: 0;
        overflow-y: auto;
        padding: 10px 12px;
    }
    .comment-list::-webkit-scrollbar { width: 8px; }
    .comment-list::-webkit-scrollbar-thumb { background: #2b323d; border-radius: 8px; }
    .msg { margin-bottom: 13px; font-size: 13.5px; line-height: 1.5; word-break: break-word; }
    .msg .name { color: #63e2ff; font-weight: 600; margin-right: 5px; }
    .msg .text { color: #d5dae3; }
    .msg.kind-gift .text { color: #ffcf5c; }
    .msg.kind-follow .text { color: #6ef0b8; }
    .msg.kind-like .text { color: #ff9cb0; }
    .msg.kind-enter .text { color: #b7a6ff; }

    .toast {
        position: fixed;
        left: 50%;
        bottom: 32px;
        transform: translateX(-50%);
        background: #1d222b;
        border: 1px solid #262c36;
        color: #eef1f6;
        padding: 10px 18px;
        border-radius: 999px;
        font-size: 13px;
        z-index: 60;
        box-shadow: 0 12px 34px rgba(0, 0, 0, .4);
    }
    .toast-fade-enter-active, .toast-fade-leave-active { transition: opacity .18s; }
    .toast-fade-enter-from, .toast-fade-leave-to { opacity: 0; }

    @media (max-width: 960px) {
        .count { padding: 7px 10px; font-size: 12px; }
        .comments { width: 290px; }
    }
</style>
