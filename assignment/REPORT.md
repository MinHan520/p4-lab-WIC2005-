# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **5016**

All addresses below are my own block, `10.16.0.0/16` (h1 `10.16.0.1`, h2 `10.16.0.2`, h3 `10.16.0.3`).

## 1. Analysis: the loop

**How I made the loop.** I started the network with the finished lab program (`python3 topo.py --id 5016`), which I had saved as `tunnel_v1.p4`. On s2 I dumped `MyIngress.myTunnel_exact`: the entry for key `0003` (tunnel exit 3, h3) had handle `2` and sent packets out port 3. I changed it to port 1, back towards s1:

```
./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"
```

s1 already sends exit 3 out port 2 towards s2, so a packet for exit 3 now goes s1 → s2 → s1 → s2 … forever. I sent one packet with `h1 python3 send.py 10.16.0.3 "loop" --tunnel 3`.

**What the logs showed** (`evidence/loop-before.txt`). I counted `Processing packet received` lines in each switch log twice, 5 seconds apart:

| Reading | s1 | s2 |
|---|---|---|
| Count 1 | 649 | 647 |
| Count 2 (5 s later) | 1318 | 1316 |
| Growth in 5 s | +669 | +669 |

One packet produced about 134 receptions per second on each switch, and the two switches grew in lockstep because every bounce passes through both. The packet never stops. On the tunnel path `myTunnel_forward` only sets `egress_spec`: nothing in the packet changes from hop to hop, so nothing can run out.

**Does the IPv4 TTL stop an IPv4 loop in this program? No.** `ipv4_forward` does:

```p4
standard_metadata.egress_spec = port;
hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
```

It decrements the TTL but never checks it and never drops. `ttl` is `bit<8>`, and P4 arithmetic wraps at the field's width. A packet arriving with TTL 1 is forwarded with TTL 0. One arriving with TTL 0 is forwarded with TTL 255. In a loop the TTL just cycles 255 → 0 → 255 and the packet circulates forever, exactly like the tunnel packet. A real router drops a packet whose TTL would reach 0 and returns ICMP Time Exceeded (Week 3); this program has neither the check nor the drop. Also, a tunnelled packet never runs `ipv4_forward` at all, so its inner TTL stays at 64 (confirmed in my captures).

## 2. Design: the hop limit

I added one field to the tunnel header:

```
 0               16              32      40
 +---------------+---------------+-------+
 |   proto_id    |    dst_id     | hop_  |
 |   (16 bits)   |   (16 bits)   | limit |
 +---------------+---------------+-------+
```

- **Size: `bit<8>`.** One byte allows up to 255 switch hops. That is far more than this 2-switch path, or most real paths, need. It is the same size as the IPv4 TTL, and it keeps the header a whole number of bytes. A 16-bit field would waste a byte on every packet for no benefit.
- **Position: after `dst_id`, at the end of the header.** `proto_id` and `dst_id` stay at the same byte offsets as before, so a switch still running the old program can still read `dst_id` and choose the right exit. Putting the new field first would move `dst_id` and break every old switch immediately. (Section 5 tests how safe this really is.)
- **Starting value: chosen by the sender, default 64.** `send.py --hops H` sets it. 64 matches the usual IPv4 starting TTL and is far above this network's 2 switches, so normal traffic is never dropped. For loop tests I used a small value (`--hops 8`) so the number of passes is easy to check.
- **Where it is decremented: at ingress, on the tunnel path only, before the `myTunnel_exact` lookup.** Every switch that forwards a tunnelled packet uses one hop. Plain IPv4 packets are not affected.
- **What happens when it runs out: a packet that *arrives* with `hop_limit == 0` is dropped. Otherwise the switch subtracts 1 and forwards.** I check before subtracting so the 8-bit field can never wrap from 0 to 255, which is exactly the bug I found in the IPv4 path in section 1. With this rule a packet sent with hop limit H arrives at h2 or h3 showing H − 2. In a loop through upgraded switches it is dropped after H + 1 switch receptions: H forwards, then the drop. (The alternative rule, decrement first and drop when the result is 0, would need H ≥ 3 just to cross 2 switches.)
- **What else had to agree.** The parser and deparser needed no new code: `packet.extract(hdr.myTunnel)` and `packet.emit(hdr.myTunnel)` follow the header definition, so they now read and write 5 bytes. The two host scripts define their own copy of the header in scapy, so `send.py` and `receive.py` both had to change too. If hosts and switches disagree on the layout, the bytes are misread, as in lab Step 6.

## 3. Implementation

