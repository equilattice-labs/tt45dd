<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'
import {
  Activity,
  ArrowUpRight,
  Check,
  CircleAlert,
  Clock3,
  Copy,
  Database,
  ExternalLink,
  Gauge,
  RefreshCw,
  Waves,
} from '@lucide/vue'

const TOKEN_ADDRESS = '0xbeea1d618e533a387d941f58a7d4c9b7bd377777'
const EXPLORER_URL = `https://bscscan.com/token/${TOKEN_ADDRESS}`
const PANCAKE_SWAP_URL = `https://pancakeswap.finance/swap?outputCurrency=${TOKEN_ADDRESS}`
const PANCAKE_FACTORY = '0xca143ce32fe78f1f7019d7d551a6402fc5350c73'
const WBNB_ADDRESS = '0xbb4cdb9cbd36b01bd1cbaebf2de08d9173bc095c'
const USDT_ADDRESS = '0x55d398326f99059ff775485246999027b3197955'
const BUSD_ADDRESS = '0xe9e7cea3dedca5984780bafc599bd69add087d56'
const RPC_URLS = [
  'https://bsc-dataseed.bnbchain.org',
  'https://bsc-dataseed.binance.org',
]
const REFRESH_INTERVAL = 5

const loading = ref(true)
const refreshing = ref(false)
const error = ref('')
const copied = ref(false)
const secondsUntilRefresh = ref(REFRESH_INTERVAL)
const lastUpdated = ref(null)
const token = ref({
  name: 'BSC Token',
  symbol: 'TOKEN',
  decimals: 18,
  totalSupply: null,
  priceUsd: null,
  priceChange24h: null,
  liquidityUsd: null,
  volume24h: null,
  marketCap: null,
  fdv: null,
  pairAddress: '',
  pairUrl: PANCAKE_SWAP_URL,
  pairName: '',
  marketCapSource: '',
})

let refreshTimer
let countdownTimer

const shortAddress = computed(() => `${TOKEN_ADDRESS.slice(0, 6)}...${TOKEN_ADDRESS.slice(-4)}`)
const hasData = computed(() => token.value.priceUsd !== null || token.value.marketCap !== null)
const changePositive = computed(() => token.value.priceChange24h !== null && Number(token.value.priceChange24h) >= 0)
const formattedUpdated = computed(() => {
  if (!lastUpdated.value) return '--'
  return new Intl.DateTimeFormat('zh-CN', {
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit',
  }).format(lastUpdated.value)
})

function withTimeout(url, options = {}, timeout = 6500) {
  const controller = new AbortController()
  const timeoutId = window.setTimeout(() => controller.abort(), timeout)
  return fetch(url, { ...options, signal: controller.signal }).finally(() => window.clearTimeout(timeoutId))
}

async function rpcCall(method, params = []) {
  let lastError
  for (const url of RPC_URLS) {
    try {
      const response = await withTimeout(url, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ jsonrpc: '2.0', id: Date.now(), method, params }),
      })
      if (!response.ok) throw new Error(`RPC ${response.status}`)
      const payload = await response.json()
      if (payload.error) throw new Error(payload.error.message || 'RPC error')
      return payload.result
    } catch (requestError) {
      lastError = requestError
    }
  }
  throw lastError || new Error('BSC RPC unavailable')
}

function encodeCall(selector) {
  return selector
}

function encodeAddressCall(selector, address) {
  return `${selector}${address.slice(2).padStart(64, '0')}`
}

function decodeUint(hex) {
  if (!hex || hex === '0x') return null
  try {
    return BigInt(hex)
  } catch {
    return null
  }
}

function decodeAddress(hex) {
  if (!hex || hex.length < 42) return ''
  return `0x${hex.slice(-40).toLowerCase()}`
}

function decodeReserves(hex) {
  if (!hex || hex.length < 130) return null
  return {
    reserve0: decodeUint(`0x${hex.slice(2, 66)}`),
    reserve1: decodeUint(`0x${hex.slice(66, 130)}`),
  }
}

