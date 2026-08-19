<script setup>
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'
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
  Search,
  Waves,
} from '@lucide/vue'

const DEFAULT_TOKEN_ADDRESS = '0xbeea1d618e533a387d941f58a7d4c9b7bd377777'
const BASE_PAGE_TITLE = 'BSC 市值监控'
const TOKEN_ADDRESS_PATTERN = /^0x[0-9a-f]{40}$/i
const TOKEN_ADDRESS_STORAGE_KEY = 'bsc-token-address'
const PANCAKE_V2_FACTORY = '0xca143ce32fe78f1f7019d7d551a6402fc5350c73'
const PANCAKE_V3_FACTORY = '0x0bfbCf9fa4f9C56B0F40a671Ad40E0805A091865'
const PANCAKE_V3_FEES = [100, 500, 2500, 10000]
const WBNB_ADDRESS = '0xbb4cdb9cbd36b01bd1cbaebf2de08d9173bc095c'
const USDT_ADDRESS = '0x55d398326f99059ff775485246999027b3197955'
const BUSD_ADDRESS = '0xe9e7cea3dedca5984780bafc599bd69add087d56'
const MARKET_QUOTES = [
  { address: USDT_ADDRESS, symbol: 'USDT', decimals: 18, isStable: true },
  { address: BUSD_ADDRESS, symbol: 'BUSD', decimals: 18, isStable: true },
  { address: WBNB_ADDRESS, symbol: 'WBNB', decimals: 18, isStable: false },
]

function parseEndpointList(value) {
  return [...new Set((value || '').split(',').map((url) => url.trim()).filter(Boolean))]
}

const PUBLIC_BSC_RPC_URLS = [
  'https://bsc-dataseed.binance.org',
  'https://bsc-dataseed1.defibit.io',
  'https://bsc-dataseed1.ninicoin.io',
]
const configuredRpcUrls = parseEndpointList(import.meta.env.VITE_BSC_HTTPS_URL)
const RPC_URLS = configuredRpcUrls.length ? configuredRpcUrls : PUBLIC_BSC_RPC_URLS
const WS_URLS = parseEndpointList(import.meta.env.VITE_BSC_WS_URL)
const PANCAKE_SYNC_TOPIC = '0x1c411e9a96e071241c2f21f7726b17ae89e3cab4c78be50e062b03a9fffbbad1'
const PANCAKE_PAIR_CREATED_TOPIC = '0x0d3648bd0f6ba80134a33ba9275ac585d9d315f0ad8355cddefde31afa28d0e9'
// PancakeSwap V3 emits a seven-word Swap payload (the final protocol-fee
// fields are not needed here). This is different from Uniswap V3's topic.
const PANCAKE_V3_SWAP_TOPIC = '0x19b47279256b2a23a1665c810c8d55a1758940ee09377d4f8d26497a3577dc83'
const REFRESH_INTERVAL = 5
const RPC_MAX_CONCURRENCY = 4
const PRICE_POOL_MIN_LIQUIDITY_USD = 10000
const PRICE_POOL_MIN_PRIMARY_RATIO = 0.01
const PRICE_POOL_MAX_DEVIATION = 0.15
const INITIAL_LOG_LOOKBACK_BLOCKS = 2000
const PAIR_DISCOVERY_LOOKBACK_BLOCKS = 30000
const PAIR_DISCOVERY_BLOCK_CHUNK = 9000

const loading = ref(true)
const refreshing = ref(false)
const error = ref('')
const addressError = ref('')
const activeTokenAddress = ref(DEFAULT_TOKEN_ADDRESS)
const addressInput = ref(DEFAULT_TOKEN_ADDRESS)
const copied = ref(false)
const secondsUntilRefresh = ref(REFRESH_INTERVAL)
const lastUpdated = ref(null)
const latestBlockNumber = ref(null)
const latestBlockTimestamp = ref(null)
const currentTimestamp = ref(Date.now())
const refreshMode = ref('websocket')
const realtimeStatus = ref('idle')

function pancakeSwapUrl(address) {
  return `https://pancakeswap.finance/swap?outputCurrency=${address}`
}

function createEmptyToken(address) {
  return {
    name: 'BSC Token',
    symbol: 'TOKEN',
    decimals: 18,
    totalSupply: null,
    priceUsd: null,
    bnbUsd: null,
    priceChange24h: null,
    liquidityUsd: null,
    quoteReserveAmount: null,
    quoteSymbol: '',
    volume24h: null,
    marketCap: null,
    fdv: null,
    pairAddress: '',
    pairUrl: pancakeSwapUrl(address),
    pairName: '',
    poolVersion: '',
    pricePoolCount: 0,
    priceSourcePoolAddress: '',
    priceSourcePoolVersion: '',
    priceSourceQuoteSymbol: '',
    marketCapSource: '',
  }
}

const token = ref(createEmptyToken(activeTokenAddress.value))

let refreshTimer
let countdownTimer
let clockTimer
let realtimeSocket
let realtimeReconnectTimer
let realtimeRefreshTimer
let realtimeUrlIndex = 0
let realtimeRequestId = 1
let realtimeLogSubscriptionId = ''
let realtimeHeadSubscriptionId = ''
let realtimePairContexts = new Map()
const realtimeV3RefreshesInFlight = new Set()
let selectedPoolDescriptor = null
let pricePoolDescriptors = []
let poolDiscoveryPromise = null
let discoveredMarketCandidates = null
let quoteBridgeMarkets = new Map()
let discoveredV2PairDescriptors = null
let quoteBridgeMarketsLoaded = false
let latestPricePoolAddress = ''
let latestMarketEventPosition = null
let activeRpcRequests = 0
const rpcWaiters = []

const shortAddress = computed(() => `${activeTokenAddress.value.slice(0, 6)}...${activeTokenAddress.value.slice(-4)}`)
const explorerUrl = computed(() => `https://bscscan.com/token/${activeTokenAddress.value}`)
const primaryPoolExplorerUrl = computed(() => (
  token.value.pairAddress ? `https://bscscan.com/address/${token.value.pairAddress}` : ''
))
const hasData = computed(() => token.value.priceUsd !== null || token.value.marketCap !== null)
const changePositive = computed(() => token.value.priceChange24h !== null && Number(token.value.priceChange24h) >= 0)
const updateStatusText = computed(() => {
  if (refreshMode.value === 'polling') return `${secondsUntilRefresh.value}s 后更新`
  if (realtimeStatus.value === 'connected') return `${token.value.pricePoolCount || 1} 个价格池 + WBNB 池实时推送`
  if (realtimeStatus.value === 'connecting') return `WS 连接中 · ${secondsUntilRefresh.value}s 轮询兜底`
  return `WS 重连中 · ${secondsUntilRefresh.value}s 轮询兜底`
})
const blockDelayMs = computed(() => {
  if (latestBlockTimestamp.value === null) return null
  return Math.max(0, currentTimestamp.value - latestBlockTimestamp.value * 1000)
})
const blockDelayLevel = computed(() => {
  if (blockDelayMs.value === null) return 'unknown'
  if (blockDelayMs.value > 15000) return 'danger'
  if (blockDelayMs.value > 5000) return 'warning'
  return 'healthy'
})
const formattedBlockNumber = computed(() => {
  if (latestBlockNumber.value === null) return '--'
  return new Intl.NumberFormat('en-US').format(latestBlockNumber.value)
})
const formattedUpdated = computed(() => {
  if (!lastUpdated.value) return '--'
  return new Intl.DateTimeFormat('zh-CN', {
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit',
  }).format(lastUpdated.value)
})

function updateDocumentTitle() {
  const marketCap = Number(token.value.marketCap)
  if (Number.isFinite(marketCap)) {
    const tokenLabel = token.value.symbol && token.value.symbol !== 'TOKEN'
      ? token.value.symbol
      : shortAddress.value
    document.title = `${tokenLabel} · ${formatMarketCap(marketCap)} · ${BASE_PAGE_TITLE}`
    return
  }
  document.title = BASE_PAGE_TITLE
}

