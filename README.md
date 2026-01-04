This project demonstrates bare-metal one-wire style communication between an STM32 (STM32F411) and a DHT22/AM2302 temperature-humidity sensor. The DHT22 uses a timing-based digital protocol (no I²C/SPI/UART), so the STM32 must generate an accurate start pulse and then decode 40 bits by measuring microsecond-level HIGH pulse widths (≈24 µs for ‘0’, ≈65 µs for ‘1’).

