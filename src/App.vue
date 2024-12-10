<script setup>
import {computed, onMounted, provide, ref} from "vue";
import moment from 'moment'
import CPU from "@/components/CPU.vue";
import Mem from "@/components/Mem.vue";
import NetIn from "@/components/NetIn.vue";
import NetOut from "@/components/NetOut.vue";
import axios from "axios";
import { Message } from "@arco-design/web-vue";
import StatsCard from "@/components/StatsCard.vue";
import {formatBytes, formatTimeStamp, formatUptime, calculateRemainingDays} from '@/utils/utils'

const socketURL = ref('')
const apiURL = ref('')

const theme = window.localStorage.getItem('theme') || 'light'
const dark = ref(theme !== 'light')

const handleChangeDark = () => {
  dark.value = !dark.value

  if (dark.value) {
    window.localStorage.setItem('theme','dark')
    document.body.setAttribute('arco-theme', 'dark')
  } else {
    // 恢复亮色主题
    window.localStorage.setItem('theme','light')
    document.body.removeAttribute('arco-theme');
   }
}

const area = ref([])
const selectArea = ref('all')

const type = ref('all')

const data = ref([])

const selectHost = ref('')

const charts = ref({})

const cpuRef = ref(null)
const memRef = ref(null)
const netInRef = ref(null)
const netOutRef = ref(null)

const host = computed(() => {
  if (selectArea.value === 'all') {
    return data.value
  }

  return data.value.filter(item => item.Host.Name.slice(0, 2) === selectArea.value)
})

const hosts = computed(() => {
  if (type.value === 'all') {
    return host.value
  } else if (type.value === 'online') {
    return host.value.filter(item => item.status)
  } else {
    return host.value.filter(item => !item.status)
  }
})

const stats = computed(() => {
  const online = host.value.filter(item => item.status)
  let bandwidth_up = 0
  let bandwidth_down = 0
  let traffic_up = 0
  let traffic_down = 0

  host.value.forEach((item) => {
    bandwidth_up += item.State.NetOutSpeed
    bandwidth_down += item.State.NetInSpeed
    traffic_up += item.State.NetOutTransfer
    traffic_down += item.State.NetInTransfer
  })

  return {
    total: host.value.length,
    online: online.length,
    offline: host.value.length - online.length,
    bandwidth_up: bandwidth_up,
    bandwidth_down: bandwidth_down,
    traffic_up: traffic_up,
    traffic_down: traffic_down
  }
})

let socket = null

let nowtime = (Math.floor(Date.now() / 1000))

const fetchConfig = async () => {
  try {
    const res = await axios.get('/config.json')
    socketURL.value = res.data.socket
    apiURL.value = res.data.apiURL
  } catch (e) {
    Message.error('获取配置失败')
  }
}

const initScoket = async () => {
  await fetchConfig()

  socket = new WebSocket(socketURL.value);  // 替换为实际的 WebSocket 服务器 URL

  socket.onmessage = function(event) {
    try {
      const message = event.data;
      const res = JSON.parse(message.replace('data: ', ''))
      if (res != null) {
        area.value = Array.from(new Set(res.map(item => item.Host.Name.slice(0, 2))))
      }
      data.value = res.map((host) => {
        if (!charts.value[host.Host.Name]) {
          charts.value[host.Host.Name] = {
            cpu: [],
            mem: [],
            net_in: [],
            net_out: []
          }
        }

        if (charts.value[host.Host.Name].cpu.length == 2) {
          charts.value[host.Host.Name].cpu = charts.value[host.Host.Name].cpu.slice(1)
          charts.value[host.Host.Name].mem = charts.value[host.Host.Name].mem.slice(1)
          charts.value[host.Host.Name].net_in = charts.value[host.Host.Name].net_in.slice(1)
          charts.value[host.Host.Name].net_out = charts.value[host.Host.Name].net_out.slice(1)
        }

        charts.value[host.Host.Name].cpu.push([host.TimeStamp * 1000, host.State.CPU])
        charts.value[host.Host.Name].mem.push([host.TimeStamp * 1000, host.State.MemUsed])
        charts.value[host.Host.Name].net_in.push([host.TimeStamp * 1000, host.State.NetOutSpeed])
        charts.value[host.Host.Name].net_out.push([host.TimeStamp * 1000, host.State.NetInSpeed])

        return {
          ...host,
          status: (nowtime - host.TimeStamp <= 10) ? 1 : 0
        }
      })

      setTimeout(() => sendPing(), 1000)

    } catch (error) {
      console.error('解析 WebSocket 消息时出错:', error);
    }
  };

  socket.onopen = function () {
    sendPing()
  }

  socket.onclose = function () {
    Message.warning('WebSocket已断连，正在重连中...')

    initScoket()
  }
}

