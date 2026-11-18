# Google Edgar speaker fix: missing Intel SST firmware

## TL;DR

On this Google Edgar Chromebook running Debian 13, PipeWire only offered **Dummy Output**, and every attempt to open the `chtrt5650` sound card failed with `Device or resource busy` (`EBUSY`). The cause was a missing Intel SST DSP firmware file:

```
intel/fw_sst_22a8.bin
```

Installing Debian's `firmware-intel-sound` package provided the file. The SST DSP then booted (`v01.0b.02.02`), WirePlumber exposed the **Built-in Audio Speaker** profile, and playback worked through both ALSA and PipeWire.

No reboot was required: the SST driver retries the firmware request on the next PCM open.

---

## Environment

| Item | Value |
|---|---|
| Device | Google **Edgar** (board: Edgar, family: Intel_Strago) |
| Firmware | MrChromebox-2408.1 |
| OS | Debian GNU/Linux 13 (trixie) |
| Kernel | `6.12.117+deb13-amd64` |
| Sound card | card 1: `chtrt5650` — `GOOGLE-Edgar-1.0` |
| ACPI device | `808622A8:00` (Intel SST / LPE) |
| Machine driver | `snd_soc_sst_cht_bsw_rt5645` (`cht-bsw-rt5645`) |
| Codec | Realtek RT5650 (`snd_soc_rt5645`) |
| Audio stack | PipeWire 1.4.2, WirePlumber 0.5.8 |
| UCM | `/usr/share/alsa/ucm2/Intel/chtrt5650/HiFi.conf` |

---

## Symptoms

- `wpctl status` showed only a **Dummy Output** sink.
- Card 1 (`chtrt5650`) exposed only the `off` and `pro-audio` profiles; the `HiFi` profiles were rejected during probing.
- Direct playback failed:

```console
$ aplay -D hw:1,0 /usr/share/sounds/alsa/Front_Center.wav
aplay: main:850: audio open error: Device or resource busy
```

- Capture failed the same way on `hw:1,0`.
- `/proc/asound/card1/pcm0p/sub0/status` reported `closed`, and neither `fuser` nor `lsof` showed another process holding the PCM. The device was not actually in use by an application.
- The error was reproducible even with a plain `open(2)` of `/dev/snd/pcmC1D0p`, and every PCM node on card 1 returned `EBUSY`.

---

## Root cause

### 1. The firmware file was absent

The SST driver requests:

```
intel/fw_sst_22a8.bin
```

Nothing provided that file:

```console
$ find /lib/firmware /usr/lib/firmware -iname '*sst*'
# (no output)
```

Because the firmware request failed at boot, the DSP never started:

```console
$ cat /sys/devices/pci0000:00/808622A8:00/firmware_version
FW not yet loaded
```

The UCM configuration itself was fine. The raw ALSA UCM API loaded the `HiFi` verb successfully:

```text
UCM available for card <<<SplitPCM=1>>>hw:1
Set UCM verb to HiFi
```

### 2. Opening the PCM then timed out waiting for the dead DSP

When a PCM is opened, the SST driver tries to power up the DSP and waits for a firmware/DSP reply. The relevant kernel timeout is:

```c
#define SST_BLOCK_TIMEOUT 1000
```

Measured open attempts failed after almost exactly that timeout:

```text
attempt 0  errno 16 EBUSY  elapsed 1.015s
attempt 1  errno 16 EBUSY  elapsed 1.020s
attempt 2  errno 16 EBUSY  elapsed 1.024s
```

So userspace saw `EBUSY`, but the device was not held by another process: the DSP simply could not answer.

### 3. WirePlumber/ACP marked every HiFi profile unsupported

With PCM open failing, the ACP probe rejected each `HiFi` profile:

```text
I spa.alsa [alsa-ucm.c:2567:ucm_probe_profile_set]: Probing profile HiFi (Mic2, Speaker)
I spa.alsa [pcm_hw.c:1803:snd_pcm_hw_open]: open '/dev/snd/pcmC1D0p' failed (-16)
I spa.alsa [alsa-util.c:721:pa_alsa_open_by_device_string]: Error opening PCM device _ucm0003.hw:chtrt5650: Device or resource busy
I spa.alsa [alsa-ucm.c:2591:ucm_probe_profile_set]: Profile 'HiFi (Mic2, Speaker)' mapping 'HiFi: Speaker: sink': output PCM open failed
I spa.alsa [alsa-ucm.c:2621:ucm_probe_profile_set]: Profile HiFi (Mic2, Speaker) not supported
```