watch(
  [() => token.value.symbol, () => token.value.marketCap],
  updateDocumentTitle,
  { immediate: true },
)

function withTimeout(url, options = {}, timeout = 6500) {
  const controller = new AbortController()
  const timeoutId = window.setTimeout(() => controller.abort(), timeout)
  return fetch(url, { ...options, signal: controller.signal }).finally(() => window.clearTimeout(timeoutId))
}

function wait(milliseconds) {
  return new Promise((resolve) => window.setTimeout(resolve, milliseconds))
}

async function withRpcSlot(task) {
  if (activeRpcRequests >= RPC_MAX_CONCURRENCY) {
    await new Promise((resolve) => rpcWaiters.push(resolve))
  }
  activeRpcRequests += 1
  try {
    return await task()
  } finally {
    activeRpcRequests -= 1
    rpcWaiters.shift()?.()
  }
}

async function rpcCall(method, params = []) {
  if (!RPC_URLS.length) throw new Error('未配置 BSC RPC，请设置 VITE_BSC_HTTPS_URL')
  return withRpcSlot(async () => {
    let lastError
    for (const url of RPC_URLS) {
      for (let attempt = 0; attempt < 3; attempt += 1) {
        try {
          const response = await withTimeout(url, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ jsonrpc: '2.0', id: Date.now(), method, params }),
          })
          if (response.status === 429 && attempt < 2) {
            await wait(300 * (2 ** attempt))
            continue
          }
          if (!response.ok) throw new Error('RPC ' + response.status)
          const payload = await response.json()
          if (payload.error) throw new Error(payload.error.message || 'RPC error')
          return payload.result
        } catch (requestError) {
          lastError = requestError
        }
      }
    }
    throw lastError || new Error('BSC RPC unavailable')
  })
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

async function loadTokenContract(targetAddress) {
  const contractCode = await rpcCall('eth_getCode', [targetAddress, 'latest'])
  if (!contractCode || /^0x0*$/i.test(contractCode)) throw new Error('该地址不是 BSC 合约地址')

  const [nameHex, symbolHex, decimalsHex, supplyHex] = await Promise.all([
    rpcCall('eth_call', [{ to: targetAddress, data: encodeCall('0x06fdde03') }, 'latest']).catch(() => '0x'),
    rpcCall('eth_call', [{ to: targetAddress, data: encodeCall('0x95d89b41') }, 'latest']).catch(() => '0x'),
    rpcCall('eth_call', [{ to: targetAddress, data: encodeCall('0x313ce567') }, 'latest']),
    rpcCall('eth_call', [{ to: targetAddress, data: encodeCall('0x18160ddd') }, 'latest']),
  ])
  const decimals = Number(decodeUint(decimalsHex))
  const totalSupply = decodeUint(supplyHex)
  if (!Number.isInteger(decimals) || decimals < 0 || decimals > 255 || totalSupply === null) {
    throw new Error('该合约不支持标准 ERC-20 元数据读取')
  }
  return {
    name: decodeText(nameHex) || 'BSC Token',
    symbol: decodeText(symbolHex) || 'TOKEN',
    decimals,
    totalSupply,
  }
}

function updateBlockStatus(block) {
  if (!block?.number || !block?.timestamp) return
  latestBlockNumber.value = Number.parseInt(block.number, 16)
  latestBlockTimestamp.value = Number.parseInt(block.timestamp, 16)
  currentTimestamp.value = Date.now()
}

async function loadLatestBlock() {
  const block = await rpcCall('eth_getBlockByNumber', ['latest', false])
  updateBlockStatus(block)
}

async function loadPairSnapshot(pairAddress, blockTag = 'latest') {
  const [token0Hex, token1Hex, reservesHex] = await Promise.all([
    rpcCall('eth_call', [{ to: pairAddress, data: encodeCall('0x0dfe1681') }, blockTag]),
    rpcCall('eth_call', [{ to: pairAddress, data: encodeCall('0xd21220a7') }, blockTag]),
    rpcCall('eth_call', [{ to: pairAddress, data: encodeCall('0x0902f1ac') }, blockTag]),
  ])
  const reserves = decodeReserves(reservesHex)
  if (!reserves?.reserve0 || !reserves?.reserve1) throw new Error('PancakeSwap 池储备为空')
  return { token0: decodeAddress(token0Hex), token1: decodeAddress(token1Hex), ...reserves }
}

async function getPancakePair(addressA, addressB) {
  const result = await rpcCall('eth_call', [{
    to: PANCAKE_V2_FACTORY,
    data: encodeAddressCall('0xe6a43905', addressA) + addressB.slice(2).padStart(64, '0'),
  }, 'latest'])
  const pairAddress = decodeAddress(result)
  return /^0x0{40}$/i.test(pairAddress) ? '' : pairAddress
}

async function getPancakeV3Pool(addressA, addressB, fee) {
  const result = await rpcCall('eth_call', [{
    to: PANCAKE_V3_FACTORY,
    data: encodeAddressCall('0x1698ee82', addressA)
      + addressB.slice(2).padStart(64, '0')
      + BigInt(fee).toString(16).padStart(64, '0'),
  }, 'latest'])
  const poolAddress = decodeAddress(result)
  return /^0x0{40}$/i.test(poolAddress) ? '' : poolAddress
}

function decodeSlot0(hex) {
  if (!hex || hex.length < 66) return null
  return decodeUint('0x' + hex.slice(2, 66))
}

function decodeSignedWord(hex, index) {
  const start = 2 + index * 64
  const unsigned = decodeUint('0x' + hex.slice(start, start + 64))
  if (unsigned === null) return null
  return unsigned >= (1n << 255n) ? unsigned - (1n << 256n) : unsigned
}

function decodeV3Swap(hex) {
  if (!hex || hex.length < 2 + 64 * 5) return null
  return {
    amount0: decodeSignedWord(hex, 0),
    amount1: decodeSignedWord(hex, 1),
    sqrtPriceX96: decodeUint('0x' + hex.slice(2 + 64 * 2, 2 + 64 * 3)),
  }
}

async function loadV3PoolSnapshot(poolAddress, quoteAddress, quoteDecimals, targetDecimals, targetAddress, blockTag = 'latest') {
  const quoteBalanceData = encodeAddressCall('0x70a08231', poolAddress)
  const [token0Hex, token1Hex, slot0Hex, quoteBalanceHex, targetBalanceHex] = await Promise.all([
    rpcCall('eth_call', [{ to: poolAddress, data: encodeCall('0x0dfe1681') }, blockTag]),
    rpcCall('eth_call', [{ to: poolAddress, data: encodeCall('0xd21220a7') }, blockTag]),
    rpcCall('eth_call', [{ to: poolAddress, data: encodeCall('0x3850c7bd') }, blockTag]),
    rpcCall('eth_call', [{ to: quoteAddress, data: quoteBalanceData }, blockTag]),
    rpcCall('eth_call', [{ to: targetAddress, data: quoteBalanceData }, blockTag]),
  ])
  const sqrtPriceX96 = decodeSlot0(slot0Hex)
  const quoteReserve = decodeUint(quoteBalanceHex)
  const tokenReserve = decodeUint(targetBalanceHex)
  if (!sqrtPriceX96 || !quoteReserve || !tokenReserve) throw new Error('PancakeSwap V3 pool data is empty')
  return {
    token0: decodeAddress(token0Hex),
    token1: decodeAddress(token1Hex),
    sqrtPriceX96,
    quoteReserve,
    tokenReserve,
    quoteDecimals,
    targetDecimals,
  }
}

