# Optimum XGS-PON bypass: WAS-110 step-by-step — 100% WEB UI, no SSH needed

> 🔥 **BEFORE ANYTHING ELSE — COOLING IS MANDATORY, NOT OPTIONAL.**
> The WAS-110 runs brutally hot: **~110 °C** under load with no airflow — and at that
> point it's not just aging the hardware, **the internet starts dropping** (thermal
> protection kicks in on the CPU). A small fan / any forced airflow keeps it in the
> **~60 °C** range and stable. Do this on day one, not after the first heat death.

Replace the Optimum XSR150DX all-in-one with a WAS-110 SFP+ stick running the 8311
community firmware (v2.8.3+). Everything below is done in the stick's web interface.
Tested end-to-end: full 5 Gbps, router untagged, survives reboots.

> 📺 **Scope: INTERNET ONLY.** This bypass does not deliver IPTV (the multicast side is
> intentionally left isolated) and **cannot replace Optimum Voice** — the gateway's SIP
> phone client is built into the Optimum box and can't be moved to the stick. Streaming
> TV apps (Optimum TV app, etc.) work fine over plain internet. If you have a
> TV/phone bundle, this guide is not for you (yet).
>
> ↩️ **Safety net:** you never modify the Optimum box. If anything goes wrong, plug the
> fiber back into it and you're back online in ~2 minutes — experiment fearlessly.

**You need:**
1. WAS-110 with 8311 community firmware already installed (if not, follow
   https://github.com/djGrrr/8311-was-110-firmware-builder first)
2. The bottom label of your Optimum box (or a photo of it)
3. A router with an SFP+ WAN port

---

## STEP 1 — Write down the values from your Optimum box's label

The label gives you almost everything:

```
S/N  : 5054494E-XXXXXXXX   →  PON serial = PTIN + last 8 hex digits
MAC  : 68:AA:C4:__:__:__
HW   : 3NTRGW22161P00      →  Hardware Version (copy yours from the label)
```

⚠️ **Decoding the S/N:** the label does NOT say "PTIN" — it prints the serial in raw
hex, like `5054494E1A2B3C4D`. The first 8 hex chars `50 54 49 4E` are ASCII for `PTIN`.
So: label `5054494E1A2B3C4D` → PON serial `PTIN1A2B3C4D`.
(Handy cross-check: the last 8 hex of the S/N always match the last 8 hex of the MAC.)

Only the **Software Version** is not on the label — use the platform constant below
(Optimum shows it nowhere; no app, no web UI). Everything else you need is either on the
label or identical on every XSR150DX, so no serial cable is needed to get started.

## STEP 2 — Log into the stick

Plug the stick into your router's SFP+ WAN port (or straight into your PC's SFP+ NIC if
it has one). Browse to **https://192.168.11.1** and log in with your 8311 root password.