function decodeText(hex) {
  if (!hex || hex === '0x') return ''
  const value = hex.slice(2)
  try {
    const offset = Number.parseInt(value.slice(0, 64), 16)
    const dynamicLength = Number.parseInt(value.slice(offset * 2, offset * 2 + 64), 16)
    const dynamicStart = offset * 2 + 64
    if (Number.isFinite(dynamicLength) && dynamicLength > 0 && dynamicStart + dynamicLength * 2 <= value.length) {
      return hexToText(value.slice(dynamicStart, dynamicStart + dynamicLength * 2))
    }
  } catch {
    // Some tokens return bytes32 instead of a dynamic string.
  }
  return hexToText(value.slice(0, 64))
}

function hexToText(value) {
  const bytes = []
  for (let index = 0; index < value.length; index += 2) {
    const code = Number.parseInt(value.slice(index, index + 2), 16)
    if (!code) break
    bytes.push(code)
  }
  return new TextDecoder('utf-8').decode(new Uint8Array(bytes)).trim()
}

async function loadTokenContract() {
  const [nameHex, symbolHex, decimalsHex, supplyHex] = await Promise.all([
    rpcCall('eth_call', [{ to: TOKEN_ADDRESS, data: encodeCall('0x06fdde03') }, 'latest']),
    rpcCall('eth_call', [{ to: TOKEN_ADDRESS, data: encodeCall('0x95d89b41') }, 'latest']),
    rpcCall('eth_call', [{ to: TOKEN_ADDRESS, data: encodeCall('0x313ce567') }, 'latest']),
    rpcCall('eth_call', [{ to: TOKEN_ADDRESS, data: encodeCall('0x18160ddd') }, 'latest']),
  ])
  return {
    name: decodeText(nameHex) || 'BSC Token',
    symbol: decodeText(symbolHex) || 'TOKEN',
    decimals: Number(decodeUint(decimalsHex) ?? 18n),
    totalSupply: decodeUint(supplyHex),
  }
}

async function loadPairSnapshot(pairAddress) {
  const [token0Hex, token1Hex, reservesHex] = await Promise.all([
    rpcCall('eth_call', [{ to: pairAddress, data: encodeCall('0x0dfe1681') }, 'latest']),
    rpcCall('eth_call', [{ to: pairAddress, data: encodeCall('0xd21220a7') }, 'latest']),
    rpcCall('eth_call', [{ to: pairAddress, data: encodeCall('0x0902f1ac') }, 'latest']),
  ])
  const reserves = decodeReserves(reservesHex)
  if (!reserves?.reserve0 || !reserves?.reserve1) throw new Error('PancakeSwap 池储备为空')
  return { token0: decodeAddress(token0Hex), token1: decodeAddress(token1Hex), ...reserves }
}

async function getPancakePair(addressA, addressB) {
  const result = await rpcCall('eth_call', [{
    to: PANCAKE_FACTORY,
    data: encodeAddressCall('0xe6a43905', addressA) + addressB.slice(2).padStart(64, '0'),
  }, 'latest'])
  const pairAddress = decodeAddress(result)
  return /^0x0{40}$/i.test(pairAddress) ? '' : pairAddress
}

function units(raw, decimals) {
  return Number(raw) / (10 ** decimals)
}

async function loadBnbUsdPrice() {
  const stablePairs = await Promise.all([
    getPancakePair(WBNB_ADDRESS, USDT_ADDRESS),
    getPancakePair(WBNB_ADDRESS, BUSD_ADDRESS),
  ])
  for (const pairAddress of stablePairs.filter(Boolean)) {
    try {
      const pair = await loadPairSnapshot(pairAddress)
      const wbnbIsToken0 = pair.token0 === WBNB_ADDRESS
      const wbnbReserve = wbnbIsToken0 ? pair.reserve0 : pair.reserve1
      const stableReserve = wbnbIsToken0 ? pair.reserve1 : pair.reserve0
      if (wbnbReserve && stableReserve) return units(stableReserve, 18) / units(wbnbReserve, 18)
    } catch {
      // Try the next stablecoin pool.
    }
  }
  return null
}