function v3PriceQuote(snapshot, targetAddress) {
  const token0IsTarget = snapshot.token0 === targetAddress
  const sqrt = Number(snapshot.sqrtPriceX96) / (2 ** 96)
  const rawToken1PerToken0 = sqrt * sqrt
  const rawQuotePerTarget = token0IsTarget ? rawToken1PerToken0 : 1 / rawToken1PerToken0
  return rawQuotePerTarget * (10 ** snapshot.targetDecimals) / (10 ** snapshot.quoteDecimals)
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
      if (wbnbReserve && stableReserve) {
        return {
          priceUsd: units(stableReserve, 18) / units(wbnbReserve, 18),
          pairAddress,
          snapshot: pair,
        }
      }
    } catch {
      // Try the next stablecoin pool.
    }
  }
  return null
}

async function loadQuoteBridgeMarkets(quotes = MARKET_QUOTES) {
  if (quoteBridgeMarketsLoaded) return
  quoteBridgeMarkets = new Map()
  const bridgeQuotes = quotes
    .filter((quote) => quote.address !== activeTokenAddress.value && !quote.isStable && quote.address !== WBNB_ADDRESS)
    .map((quote) => ({ ...quote, bridgeAddress: quote.bridgeAddress || WBNB_ADDRESS }))
  await Promise.all(bridgeQuotes.map(async (quote) => {
    try {
      const pairAddress = await getPancakePair(quote.address, quote.bridgeAddress)
      if (!pairAddress) return
      const snapshot = await loadPairSnapshot(pairAddress)
      const quoteIsToken0 = snapshot.token0 === quote.address
      const quoteReserve = quoteIsToken0 ? snapshot.reserve0 : snapshot.reserve1
      const bridgeReserve = quoteIsToken0 ? snapshot.reserve1 : snapshot.reserve0
      if (!quoteReserve || !bridgeReserve) return
      quoteBridgeMarkets.set(quote.address, {
        quoteAddress: quote.address,
        quoteDecimals: quote.decimals,
        bridgeAddress: quote.bridgeAddress,
        pairAddress: pairAddress.toLowerCase(),
        snapshot,
        bnbPerQuote: units(bridgeReserve, 18) / units(quoteReserve, quote.decimals),
      })
    } catch {
      // A quote bridge is optional; stablecoin and WBNB pools can still work.
    }
  }))
  quoteBridgeMarketsLoaded = true
}

function decodeEventAddress(value) {
  return decodeAddress(value || '')
}

async function readQuoteMetadata(address) {
  const known = MARKET_QUOTES.find((quote) => quote.address === address)
  if (known) return known
  const [symbolHex, decimalsHex] = await Promise.all([
    rpcCall('eth_call', [{ to: address, data: encodeCall('0x95d89b41') }, 'latest']).catch(() => '0x'),
    rpcCall('eth_call', [{ to: address, data: encodeCall('0x313ce567') }, 'latest']).catch(() => '0x'),
  ])
  const decimals = Number(decodeUint(decimalsHex))
  if (!Number.isInteger(decimals) || decimals < 0 || decimals > 255) return null
  return {
    address,
    symbol: decodeText(symbolHex) || `${address.slice(0, 6)}...${address.slice(-4)}`,
    decimals,
    isStable: false,
    bridgeAddress: WBNB_ADDRESS,
  }
}

async function discoverAdditionalV2Pairs(targetAddress) {
  const blockNumber = latestBlockNumber.value
  if (!blockNumber) return []
  const indexedTarget = `0x${targetAddress.slice(2).padStart(64, '0')}`
  const fromBlock = Math.max(0, blockNumber - PAIR_DISCOVERY_LOOKBACK_BLOCKS)
  const logs = []
  for (let start = fromBlock; start <= blockNumber; start += PAIR_DISCOVERY_BLOCK_CHUNK) {
    const end = Math.min(blockNumber, start + PAIR_DISCOVERY_BLOCK_CHUNK - 1)
    const filter = {
      address: PANCAKE_V2_FACTORY,
      fromBlock: `0x${start.toString(16)}`,
      toBlock: `0x${end.toString(16)}`,
    }
    const [token0Logs, token1Logs] = await Promise.all([
      rpcCall('eth_getLogs', [{ ...filter, topics: [PANCAKE_PAIR_CREATED_TOPIC, indexedTarget] }]),
      rpcCall('eth_getLogs', [{ ...filter, topics: [PANCAKE_PAIR_CREATED_TOPIC, null, indexedTarget] }]),
    ])
    logs.push(...token0Logs, ...token1Logs)
  }
  const pairs = []
  const seen = new Set()
  for (const log of logs) {
    const token0 = decodeEventAddress(log.topics?.[1])
    const token1 = decodeEventAddress(log.topics?.[2])
    const pairAddress = decodeEventAddress(`0x${(log.data || '').slice(2, 66)}`)
    const quoteAddress = token0 === targetAddress ? token1 : token0
    if (!pairAddress || !quoteAddress || quoteAddress === targetAddress || seen.has(pairAddress)) continue
    seen.add(pairAddress)
    const quote = await readQuoteMetadata(quoteAddress)
    if (quote) pairs.push({ quote, version: 'V2', fee: null, pairAddress })
  }
  persistDiscoveredPairs(pairs, targetAddress)
  return pairs
}

async function getRealtimePairAddresses() {
  return [...realtimePairContexts.values()].map((context) => context.pairAddress)
}

function getTargetPoolValues(context) {
  const tokenIsToken0 = context.token0 === context.targetAddress
  const tokenReserve = context.version === 'V3'
    ? context.tokenReserve
    : tokenIsToken0 ? context.reserve0 : context.reserve1
  const quoteReserve = context.version === 'V3'
    ? context.quoteReserve
    : tokenIsToken0 ? context.reserve1 : context.reserve0
  if (!tokenReserve || !quoteReserve) return null
  const priceQuote = context.version === 'V3'
    ? v3PriceQuote(context, context.targetAddress)
    : units(quoteReserve, context.quoteDecimals) / units(tokenReserve, context.targetDecimals)
  return { tokenReserve, quoteReserve, priceQuote }
}

function getRealtimeBnbUsd() {
  let bnbUsd = token.value.bnbUsd
  const stable = [...realtimePairContexts.values()].find((context) => context.role === 'stable')
  if (stable) {
    const wbnbIsToken0 = stable.token0 === WBNB_ADDRESS
    const wbnbReserve = wbnbIsToken0 ? stable.reserve0 : stable.reserve1
    const stableReserve = wbnbIsToken0 ? stable.reserve1 : stable.reserve0
    bnbUsd = units(stableReserve, 18) / units(wbnbReserve, 18)
  }
  return bnbUsd
}

function getQuoteUsd(quote, bnbUsd) {
  if (quote.isStable) return 1
  if (quote.address === WBNB_ADDRESS) return bnbUsd
  const bridge = quoteBridgeMarkets.get(quote.address)
  return bridge?.bnbPerQuote && bnbUsd ? bridge.bnbPerQuote * bnbUsd : null
}

function getRealtimeQuoteUsd(context, bnbUsd) {
  if (context.quoteIsStable) return 1
  if (context.quoteAddress === WBNB_ADDRESS) return bnbUsd
  const bridge = [...realtimePairContexts.values()].find((item) => (
    item.role === 'quote' && item.quoteAddress === context.quoteAddress
  ))
  if (!bridge || !bnbUsd) return context.quoteUsd || null
  const quoteIsToken0 = bridge.token0 === bridge.quoteAddress
  const quoteReserve = quoteIsToken0 ? bridge.reserve0 : bridge.reserve1
  const wbnbReserve = quoteIsToken0 ? bridge.reserve1 : bridge.reserve0
  if (!quoteReserve || !wbnbReserve) return context.quoteUsd || null
  return units(wbnbReserve, 18) / units(quoteReserve, bridge.quoteDecimals) * bnbUsd
}

