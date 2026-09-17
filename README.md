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