That left only `off` and `pro-audio`, and PipeWire fell back to **Dummy Output**.

---

## Fix

Install the package that contains `intel/fw_sst_22a8.bin`:

```bash
sudo apt install firmware-intel-sound
```

Installed version in this case:

```
firmware-intel-sound 20250410-2
```

The package installs:

```
/usr/lib/firmware/intel/fw_sst_22a8.bin
```

Then restart the user audio services so WirePlumber re-probes the card:

```bash
systemctl --user restart pipewire pipewire-pulse wireplumber
```

A reboot is not required. If the card was left in a wedged state, a reboot is the simplest reset.

### If the sink still does not appear

Check that the DSP actually loaded:

```bash
cat /sys/devices/pci0000:00/808622A8:00/firmware_version
```

Expected output after the fix:

```
v01.0b.02.02
```

Then re-check:

```bash
wpctl status
```

---

## Verification

After the fix:

```console
$ cat /sys/devices/pci0000:00/808622A8:00/firmware_version
v01.0b.02.02

$ wpctl status
Audio
 ├─ Devices:
 │      49. Built-in Audio                      [alsa]
 │      50. Built-in Audio                      [alsa]
 ├─ Sinks:
 │  *   54. Built-in Audio Speaker              [vol: 0.40]
 ├─ Sources:
 │  *   55. Built-in Audio Internal Microphone  [vol: 1.00]
```

The sink uses the correct UCM profile:

```text
node.name = "alsa_output.platform-cht-bsw-rt5645.HiFi__Speaker__sink"
node.description = "Built-in Audio Speaker"
device.profile.name = "HiFi: Speaker: sink"
device.profile.description = "Speaker"
alsa.card_name = "chtrt5650"
```

Raw PCM open now succeeds immediately:

```text
PCM open OK in 0.049s
```

Playback was verified through both paths:

```console
$ aplay -D plughw:1,0 /usr/share/sounds/alsa/Front_Center.wav
Playing WAVE ... Signed 16 bit Little Endian, Rate 48000 Hz, Mono
# exit 0

$ pw-play /usr/share/sounds/alsa/Front_Center.wav
# exit 0
```

---

## Notes and gotchas

- **Do not use `hw:1,0` with the mono ALSA test WAV directly.** The hardware expects stereo; ALSA reports `Channels count non available`. Use `plughw:1,0` or a stereo file. This is unrelated to the original failure.
- `alsaucm -c hw:1 list verbs` is unreliable on this setup and can fail with `No such file or directory`. `alsaucm -c hw:1 dump text` and the raw `snd_use_case_get_list(..., "_verbs")` API work.
- The `chtrt5650` card is card **1**; card 0 is the Intel HDA HDMI controller.
- SST is the default driver for Cherry Trail (`snd_intel_dspcfg` auto-selects it). Switching to SOF (`options snd_intel_dspcfg dsp_driver=3` plus `firmware-sof-signed`) is an alternative, but not necessary here.
- The temporary WirePlumber debug log used during diagnosis is not part of the fix. The relevant excerpt is preserved in [`evidence/wireplumber-ebusy.log`](evidence/wireplumber-ebusy.log).

---

## References

- Linux v6.12 SST sources:
  - `sound/soc/intel/atom/sst/sst_loader.c` — `sst_load_fw()`, `sst_request_fw()`
  - `sound/soc/intel/atom/sst/sst_drv_interface.c` — `sst_power_control()`
  - `sound/soc/intel/atom/sst/sst_pvt.c` — `sst_wait_timeout()`, `SST_BLOCK_TIMEOUT`
- Debian package: <https://packages.debian.org/trixie/firmware-intel-sound>
- Chromebook audio compatibility project: <https://github.com/WeirdTreeThing/chromebook-linux-audio>
- Related upstream report: <https://github.com/WeirdTreeThing/chromebook-linux-audio/issues/375>
