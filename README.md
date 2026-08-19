# BSC Market Cap Monitor

Vue 3 single-page app for inspecting a BSC Token from its contract address. The last queried address is stored locally and restored on the next visit.

The page provides two selectable update modes and remembers the last selection:

- **Realtime WS** subscribes to the selected PancakeSwap V2 or V3 token pool, the WBNB/USDT pool, and any non-standard quote bridge discovered for the selected Token. V2 reserves are decoded from `Sync` logs; V3 events trigger fresh `slot0` and token-balance reads. While WebSocket is connecting or reconnecting, full 5-second polling remains active as a fallback.
- **Token lookup** validates the entered address as a BSC contract and reads its ERC-20 name, symbol, decimals, and total supply directly from the chain.
- **Pool selection** checks PancakeSwap V2 and the standard V3 fee tiers for USDT, BUSD, and WBNB, then scans recent PancakeSwap V2 `PairCreated` events for additional quote assets. An unknown quote asset is valued through its quote/WBNB bridge pool, so launchpad pairs such as `FAKER/SKHYB` are supported without a fixed quote-token whitelist. The deepest on-chain pool is selected, and pool selections are cached independently for each Token.
- **5-second polling** closes the WebSocket and refreshes both pools exclusively through BSC JSON-RPC every 5 seconds.

Market cap is calculated as:

`ERC-20 totalSupply x PancakeSwap spot USD price`

For the WBNB pair, the USDT-denominated price is derived from the token/WBNB pool multiplied by the PancakeSwap WBNB/USDT pool price. For an additional quote asset, the quote-denominated price is multiplied by the quote/WBNB bridge and the WBNB/USDT price. If the WBNB/USDT pair is unavailable, the implementation falls back to WBNB/BUSD. The current WBNB/USDT spot price is shown next to the market cap.

Both BSC transports are read from `.env`:

```env
VITE_BSC_HTTPS_URL=https://your-bsc-json-rpc
VITE_BSC_WS_URL=wss://your-bsc-websocket-rpc
```

Multiple endpoints can be separated with commas and are tried in order. Vite exposes every `VITE_*` value to the browser bundle, so do not put a private API key here unless browser exposure is acceptable.

The footer shows the latest block number and block delay in milliseconds, calculated as `browser time - block timestamp`. WebSocket mode receives block timestamps from `newHeads`; polling mode reads `eth_getBlockByNumber("latest")` every 5 seconds. BSC block timestamps have second-level precision, so the millisecond value includes the local sampling time.

## Run

```bash
npm install
npm run dev
```

Production build:

```bash
npm run build
```