function updateFromRealtimeReserves(sourceContext = null) {
  const pricePools = [...realtimePairContexts.values()].filter((context) => context.role === 'price')
  const primary = pricePools.find((context) => context.isPrimary) || pricePools[0]
  const source = sourceContext?.role === 'price'
    ? sourceContext
    : pricePools.find((context) => context.pairAddress === latestPricePoolAddress) || primary
  if (!primary || !source) return

  const primaryValues = getTargetPoolValues(primary)
  const sourceValues = getTargetPoolValues(source)
  if (!primaryValues || !sourceValues) return
  const bnbUsd = getRealtimeBnbUsd()
  const sourceQuoteUsd = getRealtimeQuoteUsd(source, bnbUsd)
  const primaryQuoteUsd = getRealtimeQuoteUsd(primary, bnbUsd)
  if (!sourceQuoteUsd || !primaryQuoteUsd) return

  const priceUsd = sourceValues.priceQuote * sourceQuoteUsd
  const quoteAmount = units(primaryValues.quoteReserve, primary.quoteDecimals)
  const tokenAmount = units(primaryValues.tokenReserve, primary.targetDecimals)
  const liquidityUsd = (quoteAmount + tokenAmount * primaryValues.priceQuote) * primaryQuoteUsd
  const marketCap = token.value.totalSupply
    !== null && token.value.totalSupply !== undefined
    ? Number(token.value.totalSupply) / (10 ** token.value.decimals) * priceUsd
    : token.value.marketCap
  token.value = {
    ...token.value,
    priceUsd,
    bnbUsd,
    liquidityUsd,
    quoteReserveAmount: quoteAmount,
    quoteSymbol: primary.quoteSymbol,
    marketCap,
    priceSourcePoolAddress: source.pairAddress,
    priceSourcePoolVersion: source.version,
    priceSourceQuoteSymbol: source.quoteSymbol,
  }
  lastUpdated.value = new Date()
}

async function refreshRealtimeV3Context(context, blockTag = 'latest') {
  if (realtimeV3RefreshesInFlight.has(context.pairAddress)) return
  realtimeV3RefreshesInFlight.add(context.pairAddress)
  try {
    const snapshot = await loadV3PoolSnapshot(
      context.pairAddress,
      context.quoteAddress,
      context.quoteDecimals,
      context.targetDecimals,
      context.targetAddress,
      blockTag,
    )
    Object.assign(context, snapshot)
    updateFromRealtimeReserves()
  } finally {
    realtimeV3RefreshesInFlight.delete(context.pairAddress)
  }
}

function logPosition(log) {
  return {
    blockNumber: Number.parseInt(log.blockNumber || '0x0', 16),
    transactionIndex: Number.parseInt(log.transactionIndex || '0x0', 16),
    logIndex: Number.parseInt(log.logIndex || '0x0', 16),
  }
}

function compareLogPositions(left, right) {
  if (!left && !right) return 0
  if (!left) return -1
  if (!right) return 1
  return left.blockNumber - right.blockNumber
    || left.transactionIndex - right.transactionIndex
    || left.logIndex - right.logIndex
}

function selectPriceSourceFromLog(log, context) {
  if (context.role !== 'price') return false
  const position = logPosition(log)
  if (compareLogPositions(position, latestMarketEventPosition) < 0) return false
  latestMarketEventPosition = position
  latestPricePoolAddress = context.pairAddress
  return true
}

function applyRealtimeSync(log) {
  const context = realtimePairContexts.get(log.address?.toLowerCase())
  if (!context) return
  if (context.version === 'V3') {
    const topic = log.topics?.[0]?.toLowerCase()
    if (topic === PANCAKE_V3_SWAP_TOPIC) {
      const swap = decodeV3Swap(log.data)
      if (!swap?.sqrtPriceX96 || swap.amount0 === null || swap.amount1 === null) return
      const token0IsTarget = context.token0 === context.targetAddress
      context.sqrtPriceX96 = swap.sqrtPriceX96
      context.tokenReserve += token0IsTarget ? swap.amount0 : swap.amount1
      context.quoteReserve += token0IsTarget ? swap.amount1 : swap.amount0
      if (selectPriceSourceFromLog(log, context)) updateFromRealtimeReserves(context)
      return
    }
    refreshRealtimeV3Context(context, log.blockNumber || 'latest').catch(() => {})
    return
  }
  if (log.topics?.[0]?.toLowerCase() !== PANCAKE_SYNC_TOPIC) return
  const reserves = decodeReserves(log.data)
  if (!reserves) return
  context.reserve0 = reserves.reserve0
  context.reserve1 = reserves.reserve1
  window.clearTimeout(realtimeRefreshTimer)
  realtimeRefreshTimer = window.setTimeout(() => {
    if (context.role === 'price') selectPriceSourceFromLog(log, context)
    updateFromRealtimeReserves(context.role === 'price' ? context : null)
  }, 50)
}

function scheduleRealtimeReconnect() {
  window.clearTimeout(realtimeReconnectTimer)
  realtimeReconnectTimer = window.setTimeout(() => {
    if (refreshMode.value === 'websocket') connectRealtime()
  }, 3000)
}

function stopPolling() {
  window.clearInterval(refreshTimer)
  window.clearInterval(countdownTimer)
  refreshTimer = undefined
  countdownTimer = undefined
}

function startPolling() {
  if (refreshTimer) return
  secondsUntilRefresh.value = REFRESH_INTERVAL
  refreshTimer = window.setInterval(() => refreshData(), REFRESH_INTERVAL * 1000)
  countdownTimer = window.setInterval(() => {
    secondsUntilRefresh.value = secondsUntilRefresh.value > 1 ? secondsUntilRefresh.value - 1 : REFRESH_INTERVAL
  }, 1000)
}

function stopRealtime() {
  window.clearTimeout(realtimeReconnectTimer)
  window.clearTimeout(realtimeRefreshTimer)
  realtimeLogSubscriptionId = ''
  realtimeHeadSubscriptionId = ''
  const socket = realtimeSocket
  realtimeSocket = undefined
  if (socket && socket.readyState < WebSocket.CLOSING) socket.close()
}

async function setRefreshMode(mode) {
  if (!['websocket', 'polling'].includes(mode)) return
  refreshMode.value = mode
  window.localStorage.setItem('bsc-refresh-mode', mode)
  stopRealtime()
  stopPolling()
  if (mode === 'polling') {
    realtimeStatus.value = 'idle'
    startPolling()
    return
  }
  realtimeStatus.value = 'connecting'
  startPolling()
  await connectRealtime()
}