async function loadMarketData(targetDecimals = 18) {
  const quotes = [
    { address: USDT_ADDRESS, symbol: 'USDT', decimals: 18, isStable: true },
    { address: BUSD_ADDRESS, symbol: 'BUSD', decimals: 18, isStable: true },
    { address: WBNB_ADDRESS, symbol: 'WBNB', decimals: 18, isStable: false },
  ]
  const pairResults = await Promise.allSettled(quotes.map((quote) => getPancakePair(TOKEN_ADDRESS, quote.address)))
  const pairs = quotes.map((quote, index) => ({ quote, pairAddress: pairResults[index].status === 'fulfilled' ? pairResults[index].value : '' }))
    .filter((candidate) => candidate.pairAddress)
  if (!pairs.length) throw new Error('PancakeSwap 暂未找到 BSC 交易对')

  const bnbUsd = pairs.some(({ quote }) => !quote.isStable) ? await loadBnbUsdPrice() : null
  const snapshots = await Promise.all(pairs.map(async (candidate) => ({ ...candidate, snapshot: await loadPairSnapshot(candidate.pairAddress) })))
  const candidates = snapshots.map(({ quote, pairAddress, snapshot }) => {
    const tokenIsToken0 = snapshot.token0 === TOKEN_ADDRESS
    const tokenReserve = tokenIsToken0 ? snapshot.reserve0 : snapshot.reserve1
    const quoteReserve = tokenIsToken0 ? snapshot.reserve1 : snapshot.reserve0
    const quoteAmount = units(quoteReserve, quote.decimals)
    const priceQuote = units(quoteReserve, quote.decimals) / units(tokenReserve, targetDecimals)
    const priceUsd = quote.isStable ? priceQuote : bnbUsd ? priceQuote * bnbUsd : null
    const liquidityUsd = quote.isStable ? quoteAmount * 2 : bnbUsd ? quoteAmount * bnbUsd * 2 : null
    return { pairAddress, pairName: quote.symbol, priceUsd, liquidityUsd, tokenReserve }
  }).filter((candidate) => candidate.priceUsd && candidate.liquidityUsd)

  if (!candidates.length) throw new Error('PancakeSwap 交易对暂无可用流动性')
  const best = candidates.sort((a, b) => b.liquidityUsd - a.liquidityUsd)[0]
  return {
    priceUsd: best.priceUsd,
    priceChange24h: null,
    liquidityUsd: best.liquidityUsd,
    volume24h: null,
    marketCap: null,
    fdv: null,
    pairAddress: best.pairAddress,
    pairUrl: PANCAKE_SWAP_URL,
    pairName: best.pairName,
  }
}

async function refreshData(isManual = false) {
  if (refreshing.value) return
  refreshing.value = true
  if (!hasData.value) loading.value = true
  error.value = ''
  try {
    const [contractResult] = await Promise.allSettled([loadTokenContract()])
    const contract = contractResult.status === 'fulfilled' ? contractResult.value : {}
    const [marketResult] = await Promise.allSettled([loadMarketData(contract.decimals ?? 18)])
    const market = marketResult.status === 'fulfilled' ? marketResult.value : {}

    if (!Object.keys(market).length && !Object.keys(contract).length) {
      throw new Error('BSC 数据暂时不可用，请稍后重试')
    }

    const supplyMarketCap = contract.totalSupply && market.priceUsd
      ? Number(contract.totalSupply) / (10 ** (contract.decimals ?? 18)) * market.priceUsd
      : null
    const marketCap = supplyMarketCap || market.marketCap || market.fdv || null
    token.value = {
      ...token.value,
      ...contract,
      ...market,
      marketCap,
      marketCapSource: supplyMarketCap ? '链上总供应量 × PancakeSwap 现货' : '等待链上供应量',
    }
    if (marketResult.status === 'rejected' && contractResult.status === 'fulfilled') {
      error.value = '行情接口暂时不可用，已保留链上合约数据'
    } else if (contractResult.status === 'rejected' && marketResult.status === 'fulfilled') {
      error.value = 'BSC RPC 暂时不可用，市值使用行情接口数据'
    }
    lastUpdated.value = new Date()
    secondsUntilRefresh.value = REFRESH_INTERVAL
  } catch (requestError) {
    error.value = requestError?.message || '读取数据失败'
  } finally {
    loading.value = false
    refreshing.value = false
  }
}