- **`tunnel.p4`:** added `bit<8> hop_limit` as the last field of `myTunnel_t`. In the ingress `apply` block, the tunnel branch now drops the packet if `hop_limit == 0`. Otherwise it does `hdr.myTunnel.hop_limit = hdr.myTunnel.hop_limit - 1` and then applies `myTunnel_exact`. The IPv4 branch is unchanged. The check sits before the decrement, not inside `myTunnel_forward`, so the limit is enforced whatever action the control plane installs in `myTunnel_exact`: it doesn't depend on every table entry using the right action.
- **`send.py`:** imported `ByteField`, added `ByteField("hop_limit", 64)` to the `MyTunnel` scapy class, and added a `--hops H` option (default 64, rejected outside 0–255). Its value goes into the tunnel header.
- **`receive.py`:** same `MyTunnel` class, and tunnelled packets are now printed as `tunnelled, dst_id=…, hop_limit=…`.
- `tunnel_v1.p4` is the untouched finished lab program, kept for the mixed-rollout test.

## 4. Evidence: it works

**Delivery** (`evidence/delivery.txt`). I sent tunnelled packets with `--hops 10`. `logs/h2.txt` shows `tunnelled, dst_id=2, hop_limit=8` with message `'to h2'`, and `logs/h3.txt` shows `tunnelled, dst_id=3, hop_limit=8` with `'to h3'`. Each packet crossed two switches, so 10 − 2 = 8, as designed. The inner IPv4 TTL is still 64 because the tunnel path never runs `ipv4_forward`. Plain IPv4 still works: `h1 ping -c 2 h3` got 2 of 2 replies with TTL 62.

**The loop now ends** (`evidence/loop-after.txt`). In a freshly started network I made the same loop as in section 1 and sent one packet with `--hops 8`:

| Reading | s1 | s2 |
|---|---|---|
| Before sending | 782 | 779 |
| After | 787 | 783 |
| Caused by the loop packet | +5 | +4 |
| 5 s later | 787 (unchanged) | 783 (unchanged) |

My design predicts H + 1 = 9 receptions: s1 sees the hop limit arrive as 8, 6, 4, 2 and finally 0, where it drops the packet (5 receptions); s2 sees 7, 5, 3, 1 (4 receptions). I measured 5 + 4 = 9, which matches the design exactly. The counts did not change 5 seconds later, so the loop stopped. Compare section 1, where the same loop grew by 669 receptions per switch every 5 seconds and never ended.

## 5. Evaluation

### a) Mixed rollout: s1 new, s2 old

I ran `python3 topo.py --id 5016 --s2-program tunnel_v1.p4`, so s1 ran my new program and s2 ran the old lab program, which only knows the 4-byte header. Evidence: `evidence/mixed.txt`.

**Prediction (worked out before the run; added to the report afterwards).**

- *Delivery to h2 (`--tunnel 2 --hops 10`):*
    - s1 reads all 5 bytes, decrements `hop_limit` to 9, and forwards.
    - s2 extracts only `proto_id` and `dst_id`. They are at the old offsets, so it reads `dst_id = 2` and forwards to h2: delivery should still happen.
    - s2 then sees `proto_id = 0x0800` and parses IPv4 starting one byte too early, at the `hop_limit` byte.
    - s2's `MyComputeChecksum` still runs, because its `hdr.ipv4` is valid. It writes a checksum into its shifted `hdrChecksum` field, which lands on the real IPv4 protocol byte and the first real checksum byte.
    - I expected the protocol byte to no longer be `0x11` (UDP), so `receive.py`, which only prints UDP packets, would show nothing even though the frame arrives.
- *The loop (`--tunnel 3 --hops 8`):*
    - s2 never decrements, but s1 still does on every pass, so the loop should still end.
    - s1 should see 8, 7, …, 1 (forward each time) and then 0 (drop): 9 receptions. s2 handles the 8 in between: about 17 in total, roughly twice as many as with both switches upgraded.

**Result: delivery.** `logs/h2.txt` shows only `Listening on h2-eth0...`, so `receive.py` printed nothing. The tcpdump capture at h2 shows the 52-byte frame *did* arrive:

```
0x0000:  ffff ffff ffff 0000 0000 0001 1212 0800
0x0010:  0002 0945 0000 2100 0100 0040 0b66 a80a
0x0020:  1000 010a 1000 0310 e110 e100 0d80 1f6d
0x0030:  6978 6564                                (ixed)
```

| Byte(s) | Value | Meaning |
|---|---|---|
| 12–13 | `1212` | Tunnel EtherType |
| 16–17 | `0002` | `dst_id` = 2: s2 forwarded on the correct exit |
| 18 | `09` | `hop_limit` = 9: only s1 decremented it (sent as 10) |
| 19 | `45` | Real start of the IPv4 header |
| 27 | `40` | IPv4 TTL 64, untouched on the tunnel path |
| **28** | **`0b`** | **IPv4 protocol: sent as `0x11` (UDP), arrived as `0x0b`** |
| 28–29 | `0b66` | The checksum s2 computed over its one-byte-shifted view, written over the real protocol byte (28) and the first real checksum byte (29). Byte 29 happens to keep its original value (`0x66`, from the sent checksum `0x66a8`), so the visible change is the protocol byte; the checksum `0x66a8` no longer matches the modified header |
| 47–51 | `mixed` | Payload intact |

