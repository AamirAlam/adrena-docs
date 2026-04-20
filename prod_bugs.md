## Prod bugs 

```
 ---
 Remaining Bugs (verified by reading source)

 🔴 DEFINITELY HAPPENS

 1. mrSablierStaking — resolve error silently continues
 - src/client.rs:685-696 (read directly)
 if let Err(e) = handlers::resolve_staking_round::resolve_staking_round(...).await {
     log::error!("Error resolving staking round: {}", e);
     // cache never updated, Ok(()) returned — same round retried every 1 second
 }
 - process_resolve_staking_rounds is called on every tick; on persistent failure, infinite 1-per-second TX spam loop
 - Fix: on error path, still advance next_resolve_time in cache (or add per-key backoff) to avoid retry storm

 2. mrSablierStaking — TX sent with no confirmation wait
 - src/handlers/resolve_staking_round.rs:78-80 (read directly)
 // TODO wait for confirmation and retry if needed
 Ok(())
 - TX is sent, signature logged, function returns Ok(()) immediately — no await on confirmation
 - Combined with bug #1: if TX lands but takes >1s, the same round gets retried on next tick
 - Fix: confirm TX signature before updating cache / returning success

 🟠 HIGH PROBABILITY ON RECONNECT (70-85%)

 3. mrSablier — panic on stream reconnect
 - src/process_stream_message.rs:115-119 (read directly)
 } else if msg.filters.contains(&"positions_close".to_owned()) {
     match update {
         PositionUpdate::Created(_) => { panic!("New position created in positions_close filter"); }
         PositionUpdate::Modified(_) => { panic!("New position created in positions_close filter"); }
 - gRPC reconnects can replay Created events; positions_close filter uses account whitelist but Yellowstone can still emit Created on reconnect
 - Fix: replace both panic! with log::warn!(...) and continue (or return Ok(()))

 4. mrSablierStaking — panic on stream reconnect
 - src/process_stream_message.rs:68,81 (read directly)
 StakingAccountUpdate::Created(_) => {
     panic!("Staking account created in staking_create_update filter");
 }
 StakingAccountUpdate::Closed => {
     panic!("Staking account closed in staking_create_update filter");
 }
 - Same replay-on-reconnect risk
 - Fix: same — replace both panic! with log::warn! + graceful return

 🟡 PLAUSIBLE (IDL mismatch / catchup replay)

 5. adrena-data processor — eventData null dereference
 - processor/src/parse/position/parseAdditionalInfosPosition.ts:197-200 (read directly)
 // eventData comes from getEventData(...) which CAN return null
 // Lines 163-183 correctly guard with: if (eventData && ...)
 // But line 197 has NO guard:
 additionalInfos.size = nativeToUi(eventData.sizeUsd, 6);
 additionalInfos.collateralAmount = nativeToUi(eventData.collateralAmountUsd, 6);
 // Then line 208 correctly guards again: if (eventData && typeof eventData.poolType...)
 - eventData is null when the event is missing from the transaction (IDL mismatch, old catchup txns)
 - TypeError: "Cannot read properties of null" — drops the whole instruction parse for that TX
 - Fix: wrap lines 197-201 in if (eventData) { ... } to match the guard pattern used on lines 163-210

 🟢 WASTEFUL / EDGE CASE

 6. mrSablier — fee tier thresholds extremely low
 - src/handlers/liquidate_long.rs:341-365 (read directly)
 const MAX_FEE_CALC_SIZE: u64 = 2_500_000_000000; // $2.5M cap (at 1e6 scale)
 loss if loss > 2_500_000000 => ultra,  // $2,500 at 1e6 scale (was probably intended ~$250k)
 loss if loss > 500_000000  => high,    // $500 at 1e6 scale (was probably intended ~$50k)
 - Effect: nearly every liquidation with any unrealized loss gets ultra priority fee → service overpays fees, not correctness failure
 - Fix: clarify intended thresholds with team, then correct multipliers

 7. adrena-data cron — Infinity timestamp infinite loop
 - cron/process/processAutonomAssetPrices.ts:124-130
 while (normalized > YEAR_2100_SECONDS) {
   normalized = Math.floor(normalized / 1000); // Infinity / 1000 = Infinity
 }
 - Only triggers on malformed Infinity from Autonom API response
 - Fix: if (!isFinite(normalized)) throw new Error('invalid timestamp')

 ---
 Recommended Fix Order

 1. mrSablierStaking client.rs:685-696 — add backoff/cache-update on resolve failure (DEFINITELY HAPPENS)
 2. mrSablier process_stream_message.rs:115-119 — panic! → log::warn! (70-85% on reconnect)
 3. mrSablierStaking process_stream_message.rs:68,81 — same (plausible on reconnect)
 4. mrSablierStaking resolve_staking_round.rs:78 — await TX confirmation
 5. processor parseAdditionalInfosPosition.ts:197-200 — add if (eventData) guard
 6. mrSablier liquidate_long.rs thresholds — fix fee tier values (confirm intent first)
 7. cron processAutonomAssetPrices.ts — add isFinite check
```
