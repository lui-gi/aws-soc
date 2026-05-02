# Phase 3 -- Splunk Ingestion and SPL Fundamentals

Skills demonstrated through hands-on lab work and self-assessment (2026-05-02).

---

## Q1: Splunk field extraction failure with KV_MODE = none

**Question:** A custom Splunk TA has `KV_MODE = none` and `EXTRACT-` regex stanzas in `props.conf`. Events land in the index but all fields are empty. What is the root cause and fix?

**My answer:** The query is wrong -- the fields are not matching the object type. You need to run a query that returns all default fields first, then go from there.

**Correction:** The query is not the issue. `KV_MODE = none` is an explicit instruction to Splunk to skip all key-value extraction. With that set, the `EXTRACT-` regex stanzas do not fire reliably against EVE JSON, so nothing gets parsed at index time -- there are no fields to query in the first place.

**Correct answer:** Replace `KV_MODE = none` and all `EXTRACT-` stanzas with `KV_MODE = json`. This tells Splunk to natively parse the EVE JSON format and automatically extract all top-level and nested fields (including `alert.signature`, `alert.severity`, etc.). Running `| head 1 | table *` is useful to confirm fields exist after the fix, but not before -- there is nothing to confirm until extraction is working.

---

## Q2: CloudTrail index empty despite active logging -- two root causes

**Question:** After restarting Splunk, `index=aws_flowlogs` works but `index=aws_cloudtrail` returns zero results. What are the two distinct root causes?

**My answer (Root Cause 1):** Check whether the S3 bucket is actively sending CloudTrail notifications to SQS.

**Correct (Root Cause 1):** The S3 bucket had an event notification for the VPC Flow Logs prefix but no notification for the CloudTrail prefix. SQS never received messages, so Splunk had nothing to pull. Fix: add a new S3 event notification scoped to `AWSLogs/.../CloudTrail/` pointing at the `soc-lab-cloudtrail-notifications` SQS queue.

**My answer (Root Cause 2):** The KV Store did not fully initialize when the CloudTrail input tried to register its task. We disabled and re-enabled the data input in the Splunk UI and restarted Splunk.

**Correction:** No Splunk restart was needed. Disabling and re-enabling the `cloudtrail-logs` input in the Splunk UI (Settings -> Data Inputs -> AWS SQS-Based S3) was sufficient. That forced the input to re-register its task in KV Store on the next poll cycle. The flowlogs input continued working because it had already registered before the 503 error occurred.

---

## Q3: VPC Flow Logs saved search returning no results due to wrong field names

**Question:** A saved search filters on `action=REJECT` and references fields `src_addr`, `dst_addr`, `dst_port`, and `protocol`. It returns no results. Why, and what are the correct field names?

**My answer:** The field names are incorrect. You can discover the correct ones by running `action=REJECT | head 1 | table *` to see the full field list for one event.

**Correction:** `action=REJECT` is also wrong -- the Splunk_TA_aws add-on normalizes REJECT to `blocked`, so the filter must be `action=blocked`. Without that fix, even `| head 1 | table *` would return nothing. The correct approach is `index=aws_flowlogs | head 1 | table *` with no action filter, then read the field names from the result.

**Correct field names:**
- `src_ip` (not `src_addr`)
- `dest_ip` (not `dst_addr`)
- `dest_port` (not `dst_port`)
- `transport` (not `protocol`)
- `action=blocked` (not `action=REJECT`)

---

## Q4: SPL regex filter to isolate external IP traffic

**Question:** What does `| where NOT match(src_ip, "^10\.")` do, and why is it useful in this lab?

**My answer:** It returns all IP addresses that do not begin with `10.` The regex anchors to the start of the string and escapes the dot. This isolates public/external IPs because the lab runs in a private `10.0.0.0/16` subnet.

**Result:** Correct. The `NOT match(...)` inverts the regex so only non-RFC-1918 addresses from the `10.*` range pass through. This filters out internal lab traffic and surfaces external internet sources -- useful for the "External IPs Hitting Network" dashboard panel.

---

## Q5: Traffic mirroring as a false positive in Suricata

**Question:** `10.0.137.180` appears in Suricata alerts sending ICMP to internal hosts. Why is this not a threat?

**My answer:** Unsure.

**Correct answer:** This IP is the AWS VPC Traffic Mirroring infrastructure. Traffic mirroring works by copying packets from a source ENI and delivering them (VXLAN-encapsulated) to the mirror target -- in this case the Suricata instance. Suricata sees the mirror target's IP as the source of the copied packets, which looks like internal ICMP scanning but is just the mirroring mechanism. The tell is a consistent source IP hitting multiple internal hosts in a sweep-like pattern. In a production SOC this would be documented as a known false positive and suppressed in alerting rules.

---

## Summary

| Area | Result |
|------|--------|
| Splunk KV extraction modes | Needs review -- confused query debugging with config root cause |
| CloudTrail dual root cause diagnosis | Strong -- identified both causes; minor correction on restart vs. disable/re-enable |
| Field name discovery via `table *` | Strong on approach; missed that `action=REJECT` also needed correction |
| SPL regex for traffic filtering | Correct |
| VPC traffic mirroring as false positive | Not yet familiar -- review AWS Traffic Mirroring architecture |
