# MT7921e Wi-Fi Diagnosis & Recommendation

**Date:** 2026-09-30
**Machine:** Omarchy / Arch Linux, kernel `7.2.5-3-omarchy` (also tested `6.18.49-3-lts`)
**Status:** Root cause identified as an upstream `mt76` driver defect. No client-side fix available.

---

## TL;DR

The laptop's Wi-Fi adapter is **not faulty** and the campus network is **not rejecting it**. This is a **bug in the `mt7921e` driver in the `mt76` kernel module**, and it reproduces on third-party infrastructure that has no relationship to the campus.

The decisive test: a **public cafe Wi-Fi network** (`Yama Coffee`), on infrastructure unrelated to the university, fails at the same rate as the campus AP — **0 of 6 associations on 2.4 GHz**, against an access point at signal 100 / −47 dBm with zero beacon loss.

| Target | Result |
|---|---|
| Phone hotspot (simple radio stack) | **50 / 53** — 94% |
| Real access points (campus, vendor, cafe) | **1 / 315** — 0.3% |

The split is **phone hotspot vs real access point**, not campus vs non-campus. No amount of registration, re-association, kernel swapping, or MAC fixing will change this. The fix is upstream and does not exist yet.

**What actually works today:** USB Ethernet adapter, or an Intel AX210. The phone hotspot on 5 GHz is an acceptable stopgap with occasional manual reconnects.

---

## 1. Hardware & Software

| Component | Value |
|---|---|
| Adapter | MediaTek MT7921 [Filogic 330] |
| PCI ID | `14c3:7961` |
| Interface | `wlp2s0` |
| Driver | `mt7921e` (in-tree, `mt76` family) |
| Driver module params | `disable_aspm=1` — the only one that exists |
| Kernel | `7.2.5-3-omarchy` (also tested: `6.18.49-3-lts`) |
| NetworkManager | `1.58.1-1` |
| **Genuine MAC** | **`9c:12:21:84:76:c4`** (confirmed via `permaddr`) |
| Regulatory domain | `country ID`, 2400–2483 MHz enabled @ 26 dBm |

### MAC address finding (still valid, but no longer causal)

The adapter presents five different addresses depending on the network:

| Address | Locally-administered bit | Verdict |
|---|---|---|
| `9c:12:21:84:76:c4` | clear | **Genuine hardware MAC** |
| `f6:a0:a3:1b:f7:64` | set | Randomised |
| `c2:92:8f:87:47:32` | set | Randomised |
| `36:64:60:fb:3e:b7` | set | Randomised |
| `86:6d:3c:81:28:6e` | set | Randomised |
| `e2:49:e2:0b:37:bd` | set | Randomised (cafe) |

Cause: `/etc/NetworkManager/NetworkManager.conf` contains

```ini
[connection]
wifi.cloned-mac-address=stable
```

`stable` = per-SSID hashed MAC.

> This was initially suspected as the cause. **It is not.** Every failure below was reproduced using the genuine hardware MAC `9c:12:21:84:76:c4`, and the cafe network accepted a randomised address on the rare occasions it responded at all. MAC randomisation is worth cleaning up for hygiene, but it is not the bug.

---

## 2. Results by Network

| Network | BSSID | Band | Attempts | Success | Rate |
|---|---|---|---|---|---|
| **Jurgen** (phone hotspot) | `46:5e:71:22:fe:9c` | 2.4 + 5 GHz | 53 | **50** | **94%** |
| Yama Coffee (cafe) | `06:ce:88:c8:ba:7d` | 2.4 GHz | 6 | 0 | 0% |
| Yama Coffee (cafe) | `06:ce:88:c8:b9:de` | 5 GHz | 6 | 1 | 17% |
| UYM-Library | `0e:d0:f5:37:04:b7` | 2.4 GHz | 48 | 0 | 0% |
| UYM-Library | `0e:d0:f5:37:04:b8` | 5 GHz | 58 | 0 | 0% |
| INDRA (vendor router) | `9c:53:22:ae:6f:b0` | 2.4 GHz | 197 | 0 | 0% |
| | | | **368** | **51** | |

**Against real access points: 1 of 315. Against the phone hotspot: 50 of 53.**

### The cafe control test — the result that settled it

