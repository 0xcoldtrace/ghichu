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

Đo xong latency/geo RPC Arc (evidence-only).

**Vantage đo:** máy mình đang egress AWS **us-east-1 / Ashburn, VA** (`184.193.214.93`). Số ms dưới đây là từ điểm đó — không phải từ VN.

## Kết luận đặt VPS
**Không có docs công bố city/region của validator** (chỉ nói PoA multi-region). **Không** chọn “colo cạnh validator” chỉ từ public RPC.

| Ứng viên | Độ tin | Áp dụng |
|---|---|---|
| **us-east-1 / Ashburn** | Mạnh cho **CDN edge** + PublicNode origin | `cf-ray: *-IAD`; PublicNode `x-pn-region: ash` |
| **Frankfurt (eu-central-1)** | Mạnh cho **Blockdaemon only** | TeraSwitch geofeed FRA |
| **Ottawa/Kanata, Ontario** | Mạnh cho **Pinax** | PTR + `x-pinax-node-id: kan-…` |

## Bảng đo (median ms, n=5 HTTPS)

| Endpoint | Infra | Geo | connect / TLS / TTFB |
|---|---|---|---|
| `rpc.mainnet.arc.io` | Cloudflare | Edge **IAD** (IP DB Toronto = anycast artifact) | 0.9 / 30 / **68** |
| `rpc.quicknode.mainnet.arc.io` | **cùng VIP CF với official** | IAD | 1.0 / 23 / **85** |
| `rpc.drpc.mainnet.arc.io` / `arc.drpc.org` | CF | IAD, origin unknown | TTFB ~90–108 |
| `arc-rpc.publicnode.com` | CF + PN | Edge IAD, **origin ash** | **76** |
| `rpc.arc-scan.org` | CF | IAD, origin chậm hơn | **226** |
| `5042.rpc.thirdweb.com` | CF | IAD | **109** |
| `rpc.blockdaemon.mainnet.arc.io` | **Dedicated** TeraSwitch | **Frankfurt** | TLS 275 / TTFB **548** (xa từ Ashburn) |
| `arc.rpc.pinax.network` | Dedicated | **Ontario / Kanata** | TLS 120 / TTFB **248** |
| `arc.gateway.tenderly.co` | GCP | Gần path us-east-1; city DB **UNCERTAIN** | TTFB **240** |

WSS official/QuickNode cũng OK từ Ashburn (~open 120–140 ms + rpc ~40 ms).

## CDN vs dedicated
- **CF anycast (hug user, không = validator):** official, QuickNode, dRPC, PublicNode, arc-scan, thirdweb
- **Dedicated (geo rõ hơn):** Blockdaemon = FRA; Pinax = Ontario
- Tenderly “Kansas City” trên IP DB — **bác bỏ làm city** (latency/traceroute mâu thuẫn)

## Giới hạn quan trọng
1. Edge RPC ≠ vị trí proposer/validator / private mempool  
2. Chỉ 1 vantage (Ashburn)  
3. Validator cities: **NOT FOUND** trong docs/press  

## Gợi ý thực dụng
- Nếu bot chủ yếu **submit + theo `newHeads` qua RPC public CF**: VPS **us-east-1** hợp lý để gần edge IAD (đã đo nhanh từ đó).  
- Nếu ưu tiên **Blockdaemon**: cân nhắc **Frankfurt**.  
- Nếu Pinax: **ca-central-1** hoặc Ontario.  
- Muốn chắc: đo lại từ vài VPS ứng viên (us-east-1 / eu-central-1 / ca-central-1) rồi so TTFB `eth_chainId`.

Raw: `/workspace/arc-rpc-latency/` (`measurements.csv`, `measurements.json`). Muốn mình gửi file ra chat hoặc đo thêm từ region khác không?


# Free / Public VPS & Cloud cho đo latency Arc RPC

