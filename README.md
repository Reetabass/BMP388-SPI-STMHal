# BMP388 STM32 HAL SPI Driver

A low-level SPI driver for the Bosch BMP388 barometer, written in C for STM32 microcontrollers using STM32 HAL.

Tested on: **STM32F405RGT6**

---

## Credits

This driver was written with heavy reference to the I2C implementation by [DuePonto](https://github.com/DuePonto/BMP388-STM32-HAL). The register definitions, configuration logic, and compensation formulas are largely derived from that work — the primary contribution here is the SPI transport layer and associated HAL wiring.

---

## Differences from the I2C Version

The BMP388 SPI interface requires setting the MSB of the address byte high for reads (`reg | 0x80`), and a dummy byte must be discarded after the address phase. These are handled internally in the `platform_read` function.

| | I2C | SPI |
|---|---|---|
| Interface | `HAL_I2C_Mem_Read/Write` | `HAL_SPI_Transmit/Receive` |
| Read addressing | Device address + reg | `reg \| 0x80` |
| Dummy byte on read | No | Yes — 1 byte discarded |
| CS pin | Not required | Required (user-defined GPIO) |

---

## Integration

### 1. Add the files to your project

Copy `bmp388.c` and `bmp388.h` into your project.

### 2. Configure SPI in CubeMX

- **Mode**: Full-Duplex Master
- **CPOL**: Low (Mode 0) or High (Mode 3) — either works, pick one and be consistent across your bus
- **CPHA**: Match CPOL selection
- **Data Size**: 8 bits
- **First Bit**: MSB First

### 3. Define your CS pin

In `bmp388.h`, set your chip select GPIO:

```c
#define BMP388_CS_GPIO_Port   GPIOA
#define BMP388_CS_Pin         GPIO_PIN_4
```

### 4. Pass your SPI handle

```c
BMP388_Init(&hspi1);
```

### 5. Read data

```c
float temperature, pressure;
BMP388_ReadData(&temperature, &pressure);
```

---

## SPI Read/Write Implementation

```c
static int32_t platform_write(void *handle, uint8_t reg, const uint8_t *bufp, uint16_t len)
{
    HAL_GPIO_WritePin(BMP388_CS_GPIO_Port, BMP388_CS_Pin, GPIO_PIN_RESET);
    HAL_SPI_Transmit(handle, &reg, 1, HAL_MAX_DELAY);
    HAL_SPI_Transmit(handle, (uint8_t*)bufp, len, HAL_MAX_DELAY);
    HAL_GPIO_WritePin(BMP388_CS_GPIO_Port, BMP388_CS_Pin, GPIO_PIN_SET);
    return 0;
}

static int32_t platform_read(void *handle, uint8_t reg, uint8_t *bufp, uint16_t len)
{
    uint8_t dummy;
    reg |= 0x80;  // set MSB for read
    HAL_GPIO_WritePin(BMP388_CS_GPIO_Port, BMP388_CS_Pin, GPIO_PIN_RESET);
    HAL_SPI_Transmit(handle, &reg, 1, HAL_MAX_DELAY);
    HAL_SPI_Receive(handle, &dummy, 1, HAL_MAX_DELAY);  // discard dummy byte
    HAL_SPI_Receive(handle, bufp, len, HAL_MAX_DELAY);
    HAL_GPIO_WritePin(BMP388_CS_GPIO_Port, BMP388_CS_Pin, GPIO_PIN_SET);
    return 0;
}
```

> **Note:** The BMP388 requires a dummy byte to be read and discarded immediately after the address byte on all SPI reads. This is unique to the BMP388 and is not required on most other ST sensors (LSM6DSOX, H3LIS333 etc.).

---

## License

MIT — see `LICENSE`
