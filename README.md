# điều tra trinh sát cách Arc build block / xếp tx (mempool, FCFS, builder, PBS…). Chỉ lấy từ docs + on-chain/evidence thật, không đoán.

Đang đọc docs Arc/Circle + probe read-only on-chain. Xong sẽ gửi báo cáo: CONFIRMED / UNCONFIRMED / NOT FOUND — tách rõ.

Báo cáo Arc block building / tx ordering — chỉ fact + nguồn. Không đoán. Không bịa builder.

## Tóm tắt
- **Stack CONFIRMED:** PoA validator permissioned + **Malachite** (Tendermint BFT) + **Reth** EL, nối bằng Engine API. Nguồn: [system overview](https://docs.arc.io/arc/concepts/system-overview), [consensus](https://docs.arc.io/arc/concepts/consensus-layer), [arc-node ARCHITECTURE](https://github.com/circlefin/arc-node/blob/main/docs/ARCHITECTURE.md).
- **Ai build block CONFIRMED:** **validator proposer xoay vòng** lấy tx từ mempool/txpool → propose → ≥2/3 finalize. **Không** thấy PBS / external builder / single sequencer trong docs.
- **Ordering:** docs **không** viết FCFS hay auction kiểu 48club. Fee docs nói tip giúp inclusion; lifecycle nói dưới tải tx giá thấp có thể bị **evict**. Code payload builder dùng Reth `best_transactions_with_attributes` (tip-aware). Trang payments lại claim “theo thứ tự submit / chống front-run” — **mâu thuẫn** với fee + code.
- **MEV infra:** **CHƯA TÌM THẤY** builder club, relay, bundle API, Flashbots-like trên Arc.
- **AMP** (multi-proposer): blog Circle 28/05/2026 = **research**, adoption không đảm bảo — **không confirmed live**.

## So với BSC / Robinhood (chỉ bằng chứng)
| | Arc (evidence) | Không được claim |
|---|---|---|
| Build | PoA validator xoay vòng + BFT | Giống 48club / PBS mở |
| Order | Mempool + proposer chọn; tip + eviction documented | Pure FCFS kiểu Robinhood (payments claim **không** được consensus/fee docs chống lưng) |
| Mempool public | Docs có mempool; RPC official **không** hỗ trợ `txpool_*` | Public gossip kiểu ETH — **chưa tìm thấy bằng chứng** |

## CONFIRMED
1. Mainnet `5042`, RPC `https://rpc.mainnet.arc.io`, gas **USDC**, sàn `baseFee` tối thiểu **20 Gwei** (dưới thì silent drop).
2. Tip (`maxPriorityFeePerGas`) optional; 0 thường OK; tip nhỏ có thể giúp khi tắc — [gas-and-fees](https://docs.arc.io/arc/references/gas-and-fees).
3. Fee (base + tip) về **block beneficiary**, không burn.
4. Finality deterministic / no reorg theo docs.
5. Probe RPC 17/09: `web3_clientVersion` = `arc/v1`; `txpool_*` = method not supported; nhiều `miner` khác nhau (xoay vòng).
6. Validator founding cohort (Circle, BlackRock, DTCC, Visa, …) — press Circle 08–09/2026.

## UNCONFIRMED / CONFLICT
- Mempool live: private giữa validators vs gossip rộng hơn? Blog AMP nói “private-mempool from day one” nhưng **không có** docs vận hành.
- Thứ tự trong block: tip-desc vs FCFS vs discretionary — **không có** spec ordering chính thức.
- Chữ “sequencer” trong fee docs vs kiến trúc multi-validator PoA.
- Một số trang gas/`llms.txt` vẫn wording testnet dù mainnet đã live 16/09.

## NOT FOUND
- External builders, PBS, bundle endpoint, MEV-Share, bloXroute/EigenPhi Arc, club kiểu 48club.
- Fair ordering algorithm live (ngoài marketing).
- APS / encrypted mempool path đang dùng để include tx.

## Ý nghĩa cho bot (chỉ evidence-backed)
1. Gửi qua RPC public đã document — không có bundle/relay API documented.
2. `maxFeePerGas` ≥ 20 Gwei.
3. Tip có thể ảnh hưởng inclusion khi tắc; **đừng giả định FCFS**.
4. Đừng dựa `txpool_*` trên RPC official.
5. Order sự kiện dùng `blockNumber` + `logIndex`, không tin `block.timestamp` (sub-second, timestamp có thể trùng).
6. Sau khi vào block: docs nói irreversible.

## Việc nên verify tiếp (on-chain / full node)
- Map `miner` → tên validator.
- Sample nhiều block: tip vs vị trí tx (quan sát ≠ policy).
- Chạy full node xem pending đến bằng P2P hay chỉ RPC.
- Theo dõi AMP / private-mempool spec thật.

Nguồn chính đã đọc: docs.arc.io, Circle press/blog AMP, `circlefin/arc-node`, malachite-mempool (README: chưa production-ready), probe RPC read-only.

Muốn mình đào sâu phần nào tiếp: map validator/`miner`, sample tip-ordering trên N block, hay so sánh cụ thể với code `payload.rs`?

Intel tiếp theo cho bot backrun/arb trên Arc `5042` — probe 17/09 ICT, RPC `https://rpc.mainnet.arc.io`.

## Điểm quan trọng nhất
1. **Không có pending public** trên RPC đã probe: `txpool_*` / pending filter không hỗ trợ; WS `newPendingTransactions` fail hoặc bị chặn. **Backrun từ mempool public ≈ không làm được** trừ khi chạy node / feed riêng. **OBSERVATION**
2. **Tip bias, không phải tip-lock:** ~28–33% block tip-desc nghiêm; ~96% cặp kề tip không tăng. Tip=0 thường nằm **cuối** block. **OBSERVATION** ≠ policy.
3. **Uniswap v2/v3/v4 + Universal Router + UniswapX live** (bytecode OK). V2 `allPairsLength = 480`. **WETH: NOT FOUND.**
4. Block ~**0.52s**; 1 conf = final; timestamp hay trùng → order bằng `blockNumber` + `logIndex`.

## Tip vs vị trí (100 + 80 block)
| Metric | Kết quả |
|---|---|
| Empty | 0% |
| Avg txs/block | ~32 |
| Strict tip-desc | 28–33% |
| Adjacent tip ≥ next | ~96% |
| First tx = max tip | ~73% |
| baseFee | luôn 20 Gwei |

Có counterexample tip cao nằm sau tip thấp — đừng tin tip = slot chắc chắn.

## Miner (100 block) — 17 địa chỉ, gần round-robin
Ví dụ (mỗi cái ~5–6/100): `0xb1a1…9a1b`, `0xefd7…a4fd`, `0x7f07…db05`, … (đủ 17). **Map tên org (Visa/BlackRock…): NOT FOUND** — press chỉ có tên cohort, không gắn address.

## Địa chỉ hệ thống (CONFIRMED docs + bytecode)
- USDC: `0x3600000000000000000000000000000000000000`
- EURC: `0xbEf5f6d51CB62b58e6A8f77868681825C6fe21c1`
- CCTP domain 26 TokenMessengerV2: `0x28b5a0e9C621a5BadaA536219b3a228C8168cf5d`
- MessageTransmitterV2: `0x81D40F21F12A8F0E3252Bccb954D722d4c464B64`
- GatewayWallet / Minter: `0x77777777…00eE` / `0x2222222d…C205`
- Multicall3 / Permit2 / CREATE2: canonical

## DEX (CONFIRMED)
| | Address |
|---|---|
| Uni v2 Factory | `0x89e5db8b5aa49aa85ac63f691524311aeb649eba` |
| Uni v2 Router | `0x1f7d7550b1b028f7571e69a784071f0205fd2efa` |
| Uni v3 Factory | `0xf0db7b58379503491d857db50ac9ece64c653918` |
| SwapRouter02 | `0x53bf6b0684ec7ef91e1387da3d1a1769bc5a6f77` |
| Uni v4 PoolManager | `0x8366a39c…0951` |
| Universal Router | `0x4fca4a51ab4f23a7447b3284fbd7d73289a89fb1` |
| UniswapX Reactor | `0x0000000015134054eA82AE0bb9fda66b36402C36` |

Curve / Aero / fomo factory: **NOT FOUND** (press có tên, chưa có address chính thức).

## RPC matrix (thực dụng)
- **OK:** `eth_call`, `eth_getLogs`, `eth_getBlockReceipts`, `eth_feeHistory`, WS `newHeads`
- **Không:** `txpool_*`, pending filters, hầu hết `debug_*` / `trace_*` / `eth_simulateV1`
- Path public ≈ **newHeads → receipts/logs → simulate sau khi đã vào block**

## Code chọn tx (`payload.rs`) — CONFIRMED
Reth `best_transactions_with_attributes(base_fee, blob=None)` → không blob; seal sớm theo **deadline**; có blocklist/denylist; proposer ăn `effective_gas_price * gas_used` (base không burn).

## Fee / replace — CONFIRMED docs
- `maxFeePerGas` ≥ 20 Gwei hoặc silent drop
- Tip 0 OK; docs gợi ý ~1 Gwei khi tắc
- Nonce gap có thể treo; cancel = same-nonce resubmit (không thấy RBF % riêng của Arc)

## Next probes đáng giá
1. Chạy **arc-node** riêng để thấy pending thật
2. Map 17 miner → validator ID từ genesis/config
3. Top `to` ngoài Uniswap (có địa chỉ nóng chưa identify)
4. Alchemy auth WS pending (chưa probe có key)

Muốn đào tiếp hướng nào: full-node pending, map miner, hay quét top contract ngoài Uni?
