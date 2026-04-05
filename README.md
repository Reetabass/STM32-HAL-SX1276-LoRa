# STM32-HAL-SX1276-LoRa

A bare-metal LoRa driver for the Semtech SX1276 written in C using the STM32 HAL library. Written from scratch against the SX1276 datasheet with no third-party dependencies beyond STM32 HAL.

> ⚠️ **Work in progress**, core TX and RX are functional. Utility functions and continuous RX mode are still being completed.

---

## Hardware

| Item | Detail |
|------|--------|
| Target MCU | STM32F405RGT6 |
| Radio Module | RFM95 (SX1276) |
| Interface | SPI (SPI2) |
| HAL | STM32 HAL |

---

## Compatibility

| Chip | Support |
|------|---------|
| SX1276 | Full |
| SX1279 | Full |
| SX1277 | SF6–SF9 only, SF10/11/12 not supported on this variant |
| SX1278 | Frequencies below 525 MHz only |

---

## Features

- LoRa mode only, no FSK/OOK
- Single packet TX with TxDone polling
- Single packet RX with RxDone polling and CRC check
- Configurable spreading factor, bandwidth, coding rate, TX power, sync word, preamble length
- PA_BOOST and RFO pin support, including +20 dBm mode
- Hardware reset sequence on init
- SPI version register sanity check on init (reads REG_VERSION, expects 0x12)

---

## File Structure

```
STM32-HAL-SX1276-LoRa/
├── LoRa.h        # Register defines, bitmasks, config struct, function prototypes
├── LoRa.c        # Driver implementation
└── README.md
```

---

## SPI Configuration (CubeMX)

Configure your SPI peripheral with these settings before using the driver:

| Setting | Value |
|---------|-------|
| Mode | Full-Duplex Master |
| CPOL | Low (0) |
| CPHA | 1 Edge (0), SPI Mode 0 |
| Data Size | 8 bits |
| First Bit | MSB First |
| NSS | Software |
| Baud Rate | ≤ 10 MHz |

NSS must be software-controlled and passed to the driver handle via the NSS GPIO port and pin; do not use hardware NSS.

---

## Usage

### 1. Define your hardware handle and config

```c
SX1276_HandleTypedef lora_handle = {
    .hspi     = &hspi2,
    .nss_port = NSS_GPIO_Port,
    .nss_pin  = NSS_Pin,
    .rst_port = RST_GPIO_Port,
    .rst_pin  = RST_Pin
};

LORA_CONFIG_TYPEDEF lora_config = {
    .frequency       = 915000000UL,  // 915 MHz
    .spreading_Factor = 7,
    .bandwidth       = 125000,
    .coding_Rate     = 5,            // 4/5
    .tx_Power        = 17,
    .use_Pa_Boost    = 1,
    .preamble_Length = 8,
    .sync_Word       = 0x12,         // private LoRa network
    .enable_CRC      = 1,
    .implicit_header = 0             // explicit header mode
};
```

### 2. Initialise

```c
if (lora_init(&lora_handle, &lora_config) != HAL_OK) {
    // handle error — SPI not working or chip not found
}
```

### 3. Transmit

```c
uint8_t payload[] = {0x01, 0x02, 0x03, 0x04};

if (lora_TX(payload, sizeof(payload), 5000) != HAL_OK) {
    // handle error or timeout
}
```

### 4. Receive

```c
uint8_t rx_buf[64];
uint8_t rx_len = 0;

HAL_StatusTypeDef result = lora_RX(rx_buf, &rx_len, sizeof(rx_buf), 5000);

if (result == HAL_OK) {
    // rx_buf contains rx_len bytes of received data
} else if (result == HAL_TIMEOUT) {
    // no packet received within 5000 ms
} else {
    // SPI error or CRC failure
}
```

---

## Init Sequence

The driver follows the sequence from the SX1276 datasheet section 4.1.3:

1. Hardware reset via NRESET pin
2. Read REG_VERSION (0x42), must return 0x12
3. Set Sleep mode
4. Enable LoRa mode (can only be set in Sleep)
5. Set FIFO base addresses
6. Set RF frequency
7. Set LNA gain
8. Configure modem (BW, SF, CR, CRC)
9. Set preamble length
10. Set sync word
11. Set TX power
12. Enter Standby

---

## Known Limitations

- Utility functions (`lora_sleep`, `lora_standby`, `lora_packet_rssi`, `lora_packet_snr`, `lora_version`) are not yet implemented
- Continuous RX mode (`lora_receive_cont`) is not yet implemented
- TX and RX use polling — DIO0 interrupt support is planned
- SF6 requires extra register writes and is not currently handled
- No frequency hopping support

---

## Dependencies

- STM32 HAL (SPI and GPIO)
- Tested on STM32F405RGT6, should work on any STM32 with HAL SPI

---

## Reference

Based on: **SX1276/77/78/79 Datasheet Rev.7 — Semtech Corporation, May 2020**

---

## License

MIT License — free to use, modify and distribute with attribution.