const sendPing = () => {
  nowtime = (Math.floor(Date.now() / 1000))
  socket.send('ping')
}

onMounted(async() => {
  if (dark.value) {
    document.body.setAttribute('arco-theme', 'dark')
  }

  await initScoket()
  handleFetchHostInfo()
})

const progressStatus = (value) => {
  if (value < 80) {
    return 'success';
  } else if (value < 90) {
    return 'warning';
  } else {
    return 'danger';
  }
}

const handleSelectArea = (area) => {
  selectArea.value = area
}

const handleSelectHost = (host) => {
  if (selectHost.value === host) {
    selectHost.value = ''
    return
  }

  selectHost.value = host
}

// 删除主机
const deleteVisible = ref(false)
const authSecret = ref('')
const deleteHostName = ref('')

const handleShowDelete = (name) => {
  authSecret.value =  window.localStorage.getItem('auth_secret') || ''
  deleteHostName.value = name
  deleteVisible.value = true
}

const handleDeleteHost = async () => {
  try {
    await axios.post(apiURL.value + '/delete', {
      "auth_secret": authSecret.value,
      "name": deleteHostName.value
    })

    window.localStorage.setItem('auth_secret', authSecret.value)

    Message.success('删除成功')

    deleteVisible.value = false
  } catch (e) {
    Message.error('删除失败，管理密钥错误')
  }
}

const handleClose = () => {
  deleteVisible.value = false
}

const hostInfo = ref({})

const handleFetchHostInfo = async () => {
  try {
    const res = await axios.get(apiURL.value + '/info')
    res.data.forEach((item) => {
      hostInfo.value[item.name] = item
    })
  } catch (e) {
    // Message.error('删除失败，管理密钥错误')
  }
}

const handleChangeType = (value) => {
  type.value = value
}

const editHostName = ref('')
const duetime = ref(0)
const buy_url = ref('')
const seller = ref('')
const price = ref('')

const editVisible = ref(false)

const handleShowEdit = (name) => {
  if (!hostInfo.value[name]) {
    editHostName.value = name
    duetime.value = 0
    buy_url.value = ''
    seller.value = ''
    price.value = ''
    editVisible.value = true
    authSecret.value = window.localStorage.getItem('auth_secret') || ''
    return
  }

  editHostName.value = name
  duetime.value = hostInfo.value[name].due_time
  buy_url.value = hostInfo.value[name].buy_url
  seller.value = hostInfo.value[name].seller
  price.value = hostInfo.value[name].price
  editVisible.value = true
  authSecret.value = window.localStorage.getItem('auth_secret') || ''

}

const handleEditHost = async () => {
  try {
    await axios.post(apiURL.value + '/info', {
      "name": editHostName.value,
      "due_time": new Date(duetime.value).getTime(),
      "buy_url": buy_url.value,
      "seller": seller.value,
      "auth_secret": authSecret.value,
      "price": price.value
    })

    window.localStorage.setItem('auth_secret', authSecret.value)

    Message.success('更新成功')

    handleFetchHostInfo()

    editVisible.value = false
  } catch (e) {
    Message.error('更新失败，管理密钥错误')
  }

}

