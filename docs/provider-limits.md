# Provider limits: recent single-contract logs

`runLogCursor` defaults to `5000n` blocks per `getLogs` request. A public RPC may
reject that range. This optional setup uses 100-block chunks, one contract address,
and a shared request pacer for a recent-log demonstration. It does not change
hoodkit's defaults or cursor behavior. It does not make the SDK available for
commercial use; see the repository [LICENSE](../LICENSE).

The [BlockVectra public chain catalog](https://api.blockvectra.com/v1/chains)
lists Robinhood mainnet as chain `4663`; its public logs endpoint documents an
inclusive maximum range of 100 blocks and requires one address. The catalog also
documents a 900-block history window, a 3 requests/second per-IP/instance limit
with burst 20, and no `blockHash` log filter support. These are provider limits,
not a history or compatibility guarantee.

Pass your own contract address and a start block you intend to scan. The example
only constructs a client; it makes no RPC calls until `watchRecentTransfers` is
called. Its `Transfer` event is an example filter, not a holder indexer.

```ts
import { runLogCursor, Stream } from 'hoodkit'
import { createPublicClient, http, parseAbiItem, type Address } from 'viem'

const endpoint = 'https://api.blockvectra.com/v1/robinhood_mainnet/public'

// Pace request starts across all methods using this one client.
let requestQueue = Promise.resolve()
let nextStart = 0
const pacedFetch: typeof fetch = async (input, init) => {
  let release!: () => void
  const turn = new Promise<void>((resolve) => { release = resolve })
  const previous = requestQueue
  requestQueue = previous.then(() => turn)
  await previous
  const delay = Math.max(0, nextStart - Date.now())
  if (delay > 0) await new Promise<void>((resolve) => setTimeout(resolve, delay))
  nextStart = Date.now() + 550
  release()
  return fetch(input, init)
}

const client = createPublicClient({
  transport: http(endpoint, { fetchFn: pacedFetch, retryCount: 0, timeout: 15_000 }),
})
const transferEvent = parseAbiItem(
  'event Transfer(address indexed from, address indexed to, uint256 value)',
)

export async function watchRecentTransfers(contract: Address, fromBlock: bigint) {
  const chainId = await client.getChainId()
  if (chainId !== 4663) throw new Error(`Expected chain 4663, received ${chainId}`)

  // Disable viem's block-number cache so this horizon check uses a fresh head.
  const head = await client.getBlockNumber({ cacheTime: 0 })
  if (fromBlock > head || head - fromBlock >= 900n) {
    throw new Error('Start block is outside the provider recent-history window; use a suitable historical source.')
  }

  const stream = new Stream()
  stream.on('error', (error) => console.error('Recent log stream stopped:', error))
  let stopCursor = () => {}
  const stop = () => {
    stopCursor()
    stream.end()
  }

  stopCursor = runLogCursor({
    source: { getBlockNumber: () => client.getBlockNumber({ cacheTime: 0 }) },
    fromBlock,
    chunkSize: 100n,
    confirmations: 1n,
    pollingIntervalMs: 15_000,
    getLogs: (fromBlock, toBlock) => client.getLogs({
      address: contract,
      event: transferEvent,
      fromBlock,
      toBlock,
    }),
    decode: (log) => log,
    stream,
    onError: (error) => {
      // runLogCursor starts its first tick asynchronously; stopCursor is assigned
      // before its awaited RPC can call onError. Stop so the failed range is not retried.
      stopCursor()
      stream.destroy(error)
    },
  })

  return { stream, stop }
}
```

The wrapper spaces request starts by 550 ms, including the bootstrap chain ID,
head reads, and every chunk read. Keep these calls on this one client; the
15-second poll interval alone does not pace requests inside a catch-up loop.
`retryCount: 0` and the terminal `onError` handler ensure a rejected range, 429,
or RPC error stops this example instead of becoming an empty result or an
automatic cursor retry. Choose a suitable historical source if `fromBlock` is
outside the 900-block window; do not jump to the head and treat that as a complete
scan. If the head moves beyond the provider window during a long catch-up, stop
on the resulting provider error as well.

The one-block confirmation delay is not finality. This is a recent-log example,
not a holder or OHLCV full-history indexer, a hash-pinned consistency method, or
an end-to-end compatibility claim.