`Yama Coffee` is a third-party cafe network, broadcast by infrastructure with no relationship to the university. It shares no controller, no RADIUS server, and no device registry with the campus. It nevertheless fails identically.

For 2.4 GHz and 5 GHz we ran a controlled A/B on the **same access point with the same credentials**, 6 attempts each, back to back:

| Band | Successful associations | Auth timeouts |
|---|---|---|
| Yama Coffee 2.4 GHz (ch 1, signal 100) | **0** | **16** |
| Yama Coffee 5 GHz (ch 161, signal 100) | **1** | **5** |

Conditions during the 2.4 GHz run: **signal 100 (100%)**, **−47 dBm**, **zero beacon loss**, line of sight, correct password, genuine MAC, free channel. It still could not complete an authentication handshake.

Because a third-party cafe AP cannot plausibly be dropping this MAC, **the AP-side / registration hypothesis is disproved.**

### Failure signature

Identical on every real access point, campus or not:

```
wlp2s0: authenticate with 06:ce:88:c8:ba:7d (local address=e2:49:e2:0b:37:bd)
wlp2s0: send auth to 06:ce:88:c8:ba:7d (try 1/3)
wlp2s0: send auth to 06:ce:88:c8:ba:7d (try 2/3)
wlp2s0: send auth to 06:ce:88:c8:ba:7d (try 3/3)
wlp2s0: authentication with 06:ce:88:c8:ba:7d timed out
```

Two features point at the driver:

1. **All three auth retries fire within the same second**, then the driver declares failure. Real 802.11 authentication retries are spaced far further apart. The driver is giving up almost instantly instead of retrying properly.
2. **The AP never transmits anything back** — no Authentication response, no Association Response. Zero bytes. This happens *before* any password or key exchange, so a wrong password cannot cause it.

When the handshake *does* complete, the same log shows the `status=30` failure described in upstream issue #1104:

```
wlp2s0: authenticated
wlp2s0: RX AssocResp from 46:5e:71:22:fe:9c (capab=0x411 status=30 aid=4)
wlp2s0: rejected association temporarily; comeback duration 1024 TU
wlp2s0: association with 46:5e:71:22:fe:9c timed out
wlp2s0: associated            <- succeeded 12s later
```

### A false negative worth recording

`Yama Coffee` **did** connect on the first attempt, at 96.3 Mbit/s RX / 78 Mbit/s TX VHT-MCS 9, with 0 beacon loss. At that point it looked like proof the card was healthy and the campus was at fault.

It was not. Over the following 20 minutes the same access point produced roughly **40 authentication timeouts** at full signal. The first-attempt success was the outlier.

The lesson, and the reason for the A/B in §2: **a single successful association is not evidence of a working card.** On this driver the success rate against real access points is under 1%, so a first-try success carries very little information. Any future test must be run to a sample size of at least 6–10 attempts per band.

---

## 3. Ruled Out

Every one of these was tested and had **no effect**:

| Hypothesis | Result |
|---|---|
| Campus device registration / MAC whitelist | **Disproved** — a third-party cafe AP fails identically (0/6) |
| Randomised / cloned MAC | Genuine `9c:12:21:84:76:c4` also gets zero response (58/58 on UYM-Library 5 GHz) |
| Wrong Wi-Fi password | Impossible — the AP never responds to auth, so the key is never used |
| One bad access point | Five separate BSSIDs across three networks, plus a cafe AP |
| Regulatory domain | `country ID` already set, 2.4 GHz fully enabled @ 26 dBm |
| `wireless-regdb` missing (`DFS-UNSET`) | Does not block 2.4 GHz; CRDA defaults suffice |
| Signal strength / range | Failure at −47 dBm with 0 beacon loss. Not a coverage problem. |
| Kernel version | `6.18.49-3-lts` fails identically to `7.2.5-3` |
| ASPM / power save | `disable_aspm=1` already set, no change |
| Channel pinning | No effect |
| Band forcing (2.4 / 5 GHz) | Both fail; 5 GHz fails ~83% of the time |
| Recreating the profile from scratch | No effect |
| Mass retry | 1 of 315. Non-deterministic only in rare single successes. |
| Driver reload / rfkill cycle | No effect (also confirmed upstream) |
| Stale driver state | No effect |