async function connectRealtime() {
  if (refreshMode.value !== 'websocket') return
  if (!window.WebSocket || !WS_URLS.length) {
    realtimeStatus.value = 'fallback'
    startPolling()
    return
  }
  const pairAddresses = await getRealtimePairAddresses()
  if (refreshMode.value !== 'websocket') return
  if (!pairAddresses.length) {
    realtimeStatus.value = 'fallback'
    startPolling()
    scheduleRealtimeReconnect()
    return
  }
  window.clearTimeout(realtimeReconnectTimer)
  stopRealtime()
  const wsUrl = WS_URLS[realtimeUrlIndex % WS_URLS.length]
  realtimeStatus.value = 'connecting'
  startPolling()
  try {
    const socket = new WebSocket(wsUrl)
    realtimeSocket = socket
    socket.addEventListener('open', () => {
      if (realtimeSocket !== socket || refreshMode.value !== 'websocket') return
      const logRequestId = realtimeRequestId++
      const headRequestId = realtimeRequestId++
      socket.subscriptionRequests = new Map([
        [logRequestId, 'logs'],
        [headRequestId, 'heads'],
      ])
      socket.send(JSON.stringify({
        jsonrpc: '2.0',
        id: logRequestId,
        method: 'eth_subscribe',
        params: ['logs', { address: pairAddresses }],
      }))
      socket.send(JSON.stringify({
        jsonrpc: '2.0',
        id: headRequestId,
        method: 'eth_subscribe',
        params: ['newHeads'],
      }))
    })
    socket.addEventListener('message', (event) => {
      try {
        const message = JSON.parse(event.data)
        const subscriptionType = socket.subscriptionRequests?.get(message.id)
        if (subscriptionType) {
          if (!message.result) throw new Error(message.error?.message || '订阅失败')
          if (subscriptionType === 'logs') realtimeLogSubscriptionId = message.result
          if (subscriptionType === 'heads') realtimeHeadSubscriptionId = message.result
          socket.subscriptionRequests.delete(message.id)
          if (realtimeLogSubscriptionId && realtimeHeadSubscriptionId) {
            realtimeStatus.value = 'connected'
            stopPolling()
          }
          return
        }
        if (message.method !== 'eth_subscription') return
        if (message.params?.subscription === realtimeHeadSubscriptionId) {
          updateBlockStatus(message.params?.result)
          return
        }
        if (message.params?.subscription !== realtimeLogSubscriptionId) return
        if (message.params?.result?.removed) {
          latestMarketEventPosition = null
          latestPricePoolAddress = ''
          refreshData(true)
          return
        }
        applyRealtimeSync(message.params?.result || {})
      } catch {
        realtimeStatus.value = 'fallback'
        startPolling()
        socket.close()
      }
    })
    socket.addEventListener('error', () => {
      if (realtimeSocket !== socket) return
      realtimeStatus.value = 'fallback'
      startPolling()
    })
    socket.addEventListener('close', () => {
      if (realtimeSocket !== socket) return
      realtimeSocket = undefined
      realtimeLogSubscriptionId = ''
      realtimeHeadSubscriptionId = ''
      if (refreshMode.value !== 'websocket') return
      realtimeStatus.value = 'fallback'
      startPolling()
      realtimeUrlIndex += 1
      scheduleRealtimeReconnect()
    })
  } catch {
    realtimeStatus.value = 'fallback'
    startPolling()
    realtimeUrlIndex += 1
    scheduleRealtimeReconnect()
  }
}

function normalizePoolDescriptor(value) {
  if (!value || !/^0x[0-9a-f]{40}$/i.test(value.pairAddress || '')) return null
  if (!['V2', 'V3'].includes(value.version)) return null
  const quote = MARKET_QUOTES.find((item) => item.address === value.quote?.address?.toLowerCase()) || (
    /^0x[0-9a-f]{40}$/i.test(value.quote?.address || '') && Number.isInteger(Number(value.quote?.decimals))
      ? {
          address: value.quote.address.toLowerCase(),
          symbol: String(value.quote.symbol || 'QUOTE').slice(0, 24),
          decimals: Number(value.quote.decimals),
          isStable: false,
          bridgeAddress: WBNB_ADDRESS,
        }
      : null
  )
  if (!quote) return null
  return {
    pairAddress: value.pairAddress.toLowerCase(),
    version: value.version,
    fee: value.version === 'V3' ? Number(value.fee) || null : null,
    quote,
  }
}

function selectedPoolStorageKey(targetAddress) {
  return `bsc-fixed-pool:${targetAddress}`
}

function pricePoolsStorageKey(targetAddress) {
  return `bsc-price-pools:${targetAddress}`
}

function discoveredPairsStorageKey(targetAddress) {
  return `bsc-discovered-v2-pairs:${targetAddress}`
}

function readStoredPoolDescriptor(targetAddress) {
  try {
    return normalizePoolDescriptor(JSON.parse(window.localStorage.getItem(selectedPoolStorageKey(targetAddress))))
  } catch {
    return null
  }
}

function persistPoolDescriptor(descriptor, targetAddress) {
  window.localStorage.setItem(selectedPoolStorageKey(targetAddress), JSON.stringify(descriptor))
}

function readStoredPricePoolDescriptors(targetAddress) {
  try {
    const values = JSON.parse(window.localStorage.getItem(pricePoolsStorageKey(targetAddress)))
    if (!Array.isArray(values)) return []
    return values.map(normalizePoolDescriptor).filter(Boolean)
  } catch {
    return []
  }
}

function persistPricePoolDescriptors(descriptors, targetAddress) {
  window.localStorage.setItem(pricePoolsStorageKey(targetAddress), JSON.stringify(descriptors))
}

function readStoredDiscoveredPairs(targetAddress) {
  try {
    const values = JSON.parse(window.localStorage.getItem(discoveredPairsStorageKey(targetAddress)))
    if (!Array.isArray(values)) return null
    return values.map(normalizePoolDescriptor).filter((item) => item?.version === 'V2')
  } catch {
    return null
  }
}

function persistDiscoveredPairs(descriptors, targetAddress) {
  window.localStorage.setItem(discoveredPairsStorageKey(targetAddress), JSON.stringify(descriptors))
}

async function loadMarketCandidate(descriptor, targetDecimals, bnbUsd, targetAddress) {
  const { quote, version, pairAddress } = descriptor
  const snapshot = version === 'V3'
    ? await loadV3PoolSnapshot(pairAddress, quote.address, quote.decimals, targetDecimals, targetAddress)
    : await loadPairSnapshot(pairAddress)
  if (![snapshot.token0, snapshot.token1].includes(targetAddress)
    || ![snapshot.token0, snapshot.token1].includes(quote.address)) {
    throw new Error('PancakeSwap 交易池资产与 Token 不匹配')
  }
  const tokenIsToken0 = snapshot.token0 === targetAddress
  const tokenReserve = version === 'V3' ? snapshot.tokenReserve : tokenIsToken0 ? snapshot.reserve0 : snapshot.reserve1
  const quoteReserve = version === 'V3' ? snapshot.quoteReserve : tokenIsToken0 ? snapshot.reserve1 : snapshot.reserve0
  const quoteAmount = units(quoteReserve, quote.decimals)
  const priceQuote = version === 'V3'
    ? v3PriceQuote(snapshot, targetAddress)
    : units(quoteReserve, quote.decimals) / units(tokenReserve, targetDecimals)
  const tokenAmount = units(tokenReserve, targetDecimals)
  const quoteUsd = getQuoteUsd(quote, bnbUsd)
  const priceUsd = quoteUsd ? priceQuote * quoteUsd : null
  const liquidityQuote = quoteAmount + tokenAmount * priceQuote
  const liquidityUsd = quoteUsd ? liquidityQuote * quoteUsd : null
  if (!priceUsd || !liquidityUsd) throw new Error('PancakeSwap pool has no usable liquidity')
  return {
    ...descriptor,
    poolVersion: version,
    pairName: quote.symbol,
    priceUsd,
    liquidityUsd,
    quoteReserveAmount: quoteAmount,
    quoteSymbol: quote.symbol,
    tokenReserve,
    snapshot,
  }
}

async function discoverMarketCandidates(targetDecimals, bnbUsd, targetAddress) {
  const marketQuotes = [...MARKET_QUOTES, ...(discoveredV2PairDescriptors || []).map((item) => item.quote)]
    .filter((quote, index, values) => quote.address !== targetAddress
      && values.findIndex((item) => item.address === quote.address) === index)
  const v2Addresses = await Promise.all(MARKET_QUOTES.map((quote) => getPancakePair(targetAddress, quote.address)))
  const pairs = MARKET_QUOTES.map((quote, index) => ({
    quote,
    version: 'V2',
    fee: null,
    pairAddress: v2Addresses[index],
  })).filter((candidate) => candidate.pairAddress)
  pairs.push(...(discoveredV2PairDescriptors || []))
  const v3Pools = await Promise.all(marketQuotes.flatMap((quote) => PANCAKE_V3_FEES.map(async (fee) => ({
    quote,
    version: 'V3',
    fee,
    pairAddress: await getPancakeV3Pool(targetAddress, quote.address, fee),
  }))))
  pairs.push(...v3Pools.filter((candidate) => candidate.pairAddress))
  if (!pairs.length) throw new Error('PancakeSwap 暂未找到 BSC 交易对')

  const candidates = (await Promise.all(pairs.map(async (descriptor) => {
    try {
      return await loadMarketCandidate(descriptor, targetDecimals, bnbUsd, targetAddress)
    } catch (candidateError) {
      if (/empty|no usable liquidity/i.test(candidateError?.message || '')) return null
      throw candidateError
    }
  }))).filter(Boolean)
  if (!candidates.length) throw new Error('PancakeSwap 交易对暂无可用流动性')
  discoveredMarketCandidates = candidates.sort((a, b) => b.liquidityUsd - a.liquidityUsd)
  return discoveredMarketCandidates
}