function formatUsd(value, digits = 2) {
  if (value === null || value === undefined || Number.isNaN(Number(value))) return '--'
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
    notation: Math.abs(Number(value)) >= 1000 ? 'compact' : 'standard',
    maximumFractionDigits: digits,
  }).format(Number(value))
}

function formatPrice(value) {
  if (value === null || value === undefined || Number.isNaN(Number(value))) return '--'
  const number = Number(value)
  if (number < 0.01) return `$${number.toFixed(8)}`
  return formatUsd(number, 6)
}

function formatPercent(value) {
  if (value === null || value === undefined || Number.isNaN(Number(value))) return '--'
  return `${Number(value) >= 0 ? '+' : ''}${Number(value).toFixed(2)}%`
}

function formatSupply(value, decimals) {
  if (value === null || value === undefined) return '--'
  const units = Number(value) / (10 ** (decimals || 18))
  return new Intl.NumberFormat('en-US', { notation: 'compact', maximumFractionDigits: 2 }).format(units)
}

async function copyAddress() {
  try {
    await navigator.clipboard.writeText(TOKEN_ADDRESS)
    copied.value = true
    window.setTimeout(() => { copied.value = false }, 1600)
  } catch {
    error.value = '复制失败，请手动选择地址'
  }
}

function startTimers() {
  refreshTimer = window.setInterval(() => refreshData(), REFRESH_INTERVAL * 1000)
  countdownTimer = window.setInterval(() => {
    secondsUntilRefresh.value = secondsUntilRefresh.value > 1 ? secondsUntilRefresh.value - 1 : REFRESH_INTERVAL
  }, 1000)
}

onMounted(async () => {
  await refreshData()
  startTimers()
})

onBeforeUnmount(() => {
  window.clearInterval(refreshTimer)
  window.clearInterval(countdownTimer)
})
</script>