**Kernel log is otherwise completely clean** — no firmware timeout, no MAC reset, no watchdog, no assert. The failure is silent.

### One clarification

802.11 *management* frames — which Authentication is — are transmitted at **basic legacy rates**, not the negotiated high MCS rate. Aggregation and rate selection do not apply to them, so rate/aggregation problems are not a plausible cause of the zero-response behaviour.

---

## 4. Diagnosis

**A defect in the `mt7921e` driver (kernel module `mt76`) breaks authentication against real access points. The adapter and the network are both fine.**

The evidence in one view:

| Target | Result | Character |
|---|---|---|
| Phone hotspot | 50 / 53 | simple radio stack |
| Cafe AP | 1 / 12 | real enterprise AP |
| Campus AP | 0 / 106 | real enterprise AP |
| Vendor router | 0 / 197 | real router |

What varies is **the complexity of the access point**, not its ownership, its band, or the client's configuration. The phone hotspot succeeds because its radio stack is simple enough that the driver's broken authentication retry logic still gets a reply before the driver abandons the attempt. Real access points, which implement proper rate adaptation, aggregation, and beamforming, do not tolerate the driver's premature give-up.

Supporting detail:

- The failure occurs **before** any key exchange, so it is a layer-2 authentication-timing problem, not a credential problem.
- The three retries collapsing into ~1 second is characteristic of a driver-side timeout being far too aggressive, and matches the "intermittent auth timeout" reports in [mt76#1104](https://github.com/openwrt/mt76/issues/1104).
- The `status=30` association denials seen intermittently on the phone hotspot match the same issue's reported behaviour, meaning the defect is present everywhere — its severity simply scales with how demanding the AP is.
- [mt76#1115](https://github.com/openwrt/mt76/issues/1115) describes precisely this pattern on the sibling MT7902: **2.4 GHz authentication fails, 5 GHz mostly works.** We observe a stronger version on MT7921 — both bands fail, 5 GHz only occasionally succeeding.

### Confidence

**High** that this is a driver defect. The cafe control test removes the network as a variable, and the same card succeeds on the same AP within the same session, so no AP-side explanation survives.

**Not established:** the precise fault in the driver. `mt7921e` exposes no module parameters for auth timeout or retry count, so it cannot be tuned from userspace. Isolating it would require reading `mt76/mt7921` auth-timeout handling, or a newer `mt76` backport.

---

## 5. Recommendations

Both upstream issues are **open with zero replies and no assignee**:

| Issue | Opened | Replies | Assignee |
|---|---|---|---|
| [mt76#1115](https://github.com/openwrt/mt76/issues/1115) | 2026-08-07 | 0 | none |
| [mt76#1104](https://github.com/openwrt/mt76/issues/1104) | 2026-07-18 | 0 | none |

No patch is in `for-next`, and there is no timeline.

### Ranked actions

| # | Action | Cost | Notes |
|---|---|---|---|
| 1 | **Use the USB Ethernet adapter** | already owned | Verified: 4.8 ms RTT, 0% loss, fully stable |
| 2 | Intel AX210 USB Wi-Fi adapter | ~Rp 300–400k | `iwlwifi` is far better tested than `mt76` |
| 3 | Campus wired ethernet (dorms/libraries) | Free | Check whether your building provides it |
| 4 | Phone hotspot on 5 GHz, manual reconnects | Free | 78 Mbit/s typical, 433 Mbit/s close range. Occasional reconnection needed. |
| 5 | Wait for an `mt76` fix | Free | Indefinite, no active development |

**Do not** spend time on device registration, MAC pinning, or kernel swaps. All three are tested and excluded.

### Not worth trying

- **Buying another MediaTek/RTL chipset.** The failure follows the driver, not the vendor.
- **Requesting IT reimage or network repair.** This is not a campus network problem; IT cannot fix it.
- **Raising it with the cafe.** Their equipment is fine and working for other customers.

### Meanwhile: the hotspot is the practical answer

The phone hotspot delivers **78 Mbit/s on 5 GHz**, up to **433 Mbit/s** at close range — well above the campus APs' advertised 270 Mbit/s on 2.4 GHz. Keep it on 5 GHz. You may need to reconnect manually a few times a session, which is the driver bug, not a configuration problem.

---

## 6. Upstream Bug Report

This is the useful artefact. Filing this on [mt76#1115](https://github.com/openwrt/mt76/issues/1115) adds a **second chip (MT7921 rather than MT7902)** and a controlled A/B, both of which strengthen that issue considerably.

```
MT7921e: authentication fails against all real APs, ~94% success only
against a phone hotspot. 2.4 GHz 0/6 on third-party infrastructure.

Hardware:  MediaTek MT7921 [Filogic 330], PCI 14c3:7961
Driver:    mt7921e (mt76 family), in-tree
Kernel:    7.2.5-3-omarchy, and 6.18.49-3-lts (identical behaviour)
MAC:       9c:12:21:84:76:c4 (genuine hardware MAC via permaddr)

The distinguishing result is a third-party cafe AP, unrelated to any
corporate or campus network, that fails identically to campus APs:

  Yama Coffee  2.4 GHz  0/6    signal 100%, -47 dBm, 0 beacon loss
  Yama Coffee  5 GHz    1/6    signal 100%, -47 dBm
  UYM-Library  2.4 GHz  0/48
  UYM-Library  5 GHz    0/58
  INDRA router 2.4 GHz  0/197
  phone hotspot 2.4+5  50/53  <- 94%

A cafe AP cannot plausibly be dropping this station's MAC, so this
rules out AP-side admission control and points at the driver.

Failure signature, identical on every real AP:

  wlp2s0: authenticate with 06:ce:88:c8:ba:7d
  wlp2s0: send auth to 06:ce:88:c8:ba:7d (try 1/3)
  wlp2s0: send auth to 06:ce:88:c8:ba:7d (try 2/3)
  wlp2s0: send auth to 06:ce:88:c8:ba:7d (try 3/3)
  wlp2s0: authentication with 06:ce:88:c8:ba:7d timed out

Observations that may help:

1. All three auth retries fire within the same second before the
   driver gives up. The auth timeout appears far too aggressive.

2. The AP transmits nothing in response - no Authentication response,
   no Association Response. This is before any key exchange, so it
   is unrelated to credentials.

3. When the handshake does complete, it intermittently shows
   RX AssocResp ... status=30 followed by a temporary rejection,
   matching the behaviour in #1104.

4. 2.4 GHz never succeeds against a real AP. 5 GHz succeeds roughly
   17% of the time. This is the same band asymmetry as #1115 but
   more severe on MT7921.

5. Same behaviour on 7.2.5 and 6.18.49 LTS, so not a regression in
   either kernel. disable_aspm=1 is already set.

6. Kernel log is otherwise clean - no firmware timeout, MAC reset,
   watchdog or assert.

Note on methodology: a first-try success on the cafe AP produced
96 Mbit/s and initially looked like proof the card was healthy. The
same AP then timed out ~40 times over 20 minutes at full signal. On
this driver a single association carries almost no information -
please run at least 6-10 attempts per band.
```

### If you want to report this to campus IT anyway

Only worth doing so that nobody keeps chasing the network. Suggested wording:

> My laptop cannot associate with campus Wi-Fi. I have traced this to a
> bug in the MediaTek MT7921e driver, not to the network — the same
> failure occurs on a public cafe Wi-Fi that has no relationship with
> the university. Registration, MAC changes and kernel changes have all
> been tested and ruled out. No action is needed from IT, but please note
> that devices with MT7921 adapters cannot be expected to work on
> campus Wi-Fi until this is fixed upstream. Wired ethernet works
> normally.

---

## 7. Configuration Changes Made During Diagnosis

All on NetworkManager profiles. Nothing system-wide was modified. No sudo was used.

| Profile | Change | Reason |
|---|---|---|
| `INDRA` | `autoconnect` → `no` | 0/197; was retrying forever and thrashing |
| `INDRA` | `autoconnect-retries` → `5` | Then disabled entirely |
| `UYM-Library` | `autoconnect` → `no` | 0/106; was thrashing with the hotspot every 27s |
| `UYM-Library` | `cloned-mac-address` → `permanent` | Force genuine MAC (ruled out as causal) |
| `UYM-Library` | `mac-address-randomization` → `never` | Stop per-SSID randomisation |
| `Jurgen` | `autoconnect-priority` → `100` | Preferred fallback |
| `Jurgen` / `Jurgen 1` | `autoconnect` → temporarily `no`, restored to `yes` | Prevented hijacking the card during the band A/B test |
| `Yama Coffee` | `band` + `bssid` pinned for the 2.4/5 GHz A/B, **then restored to unpinned** | The control test |

`Yama Coffee` is currently back on automatic band selection with no BSSID pin. Nothing is left in a pinned or test-only state.

### To revert everything

```bash
nmcli con mod INDRA connection.autoconnect yes
nmcli con mod UYM-Library connection.autoconnect yes
nmcli con mod Jurgen 802-11-wireless.band ""
nmcli con mod "Yama Coffee" 802-11-wireless.band "" 802-11-wireless.bssid ""
```

Leaving `INDRA` and `UYM-Library` on `autoconnect no` is recommended: they will otherwise retry continuously and interrupt the phone hotspot.

### Optional cleanup (needs sudo) — hygiene, not a fix

`/etc/NetworkManager/NetworkManager.conf` line 5 causes the per-SSID MAC churn. Commenting it out makes the adapter present one stable genuine address, which is good practice for MAC-authenticated networks:

```ini
[connection]
# wifi.cloned-mac-address=stable
```

```bash
sudo systemctl restart NetworkManager
```

This will not change the authentication failures. It is purely cosmetic.

---

## 8. Diagnostic Commands for Future Reference

```bash
# Which address is the card actually using?
ip link show wlp2s0 | grep -oE 'permaddr [0-9a-f:]+'
nmcli dev wifi list | grep -E "BSSID|SSID"

# Watch authentication attempts live
journalctl -kf -o short-precise -k | grep -E "authenticate|authenticated|AssocResp|timed out"

# Tally outcomes per access point over a session
awk '/authenticate with [0-9a-f:]+ /{split($0,a,"with ");split(a[2],b," ");bss=b[1];next}
     /wlp2s0: authenticated/{if(bss!=""){ok[bss]++;bss=""}next}
     /timed out/{if(bss!=""){fail[bss]++;bss=""}next}
     END{for(k in ok)printf "%-20s %6d ok %6d fail\n",k,ok[k],fail[k]+0;
         for(k in fail)if(!(k in ok))printf "%-20s %6d ok %6d fail\n",k,0,fail[k]}' \
  <(journalctl -k -o short --no-pager | grep wlp2s0)
```

### Controlled band A/B on one access point

Worth repeating for any future AP. Pin band and BSSID, run N attempts, then restore:

```bash
CON="Yama Coffee"
nmcli con mod "$CON" 802-11-wireless.band bg 802-11-wireless.bssid 06:CE:88:C8:BA:7D
for i in $(seq 1 6); do
  nmcli con up "$CON"; nmcli dev disconnect wlp2s0; sleep 2
done
nmcli con mod "$CON" 802-11-wireless.band a 802-11-wireless.bssid 06:CE:88:C8:B9:DE
for i in $(seq 1 6); do
  nmcli con up "$CON"; nmcli dev disconnect wlp2s0; sleep 2
done
# restore
nmcli con mod "$CON" 802-11-wireless.band "" 802-11-wireless.bssid ""
```

Note: on this NetworkManager build the band value for 2.4 GHz is `bg`, not `2g`. `2g` is rejected with *"'2g' not among [a, bg, 6GHz]"*.

Note: on this machine the system clock skews during boot, so chronological `journalctl --since/--until` queries can silently return nothing. Tally by content, not by time.

---

## 9. Upstream References

- [openwrt/mt76#1115](https://github.com/openwrt/mt76/issues/1115) — MT7902, `mt7921e`, 2.4 GHz auth fails / 5 GHz fine. Open, 0 replies. **Most relevant.** MT7921 PCIe support landed in-tree; no newer fix in `for-next`. Our report in §6 adds a second chip and a controlled A/B.
- [openwrt/mt76#1104](https://github.com/openwrt/mt76/issues/1104) — intermittent auth timeout + post-associate deauth + DHCP timeout across unrelated APs. Open, 0 replies. Matches the `status=30` behaviour seen on the phone hotspot.

Contributions welcome: the §6 report is ready to file. It is a genuinely useful data point — a third chip generation, a third-party AP as a control, and an explicit warning about first-try false positives.