function candidateDescriptor(candidate) {
  return {
    pairAddress: candidate.pairAddress.toLowerCase(),
    version: candidate.poolVersion,
    fee: candidate.fee,
    quote: candidate.quote,
  }
}

function filterPricePoolCandidates(candidates, primary) {
  const minimumLiquidity = Math.max(
    PRICE_POOL_MIN_LIQUIDITY_USD,
    primary.liquidityUsd * PRICE_POOL_MIN_PRIMARY_RATIO,
  )
  const eligible = candidates.filter((candidate) => {
    const deviation = Math.abs(candidate.priceUsd / primary.priceUsd - 1)
    return candidate.liquidityUsd >= minimumLiquidity && deviation <= PRICE_POOL_MAX_DEVIATION
  })
  if (!eligible.some((candidate) => candidate.pairAddress.toLowerCase() === primary.pairAddress.toLowerCase())) {
    eligible.unshift(primary)
  }
  return eligible
}

async function discoverFixedPool(targetDecimals, bnbUsd, targetAddress) {
  const candidates = discoveredMarketCandidates || await discoverMarketCandidates(targetDecimals, bnbUsd, targetAddress)
  const best = candidates[0]
  const eligible = filterPricePoolCandidates(candidates, best)
  pricePoolDescriptors = eligible.map(candidateDescriptor)
  persistPricePoolDescriptors(pricePoolDescriptors, targetAddress)
  return candidateDescriptor(best)
}

async function ensureSelectedPool(targetDecimals, bnbUsd, targetAddress) {
  if (selectedPoolDescriptor) return selectedPoolDescriptor
  if (!poolDiscoveryPromise) {
    poolDiscoveryPromise = discoverFixedPool(targetDecimals, bnbUsd, targetAddress)
      .then((descriptor) => {
        selectedPoolDescriptor = descriptor
        persistPoolDescriptor(descriptor, targetAddress)
        return descriptor
      })
      .finally(() => { poolDiscoveryPromise = null })
  }
  return poolDiscoveryPromise
}

async function loadPricePoolCandidates(primary, targetDecimals, bnbUsd, targetAddress) {
  let descriptors = pricePoolDescriptors
  if (!descriptors.length) {
    const discovered = discoveredMarketCandidates || await discoverMarketCandidates(targetDecimals, bnbUsd, targetAddress)
    const eligible = filterPricePoolCandidates(discovered, primary)
    descriptors = eligible.map(candidateDescriptor)
    pricePoolDescriptors = descriptors
    persistPricePoolDescriptors(descriptors, targetAddress)
    return eligible
  }

  const primaryDescriptor = candidateDescriptor(primary)
  const uniqueDescriptors = [...descriptors, primaryDescriptor].filter((descriptor, index, values) => (
    values.findIndex((item) => item.pairAddress === descriptor.pairAddress) === index
  ))
  const loaded = (await Promise.all(uniqueDescriptors.map(async (descriptor) => {
    try {
      return await loadMarketCandidate(descriptor, targetDecimals, bnbUsd, targetAddress)
    } catch {
      return null
    }
  }))).filter(Boolean)
  const eligible = filterPricePoolCandidates(loaded, primary)
  pricePoolDescriptors = eligible.map(candidateDescriptor)
  persistPricePoolDescriptors(pricePoolDescriptors, targetAddress)
  return eligible
}

function createPricePoolContext(candidate, primaryAddress, targetDecimals, targetAddress, bnbUsd) {
  return {
    role: 'price',
    isPrimary: candidate.pairAddress.toLowerCase() === primaryAddress,
    version: candidate.poolVersion,
    pairAddress: candidate.pairAddress.toLowerCase(),
    token0: candidate.snapshot.token0,
    token1: candidate.snapshot.token1,
    reserve0: candidate.snapshot.reserve0,
    reserve1: candidate.snapshot.reserve1,
    sqrtPriceX96: candidate.snapshot.sqrtPriceX96,
    tokenReserve: candidate.snapshot.tokenReserve,
    quoteReserve: candidate.snapshot.quoteReserve,
    targetDecimals,
    quoteDecimals: candidate.quote.decimals,
    quoteSymbol: candidate.quote.symbol,
    quoteAddress: candidate.quote.address,
    quoteIsStable: candidate.quote.isStable,
    quoteUsd: getQuoteUsd(candidate.quote, bnbUsd),
    targetAddress,
  }
}

function isPriceEventLog(log, context) {
  const topic = log.topics?.[0]?.toLowerCase()
  return context.version === 'V3' ? topic === PANCAKE_V3_SWAP_TOPIC : topic === PANCAKE_SYNC_TOPIC
}

async function selectLatestPricePool(priceContexts) {
  const blockNumber = latestBlockNumber.value
  if (!blockNumber || !priceContexts.length) return priceContexts.find((context) => context.isPrimary) || priceContexts[0]
  const fromBlock = latestMarketEventPosition
    ? Math.max(0, latestMarketEventPosition.blockNumber)
    : Math.max(0, blockNumber - INITIAL_LOG_LOOKBACK_BLOCKS)
  try {
    const logs = await rpcCall('eth_getLogs', [{
      address: priceContexts.map((context) => context.pairAddress),
      fromBlock: '0x' + fromBlock.toString(16),
      toBlock: '0x' + blockNumber.toString(16),
    }])
    const contextsByAddress = new Map(priceContexts.map((context) => [context.pairAddress, context]))
    const newest = (logs || [])
      .map((log) => ({ log, context: contextsByAddress.get(log.address?.toLowerCase()) }))
      .filter(({ log, context }) => context && isPriceEventLog(log, context))
      .sort((left, right) => compareLogPositions(logPosition(right.log), logPosition(left.log)))[0]
    if (newest && selectPriceSourceFromLog(newest.log, newest.context)) return newest.context
  } catch {
    // Keep the last known source when a provider throttles the optional log query.
  }
  return priceContexts.find((context) => context.pairAddress === latestPricePoolAddress)
    || priceContexts.find((context) => context.isPrimary)
    || priceContexts[0]
}

