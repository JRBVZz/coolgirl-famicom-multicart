# COOLGIRL - ultimate multigame cartridge for Famicom

The goal of the project is to create a open source Famicom cartridge that will not be too expensive and can contain up to ~700 games of various mappers.

It's inspired by ultra cheap "COOLBOY" multicarts but instead of using single mapper (MMC3) "COOLGIRL" uses EPM1270T144 CPLD chip to simulate many different mappers. Both PRG and CHR data stored on a single flash memory chip. Before game starts loader loads CHR data from flash to CHR RAM and sets PRG offset, banking modes, mapper code and other settings via registers. There is lockout register bit which prevents further writes.

Full characteristics:
* PRG ROM: up to 128 MiB
* CHR RAM: up to 512 KiB
* PRG RAM: 32 KiB, non-volatile (FRAM) (optional)

--> You can use FRAM or SRAM with Battery. If you select FRAM (18W08 or 18L08), just solder jumper "FRAM". With SRAM (e.g. CY62256VLL)+Battery you need solder D2, D3, R4.
## How to build
### Hardware - KiCad 8 Version
#### Bill of materials
Dowload [BoM](CoolGirl_rev6.4-KiCad/Coolgirl-iBOM.html) and use interactive map.

#### Schematic
[Schematic](CoolGirl_rev6.4-KiCad/CG-Schematic.jpg) picture

