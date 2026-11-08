# Galaxy S20 FE 4G (SM-G780G, `r8q`) — hardware notes

Source: live device tree (`/sys/firmware/fdt`), `/proc/iomem`, `/proc/interrupts`
and `dmesg`, captured from the stock Android kernel while booted in TWRP.
Serial numbers and device IDs are deliberately NOT recorded here.

Status legend: **[DT]** read directly from the device tree, **[LOG]** observed in
dmesg/interrupts, **[INFERRED]** my reasoning, needs a test, **[UNKNOWN]**.

## SoC / platform
- Snapdragon 865 (SM8250), 8 cores, ~6 GB RAM. **[DT]**
- Samsung boot flow, bootloader unlocked (`verifiedbootstate=orange`). **[LOG]**
- UFS host `1d84000.ufshc` (phy `1d87000`), same SM8250 layout as the LG V60. **[LOG]**
- TLMM (pinctrl) at `0xf000000`. DPU/MDP at `0xae00000`, DSI ctrl at
  `0xae94000` / `0xae96000`. **[LOG]**

## Block devices (UFS)
| Partition | Device |
|-----------|--------|
| boot | `/dev/block/sda23` |
| recovery (TWRP lives here; do not overwrite) | `/dev/block/sda24` |
| userdata | `/dev/block/sda36` |
| abl | `/dev/block/sdd8` |
| dtbo / vbmeta | `sda22` / `sdd25` (+ `vbmeta_samsung` `sda27`) |

## Touchscreen — STMicro FTS5CU56A (NOT the V60's SW42902)
- Node: `/soc/i2c@994000/touchscreen@49`, compatible `stm,fts_touch`. **[DT]**
- I²C bus = QUP SE5 (`0x994000`), 7-bit address `0x49`. **[DT]**
- SE5 pins: **GPIO12 and GPIO13**, function `qup5` (sleep state = plain GPIO). **[DT]**
  SDA/SCL order is not stated by the DT. **[UNKNOWN]** — probe both orders for an ACK at `0x49`.
- Interrupt: **TLMM GPIO15**, level-triggered, active low (`0x2008`), input, no bias. **[DT]**
- Power ("avdd"): fixed regulator `tsp_ldo_en` = **TLMM GPIO92, active high, 3.0 V**,
  marked `regulator-boot-on`. A plain GPIO, no PMIC/RPMh needed. **[DT]**
  Whether the bootloader leaves it high at handoff is **[UNKNOWN]**; drive it high ourselves.
- Coordinate range 0..4095 on both axes. **[DT]**
- Samsung's driver stack uses an event FIFO (events seen starting `0x43`, `0x03`, ...). **[LOG]**
  Protocol details must come from Samsung's GPL kernel source for SM-G780G.
- The bus transfers go through the GPI DMA engine (`gpi_dma900000_gpii0`, ~600 IRQs
  while the SE5 IRQ stays at 0). **[LOG]** Same "GSI-only" situation the V60 README
  describes, so GPIO bit-banging is the expected approach. **[INFERRED]**

## Display
- Panel: Samsung AMOLED `S6E3FC3_AMS646YD01_FHD`, 1080x2400, up to 120 Hz, DSI with
  compression (2 layer mixers / 2 encoders). **[LOG]**
- `cont_splash_region` at **`0x9C000000`, size `0x2300000`**; `disp_rdump_region`
  at `0x9C000000`, size `0x1200000`; `dfps_data_region` at `0x9E300000`. **[DT]**
- A `TE_GPIO` interrupt exists on **GPIO66** and dmesg reports `mdp_transfer_time_us`.
  Both suggest a **command-mode panel**, which usually needs the display controller
  to be triggered to push each new frame; writing pixels to memory alone may not
  update the screen. **[INFERRED]** — this is the biggest open risk for the port.
- Whether the bootloader leaves the pipeline running at kernel entry, and the
  buffer pixel format/stride, are **[UNKNOWN]**.
- TWRP exposes no `/dev/fb0` (it uses DRM), so there is no fbdev info to read.

## Memory map
`/memory` **[DT]**:
- `0x80000000` .. `0xBCC00000` (0x3CC00000 bytes)
- `0xC0000000` .. `0x200000000` (0x140000000 bytes)

Large no-map / reserved blocks exist inside these (modem, DSP, secure display,
Samsung debug at `0x9F000000`+, ramoops at `0x9FA00000`, etc.) — see `/reserved-memory`
in the DT. The stock kernel (TWRP) was loaded at `0xA01E0000`, ramdisk at `0xAEAEE000`.
A free contiguous region for XNU still has to be chosen. **[UNKNOWN]**

## Buttons
- Volume Up is a PMIC (SPMI/`pmic_arb`) interrupt. **[LOG]** Same mechanism family the
  V60 port used. Power/Volume Down mappings not yet read. **[UNKNOWN]**

## Open questions (in priority order)
1. Does the Samsung bootloader leave a live, scanning-out display at kernel entry?
   Can we trigger frame updates on this command-mode panel?
2. Boot image header version and how our payload is loaded (inspect stock `boot`).
3. SDA/SCL order on GPIO12/13; does GPIO92 need to be driven high?
4. Free RAM region for XNU / shim given the reservations above.
5. Power and Volume Down button wiring.
