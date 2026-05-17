---
layout: ../../layouts/Article.astro
title: "Decoding TicketOptions: Bitmask Analysis for Silver Ticket Detection"
date: 2026-04-15
tags: [Kerberos, Active Directory, Splunk]
excerpt: "Deep dive into Kerberos TicketOptions bitmask forensics — what the bits reveal, and why join-based Silver Ticket detection fails in production Splunk."
draft: true

---

Silver Ticket attacks are notoriously difficult to detect at scale. Unlike Golden Tickets, they never touch the Domain Controller — the forged TGS goes directly to the target service. The only artifacts you have are on the host and in Windows Security Event logs.

## What TicketOptions Tells Us

Event ID 4769 (Kerberos Service Ticket Requested) includes a `TicketOptions` field — a 32-bit bitmask. Each bit encodes a flag from RFC 4120. Legitimate Kerberos requests have predictable patterns; forged tickets often don't.

```spl
| eval ticket_flags  = tonumber(TicketOptions, 16)
| eval forwardable   = if(bitwiseAND(ticket_flags, 0x40000000) > 0, 1, 0)
| eval renewable     = if(bitwiseAND(ticket_flags, 0x00800000) > 0, 1, 0)
| eval pre_authent   = if(bitwiseAND(ticket_flags, 0x00200000) > 0, 1, 0)
```

The key insight: Silver Tickets minted with Mimikatz default to `0x40810010`. This value has `forwardable` set but `pre-authent` absent — unusual for normal service ticket requests from domain-joined machines.

## Why the Join Approach Fails

The intuitive detection joins 4769 events missing a corresponding 4768 (no TGT = forged ticket). In theory, elegant.

In production Splunk across a multi-tenant MSSP environment, the join reliability breaks down beyond ~10k events. The subsearch window caps out, and you get false negatives at exactly the volume where it matters.

```spl
| join type=outer ServiceName
    [ search index=wineventlog EventCode=4768 | fields ServiceName, _time ]
```

This works in a lab. It doesn't work on a client generating 50k auth events per hour.

## Better Approach: Baseline Distributions

Instead of correlation-by-join, baseline the `TicketOptions` distribution per `ServiceName` over 30 days. A Silver Ticket will introduce a new value into a service's distribution — detectable as a statistical outlier.

```spl
index=wineventlog EventCode=4769
| stats count by ServiceName, TicketOptions
| eventstats avg(count) as avg_count, stdev(count) as stdev_count by ServiceName
| where count < (avg_count - 2*stdev_count)
```

More robust, lower noise, and it doesn't depend on Splunk join semantics.

## Production Status

This detection exists as a hunt query, not a scheduled alert — the baseline period needs tuning per environment before it's stable enough to page on.