async function loadMarketData(targetDecimals, targetAddress) {
  const bnbMarket = await loadBnbUsdPrice()
  const bnbUsd = bnbMarket?.priceUsd || null
  if (discoveredV2PairDescriptors === null) {
    discoveredV2PairDescriptors = await discoverAdditionalV2Pairs(targetAddress)
  }
  const discoveredQuotes = discoveredV2PairDescriptors.map((item) => item.quote)
  await loadQuoteBridgeMarkets([...MARKET_QUOTES, ...discoveredQuotes])
  let descriptor = await ensureSelectedPool(targetDecimals, bnbUsd, targetAddress)
  let best
  try {
    best = await loadMarketCandidate(descriptor, targetDecimals, bnbUsd, targetAddress)
  } catch (candidateError) {
    // A cached pool can be removed or migrated. Clear only this Token's cache
    // and retry discovery once so future polling can recover automatically.
    if (!selectedPoolDescriptor || selectedPoolDescriptor.pairAddress !== descriptor.pairAddress) throw candidateError
    selectedPoolDescriptor = null
    pricePoolDescriptors = []
    discoveredMarketCandidates = null
    discoveredV2PairDescriptors = await discoverAdditionalV2Pairs(targetAddress)
    quoteBridgeMarketsLoaded = false
    await loadQuoteBridgeMarkets([...MARKET_QUOTES, ...discoveredV2PairDescriptors.map((item) => item.quote)])
    window.localStorage.removeItem(selectedPoolStorageKey(targetAddress))
    window.localStorage.removeItem(pricePoolsStorageKey(targetAddress))
    descriptor = await ensureSelectedPool(targetDecimals, bnbUsd, targetAddress)
    best = await loadMarketCandidate(descriptor, targetDecimals, bnbUsd, targetAddress)
  }
  const priceCandidates = await loadPricePoolCandidates(best, targetDecimals, bnbUsd, targetAddress)
  const primaryAddress = best.pairAddress.toLowerCase()
  const priceContexts = priceCandidates.map((candidate) => (
    createPricePoolContext(candidate, primaryAddress, targetDecimals, targetAddress, bnbUsd)
  ))
  const sourceContext = await selectLatestPricePool(priceContexts)
  const sourceCandidate = priceCandidates.find((candidate) => (
    candidate.pairAddress.toLowerCase() === sourceContext?.pairAddress
  )) || best
  latestPricePoolAddress = sourceCandidate.pairAddress.toLowerCase()
  const realtimeContexts = [...priceContexts]
  if (bnbMarket) {
    realtimeContexts.push({
      role: 'stable',
      version: 'V2',
      pairAddress: bnbMarket.pairAddress.toLowerCase(),
      token0: bnbMarket.snapshot.token0,
      token1: bnbMarket.snapshot.token1,
      reserve0: bnbMarket.snapshot.reserve0,
      reserve1: bnbMarket.snapshot.reserve1,
    })
  }
  for (const bridge of quoteBridgeMarkets.values()) {
    realtimeContexts.push({
      role: 'quote',
      version: 'V2',
      pairAddress: bridge.pairAddress,
      token0: bridge.snapshot.token0,
      token1: bridge.snapshot.token1,
      reserve0: bridge.snapshot.reserve0,
      reserve1: bridge.snapshot.reserve1,
      quoteAddress: bridge.quoteAddress,
      quoteDecimals: bridge.quoteDecimals,
    })
  }
  return {
    priceUsd: sourceCandidate.priceUsd,
    priceChange24h: null,
    liquidityUsd: best.liquidityUsd,
    quoteReserveAmount: best.quoteReserveAmount,
    quoteSymbol: best.quoteSymbol,
    volume24h: null,
    marketCap: null,
    fdv: null,
    pairAddress: best.pairAddress,
    pairUrl: pancakeSwapUrl(targetAddress),
    pairName: best.pairName,
    poolVersion: best.poolVersion,
    pricePoolCount: priceCandidates.length,
    priceSourcePoolAddress: sourceCandidate.pairAddress.toLowerCase(),
    priceSourcePoolVersion: sourceCandidate.poolVersion,
    priceSourceQuoteSymbol: sourceCandidate.quote.symbol,
    bnbUsd,
    realtimeContexts,
  }
}