The [source file](CoolGirl_rev6.4-KiCad/Coolgirl.kicad_sch) can be opened using the [KiCad 8](https://www.kicad.org/).

#### Board
Board designed for order on [jlcpcb.com](https://jlcpcb.com).

![image](CoolGirl_rev6.4-KiCad/Coolgirl.jpg)

* Layers: 2
* PCB Thickness: 1.2mm
* Golden fingers highly recommended

The [source file](CoolGirl_rev6.4-KiCad/Coolgirl.kicad_pcb) can be opened using the [KiCad 8](https://www.kicad.org/).

### Firmware
EPM1270T144 CPLD firmware can be compiled using the [Quartus 22.1](https://www.intel.com/content/www/us/en/collections/products/fpga/software/downloads.html?s=Newest&f:guidetm83741EA404664A899395C861EDA3D38B=%5BIntel%C2%AE%20MAX%C2%AE%20CPLDs%20and%20FPGAs%3BMAX%C2%AE%20II%20CPLDs%5D). Open [CoolGirl.qpf](CoolGirl_rev6.x/CoolGirl.qpf) to configure and compile project.

All mappers can't fit into the CPLD at once, so you need to select required mappers in (config file)[CoolGirl_config.vh], so they can fit into 1270 macrocells. Also, you can set RESET_COMBINATION parameter to specify software reset button combination, it works on original consoles and some famiclones. Default combination is 8'b11010010 (Left+Start+A+B). Set it to 0 to disable and free some macrocells.

There are JTAG pads on the cartridge board to connect programmer (USB Blaster).

### ROM preparing
You can use [COOLGIRL Multirom Builder](https://github.com/ClusterM/coolgirl-multirom-builder) to combine multiple ROMs into one with menu and loader. ROM can be written to assembled cartridge using [Famicom/NES Dumper/Writer](https://github.com/ClusterM/famicom-dumper-writer). Non-soldered flash memory chip can be written using programmer.

## Registers
Range: $5000-$5FFF

Mask: $5007

All registers are $00 on power-on and reset.

### $5xx0
```
 7  bit  0
 ---- ----
 PPPP PPPP
 |||| ||||
 ++++-++++-- PRG base offset (A29-A22)
```

### $5xx1
```
 7  bit  0
 ---- ----
 PPPP PPPP
 |||| ||||
 ++++-++++-- PRG base offset (A21-A14)
```

### $5xx2
```
 7  bit  0
 ---- ----
 AMMM MMMM
 |||| ||||
 |+++-++++-- PRG mask (A20-A14, inverted+anded with PRG address)
 +---------- CHR mask (A18, inverted+anded with CHR address)
```

### $5xx3
```
 7  bit  0
 ---- ----
 BBBC CCCC
 |||| ||||
 |||+-++++-- CHR bank A (bits 7-3)
 +++-------- PRG banking mode (see below)
```

### $5xx4
```
 7  bit  0
 ---- ----
 DDDE EEEE
 |||| ||||
 |||+-++++-- CHR mask (A17-A13, inverted+anded with CHR address)
 +++-------- CHR banking mode (see below)
```

### $5xx5
```
 7  bit  0
 ---- ----
 CDDE EEWW
 |||| ||||
 |||| ||++-- 8KiB WRAM page at $6000-$7FFF
 |+++-++---- PRG bank A (bits 5-1)
 +---------- CHR bank A (bit 8)
```

### $5xx6
```
 7  bit  0
 ---- ----
 FFFM MMMM
 |||| ||||
 |||+ ++++-- Mapper code (bits 4-0, see below)
 +++-------- Flags 2-0, functionality depends on selected mapper
```

### $5xx7
```
 7  bit  0
 ---- ----
 LMTR RSNO
 |||| |||+-- Enable WRAM (read and write) at $6000-$7FFF
 |||| ||+--- Allow writes to CHR RAM
 |||| |+---- Allow writes to flash chip
 |||+-+----- Mirroring (00=vertical, 01=horizontal, 10=1Sa, 11=1Sb)
 ||+-------- Enable four-screen mode
 |+-- ------ Mapper code (bit 5, see below)
 +---------- Lockout bit (prevent further writes to all registers)
```

## Mapper codes
```
| Code   | iNES mapper number(s) and name(s)  | Flags meaning                         | Notes                                     |
| ====== + ================================== + ===================================== + ========================================= |
| 000000 | 0 (NROM) †                         |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 000001 | 2 (UxROM) †                        | 0 - enable "Fire Hawk" mirroring for  | Mapper 2 is fully compartible with mapper |
|        | 71 (Codemasters) *                 |     mapper 71 (Codemasters)           | 71 but "Fire Hawk" only uses mirroring    |
|        | 30 (UNROM-512)                     | 1 - Enable one screen mirroring       | control. UNROM-512 self-writable feature  |
|        |                                    |     select for mapper 30 (UNROM-512)  | is not supported                          |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 000010 | 3 (CNROM) †                        |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 000011 | 78 (Irem) *                        |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 000100 | 97 (Irem's TAM-S1)                 |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 000101 | 93 (Sunsoft-2) *                   |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 000110 | 163 (Nanjing)                      |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 000111 | 18 (Jaleco SS 88006)               |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 001000 | 7 (AxROM) †                        | 0 - disable mirroring control, used   | Can be oversized to 512 KiB.              |
|        | 34 (BNROM, NINA-001) *             |     to select mapper 34 instead of 7  | iNES Mapper 034 is used to designate both |
|        |                                    |                                       | the BNROM and NINA-001 boards but only    |
|        |                                    |                                       | BNROM is supported                        |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 001001 | 228 (Action 52)                    |                                       | Only Cheetahmen II is supported           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 001010 | 11 (Color Dreams) *                |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 001011 | 66 (GxROM) *                       |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 001100 | 87 *                               |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 001101 | 90 (J.Y. Company) *                |                                       | Partial support only, can be used for     |
|        |                                    |                                       | "Aladdin" and "Super Mario World" only.   |
|        |                                    |                                       | "Super Mario World" requires to enable    |
|        |                                    |                                       | accurate IRQs and multiplier (disabled by |
|        |                                    |                                       | default)                                  |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 001110 | 65 (Irem's H3001) *                |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 001111 | 5 (MMC5) *                         |                                       | Experimental partically support, can be   |
|        |                                    |                                       | used for "Castlevania 3 (U)" only         |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 010000 | 1 (MMC1) †                         | 0 - enable 16KiB of WRAM              |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 010001 | 9 (MMC2) *                         |                                       |                                           |
|        | 10 (MMC4) *                        | 0 - 0=MMC2, 1=MMC4                    |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 010010 | 70 *                               | 0 - 0=70, 1=152                       |                                           |
|        | 152 *                              |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 010011 | 73 (VRC3)                          |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 010100 | 4 (MMC3/MMC6) †                    | 0 - use mapper 118 (TxROM)            | Can be oversized up to 2 MiB              |
|        | 118 (TxROM) *                      | 1 - use mapper 189                    |                                           |
|        | 189 *                              | 2 - use mapper 206                    |                                           |
|        | 206 (Namco, Tengen, others)        |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 010101 | 112                                |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 010110 | 33 (Taito) *                       | 0 - 0=33, 1=48                        |                                           |
|        | 48 (Taito) *                       |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 010111 | 42 (FDS conversions) *             |                                       | Interrupts disabled by default            |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 011000 | 21 (VRC2/VRC4) *                   | 2,0 - A0 and A1 lines configuration:  |                                           |
|        | 22 (VRC2/VRC4) *                   |     0,0 - like mapper 21              |                                           |
|        | 23 (VRC2/VRC4) *                   |     0,1 - like mapper 22              |                                           |
|        | 25 (VRC2/VRC4) *                   |     1,0 - like mapper 23              |                                           |
|        |                                    |     1,1 - like mapper 25              |                                           |
|        |                                    | 1 - divide CHR bank select by two     |                                           |
|        |                                    |     (VRC2a, mapper 22)                |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 011001 | 69 (Sunsoft FME-7) *               |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 011010 | 32 (IREM G-101) *                  |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 011011 | 79 (NINA-03/06)                    |                                       |                                           |
|        | 146 (Sachen 3015)                  |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 011100 | 133 (Sachen)                       |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 011101 | 36 (TXC's PCB 01-22000-400)        |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 011110 | Reserved                           |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 011111 | 184 (Sunsoft-1)                    |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 100000 | 38                                 |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 100001 | Reserved                           |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 100010 | 75 (VRC1)                          |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 100011 | 83                                 | 0 - DIP switch 0                      | Partial support, submapper 1 requires     |
|        |                                    | 1 - DIP switch 1                      | 512 KiB of CHR RAM                        |
|        |                                    | 2 - use submapper 1                   |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 100100 | 67 (Sunsoft-3)                     |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| 100101 | 89 (Sunsoft-2 on Sunsoft-3 PCB)    |                                       |                                           |
| ------ + ---------------------------------- + ------------------------------------- + ----------------------------------------- |
| ...... | Reserved                           |                                       |                                           |

† - very popular mappers, can't be disabled in config
* - enabled by default in config
```

## PRG banking modes
```
| Code | $8000 | $A000 | $E000 | $C000 | Notes                                    |
| ==== + ===== + ===== + ===== + ===== + ======================================== |
| 000  |       A       |       C       | UxROM, MMC4, MMC1 mode #3, etc.          |
| ---- + ------------- + ------------- + ---------------------------------------- |
| 001  |       C       |       A       | Mapper 97 (TAM-S1)                       |
| ---- + ------------- + ------------- + ---------------------------------------- |
| 010  |           Reserved            |                                          |
| ---- + ----------------------------- + ---------------------------------------- |
| 011  |           Reserved            |                                          |
| ---- + ----- + ----- + ----- + ----- + ---------------------------------------- |
| 100  |   A   |   B   |   C   |   D   | Universal, used by MMC3 mode 0, etc.     |
| ---- + ----- + ----- + ----- + ----- + ---------------------------------------- |
| 101  |   C   |   B   |   A   |   D   | MMC3 mode 1                              |
| ---- + ----- + ----- + ----- + ----- + ---------------------------------------- |
| 110  |               B               | Mapper 163                               |
| ---- + ----------------------------- + ---------------------------------------- |
| 111  |               A               | AxROM, MMC1 modes 0/1, Color Dreams      |
```
Power-on/reset state: A=0, B=~2, C=~1, D=~0

## CHR banking modes
```
| Code | $0000 | $0400 | $0800 | $0C00 | $1000 | $1400 | $1800 | $1C00 | Notes                                    |
| ==== + ===== + ===== + ===== + ===== + ===== + ===== + ===== + ===== + ======================================== |
| 000  |                               A                               | Used by many simple mappers              |
| ---- + ------------------------------------------------------------- + ---------------------------------------- |
| 001  |                          Spetial mode                         | Used by mapper 163                       |
| ---- + ----- + ----- + ----- + ----- + ----- + ----- + ----- + ----- + ---------------------------------------- |
| 010  |       A       |       C       |   E   |   F   |   G   |   H   | Used by MMC3 mode 0                      |
| ---- + ----- + ----- + ----- + ----- + ----- + ----- + ----- + ----- + ---------------------------------------- |
| 011  |   E   |   F   |   G   |   H   |       A       |       C       | Used by MMC3 mode 1                      |
| ---- + ----- + ----- + ----- + ----- + ------------- + ------------- + ---------------------------------------- |
| 100  |               A               |               E               | Used by MMC1                             |
| ---- + ----------------------------- + ------------- + ------------- + ---------------------------------------- |
| 101  |              A/B              |              E/F              | MMC2/MMC4, switched by tiles $FD or $FE  |
| ---- + ------- ----- + ------------- + ------------- + ------------- + ---------------------------------------- |
| 110  |       A       |       C       |       E       |       G       | Used by many complicated mappers         |
| ---- + ----- + ----- + ----- + ----- + ----- + ----- + ----- + ----- + ---------------------------------------- |
| 111  |   A   |   B   |   C   |   D   |   E   |  F    |   G   |   H   | Used by very complicated mappers         |
```
Power-on/reset state: A=0, B=1, C=2, D=3, E=4, F=5, G=6, H=7

## Donate
* PayPal: cluster@cluster.wtf
* [Donation Alerts](https://www.donationalerts.com/r/clustermeerkat)
* [Boosty](https://boosty.to/cluster)