**Ngày verify:** 2026-09-17 (Asia/Saigon, UTC+7)  
**Mục đích:** Multi-region ping/TTFB tới Arc RPC (ví dụ `eth_chainId`) — ưu tiên SSH hoặc chạy được `curl`.  
**Nguồn chính:** trang official + [ripienaar/free-for-dev](https://github.com/ripienaar/free-for-dev)  
**Quy tắc:** Không bịa provider. Không liệt kê “free VPS Telegram” khi không có link thật.

---

## Cảnh báo quan trọng

- **Telegram / X “free VPS giveaway”:** thường là malware, honeypot, steal SSH key/crypto wallet. **KHÔNG dùng** trừ khi bạn tự audit nguồn uy tín — trong lần search hôm nay **không tìm thấy giveaway community đáng tin** để liệt kê.
- Free tier **có thể đổi / hết capacity** bất cứ lúc nào; luôn đọc lại trang official trước khi signup.
- Card hầu hết bắt buộc để verify identity — không có nghĩa là sẽ bị charge nếu ở trong free limits (nhưng dễ charge nếu quên tắt resource).

---

## Bảng ~10 lựa chọn đã verify (2026-09-17)

| # | Name | Type | Official signup URL | What you get | Regions (latency-critical) | Limits / card / expire? | Good for latency? | Caveats / scam risk | Source verified today |
|---|------|------|---------------------|--------------|----------------------------|-------------------------|-------------------|---------------------|----------------------|
| 1 | **Oracle Cloud Always Free** | Always Free VM | https://signup.cloud.com/ | 2× `VM.Standard.E2.1.Micro` (1/8 OCPU, 1 GB) **và/hoặc** Ampere A1 Flex tới **2 OCPU + 12 GB** tổng; 200 GB block; 10 TB egress/mo | **Chỉ home region** lúc signup (chọn kỹ). Có Toronto (`ca-toronto-1`), Frankfurt (`eu-frankfurt-1`), Ashburn, v.v. trong OCI commercial regions | Card/debit (credit-like) bắt buộc verify; Always Free **không hết hạn**; idle VM có thể bị reclaim; capacity “out of host” thường gặp | **SSH: YES** — top pick VPS | Capacity khó; A1 limit đã giảm còn 2 OCPU/12 GB (Always Free); idle reclaim | [Always Free docs](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm), [FAQ](https://www.oracle.com/cloud/free/faq/) |
| 2 | **Google Cloud Always Free e2-micro** | Always Free VM | https://cloud.google.com/free | 1× non-preemptible **`e2-micro`** (~2 shared vCPU, **1 GB RAM**); 30 GB-months disk; 1 GB NA egress/mo | **Chỉ US:** `us-west1`, `us-central1`, `us-east1` | Card khi Free Trial; Always Free **không expire** nếu billing account active; cần Paid billing để giữ sau trial | **SSH: YES** — tốt cho **us-east** | **Không free** ở Europe/Canada/Asia; e2-micro ngoài 3 region = mất tiền | [Free cloud features](https://docs.cloud.google.com/free/docs/free-cloud-features) (updated 2026-09-15) |
| 3 | **AWS Free Tier EC2** | Trial / credit (không còn always-free EC2 dài hạn kiểu cũ cho account mới) | https://aws.amazon.com/free/ | Account **≥ 2025-07-15:** tới **$200 credit / tối đa 6 tháng** (Free plan); EC2 eligible: `t3.micro/small`, `t4g.micro/small`, … Account cũ: 12 tháng t2/t3.micro | **Multi-region** gồm `us-east-1`, `eu-central-1` (Frankfurt), `ca-central-1` (Canada) — chọn khi launch | Card; Free plan tự đóng sau 6 tháng / hết credit; upgrade Paid nếu muốn tiếp | **SSH: YES** — tốt multi-region **trong thời gian credit** | Model đổi mạnh 2025-07; đọc kỹ Free vs Paid plan; xóa EBS/EIP tránh bill | [AWS Free](https://aws.amazon.com/free/), [EC2 Free Tier usage](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-free-tier-usage.html) |
| 4 | **Azure Free Account VMs** | Trial 12 tháng (không Always Free VM) | https://azure.microsoft.com/en-us/pricing/purchase-options/azure-account | **750 giờ/tháng × 12 tháng** cho B-series free: **B1s**, **B2pts v2** (ARM), **B2ats v2** (AMD) | Nhiều region (East US, Germany West Central / Frankfurt area, Canada Central…) — phụ thuộc capacity & SKU subscription | Card; **12 tháng** rồi trả phí; disk/IP có thể phát sinh phí; Free Trial đôi khi `NotAvailableForSubscription` với B1s | **SSH: YES** (Linux) | Không phải always-free; SKU/region intermittent | [Free services](https://azure.microsoft.com/en-us/pricing/free-services), [Create free services](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/create-free-services) |
| 5 | **Alibaba Cloud ECS Free Trial** | Trial | https://www.alibabacloud.com/ (Free Trial Center / ECS free trial) | Quota **CNY 300** (personal) / **CNY 660** (enterprise) cho ECS+system disk; traffic 20 GB CN + 200 GB overseas/mo | **7 region trial:** Beijing, Hangzhou, Guangzhou, Chengdu, Ulanqab, Heyuan, **Hong Kong** | Identity verify; **3 tháng** validity; vượt quota = pay-as-you-go; **phải tự release** khi hết | **SSH: YES** — tốt Asia/HK | Không cover US-East/Frankfurt/Ontario; dễ quên instance → bill | [ECS free trial guide](https://help.aliyun.com/en/ecs/user-guide/ecs-free-trial) |
| 6 | **Render Free Web Service** | PaaS | https://render.com/docs/free | Free web service **0.1 CPU / 512 MB**; 750 Free instance-hours/mo | **Oregon, Ohio, Virginia, Frankfurt, Singapore** | Spin down sau **15 phút idle**; spin-up ~1 phút; **không SSH**; card optional (cần nếu vượt bandwidth) | **Curl script: YES** (HTTP service / cron-like wake). **SSH: NO** | Sleep làm lệch TTFB cold-start; không đo được như VPS ổn định trừ khi giữ warm | [Render Free](https://render.com/docs/free), [Regions](https://render.com/docs/regions) |
| 7 | **Railway Trial + Free plan** | Trial / PaaS | https://railway.com/ | Trial: **$5 / 30 ngày**; sau đó Free plan **$1 credit/tháng** (không roll-over); trial max ~1 GB RAM shared | US West, US East (Virginia), EU (Amsterdam), Singapore (theo docs Railway) | GitHub verify cho Full Trial (outbound); Limited Trial **hạn chế network** — kém cho ping RPC | Curl nếu Full Trial. **SSH: limited/no classic VPS** | $1/mo quá ít cho always-on; Limited Trial = outbound kém | [Railway free trial](https://docs.railway.com/pricing/free-trial) |
| 8 | **Cloudflare Workers Free** | Edge serverless (**≠ VPS**) | https://developers.cloudflare.com/workers/ | **100k requests/day**; 10 ms CPU/invocation | **Global edge** (hàng trăm PoP) — không chọn “1 VPS region” | Free mặc định; không SSH | **HTTP latency từ edge: YES**. **SSH/VPS: NO** | Đo được RTT edge→RPC, **không** đại diện egress VPS bot colocated | [Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/) |
| 9 | **GitHub Codespaces** | Codespace / temp | https://github.com/features/codespaces | Free personal: **120 core-hours/mo** + 15 GB-month storage (≈ 60h trên 2-core) | Host trên Azure (region do GitHub chọn; không full control như EC2) | Hết quota thì block nếu không có payment method | **Curl trong terminal: YES**. Không phải persistent VPS | Egress region không ổn định cho so sánh lâu dài; idle timeout | [Codespaces billing](https://docs.github.com/en/billing/concepts/product-billing/github-codespaces) |
| 10 | **Google Cloud Shell** | Temp shell | https://shell.cloud.google.com/ (qua GCP console) | Linux shell + **5 GB** `$HOME`; free | Auto gán region gần — **không chọn region** | **~50 giờ/tuần**; session idle/non-interactive cut; không dùng mining/scan | **Curl nhanh: YES**. Multi-region control: **NO** | Không thay VPS multi-region; chỉ smoke test | [Cloud Shell quotas](https://cloud.google.com/shell/docs/quotas-limits) |

### Bonus / hạn chế mạnh (không đếm là “solid always-free VPS”)

| Name | Status 2026-09-17 | Note |
|------|-------------------|------|
| **Fly.io** | Free tier **ENDED** cho account mới; chỉ **Free Trial: 2 machine-hours hoặc 7 ngày**, auto-stop 5 phút | [fly.io/docs/about/free-trial](https://fly.io/docs/about/free-trial/) — dùng được vài phút multi-region rồi hết |
| **IBM Cloud Lite** | **KHÔNG** có Always Free VPC/VSI thông thường; có **$200 / 30 ngày** + promo VPC 70% off (Madrid/Osaka/São Paulo tới 2026-12-31) | [ibm.com/products/cloud/free](https://www.ibm.com/products/cloud/free) — **NOT FOUND** free persistent VM kiểu Oracle/GCP |
| **Huawei Cloud free packages** | Campaign/free packages theo account — trang intl trả về ít nội dung tĩnh; **phải tự check** Billing → Free Packages sau signup | [activity.huaweicloud.com/.../free_packages](https://activity.huaweicloud.com/intl/en-us/free_packages/index.html) — không khẳng định ECS free cố định |
| **Community TG/X free VPS** | **NOT FOUND** link uy tín trong lần search này | **Scam risk cực cao** |

---

## Ranked top picks cho Arc multi-region (us-east-1, Frankfurt, Ontario)

Giả sử bạn cần so latency tới RPC gần **US East / Frankfurt / Canada (Ontario)**:

| Rank | Pick | Region fit | Vì sao |
|------|------|------------|--------|
| **1** | **Oracle Always Free** | Chọn home = **Toronto** *hoặc* **Frankfurt** *hoặc* Ashburn lúc signup | SSH thật, lâu dài, RAM A1 tốt; **1 account = 1 home region** → cần nhiều account/org hợp lệ nếu muốn 3 region (Oracle cấm multi free abuse) |
| **2** | **AWS Free credits (6 tháng)** | Launch `t3/t4g.micro` ở `us-east-1`, `eu-central-1`, `ca-central-1` | Best multi-region **trong cửa sổ credit**; đo xong **terminate** |
| **3** | **GCP e2-micro** | `us-east1` (South Carolina) | Solid cho **US East only**; không cover FRA/Ontario free |
| **4** | **Azure 12-mo B-series** | East US / Germany West Central / Canada Central | SSH multi-region trong 12 tháng; theo dõi disk/IP |
| **5** | **Render Free** | Virginia + Frankfurt (+ Oregon/Ohio/SG) | Nhanh standup HTTP probe; cold-start làm nhiễu số liệu |
| **6** | **Alibaba trial (HK)** | Hong Kong / CN | Bổ sung **Asia**; không thay US/EU/CA |
| **7** | **Workers / Codespaces / Cloud Shell** | Edge / Azure / nearest GCP | Chỉ phụ / smoke — không thay VPS bot |

### Honest shortfall

Không có “10 Always Free VPS multi-region bền” từ reputable cloud năm 2026.  
Thực tế solid SSH free/trial: **Oracle + GCP (US) + AWS credit window + Azure 12mo (+ Alibaba Asia)**. PaaS/edge thêm 3–4 slot đo phụ. **Thiếu ~3–5 “true multi-region free VMs”** so với mục tiêu 10 VPS thuần.

### Alternative trả phí rẻ (khuyến nghị khi cần số liệu sạch)

- **Hetzner Cloud** (EU: Falkenstein/Nuremberg/Helsinki) — VM nhỏ ~€5/tháng; thêm location US nếu cần (plans khác).  
- Spot/hourly **Vultr / DigitalOcean / Linode** ~$4–6/tháng × 3 region (NYC/NJ ≈ us-east, Frankfurt, Toronto nếu có) → chạy script 1 ngày rồi destroy.  
Chi phí một lần đo multi-region thường **<$20**.

---

## Phương pháp đo thực tế (cùng script `eth_chainId`)

Chạy **cùng một script** từ mỗi VPS/PaaS tới **cùng danh sách Arc RPC URL**.

```bash
#!/usr/bin/env bash
# arc-latency.sh — đo TTFB + total time eth_chainId
set -euo pipefail
RPC_URLS=(
  "https://YOUR-ARC-RPC-1"
  "https://YOUR-ARC-RPC-2"
)
REGION_LABEL="${REGION_LABEL:-unknown}"
N="${N:-20}"
OUT="arc-latency-${REGION_LABEL}-$(date -u +%Y%m%dT%H%M%SZ).csv"
echo "ts_utc,region,rpc,http_code,time_namelookup,time_connect,time_appconnect,time_starttransfer,time_total" > "$OUT"

for rpc in "${RPC_URLS[@]}"; do
  for i in $(seq 1 "$N"); do
    ts=$(date -u +%Y-%m-%dT%H:%M:%SZ)
    # shellcheck disable=SC2016
    line=$(curl -sS -o /tmp/arc_body.json -w '%{http_code},%{time_namelookup},%{time_connect},%{time_appconnect},%{time_starttransfer},%{time_total}' \
      -H 'content-type: application/json' \
      --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}' \
      --max-time 10 \
      "$rpc" || echo "000,0,0,0,0,0")
    echo "$ts,$REGION_LABEL,$rpc,$line" >> "$OUT"
    sleep 0.2
  done
done
echo "Wrote $OUT"
```

**Cách dùng:**

1. Trên mỗi VPS: `export REGION_LABEL=us-east-1` (hoặc `eu-frankfurt-1`, `ca-toronto-1`).  
2. Chạy cùng `N` (ví dụ 20–50), cùng giờ UTC nếu có thể.  
3. So cột **`time_starttransfer` (TTFB)** và **`time_total`**; bỏ mẫu `http_code != 200`.  
4. Trên Render/Workers: bọc script trong HTTP handler hoặc cron; ghi nhớ **cold start**.  
5. Ghi lại IP egress / ASN nếu cần phân biệt CDN vs origin RPC.

---

## Nguồn list cộng đồng (không phải VPS)

- https://github.com/ripienaar/free-for-dev — section Major Cloud Providers (Oracle/GCP/AWS/Azure/IBM/CF)  
- Official docs đã link trong bảng (verify 2026-09-17)

---

## Checklist nhanh cho user (VI)

1. Signup **Oracle** → chọn home region gần RPC candidate (Toronto / Frankfurt / Ashburn).  
2. Signup **GCP** → e2-micro `us-east1` cho US East.  
3. Signup **AWS** Free plan → trong 6 tháng credit launch 3 micro ở us-east-1 / eu-central-1 / ca-central-1 → đo → **terminate**.  
4. (Optional) Azure B-series 12 tháng nếu cần thêm điểm.  
5. Bổ sung Asia: Alibaba HK trial.  
6. Smoke: Cloud Shell / Codespaces / Workers.  
7. **Tránh** random free VPS Telegram/X.  
8. Nếu cần số liệu production: Hetzner + 1–2 VPS US/CA trả phí 1 ngày.