async function refreshData(isManual = false, contractOverride = null) {
  if (refreshing.value) return
  const targetAddress = activeTokenAddress.value
  refreshing.value = true
  if (!hasData.value) loading.value = true
  error.value = ''
  try {
    const contractResult = contractOverride
      ? { status: 'fulfilled', value: contractOverride }
      : (await Promise.allSettled([loadTokenContract(targetAddress)]))[0]
    await Promise.allSettled([loadLatestBlock()])
    if (contractResult.status === 'rejected') throw contractResult.reason
    const contract = contractResult.value
    const [marketResult] = await Promise.allSettled([loadMarketData(contract.decimals, targetAddress)])
    const market = marketResult.status === 'fulfilled' ? marketResult.value : {}
    const { realtimeContexts = [], ...marketData } = market

    const hasSupplyAndPrice = contract.totalSupply !== null
      && contract.totalSupply !== undefined
      && market.priceUsd !== null
      && market.priceUsd !== undefined
    const supplyMarketCap = hasSupplyAndPrice
      ? Number(contract.totalSupply) / (10 ** (contract.decimals ?? 18)) * market.priceUsd
      : null
    const marketCap = supplyMarketCap ?? market.marketCap ?? market.fdv ?? null
    token.value = {
      ...token.value,
      ...contract,
      ...marketData,
      marketCap,
      marketCapSource: supplyMarketCap ? '链上总供应量 × PancakeSwap 多池最新成交' : '等待链上供应量',
    }
    if (realtimeContexts.length) {
      realtimePairContexts = new Map(realtimeContexts.map((context) => [context.pairAddress, context]))
    }
    if (marketResult.status === 'rejected' && contractResult.status === 'fulfilled') {
      error.value = marketResult.reason?.message || '未找到可用于计算价格的 PancakeSwap 交易对'
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

function formatMarketCap(value) {
  if (value === null || value === undefined || Number.isNaN(Number(value))) return '--'
  const number = Number(value)
  const absolute = Math.abs(number)
  if (absolute >= 1e9) return `$${(number / 1e9).toFixed(3)}B`
  if (absolute >= 1e6) return `$${(number / 1e6).toFixed(3)}M`
  if (absolute >= 1e3) return `$${(number / 1e3).toFixed(2)}K`
  return formatUsd(number)
}

function formatExactUsd(value) {
  if (value === null || value === undefined || Number.isNaN(Number(value))) return '--'
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
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
  const units = Number(value) / (10 ** (decimals ?? 18))
  return new Intl.NumberFormat('en-US', { notation: 'compact', maximumFractionDigits: 2 }).format(units)
}

function formatReserve(value, symbol) {
  if (value === null || value === undefined || Number.isNaN(Number(value))) return '--'
  const normalizedSymbol = symbol || '报价资产'
  const digits = normalizedSymbol === 'WBNB' ? 6 : 2
  const formatted = new Intl.NumberFormat('en-US', { maximumFractionDigits: digits }).format(Number(value))
  return formatted + ' ' + normalizedSymbol
}

function formatPoolAddress(value) {
  if (!value) return '--'
  return value.slice(0, 6) + '...' + value.slice(-4)
}

async function copyAddress() {
  try {
    await navigator.clipboard.writeText(activeTokenAddress.value)
    copied.value = true
    window.setTimeout(() => { copied.value = false }, 1600)
  } catch {
    error.value = '复制失败，请手动选择地址'
  }
}

function normalizeTokenAddress(value) {
  const normalized = (value || '').trim().toLowerCase()
  return TOKEN_ADDRESS_PATTERN.test(normalized) ? normalized : ''
}

function resetTokenSession(targetAddress) {
  token.value = createEmptyToken(targetAddress)
  selectedPoolDescriptor = readStoredPoolDescriptor(targetAddress)
  pricePoolDescriptors = readStoredPricePoolDescriptors(targetAddress)
  poolDiscoveryPromise = null
  discoveredMarketCandidates = null
  discoveredV2PairDescriptors = readStoredDiscoveredPairs(targetAddress)
  quoteBridgeMarketsLoaded = false
  latestPricePoolAddress = ''
  latestMarketEventPosition = null
  realtimePairContexts = new Map()
  realtimeV3RefreshesInFlight.clear()
  latestBlockNumber.value = null
  latestBlockTimestamp.value = null
  lastUpdated.value = null
  copied.value = false
  error.value = ''
  loading.value = true
}

async function submitTokenAddress() {
  if (refreshing.value) return
  const targetAddress = normalizeTokenAddress(addressInput.value)
  if (!targetAddress) {
    addressError.value = '请输入有效的 BSC Token 合约地址（0x + 40 位十六进制）'
    return
  }

  addressError.value = ''
  addressInput.value = targetAddress
  if (targetAddress === activeTokenAddress.value) {
    await refreshData(true)
    return
  }

  let contract
  refreshing.value = true
  try {
    contract = await loadTokenContract(targetAddress)
  } catch (requestError) {
    addressError.value = requestError?.message || 'Token 合约校验失败'
    return
  } finally {
    refreshing.value = false
  }

  stopRealtime()
  stopPolling()
  activeTokenAddress.value = targetAddress
  window.localStorage.setItem(TOKEN_ADDRESS_STORAGE_KEY, targetAddress)
  resetTokenSession(targetAddress)
  await refreshData(true, contract)
  await setRefreshMode(refreshMode.value)
}

onMounted(async () => {
  const savedMode = window.localStorage.getItem('bsc-refresh-mode')
  if (['websocket', 'polling'].includes(savedMode)) refreshMode.value = savedMode
  const savedAddress = normalizeTokenAddress(window.localStorage.getItem(TOKEN_ADDRESS_STORAGE_KEY))
  if (savedAddress) {
    activeTokenAddress.value = savedAddress
    addressInput.value = savedAddress
  }
  resetTokenSession(activeTokenAddress.value)
  await refreshData()
  clockTimer = window.setInterval(() => { currentTimestamp.value = Date.now() }, 1000)
  await setRefreshMode(refreshMode.value)
})

onBeforeUnmount(() => {
  stopPolling()
  stopRealtime()
  window.clearInterval(clockTimer)
  document.title = BASE_PAGE_TITLE
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

    <section class="token-query-section" aria-label="Token 查询">
      <form class="token-query-form" @submit.prevent="submitTokenAddress">
        <label for="token-address">Token 合约地址</label>
        <div class="token-query-control" :class="{ invalid: addressError }">
          <input
            id="token-address"
            v-model="addressInput"
            type="text"
            inputmode="text"
            autocomplete="off"
            spellcheck="false"
            placeholder="0x..."
            aria-describedby="token-address-error"
            :aria-invalid="Boolean(addressError)"
            @input="addressError = ''"
          >
          <button type="submit" :disabled="refreshing">
            <RefreshCw v-if="refreshing" :size="16" class="spinning" />
            <Search v-else :size="16" />
            <span>{{ refreshing ? '查询中' : '查询' }}</span>
          </button>
        </div>
        <p v-if="addressError" id="token-address-error" class="input-error" role="alert">{{ addressError }}</p>
      </form>
    </section>

    <section class="hero-section">
      <div class="token-heading">
        <div class="token-avatar">{{ token.symbol.slice(0, 1) }}</div>
        <div class="token-heading-copy">
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
            <a class="explorer-link" :href="explorerUrl" target="_blank" rel="noreferrer">
              BscScan <ExternalLink :size="13" />
            </a>
          </div>
        </div>
      </div>
      <div class="hero-value">
        <p class="metric-label">MARKET CAP</p>
        <div v-if="loading" class="skeleton skeleton-value"></div>
        <p v-else class="market-cap">{{ formatMarketCap(token.marketCap) }}</p>
        <p class="source-note">{{ token.marketCapSource || '等待 BSC 数据' }}</p>
        <p v-if="token.marketCap !== null" class="exact-value">{{ formatExactUsd(token.marketCap) }}</p>
        <div class="bnb-spot">
          <span>WBNB / USDT</span>
          <strong>{{ formatPrice(token.bnbUsd) }}</strong>
        </div>
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
        <p v-else class="metric-foot muted">
          {{ token.pricePoolCount || 1 }} 池最新成交 · {{ formatPoolAddress(token.priceSourcePoolAddress) }}
        </p>
      </article>
      <article class="metric-panel accent-orange">
        <div class="panel-top"><span class="metric-label">LIQUIDITY</span><Waves :size="17" /></div>
        <div v-if="loading" class="skeleton skeleton-line"></div>
        <p
          v-else
          class="metric-value liquidity-value"
          :title="formatReserve(token.quoteReserveAmount, token.quoteSymbol)"
        >
          {{ formatReserve(token.quoteReserveAmount, token.quoteSymbol) }}
        </p>
        <p class="metric-foot muted">单侧 {{ token.quoteSymbol || '报价资产' }} 储备</p>
      </article>
      <article class="metric-panel accent-lime">
        <div class="panel-top"><span class="metric-label">POOL</span><Database :size="17" /></div>
        <div v-if="loading" class="skeleton skeleton-line"></div>
        <p v-else class="metric-value">{{ token.pairName ? `${token.symbol}/${token.pairName}` : '--' }}</p>
        <p class="metric-foot muted">PancakeSwap {{ token.poolVersion || '--' }} · {{ formatPoolAddress(token.pairAddress) }}</p>
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
        <div class="update-controls">
          <div class="mode-switch" role="group" aria-label="价格更新方式">
            <button
              type="button"
              class="mode-button"
              :class="{ active: refreshMode === 'websocket' }"
              :aria-pressed="refreshMode === 'websocket'"
              @click="setRefreshMode('websocket')"
            >
              <Activity :size="14" /> 实时 WS
            </button>
            <button
              type="button"
              class="mode-button"
              :class="{ active: refreshMode === 'polling' }"
              :aria-pressed="refreshMode === 'polling'"
              @click="setRefreshMode('polling')"
            >
              <Clock3 :size="14" /> 5 秒轮询
            </button>
          </div>
          <div class="refresh-status" :class="`realtime-${realtimeStatus}`">
            <span class="status-dot"></span>
            <span>{{ updateStatusText }}</span>
          </div>
        </div>
      </div>
      <div class="details-table">
        <div class="detail-row"><span>网络</span><strong>BNB Smart Chain <span class="tiny-badge">BSC</span></strong></div>
        <div class="detail-row"><span>Token Symbol</span><strong>{{ token.symbol }}</strong></div>
        <div class="detail-row"><span>Decimals</span><strong>{{ token.decimals }}</strong></div>
        <div class="detail-row"><span>Total Supply</span><strong>{{ formatSupply(token.totalSupply, token.decimals) }}</strong></div>
        <div class="detail-row">
          <span>主交易池</span>
          <a v-if="primaryPoolExplorerUrl" :href="primaryPoolExplorerUrl" target="_blank" rel="noreferrer">
            PancakeSwap {{ token.poolVersion }} · {{ formatPoolAddress(token.pairAddress) }} <ExternalLink :size="13" />
          </a>
          <strong v-else>--</strong>
        </div>
        <div class="detail-row">
          <span>价格来源</span>
          <strong>{{ token.priceSourcePoolVersion || '--' }} {{ token.priceSourceQuoteSymbol || '' }} · {{ formatPoolAddress(token.priceSourcePoolAddress) }}</strong>
        </div>
        <div class="detail-row"><span>数据更新</span><strong>{{ formattedUpdated }}</strong></div>
      </div>
    </section>

    <footer class="footer">
      <span><span class="footer-dot"></span> BSC RPC + PancakeSwap {{ token.poolVersion || 'V2 / V3' }}</span>
      <a v-if="token.pairUrl" :href="token.pairUrl" target="_blank" rel="noreferrer">在 PancakeSwap 交易 <ArrowUpRight :size="14" /></a>
    </footer>

    <section class="block-freshness" :class="`delay-${blockDelayLevel}`" aria-label="区块数据延时">
      <div class="block-freshness-label"><span class="status-dot"></span> 数据源区块</div>
      <div class="block-freshness-value">
        <strong>{{ blockDelayMs === null ? '--' : `${blockDelayMs}ms` }}</strong>
        <span>延时</span>
      </div>
      <div class="block-freshness-meta">Block #{{ formattedBlockNumber }} · {{ refreshMode === 'websocket' ? 'WS newHeads' : 'RPC latest' }}</div>
    </section>
  </main>
</template>