const handleEditClose = () => {
  editVisible.value = false
}

provide('handleChangeType', handleChangeType)

</script>

<template>
  <div class="max-container">
    <div class="header">
      <a class="logo" href="#">
        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 725 359.04"><defs><style>.cls-1{fill:none;stroke:#000;stroke-miterlimit:5.33;stroke-width:7px;}.cls-2{font-size:140px;font-family:MiSans-Normal, MiSans;}</style></defs><g id="图层_1" data-name="图层 1"><path d="M123.41,223V333.27a7.11,7.11,0,0,0,7.12,7.11h60.35v19.77h-73a16.06,16.06,0,0,1-16.07-16.07V223Z" transform="translate(-53 -223)"/><path d="M328.34,355a41.47,41.47,0,0,1-15-17.68Q308,325.87,308,311t5.35-26.29a41.55,41.55,0,0,1,15-17.69,42.62,42.62,0,0,1,44.44,0,41.62,41.62,0,0,1,15,17.69q5.34,11.42,5.35,26.29t-5.35,26.3a41.53,41.53,0,0,1-15,17.68,42.56,42.56,0,0,1-44.44,0Zm39.09-8.25a33.18,33.18,0,0,0,11.42-14.33,58.16,58.16,0,0,0,0-42.8,33.21,33.21,0,0,0-11.42-14.32,29.07,29.07,0,0,0-17-5.08,28.65,28.65,0,0,0-16.87,5.08,33.48,33.48,0,0,0-11.33,14.32,58.16,58.16,0,0,0,0,42.8,33.45,33.45,0,0,0,11.33,14.33,28.65,28.65,0,0,0,16.87,5.08A29.07,29.07,0,0,0,367.43,346.73Z" transform="translate(-53 -223)"/><path d="M431.81,355.08a41.52,41.52,0,0,1-15.24-17.78q-5.34-11.43-5.35-26.3,0-14.69,5.35-26a40.35,40.35,0,0,1,15.24-17.5q9.88-6.17,23.12-6.17a40.72,40.72,0,0,1,34.64,19.95l-7.8,5.44a32,32,0,0,0-11.7-11.7,30.82,30.82,0,0,0-15.68-4.26,29.83,29.83,0,0,0-17.23,5.08,33.62,33.62,0,0,0-11.61,14.24A50,50,0,0,0,421.38,311a52.74,52.74,0,0,0,4.08,21.31A33.41,33.41,0,0,0,437,346.82,29.39,29.39,0,0,0,454.21,352a31,31,0,0,0,15.86-4.26,31.87,31.87,0,0,0,11.7-11.7l7.8,5.44a38.23,38.23,0,0,1-14.69,14.6,39.8,39.8,0,0,1-19.95,5.35Q441.69,361.42,431.81,355.08Z" transform="translate(-53 -223)"/><path d="M531.75,351.89l-34-88.58H509l30.65,82.33h.91l31.91-82.33H583.7L530,396.43H519l12.66-31A18.41,18.41,0,0,0,531.75,351.89Z" transform="translate(-53 -223)"/><path d="M122.75,538.15h-21V410.24a9.2,9.2,0,0,1,9.2-9.2H122l67.47,98.3h.91V401h21V529a9.18,9.18,0,0,1-9.19,9.19H191.85l-68-98.29h-1.09Z" transform="translate(-53 -223)"/><path d="M316.08,491.18H244.26q.54,17.42,9.79,27.93T278,529.63a33.93,33.93,0,0,0,16.51-4.26,31.09,31.09,0,0,0,12.15-11.7l7.8,5.26a38,38,0,0,1-14.88,14.6,46.48,46.48,0,0,1-44.79-.82,40.82,40.82,0,0,1-15.42-17.41q-5.45-11.23-5.44-26.11,0-15.06,5.26-26.48A41,41,0,0,1,254.05,445q9.61-6.25,22.49-6.25a40.4,40.4,0,0,1,21.31,5.53A37.27,37.27,0,0,1,312,459.62a50.79,50.79,0,0,1,4.08,31.56Zm-12.31-9.07a2.77,2.77,0,0,0,2.8-2.88q-.7-13.26-8-21.78-8.07-9.43-22-9.43-13.25,0-21.76,9-7.68,8.08-9.76,22a2.78,2.78,0,0,0,2.78,3.15Z" transform="translate(-53 -223)"/><path d="M383,450.74H354.71v59.48q0,9.25,4.35,14.33t11.79,5.08a28.15,28.15,0,0,0,10-2.18v9.43a32.9,32.9,0,0,1-12,2.18q-11.42,0-17.86-7.25t-6.44-20.68V450.74H329.5v-9.43h12.2a2.86,2.86,0,0,0,2.85-2.86V413h10.16v25.44a2.85,2.85,0,0,0,2.85,2.86H383Z" transform="translate(-53 -223)"/><path d="M638.9,536.7V441.31h10v21.22h.73a48,48,0,0,1,15.5-17,39.6,39.6,0,0,1,22.77-6.71h1.63v10.88h-2.36a39.91,39.91,0,0,0-19.95,4.9,37.6,37.6,0,0,0-13.51,12.33q-4.8,7.44-4.81,14.69V536.7Z" transform="translate(-53 -223)"/><path d="M778,536.7H765.3l-38-55.25a2.32,2.32,0,0,0-3.6-.28l-7.35,7.73a4.46,4.46,0,0,0-1.23,3.07V536.7h-10V403.22h10v68a2.32,2.32,0,0,0,4,1.61l42.43-44.09h13.42L735.28,469.2a4.45,4.45,0,0,0-.46,5.66Z" transform="translate(-53 -223)"/><path class="cls-1" d="M613.54,493.53a41.46,41.46,0,1,1-41.46-41.46A41.45,41.45,0,0,1,613.54,493.53Z" transform="translate(-53 -223)"/><path class="cls-1" d="M602.39,463.42c10.42-1.92,18.09-1.09,20.63,3,5.24,8.37-13.32,27.31-41.45,42.3s-55.18,20.35-60.42,12c-2.77-4.42,1.09-11.76,9.48-19.93" transform="translate(-53 -223)"/><rect width="8" height="312"/></g><g id="图层_2" data-name="图层 2"><text class="cls-2" transform="translate(339.77 319.56)">W</text><text class="cls-2" transform="translate(145.06 138.56)">U</text></g></svg>
        <span>LuocyNetwork</span>
        <small style="font-weight: 400;opacity: .8"> ｜ 节点监控</small>
      </a>
      <a-button class="theme-btn" :shape="'round'" @click="handleChangeDark">
        <template #icon>
          <icon-sun-fill v-if="!dark" />
          <icon-moon-fill v-else />
        </template>
      </a-button>
    </div>
    <div class="area-tabs">
      <div class="area-tab-item" :class="selectArea === 'all' ? 'is-active' : ''" @click="handleSelectArea('all')">
        全部地区
      </div>
      <div class="area-tab-item" :class="selectArea === item ? 'is-active' : ''" v-for="(item, index) in area" :key="item" @click="handleSelectArea(item)">
        <span :class="`flag-icon flag-icon-${item.replace('UK', 'GB').toLowerCase()}`" style="margin-right: 3px;"></span> {{item}}
      </div>
    </div>
    <StatsCard :type="type" :stats="stats" @handleChangeType="handleChangeType" />
    <div class="monitor-card">
      <div class="monitor-item" :class="selectHost === item.Host.Name ? 'is-active' : ''" v-for="(item, index) in hosts" @click="handleSelectHost(item.Host.Name)" :key="index">
        <div class="name">
          <div class="title">
            <span :class="`flag-icon flag-icon-${item.Host.Name.slice(0, 2).replace('UK', 'GB').toLowerCase()}`"></span>
            {{item.Host.Name}}
          </div>
          <div class="status" :class="item.status ? 'online' : 'offline'">
            <span>{{item.status  ? '在线' : '离线'}}</span>
            <span style="margin-left: 6px;">{{formatUptime(item.State.Uptime)}}</span>
          </div>
        </div>
        <div class="platform">
          <div class="monitor-item-title">系统</div>
          <div class="monitor-item-value">{{item.Host.Platform}} {{item.Host.PlatformVersion}}</div>
        </div>
        <div class="cpu">
          <div class="monitor-item-title">CPU</div>
          <div class="monitor-item-value">{{item.State.CPU.toFixed(2) + '%'}}</div>
          <a-progress class="monitor-item-progress" :status="progressStatus(item.State.CPU)" :percent="item.State.CPU/100" :show-text="false" style="width: 60px" />
        </div>
        <div class="mem">
          <div class="monitor-item-title">内存使用情况</div>
          <div class="monitor-item-value">{{(item.State.MemUsed / item.Host.MemTotal * 100).toFixed(2) + '%'}}</div>
          <a-progress class="monitor-item-progress" :status="progressStatus(item.State.MemUsed / item.Host.MemTotal * 100)" :percent="item.State.MemUsed / item.Host.MemTotal" :show-text="false" style="width: 60px" />
        </div>
        <div class="network">
          <div class="monitor-item-title">网络速度（IN|OUT）</div>
          <div class="monitor-item-value">{{`${formatBytes(item.State.NetInSpeed)}/s | ${formatBytes(item.State.NetOutSpeed)}/s`}}</div>
        </div>
        <div class="average">
          <div class="monitor-item-title">负载平均值(1|5|15)</div>
          <div class="monitor-item-value">{{`${item.State.Load1} | ${item.State.Load5} | ${item.State.Load15}`}}</div>
        </div>
        <div class="uptime" style="width: 120px;">
          <div class="monitor-item-title">剩余时间</div>
          <div class="monitor-item-value">{{hostInfo[item.Host.Name] ? calculateRemainingDays(hostInfo[item.Host.Name].due_time) : '-'}}</div>
        </div>
        <div class="uptime">
          <div class="monitor-item-title">上报时间</div>
          <div class="monitor-item-value">{{formatTimeStamp(item.TimeStamp)}}</div>
        </div>
        <div class="detail" v-if="selectHost === item.Host.Name">
          <a-row>
            <a-col :span="10" :xs="24" :sm="24" :md="10" :lg="10" :sl="10">
              <div class="detail-item-list">
                <div class="detail-item">
                  <div class="name">主机名</div>
                  <div class="value">{{item.Host.Name}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">地区</div>
                  <div class="value">
                    <span :class="`flag-icon flag-icon-${item.Host.Name.slice(0, 2).replace('UK', 'GB').toLowerCase()}`"></span>
                    {{item.Host.Name.slice(0, 2).toUpperCase()}}
                  </div>
                </div>
                <div class="detail-item">
                  <div class="name">系统</div>
                  <div class="value">{{item.Host.Platform}} {{item.Host.PlatformVersion}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">架构</div>
                  <div class="value">{{item.Host.Arch}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">虚拟化</div>
                  <div class="value">{{item.Host.Virtualization || '-'}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">CPU</div>
                  <div class="value">{{item.Host.CPU.join(',')}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">CPU占用</div>
                  <div class="value">{{item.State.CPU.toFixed(2) + '%'}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">内存使用情况</div>
                  <div class="value">{{(item.State.MemUsed / item.Host.MemTotal * 100).toFixed(2) + '%'}} ({{formatBytes(item.State.MemUsed)}} / {{formatBytes(item.Host.MemTotal)}})</div>
                </div>
                <div class="detail-item">
                  <div class="name">虚拟内存(Swap)</div>
                  <div class="value">{{formatBytes(item.State.SwapUsed)}} / {{formatBytes(item.Host.SwapTotal)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">网络速度（IN|OUT）</div>
                  <div class="value">{{`${formatBytes(item.State.NetInSpeed)}/s | ${formatBytes(item.State.NetOutSpeed)}/s`}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">负载平均值(1|5|15)</div>
                  <div class="value">{{`${item.State.Load1} | ${item.State.Load5} | ${item.State.Load15}`}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">流量使用↑|↓</div>
                  <div class="value">{{formatBytes(item.State.NetOutTransfer)}} | {{formatBytes(item.State.NetInTransfer)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">开机时间</div>
                  <div class="value">{{formatTimeStamp(item.Host.BootTime)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">上报时间</div>
                  <div class="value">{{formatTimeStamp(item.TimeStamp)}}</div>
                </div>
                <div class="detail-item" v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].seller">
                  <div class="name">商家名称</div>
                  <div class="value">{{hostInfo[item.Host.Name].seller}}</div>
                </div>
                <div class="detail-item" v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].price">
                  <div class="name">主机价格</div>
                  <div class="value">{{hostInfo[item.Host.Name].price}}</div>
                </div>
                <div class="detail-item" v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].due_time">
                  <div class="name">到期时间</div>
                  <div class="value">{{moment(hostInfo[item.Host.Name].due_time).format('YYYY-MM-DD')}}</div>
                </div>
                <div class="detail-item" v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].buy_url">
                  <div class="name">购买链接</div>
                  <div class="value">
                    <a style="color: #0077ff" :href="hostInfo[item.Host.Name].buy_url" target="_blank" @click.stop="() => {}">{{hostInfo[item.Host.Name].buy_url}}</a>
                  </div>
                </div>
              </div>
            </a-col>
            <a-col :span="14" :xs="24" :sm="24" :md="14" :lg="14" :sl="14">
              <a-row :gutter="20">
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <CPU ref="cpuRef" style="margin-bottom: 20px;" :data="charts[item.Host.Name].cpu" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <Mem ref="memRef" :max="item.Host.MemTotal" style="margin-bottom: 20px;" :data="charts[item.Host.Name].mem" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <NetIn ref="netInRef" :data="charts[item.Host.Name].net_in" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <NetOut ref="netOutRef" :data="charts[item.Host.Name].net_out" />
                </a-col>
              </a-row>
            </a-col>
          </a-row>
        </div>
        <div class="edit-btn" @click.stop="handleShowEdit(item.Host.Name)">
          <icon-edit />
        </div>
        <div class="delete-btn" @click.stop="handleShowDelete(item.Host.Name)">
          <icon-delete />
        </div>
      </div>
    </div>
    <a-modal v-model:visible="deleteVisible" :footer="false" :hide-title="true" width="360px">
      <div class="akile-modal-title">
        <span>删除主机</span>
        <a-button @click="handleClose">
          <template #icon>
            <icon-close/>
          </template>
        </a-button>
      </div>
      <div class="akile-modal-content">
        <a-input-password v-model="authSecret" placeholder="请输入管理密钥"></a-input-password>
        <div class="tips">提示：删除后无法恢复，请确定后再删除操作</div>
      </div>
      <div class="akile-modal-action">
        <a-button type="primary" status="danger" :long="true" @click="handleDeleteHost">确认删除</a-button>
      </div>
    </a-modal>
    <a-modal v-model:visible="editVisible" :footer="false" :hide-title="true" width="360px">
      <div class="akile-modal-title">
        <span>编辑主机信息</span>
        <a-button @click="handleEditClose">
          <template #icon>
            <icon-close/>
          </template>
        </a-button>
      </div>
      <div class="akile-modal-content">
        <a-date-picker v-model="duetime" placeholder="到期时间" style="margin-bottom: 10px;width: 100%;"></a-date-picker>
        <a-input v-model="seller" placeholder="卖家" style="margin-bottom: 10px;"></a-input>
        <a-input v-model="price" placeholder="价格" style="margin-bottom: 10px;"></a-input>
        <a-input v-model="buy_url" placeholder="购买链接" style="margin-bottom: 10px;"></a-input>
        <a-input-password v-model="authSecret" placeholder="请输入管理密钥"></a-input-password>
      </div>
      <div class="akile-modal-action">
        <a-button type="primary" :long="true" @click="handleEditHost">更新信息</a-button>
      </div>
    </a-modal>
    <div class="footer" style="margin-bottom: 30px">Copyright © 2023-{{new Date().getFullYear()}} LuocyNetwork</div>
  </div>
</template>

<style lang="scss">
body {
  margin: 0;
  background-color: #fafafa;
  font-family: Inter,-apple-system,BlinkMacSystemFont,Roboto,PingFang SC,Noto Sans CJK,WenQuanYi Micro Hei,Microsoft YaHei;
}

a {
  text-decoration: none;
}

.max-container {
  margin: 0 auto;
  width: 100%;
  max-width: 1440px;
}

.header {
  margin: 10px;
  display: flex;
  align-items: center;
  justify-content: space-between;

  .theme-btn {
    border: 1px solid #eeeeee!important;
    background-color: #ffffff!important;
    color: #333333!important;
  }
}

.area-tabs {
  margin: 20px 10px;

  .area-tab-item {
    margin-bottom: 10px;
    margin-right: 10px;
    padding: 6px 12px;
    border-radius: 6px;
    cursor: pointer;
    border: 1px solid #e5e5e5;
    background: #ffffff;
    box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
    display: inline-block;

    .flag-icon {
      border-radius: 3px;
      margin-top: -3px;
    }

    &.is-active {
      background: #005fe710;
      color: #054db4;
      border: 1px solid #005fe7;
    }
  }

}

.monitor-card {
  position: relative;
  margin: 0 auto;
  padding: 10px;

  .monitor-item {
    position: relative;
    margin-bottom: 12px;
    padding: 12px 24px;
    border-radius: 6px;
    border: 1px solid #e5e5e5;
    display: block;
    background: #ffffff;
    box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
    cursor: pointer;

    &.is-active {
      background: #e7e7e730;

      .delete-btn,
      .edit-btn {
        display: none!important;
      }

      &>.detail {
        margin-top: 15px;
        border-top: 1px solid #eeeeee;
        padding-top: 15px;
        display: block;
      }
    }

    &:hover {
      background: #e7e7e730;

      .delete-btn,
      .edit-btn {
        display: flex;
      }
    }

    .edit-btn {
      right: 60px!important;
      background: rgba(22, 131, 255, 0.13)!important;

      &:hover {
        background: rgba(22, 131, 255, 0.19)!important;
      }

      .arco-icon {
        color: #1673ff!important;
      }
    }

    .delete-btn,
    .edit-btn {
      position: absolute;
      right: 20px;
      top: calc(50% - 16px);
      cursor: pointer;
      background: #ff161620;
      border-radius: 100px;
      width: 32px;
      height: 32px;
      display: none;
      align-items: center;
      justify-content: center;
      transition: .15s background-color ease-in-out;

      &:hover {
        background: #ff161630;
      }

      .arco-icon {
        color: #ff1616;
      }
    }

    .flag-icon {
      margin-right: 5px;
      border-radius: 3px;
    }

    .monitor-item-title {
      margin-bottom: 3px;
      font-size: 11px;
      opacity: .6;
    }

    .monitor-item-value {
      font-weight: 500;
    }

    .monitor-item-progress {
      margin-top: 4px;
      height: 4px;
      display: block;
    }

    .detail {
      width: 100%;
    }

    .name {
      display: inline-block;
      vertical-align: middle;
      width: 250px;

      .title {
        margin-bottom: 5px;
        display: flex;
        align-items: center;
        font-weight: 600;
        font-size: 16px;
      }

      .status {
        display: flex;
        align-items: center;
        &::before {
          margin-right: 10px;
          position: relative;
          display: block;
          content: '';
          width: 8px;
          height: 8px;
          border-radius: 12px;
          background-color: #fb2c36;
        }

        &.online {
          &::before {
            background-color: #00c951;
          }
        }

        span {
          font-size: 13px;
          opacity: .6;
        }
      }
    }

    .platform {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .cpu {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .mem {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .average {
      display: inline-block;
      vertical-align: top;
      width: 200px;
    }

    .network {
      display: inline-block;
      vertical-align: top;
      width: 200px;
    }

    .uptime {
      display: inline-block;
      vertical-align: middle;
      width: 200px;
    }

    .detail {
      display: none;

      .detail-item-list {
        margin-bottom: 20px;
      }

      .detail-item {
        .name {
          width: 20%;
          font-size: 12px;
          color: #666;
          margin-bottom: 5px;
          display: inline-block;
          vertical-align: top;
        }

        .value {
          width: 80%;
          font-size: 12px;
          font-weight: 500;
          display: inline-block;
          vertical-align: top;
        }
      }
    }
  }
}

.logo {
  margin-top: 20px;
  margin-bottom: 20px;
  position: relative;
  cursor: pointer;
  line-height: 54px;
  height: 54px;
  font-size: 16px;
  font-weight: 900;
  color: #333;
  display: flex;
  align-items: center;

  .arco-icon {
    margin-right: 5px;
    height: 28px;
    width: 28px;
    color: rgb(22,93,255)!important;
  }
}

.monitor-action-bar {
  margin: 0 10px;
  display: inline-block;
  border: 1px solid #e5e5e5;
  background: #ffffff;
  box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
  border-radius: 4px;

  :deep(.arco-tabs-content) {
    display: none;
  }
}

.footer {
  line-height: 22px;
  text-align: center;
  color: #565656;

  a {
    color: rgba(var(--primary-6));
  }
}

.akile-modal-title {
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 20px;
}

.akile-modal-content {
  margin-bottom: 20px;
  .tips {
    font-size: 12px;
    color: #333333;
    margin-top: 10px;
  }
}

body[arco-theme='dark'] {
  background-color: #111111;

  .arco-modal {
    background-color: #0e0e0e;
    border: 1px solid rgba(255,255,255,0.05);
  }

  .header {
    .logo {
      span,
      small {
        color: #ffffff;
      }
    }

    .theme-btn {
      border: 1px solid #333333!important;
      background-color: #000000!important;
      color: #ffffff!important;
    }
  }

  .area-tabs {
    .area-tab-item {
      border-color: #333333;
      background: #000000;
      color: #ffffff;
      box-shadow: none;

      &.is-active {
        background: #005fe705;
        color: #005fe7;
        border: 1px solid #005fe7;
      }
    }
  }

  .footer {
    color: #ffffffAA;
  }

  .monitor-card {
    .monitor-item {
      border: 1px solid rgb(255 255 255 / 16%);
      box-shadow: none;
      background-color: #000000;
      color: #ffffff;

      &:hover {
        background-color: #101010;
      }

      .detail {
        border-color: #333333AA;
        .detail-item-list {
          .detail-item {
            .name {
              color: #999999;
            }
          }
        }
      }
    }
  }

}

@media screen and (max-width: 768px) {
  .monitor-item {
    &>div {
      margin-bottom: 10px;
    }
  }

  .detail {
    .detail-item {
      .name {
        width: 25%!important;
      }
      .value {
        width: 75%!important;
      }
    }
  }
}

@media screen and (max-width: 1920px) {
  .max-container {
    max-width: 1440px;
  }
}

@media screen and (max-width: 1280px) {
  .max-container {
    max-width: 1140px;
  }
}

@media screen and (max-width: 992px) {
  .max-container {
    max-width: 960px;
  }
}

@media screen and (max-width: 768px) {
  .max-container {
    max-width: 720px;
  }
}

@media screen and (max-width: 576px) {
  .max-container {
    max-width: 540px;
  }
}
</style>