<template>
  <main class="app-shell">
    <header class="topbar">
      <div class="brand-block">
        <div class="brand-mark"><Activity :size="18" stroke-width="2.4" /></div>
        <div>
          <p class="eyebrow">ON-CHAIN MONITOR</p>
          <h1>BSC 市值监控</h1>
        </div>
      </div>
      <div class="topbar-actions">
        <div class="network-pill"><span class="live-dot"></span> BNB Smart Chain</div>
        <button class="icon-button" type="button" title="立即刷新" aria-label="立即刷新" :disabled="refreshing" @click="refreshData(true)">
          <RefreshCw :size="17" :class="{ spinning: refreshing }" />
        </button>
      </div>
    </header>

    <section class="hero-section">
      <div class="token-heading">
        <div class="token-avatar">{{ token.symbol.slice(0, 1) }}</div>
        <div>
          <div class="token-title-row">
            <h2>{{ token.name }}</h2>
            <span class="symbol-tag">{{ token.symbol }}</span>
          </div>
          <div class="address-row">
            <span class="mono">{{ shortAddress }}</span>
            <button class="copy-button" type="button" title="复制合约地址" aria-label="复制合约地址" @click="copyAddress">
              <Check v-if="copied" :size="14" />
              <Copy v-else :size="14" />
              <span>{{ copied ? '已复制' : '复制' }}</span>
            </button>
            <a class="explorer-link" :href="EXPLORER_URL" target="_blank" rel="noreferrer">
              BscScan <ExternalLink :size="13" />
            </a>
          </div>
        </div>
      </div>
      <div class="hero-value">
        <p class="metric-label">MARKET CAP</p>
        <div v-if="loading" class="skeleton skeleton-value"></div>
        <p v-else class="market-cap">{{ formatUsd(token.marketCap) }}</p>
        <p class="source-note">{{ token.marketCapSource || '等待 BSC 数据' }}</p>
      </div>
    </section>

    <div v-if="error" class="notice-bar" :class="{ warning: hasData }">
      <CircleAlert :size="16" />
      <span>{{ error }}</span>
    </div>

    <section class="metrics-grid" aria-label="市场指标">
      <article class="metric-panel accent-cyan">
        <div class="panel-top"><span class="metric-label">PRICE / USD</span><Gauge :size="17" /></div>
        <div v-if="loading" class="skeleton skeleton-line"></div>
        <p v-else class="metric-value">{{ formatPrice(token.priceUsd) }}</p>
        <p v-if="token.priceChange24h !== null" class="metric-foot" :class="changePositive ? 'positive' : 'negative'">
          <ArrowUpRight v-if="changePositive" :size="14" />
          <ArrowUpRight v-else :size="14" class="down-icon" />
          {{ formatPercent(token.priceChange24h) }} <span>24H</span>
        </p>
        <p v-else class="metric-foot muted">链上实时现货</p>
      </article>
      <article class="metric-panel accent-orange">
        <div class="panel-top"><span class="metric-label">LIQUIDITY</span><Waves :size="17" /></div>
        <div v-if="loading" class="skeleton skeleton-line"></div>
        <p v-else class="metric-value">{{ formatUsd(token.liquidityUsd) }}</p>
        <p class="metric-foot muted">BSC 交易对</p>
      </article>
      <article class="metric-panel accent-lime">
        <div class="panel-top"><span class="metric-label">POOL</span><Database :size="17" /></div>
        <div v-if="loading" class="skeleton skeleton-line"></div>
        <p v-else class="metric-value">{{ token.pairName || '--' }}</p>
        <p class="metric-foot muted">PancakeSwap V2</p>
      </article>
      <article class="metric-panel accent-violet">
        <div class="panel-top"><span class="metric-label">TOTAL SUPPLY</span><Activity :size="17" /></div>
        <div v-if="loading" class="skeleton skeleton-line"></div>
        <p v-else class="metric-value">{{ formatSupply(token.totalSupply, token.decimals) }}</p>
        <p class="metric-foot muted">ERC-20 链上总量</p>
      </article>
    </section>

    <section class="details-section">
      <div class="section-heading">
        <div>
          <p class="eyebrow">CONTRACT SNAPSHOT</p>
          <h3>链上合约信息</h3>
        </div>
        <div class="refresh-status"><span class="status-dot"></span><Clock3 :size="14" /> {{ secondsUntilRefresh }}s 后更新</div>
      </div>
      <div class="details-table">
        <div class="detail-row"><span>网络</span><strong>BNB Smart Chain <span class="tiny-badge">BSC</span></strong></div>
        <div class="detail-row"><span>Token Symbol</span><strong>{{ token.symbol }}</strong></div>
        <div class="detail-row"><span>Decimals</span><strong>{{ token.decimals }}</strong></div>
        <div class="detail-row"><span>Total Supply</span><strong>{{ formatSupply(token.totalSupply, token.decimals) }}</strong></div>
        <div class="detail-row"><span>数据更新</span><strong>{{ formattedUpdated }}</strong></div>
      </div>
    </section>

    <footer class="footer">
      <span><span class="footer-dot"></span> BSC RPC + PancakeSwap V2</span>
      <a v-if="token.pairUrl" :href="token.pairUrl" target="_blank" rel="noreferrer">查看交易对 <ArrowUpRight :size="14" /></a>
    </footer>
  </main>
</template>
