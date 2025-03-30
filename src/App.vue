<template>
  <div class="orderbook">
    <h2 class="title">Order Book</h2>
    <table>
      <thead>
        <tr>
          <th>Price(USD)</th>
          <th>Size</th>
          <th>Total</th>
        </tr>
      </thead>
      <tbody>
        <!-- 賣單 -->
        <tr v-for="ask in showAsks" 
            :key="'ask-' + ask.price" 
            class="ask-row"
            :class="{ 'highlight_red': ask.status === 'new' }">
          <td :style="{ color: '#FF5B5A' }">{{ ask.price }}</td>
          <td :class="sizeClass(ask.status)">{{ formatNumber(ask.size) }}</td>
          <td>{{ formatNumber(ask.total) }}</td>
          <div class="bg-bar-red" :style="{ width: `${ask.percentage}%` }"></div>
        </tr>

        <!-- 當前成交價 -->
        <tr>
          <td colspan="3" :class="`last-price-row ${priceClass}`">
            <div class="last-price-text">
              {{ lastPrice }}
              <svg v-if="priceClass !== 'price-neutral'" xmlns="http://www.w3.org/2000/svg" class="svg-icon" viewBox="0 0 24 24" role="presentation" fill="none" fill-rule="nonzero" stroke="currentColor" stroke-width="4" stroke-linecap="round" stroke-linejoin="round">
                <line x1="12" y1="5" x2="12" y2="19"></line>
                <polyline points="19 12 12 19 5 12"></polyline>
              </svg>
            </div>
          </td>
        </tr>

        <!-- 買單 -->
        <tr v-for="bid in showBids" 
            :key="'bid-' + bid.price" 
            class="bid-row"
            :class="{ 'highlight_green': bid.status === 'new' }">
          <td :style="{ color: '#00b15d' }">{{ bid.price }}</td>
          <td :class="sizeClass(bid.status)">{{ formatNumber(bid.size) }}</td>
          <td>{{ formatNumber(bid.total) }}</td>
          <div class="bg-bar-green" :style="{ width: `${bid.percentage}%` }"></div>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, type Ref } from 'vue';

interface Order {
  price: number
  size: number
  total: number
  status: string
  percentage: number
}

const bids = ref<Order[]>([]);
const asks = ref<Order[]>([]);
const lastPrice = ref(0);
const prevLastPrice = ref(0);
const lastSeqNum = ref(null);
const tradeSocket = ref<WebSocket | null>(null);
const orderSocket = ref<WebSocket | null>(null);

const showBids = computed(() => bids.value.slice(0, 8));
const showAsks = computed(() => asks.value.slice(0, 8).reverse());

const formatNumber = (num: number) => {
  return num.toLocaleString(undefined, { maximumFractionDigits: 0 });
};

const priceClass = computed(() => {
  if (lastPrice.value > prevLastPrice.value) {
    return 'price-up';
  } else if (lastPrice.value < prevLastPrice.value) {
    return 'price-down';
  }
  return 'price-neutral';
});

const sizeClass = (status: string) => {
  if (status === 'up') return 'highlight_green'
  else if (status === 'down') return 'highlight_red'
}

onMounted(() => {
  connectOrderSocket();
  connectTradeSocket();
});

onUnmounted(() => {
  if (tradeSocket.value) {
    tradeSocket.value.close();
    tradeSocket.value = null;
  }

  if (orderSocket.value) {
    orderSocket.value.close();
    orderSocket.value = null;
  }
})

const connectTradeSocket = () => {
  tradeSocket.value = new WebSocket("wss://ws.btse.com/ws/futures");
  tradeSocket.value.onopen = () => {
    tradeSocket.value?.send(JSON.stringify({ op: "subscribe", args: ["tradeHistoryApi:BTCPFC"] }));
  };

  tradeSocket.value.onmessage = (event) => {
    const data = JSON.parse(event.data);
    if (!data.data) return
    prevLastPrice.value = lastPrice.value;
    lastPrice.value = parseFloat(data.data[0].price);
  };
};

const connectOrderSocket = () => {
  orderSocket.value = new WebSocket('wss://ws.btse.com/ws/oss/futures');
  orderSocket.value.onopen = () => {
    orderSocket.value?.send(JSON.stringify({ op: 'subscribe', args: ['update:BTCPFC_0'] }));
  };

  const reconnectOrderSocket = () => {
    if (orderSocket.value) {
      orderSocket.value.close();
      orderSocket.value = null;
      lastSeqNum.value = null;
    }
    setTimeout(connectOrderSocket, 1000); // 1 秒後重新連接
  };

  orderSocket.value.onmessage = (event) => {
    const response = JSON.parse(event.data);
    if (!response.data) return
    const { seqNum, prevSeqNum } = response.data;
    
    // **檢查是否丟失數據**
    if (lastSeqNum.value !== null && prevSeqNum !== lastSeqNum.value) {
      console.warn("Data out of order, resubscribing...");
      reconnectOrderSocket();
      return;
    }

    lastSeqNum.value = seqNum; // 更新最新的 seqNum
    updateOrderBook(response.data);
    
    if (asks.value[0].price <= bids.value[0].price) {
      console.warn("Crossed orderbook detected, resubscribing...");
      reconnectOrderSocket();
    }
  };
};


