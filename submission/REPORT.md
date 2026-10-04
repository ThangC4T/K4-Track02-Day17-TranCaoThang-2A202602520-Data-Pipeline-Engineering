# K4-Track02-Day17 â€” Report cĂ¡ nhĂ¢n

**Há» tĂªn / MSSV:** Tran Cao Thang / 2A202602520  
**Repo:** https://github.com/ThangC4T/K4-Track02-Day17-TranCaoThang-2A202602520-Data-Pipeline-Engineering  
**Commit bai nop:** commit moi nhat tren nhanh `main` sau khi push  
**AI Ä‘Ă£ dĂ¹ng vĂ  pháº¡m vi há»— trá»£:** Codex há»— trá»£ Ä‘á»c Ä‘á», xĂ¡c Ä‘á»‹nh lá»—i, sá»­a code trong `pipeline/`, cháº¡y kiá»ƒm chá»©ng vĂ  soáº¡n report.  
**Nguá»“n tham kháº£o khĂ¡c:** README, RUBRIC, CHECKPOINTS, SUBMISSION trong repo.

## 1. Ba lá»—i

| | Lá»—i Silver | Lá»—i late data | Lá»—i xoĂ¡ (CDC) |
|---|---|---|---|
| **Triá»‡u chá»©ng** | `verify` bĂ¡o `silver_tickets` cĂ³ 24 rows cho 12 tickets; T-91 tráº£ vá» 3 tráº¡ng thĂ¡i thay vĂ¬ tráº¡ng thĂ¡i cuá»‘i. | `gold_feature_daily` lá»‡ch full recompute; u05 ngĂ y 2026-08-12 khĂ´ng Ä‘á»§ event offline; `LOOKBACK_DAYS=0 < 3`. | T-97 khĂ´ng thĂ nh tombstone vĂ¬ delete Debezium cĂ³ `after=null`; latest training snapshot vĂ  RAG index cĂ²n nguy cÆ¡ giá»¯ dá»¯ liá»‡u Ä‘Ă£ xoĂ¡. |
| **NguyĂªn nhĂ¢n gá»‘c** | `upsert_silver_tickets` append báº±ng `INSERT`, khĂ´ng cĂ³ key vĂ  khĂ´ng cháº·n batch cÅ© ghi Ä‘Ă¨ batch má»›i. | Gold chá»‰ recompute Ä‘Ăºng ngĂ y cháº¡y, trong khi Bronze Ä‘o Ä‘Æ°á»£c P99 lateness = 3 ngĂ y. | `ticket_changes_sql` láº¥y `ticket_id` vĂ  cĂ¡c field tá»« `after`; vá»›i `op='d'`, `after` null nĂªn báº£n ghi delete bá»‹ rÆ¡i máº¥t. |
| **CĂ¡ch sá»­a** | `pipeline/silver.py`: Ä‘á»•i sang `MERGE INTO silver_tickets ON ticket_id`, update chá»‰ khi `s._lsn > t._lsn`, insert khi chÆ°a cĂ³. | `pipeline/config.py`: Ä‘áº·t `LOOKBACK_DAYS = 3` theo `ceil(P99)`, giá»¯ overwrite-partition trong `gold_feature_daily`. | `pipeline/staging.py`: láº¥y key báº±ng `coalesce(after.ticket_id, before.ticket_id, key.ticket_id)`; vá»›i delete, clear PII/text Ä‘á»ƒ Silver lÆ°u tombstone. |
| **KhĂ¡i niá»‡m trĂªn slide** | Silver cĂ³ khoĂ¡, keyed MERGE, LSN guard, idempotent replay. | Late data, event time khĂ¡c ingest time, lookback Ä‘o tá»« Bronze. | CDC log-based, Debezium delete, tombstone, xoĂ¡ pháº£i lan xuá»‘ng Gold. |

## 2. CĂ¡c con sá»‘

- P99 lateness Ä‘o tá»« Bronze: `3.00` ngĂ y â†’ `LOOKBACK_DAYS = 3`
- `submission/checksums.txt`: PASS â€” Gold checksum: `39e115c510ecdf526800eac227158a4f`
- `make parity`: PARITY

## 3. Lá»±a chá»n cĂ´ng cá»¥ / ká»¹ thuáº­t

- MERGE theo khoĂ¡ cho `silver_tickets`, overwrite-partition cho `gold_feature_daily`: ticket lĂ  entity mutable cáº§n má»™t row hiá»‡n táº¡i, cĂ²n feature daily lĂ  aggregate theo partition ngĂ y nĂªn recompute cá»­a sá»• lookback Ä‘Æ¡n giáº£n vĂ  idempotent.
- Tombstone thay vĂ¬ xoĂ¡ háº³n hĂ ng trong Silver: giá»¯ dáº¥u váº¿t delete vĂ  `_lsn` Ä‘á»ƒ replay CDC cÅ© khĂ´ng lĂ m ticket sá»‘ng láº¡i.
- Snapshot training dá»±ng láº¡i tá»« Bronze "as of" ngĂ y Ä‘Ă³, khĂ´ng sá»­a snapshot cÅ©: báº£o toĂ n tĂ­nh reproducible/versioned cá»§a dataset train; náº¿u cáº§n tuĂ¢n thá»§ right-to-erasure thĂ¬ thĂªm policy redaction/expiry cho snapshot cÅ©.
- DuckDB (lite) / dbt (track dbt) cho bĂ i toĂ¡n cá»¡ nĂ y, chá»© khĂ´ng pháº£i Spark: dá»¯ liá»‡u seed nhá», cáº§n cháº¡y zero-key local nhanh; dbt dĂ¹ng Ä‘á»ƒ chá»©ng minh cĂ¹ng logic báº±ng SQL contract/microbatch.

