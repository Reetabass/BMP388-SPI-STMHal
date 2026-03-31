# BMP388 STM32 HAL SPI Driver

A low-level SPI driver for the Bosch BMP388 barometer, written in C for STM32 microcontrollers using STM32 HAL.

Tested on: **STM32F405RGT6**

---

## Credits

This driver was written with heavy reference to the I2C implementation by [DuePonto](https://github.com/DuePonto/BMP388-STM32-HAL). The register definitions, configuration logic, and compensation formulas are largely derived from that work — the primary contribution here is the SPI transport layer and associated HAL wiring.

---

## Differences from the I2C Version

| | I2C | SPI |
|---|---|---|
| Interface | `HAL_I2C_Mem_Read/Write` | `HAL_SPI_TransmitReceive` |
| Read addressing | Device address + reg | `reg \| 0x80` (bit 7 set) |
| Dummy byte on read | No | Yes - 1 byte discarded via `HAL_SPI_TransmitReceive` |
| CS pin | Not required | Required (set in `BMP388Handle_TypeDef`) |

> **Note:** The BMP388 requires a dummy byte to be discarded after the address phase on all SPI reads. This is handled internally — `BMP388_readRegister` and `BMP388_readBytes` both transmit a 2-byte buffer and discard `rxBuff[0]`, with real data starting at `rxBuff[1]`.

---

## Integration

### 1. Add the files to your project

Copy `BMP388.c` and `BMP388.h` into your project and include them in your build.

### 2. Configure SPI in CubeMX

- **Mode**: Full-Duplex Master
- **CPOL**: Low (Mode 0) or High (Mode 3) — either works, be consistent across your bus
- **CPHA**: Match CPOL selection
- **Data Size**: 8 bits
- **First Bit**: MSB First

### 3. Declare and populate the handle

The driver uses a `BMP388Handle_TypeDef` struct to hold the SPI handle, CS pin, calibration data, and config:

```c
BMP388Handle_TypeDef bmp388 = {
    .hspi    = &hspi1,
    .cs_port = GPIOA,
    .cs_pin  = GPIO_PIN_4
};
```

### 4. Initialise

```c
if (BMP388_Init(&bmp388) != HAL_OK) {
    // handle error
}
```

`BMP388_Init` will verify the chip ID, perform a soft reset, read the calibration data, and automatically configure oversampling, IIR filter, ODR, and power mode.

### 5. Read pressure and temperature

```c
uint32_t raw_pressure, raw_temperature, time;
float pressure, temperature;

BMP388_ReadRawPressTempTime(&bmp388, &raw_pressure, &raw_temperature, &time);
BMP388_CompensateRawPressTemp(&bmp388, raw_pressure, raw_temperature, &pressure, &temperature);
```

### 6. Calculate altitude (optional)

```c
float ground_pressure = 101325.0f; // Pa at sea level, or calibrate on the pad
float altitude = BMP388_FindAltitude(ground_pressure, pressure);
```

---

## Default Configuration

These defaults are applied automatically by `BMP388_Init`:

| Setting | Default | Constant |
|---|---|---|
| Temperature oversampling | 2x | `BMP388_OVERSAMPLING_2X` |
| Pressure oversampling | 8x | `BMP388_OVERSAMPLING_8X` |
| IIR filter coefficient | 3 | `BMP3_IIR_FILTER_COEFF_3` |
| Output data rate | 50 Hz | `BMP3_ODR_50_HZ` |
| Power mode | Normal | `BMP3_PWR_CTRL_MODE_NORMAL` |

To change these, call the setter functions before or after `BMP388_Init` and rewrite the relevant register:

```c
BMP388_SetPressOS(&bmp388, BMP388_OVERSAMPLING_16X);
BMP388_SetOutputDataRate(&bmp388, BMP3_ODR_100_HZ);
```

---

## License

MIT — see `LICENSE`
