# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **5016**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

Reading	s1	s2
Count 1	649	647
Count 2 (+5 s)	1318	1316
Growth in 5 s	+669	+669

## 2. Design: the hop limit

<!-- Your header change and your pipeline change, and the reason for each choice:
     - the field's size, and where in the header you put it (and why there);
     - the starting value send.py uses, and how you chose it;
     - where it's decremented, and what happens when it runs out;
     - what else had to change for every part of the path to agree (parser, deparser, hosts). -->

## 3. Implementation

<!-- Summarise what you changed in tunnel.p4, send.py and receive.py. The pull request shows
     the code itself: here, explain anything a reviewer might not see from the diff alone. -->

## 4. Evidence: it works

<!-- Show, with your evidence files:
     - tunnelled delivery to h2 and h3, and the hop limit the receivers saw (delivery.txt);
     - plain IPv4 still works (delivery.txt);
     - the same loop as in section 1 now ends, and after how many passes — does that match
       your design? (loop-after.txt) -->

## 5. Evaluation

<!-- a) The mixed rollout (mixed.txt): what did you predict before running s1 with your new
        program and s2 with tunnel_v1.p4? What actually happened to delivery, and to the loop?
        Explain every difference from your prediction, byte by byte if you need to.
     b) Propose a safe way to roll out a header change across a network of switches that can't
        all be upgraded at once. Compare at least two options (for example a version field, a
        new EtherType, or ordering fields so old switches still parse them).
     c) Loop protection in the data plane (this hop limit) versus the control plane (Week 3's
        check_paths()): what can each catch that the other can't? Why do real networks use both?
     d) What would this change cost on a hardware switch running at line rate? -->

## AI use

Level 3 AI Usage. I utilise Claude AI to understand the lab and assit in my debugging. 

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