## 4. Hai cĂ¢u há»i suy ngáº«m

1. Snapshot báº¥t biáº¿n vĂ  quyá»n Ä‘Æ°á»£c xoĂ¡ dá»¯ liá»‡u cĂ³ xung Ä‘á»™t. CĂ¡ch xá»­ lĂ½ thá»±c táº¿: giá»¯ metadata version/checksum báº¥t biáº¿n, nhÆ°ng tĂ¡ch ná»™i dung PII ra vĂ¹ng cĂ³ thá»ƒ redaction hoáº·c mĂ£ hoĂ¡ theo subject key; khi cĂ³ delete request thĂ¬ thu há»“i key hoáº·c cháº¡y redaction job cĂ³ audit trail. Snapshot cÅ© khĂ´ng nĂªn tiáº¿p tá»¥c phá»¥c vá»¥ training náº¿u cĂ²n chá»©a dá»¯ liá»‡u ngÆ°á»i dĂ¹ng Ä‘Ă£ xoĂ¡.
2. Regex chá»‰ báº¯t email/phone nĂªn chÆ°a Ä‘á»§. TĂ´i sáº½ Ä‘áº·t PII gate á»Ÿ Silver cho má»i text rá»i Bronze, bá»• sung NER/allowlist cho tĂªn ngÆ°á»i, Ä‘á»‹a chá»‰, mĂ£ Ä‘á»‹nh danh; Ä‘o báº±ng test seed cĂ³ nhĂ£n PII, sampling thá»§ cĂ´ng, tá»· lá»‡ false negative, vĂ  quarantine cĂ¡c dĂ²ng rá»§i ro cao.

## 5. Output

```text
$ python -m scripts.verify
=== verify.py â€” Day 17 pipeline contracts ===
  [OK ] Bronze  every daily batch landed as Parquet (7 days x 3 sources)
  [OK ] Bronze  re-landing a batch is a no-op (append-only, no duplicate file)
  [OK ] Bronze  Bronze keeps the raw truth: Kafka tombstone + redelivered events are still there
  [OK ] Silver  silver_tickets has exactly one row per ticket_id
  [OK ] Silver  T-91 shows its latest state: high / closed / bug
  [OK ] Silver  deleted ticket T-97 is a tombstone: is_deleted and no personal data left
  [OK ] Silver  no email / phone number survives past Bronze
  [OK ] Silver  silver_events has one row per event_id (Kafka redeliveries removed)
  [OK ] Silver  2 malformed events quarantined with a reason; the run did not halt
  [OK ] Gold    gold_feature_daily reconciles with a full recompute from Silver
  [OK ] Gold    u05's offline events of 08-12 (arrived 08-15) are counted on 08-12
  [OK ] Gold    LOOKBACK_DAYS covers measured P99 lateness (p99=3.00 days)
  [OK ] Gold    training set uses point-in-time priority (T-91 created as 'low')
  [OK ] Gold    late feedback creates a NEW snapshot version; the old one is untouched
  [OK ] Gold    latest training snapshot excludes the deleted ticket T-97
  [OK ] Gold    deletes propagate to the RAG index: no chunk of T-97
  [OK ] Gold    gold_doc_chunks: one row per chunk, and a re-run embeds 0 new chunks
  [OK ] Rerun   re-run 2026-08-12 three times -> Gold checksum identical to a fresh build

RESULT: 18/18 checks â€” ALL PASS
re-run checksums written to submission/checksums.txt

$ python -m pytest
..................................                                       [100%]
34 passed in 2.86s

$ python -m scripts.rerun_check
# Lab 17 â€” re-run check for 2026-08-12

run                     gold_feature_daily    gold_training_set     gold_doc_chunks       gold (combined)
fresh build             8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #1 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #2 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #3 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f

RESULT: PASS â€” 3 re-runs, identical checksums

$ python main.py --lateness
event lateness over 43 Bronze records (calendar days): p50=0.00 p95=2.90 p99=3.00 max=3
-> lookback must be >= ceil(p99) = 3 day(s); config.LOOKBACK_DAYS = 3

$ dbt build --profiles-dir . --event-time-start 2026-08-10 --event-time-end 2026-08-17
Completed successfully
Done. PASS=19 WARN=0 ERROR=0 SKIP=0 NO-OP=0 REUSED=0 TOTAL=19

$ python -m scripts.parity
=== parity: lite pipeline vs dbt ===
  [OK ] silver_tickets       lite 3c15dfd43701  dbt 3c15dfd43701
  [OK ] gold_feature_daily   lite 8630e04a61d1  dbt 8630e04a61d1
RESULT: PARITY â€” both implementations agree
```