💡 **No SFP+ port on your PC?** Set the router's WAN interface — the one the stick is
plugged into — to a **static IP `192.168.11.2`** (subnet `255.255.255.0`) so you can
reach the stick's web UI through it. You'll switch it back to DHCP in Step 7.
(If the stick is directly in your PC, set the PC's NIC to `192.168.11.2/24` instead.)

## STEP 3 — 8311 → Configuration → PON tab

Fill in:

| Field | Value |
|---|---|
| PON Serial Number (ONT ID) | `PTIN` + last 8 hex of the label S/N (see Step 1) |
| Vendor ID | `PTIN` |
| Equipment ID | `XSR150DX` |
| Hardware Version | `3NTRGW22161P00` *(use the `HW:` value from YOUR label)* |
| Sync Circuit Pack Version | ✅ checked |
| Software Version A | `3XGS020700R19` |
| Software Version B | `3XGS020700R19` |
| Override active firmware bank | `A` |
| Override committed firmware bank | `A` |
| PON Mode | `XGS-PON` |
| Registration ID (HEX) | `20202020202020202020` (ten spaces in hex — matches stock ONT) |
| MIB File | `/etc/mibs/prx300_1V_bell.ini` ⚠️ **NOT the 1U default!** |
| IP Host MAC Address | your `68:AA:C4:__:__:__` from the label |

Everything else on this tab: leave default/empty.

> The Hardware/Software versions above are from a production XSR150DX and are safe to
> reuse. If the stick later gets stuck at "O5.1 but no internet", Optimum may have
> pushed a newer firmware to your ONT — the serial console boot log shows your exact
> `HW_VERSION=` / `SW_VERSION=` (only then would you need a USB to TTL Serial Adapter
> Cable — see the appendix at the bottom of this guide).

## STEP 4 — ISP Fixes tab

1. **Fix VLANs** → select **`Hook script only`**.
2. Click **Edit hook script** and paste the script below into the editor (it replaces
   whatever example text is in there — that's fine, it's just a placeholder):

```sh
#!/bin/sh
# 1. Seed ME 45 (MAC bridge service profile) instance 0 — required by this OLT
omci_pipe.sh mec 45 0 0 1 1 0x8000 0x1400 0x0200 0x0f00 0 0 0x00000258 2>/dev/null

# 2. Unify datapath + clear GEM tc filters; gem29 = IPTV multicast, keep out
for dev in $(ip -o link | awk -F'[@:]' '/(gem|pmapper)[0-9]+@pon0/ {print $2}' | grep -v gem29); do
	tc filter del dev "$dev" egress 2>/dev/null
	tc filter del dev "$dev" ingress 2>/dev/null
	ip link set "$dev" master sw1 2>/dev/null
done

# 3. VLAN 12 tag translation in hardware (skip_sw = line rate)
tc filter del dev eth0_0 ingress pref 100 2>/dev/null
tc filter del dev eth0_0 egress pref 100 2>/dev/null
tc filter add dev eth0_0 ingress protocol all pref 100 flower skip_sw \
	action vlan push id 12 priority 0 protocol 802.1Q pass
tc filter add dev eth0_0 egress protocol 802.1Q pref 100 flower vlan_id 12 skip_sw \
	action vlan pop pass
exit 0
```

3. Save the hook script, then click **Save** on the config page.

## STEP 5 — Reboot, THEN move the fiber

**System → Reboot** and wait ~2 minutes.
**Now** unplug the fiber from the Optimum box and plug it into the stick.

## STEP 6 — Verify

Wait 2–3 minutes after connecting the fiber (the first provisioning round needs a
moment), then open **8311 → PON Status**:

✅ **PON PLOAM Status: `O5.1, Associated state`**

If it says O5.1, move on — the real proof is the next step (your router getting an IP).
(You can peek at **8311 → VLAN Tables** if curious; a few default-looking rows are normal.)

## STEP 7 — Router

WAN port (the same one you set to static in Step 2): switch it **back to DHCP, untagged,
no VLAN.** It should pull a public IP within a minute.
Run a speed test.

## STEP 8 — (Recommended) Fix your peering

Optimum only gives the *good* routes to MACs starting with `68:AA:C4` (their own
equipment). In your router's WAN settings, **clone the Optimum box's base MAC** from
Step 1, then renew DHCP. Verify with `ping 1.1.1.1` — single-digit ms on the good pool
(NY area) vs ~25 ms on the bad one.

⚠️ Never plug the old Optimum box back in while its MAC is cloned anywhere.

## STEP 9 — Back up your working config (2 minutes, worth it)

Once everything works, save a copy of the stick's configuration so you can recover from
any future mistake:

1. **8311 → Support** → download the support bundle (contains all fwenv settings).
2. Copy the hook file somewhere safe: open **ISP Fixes → Edit hook script**, select all,
   paste into a text file on your PC.

Also note: check your temps once — **8311 → PON Status** shows the module temperature.
You want to see it well under 80 °C with your fan running.

---

## Troubleshooting

| What you see | Fix |
|---|---|
| PON flaps forever, never reaches O5.1 | MIB File got left on `prx300_1U.ini` → set it to `prx300_1V.ini`, reboot |
| Stick reboots every ~3–4 min | Override active/committed bank not set to `A` (Step 3) |
| O5.1 but no internet | Check the hook pasted correctly (Step 4), Fix VLANs = Hook script only, wait 3 min for the OLT audit cycle |
| O5.1, internet works, stuck ~1 Gbps | Your router is tagging VLAN 12 → remove it; the stick tags for you (tagged WAN kills UniFi offload) |
| Was working, broke after web UI save | Check MIB File didn't revert to 1U |
| Worked for weeks, now O5.1 with no internet | Optimum pushed a firmware upgrade to the ONT profile → update Software Version A/B to the new version (see appendix to read it from your old ONT's boot log) |
| Random drops under load, stick hot to the touch | Thermal throttle → improve cooling (Step 0!), check PON Status temperature |

**Golden rule:** only connect the fiber after the Step 5 reboot, in that order.

---

## Appendix — the serial console (only needed for version verification)

You only need this if the published SW/HW constants don't provision (Optimum pushed a
different firmware to your unit). Everything else in this guide is label-based.

**Hardware:** a USB-to-TTL adapter with **3.3 V logic** (CP2102/FTDI/CH340 — check the
voltage before buying/using; many default to 5 V).

**Access:** the UART pins are on the bottom of the ONT, **under the FCC sticker**. Two
ways in, both cross a seal: peel the FCC sticker, or remove the bottom-mount screw
(which has its own security sticker over it). Heads-up: that's a tamper seal on
ISP-owned hardware — you're choosing to break it.

**Pinout & wiring (3 wires ONLY):**
- **Middle pin = GND** → adapter GND
- **Right-hand pins = RX and TX** → cross them (adapter RX ← ONT TX, adapter TX → ONT RX)
- ⚠️ **NEVER connect the V+/VCC pin.** The ONT is self-powered; the pin is unneeded, and
  if your adapter drives 5 V you'll fry the ONT's 3.3 V bus. GND/TX/RX only.

**Terminal:** 115200 baud, 8N1, no flow control (PuTTY works fine). Power-cycle the ONT
and watch the boot. What you're looking for:

- CFE board params: `GPON Serial Number`, `Base MAC Address`
- the `NUNO env=...` line with `EQUIP_ID=`, `HW_VERSION=`, `SW_VERSION=`

Copy those exact strings into Step 3 in place of the constants.

---
*Made by Attari Solutions LLC. Huge thanks to the 8311 community and to djGrrr
(https://github.com/djGrrr) — the developer of the 8311 community firmware this
entire project is built on.*