This matches my prediction. Forwarding worked and the payload arrived, but the inner IPv4 header was corrupted: h2 no longer sees UDP, and the IPv4 checksum is wrong. No switch dropped the packet or logged an error. It is a **silent failure**. Appending the field kept the old switch *forwarding* correctly, but not *processing* correctly, because the old program still parses and rewrites the bytes after the tunnel header at the wrong offset.

**Result: loop.** With s2's exit 3 pointed back to s1 and one packet sent with `--hops 8`, the counts went from 789 / 785 (s1 / s2) to 798 / 793 and then stayed there. That is **s1 +9 and s2 +8 = 17 receptions**, exactly as predicted. s1 saw the hop limit arrive as 8, 7, …, 1, then 0 and dropped it. s2 bounced it back 8 times without changing it. With both switches upgraded (section 4) the same loop took 9 receptions. A half-upgraded network still stops the loop, but only because one switch on the loop enforces the limit, and it takes about twice as many passes. A loop made only of old switches would never end. (A second loop send failed because of a typo, `spython3`, so only one loop packet was sent; `mixed.txt` notes this.)

### b) Rolling out a header change safely

Real networks can't upgrade every switch at once, so old and new programs share the network for a while. I compared four options:

| Option | What an old switch does with a new packet | Assessment |
|---|---|---|
| **Append the field** (what I did) | Reads `proto_id` and `dst_id` correctly but misreads everything after them | Forwarding still works, but the inner packet is silently corrupted (section a). Not safe on its own. |
| **New EtherType** for the new header | Doesn't recognise the type, so it has no valid header and drops the packet | Fails *closed* and visibly instead of corrupting. Upgraded switches parse both the old and new types. Senders must only use the new type once the whole path supports it. |
| **Version field** in the header | Could branch on the version | Cleanest long-term, but only if it was in the header from the start. The original tunnel header has none. |
| **Two-phase upgrade** | Never receives a new packet | Phase 1: upgrade every switch to accept *both* formats. Phase 2: only then switch senders to the new format. Slower, but no mixed window. It is the same idea as Week 4's "update downstream first". |

**My recommendation:** a new EtherType plus a two-phase upgrade. The new EtherType means any switch that was missed drops the packet instead of corrupting it. The two-phase order means that, if the rollout goes to plan, no switch ever sees a format it can't read. Appending a field is only acceptable when every switch is upgraded before any sender changes.

### c) Data-plane hop limit vs control-plane `check_paths()`

- **`check_paths()` (Week 3, control plane)** walks every host-to-host path through the *intended* routes before anything is installed. It catches loops and missing routes before any packet is affected, and it names the broken path.
    - It only checks what it is given. It could not see my hand-made `table_modify` in section 1.
    - It also misses failures or races while an update is half-applied, and switch bugs.
- **The hop limit (data plane)** stops *any* loop at run time, whatever caused it, including my hand-made one.
    - It only limits the damage: a looping packet still makes up to H passes (9 in section 4, 17 in the mixed network) and wastes bandwidth.
    - The drop is silent, and nothing says where the loop is.
- **Why real networks use both:** prevention plus a safety net. IP has a TTL for the same reason, even though routing protocols already try to avoid loops.

### d) Cost on a hardware switch at line rate

- **Bandwidth:** 1 extra byte per tunnelled packet. That is about 1.2% of a minimum-size frame (84 bytes on the wire) and negligible for large packets.
- **Processing:** one comparison and one subtraction per packet. These are simple ALU operations a PISA stage does at line rate, with no loops.
- **Pipeline and memory:** no new table, so no extra SRAM or TCAM. The check is a condition in front of `myTunnel_exact`, and the decrement can sit in the same stage, so it should need no extra stage. The tunnel header has no checksum to recompute.
- **The real cost is operational:** every switch must be reprogrammed and every sender updated. As section (a) showed, the period when only some switches are upgraded is where things go wrong.

## AI use

I used Claude (Anthropic) during this assignment:

- It explained the assignment steps and Mininet/BMv2 commands, and helped me debug a missing brace and command typos.
- It suggested the hop-limit design and drafted the code changes. I checked them against the lab program, applied them, compiled them and tested them myself.
- It helped me read the tcpdump bytes and drafted parts of this report from my own evidence. I checked every number against my evidence files and edited the text.

I ran every experiment and collected all evidence myself, and I can explain every part of the code and report.

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with the finished lab program: counts growing from 649/647 to 1318/1316 in 5 s |
| `evidence/delivery.txt` | Tunnelled delivery to h2 and h3 with `hop_limit` 10 → 8, and a plain IPv4 ping still working |
| `evidence/loop-after.txt` | The same loop with my hop limit: the counts stop after the predicted number of passes |
| `evidence/mixed.txt` | Mixed rollout (s1 new, s2 old): the frame reaches h2 with its IPv4 protocol byte corrupted; the loop ends after 17 receptions |