const updateOrderBook = (data: Record<string, any>) => {
  if (data.type === 'snapshot') {
    bids.value = processOrders(data.bids, true);
    asks.value = processOrders(data.asks, false);
    console.log('snapshot data');
  } else if (data.type === 'delta') {
    updateOrders(data.bids, bids, true);
    updateOrders(data.asks, asks, false);
  }
};

const processOrders = (orders: string[], isBid: boolean) => {
  let total = 0;
  const sortOrders = orders.sort((a, b) => isBid ? parseFloat(b[0]) - parseFloat(a[0]) : parseFloat(a[0]) -parseFloat(b[0]))
    .map(([price, size]) => {
      total += parseFloat(size);
      return { price: parseFloat(price), size: parseFloat(size), total, status: 'init' };
    })
  return sortOrders.map(data => ({ ...data, percentage: data.total * 100 / total }));
};

const updateOrders = (updates: string[], orderList: Ref<Order[]>, isBid: boolean) => {
  updates.forEach(([price, size]) => {
    const orderIndex = orderList.value.findIndex(order => order.price === parseFloat(price));
    if (size === '0') {
      if (orderIndex !== -1) orderList.value.splice(orderIndex, 1);
    } else {
      if (orderIndex !== -1) {
        const status = orderList.value[orderIndex].size > parseFloat(size) ? 'down' : 'up'
        orderList.value[orderIndex].size = parseFloat(size);
        orderList.value[orderIndex].status = status;
      } else {
        orderList.value.push({ price: parseFloat(price), size: parseFloat(size), total: 0, percentage: 0 ,status: 'new' });
      }
    }
  });

  orderList.value.sort((a, b) => (isBid ? b.price - a.price : a.price - b.price));
  // 重新計算累計總量
  let total = 0;
  orderList.value.forEach(order => {
    total += order.size;
    order.total = total;
  });

  orderList.value = orderList.value.map(data => ({ ...data, percentage: data.total * 100 / total }));
};
</script>

<style>
.orderbook {
  background: #131B29;
  color: #F0F4F8;
  width: 100%;
  padding: 1rem;
}

.title {
  text-align: left;
  font-size: 1.2em;
  line-height: 1;
  margin: 0 0.5em 0.5em;
}

table {
  width: 100%;
  border-collapse: collapse;
}

th, td {
  padding: 8px;
  text-align: right;
}

th {
  color: #8698aa;
}

.ask-row, .bid-row {
  position: relative;
}

.ask-row:hover, .bid-row:hover {
  background: #1E3059;
}

.bg-bar-red {
  position: absolute;
  top: 0;
  right: 0;  /* 讓背景條從右側開始 */
  height: 100%;
  background-color: rgba(255, 90, 90, 0.12);
}

.bg-bar-green {
  position: absolute;
  top: 0;
  right: 0;  /* 讓背景條從右側開始 */
  height: 100%;
  background-color: rgba(16, 186, 104, 0.12);
}

/* 當前價格樣式 */
.last-price-row {
  background: rgba(134, 152, 170, 0.12);
  font-weight: bold;
  font-size: 1.3em;
}

.last-price-text {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 0.3em;
}

.svg-icon {
  width: 1.1em;
  height: 1.1em;
}

.price-up .svg-icon {
  transform: rotate(180deg);
}

.price-up {
  color: #00b15d;
  background: rgba(16, 186, 104, 0.12);
}

.price-down {
  color: #FF5B5A;
  background: rgba(255, 90, 90, 0.12);
}

.price-neutral {
  color: #F0F4F8;
  background: rgba(134, 152, 170, 0.12);
}

/* 高亮動畫 */
.highlight_red {
  animation: flash_red 0.5s;
}

.highlight_green {
  animation: flash_green 0.5s;
}

@keyframes flash_red {
  from { background-color: rgba(255, 91, 90, 0.5); }
  to { background-color: transparent; }
}

@keyframes flash_green {
  from { background-color: rgba(0, 177, 93, 0.5); }
  to { background-color: transparent; }
}
</style>