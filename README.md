![视频演示](https://github.com/zhou19830318/watch-micropython-lvgl/blob/main/0268c58e8d72c3bb1a3dd32d673856ec.mp4)
# ESP32-S3 Smartwatch Display Project

This project creates a vibrant and functional smartwatch interface using the **ESP32-S3** microcontroller and the **GC9A01** 240x240 circular TFT display. Powered by **MicroPython** and **LVGL (Light and Versatile Graphics Library)**, it features a smooth analog clock, digital time/date display, and real-time weather updates fetched via Wi-Fi. Perfect for hobbyists and developers looking to build a custom wearable device!

## Features
- **Analog Clock**: Smoothly animated hour, minute, and second hands, updated every 100ms using trigonometric calculations for precise positioning.
- **Digital Time/Date**: Displays time in HH:MM:SS and date in YYYY-MM-DD format, synced via NTP.
- **Weather Updates**: Fetches real-time weather data (temperature and conditions) for a specified city (default: Changzhou) using the Seniverse API, updated every 3 hours.
- **Wi-Fi & NTP Sync**: Connects to Wi-Fi for internet access and synchronizes time with NTP servers via a button press.
- **Optimized Performance**: Minimizes redraws by tracking time changes, ensuring efficient use of resources for a wearable device.

## Hardware Requirements
- **ESP32-S3**: Microcontroller with Wi-Fi and sufficient memory for LVGL.
- **GC9A01 Display**: 240x240 circular TFT display with SPI interface.
- **Development Board**: Compatible with ESP32-S3 and MicroPython firmware.
- **Wi-Fi Network**: For weather updates and NTP time synchronization.

## Software Requirements
- **MicroPython Firmware**: Version with LVGL and GC9A01 driver support.
- **Libraries**:
  - `lvgl`: For the graphical user interface.
  - `gc9a01`: Display driver for the GC9A01 TFT.
  - `lcd_bus`: SPI bus communication for the display.
  - `urequests`: For HTTP requests to the weather API.
  - `ntptime`: For NTP time synchronization.
  - `machine`, `network`, `time`, `json`, `math`: Standard MicroPython libraries.

## Setup Instructions
1. **Prepare the Hardware**:
   - Connect the GC9A01 display to the ESP32-S3 using the following pins:
     - SCK: Pin 8
     - MOSI: Pin 9
     - MISO: Not used (-1)
     - DC: Pin 11
     - CS: Pin 12
     - RST: Pin 10
   - Ensure the ESP32-S3 is flashed with a MicroPython firmware supporting LVGL and GC9A01.

2. **Configure the Software**:
   - Copy `display_driver.py` and `watch.py` to your ESP32-S3.
   - Update `watch.py` with your Wi-Fi credentials and city for weather updates:
     ```python
     SSID = "Your_WiFi_SSID"
     PASSWORD = "Your_WiFi_Password"
     city = "Your_City"
     ```
   - Ensure weather icons are stored in a `weather_incons` directory on the device, with filenames matching the `weather_dict` keys (e.g., `0@1x.png` for "Sunny").

3. **Install Dependencies**:
   - Ensure all required libraries are included in your MicroPython firmware or uploaded to the device.

4. **Run the Project**:
   - Upload the code to the ESP32-S3 using a tool like `ampy` or Thonny.
   - Power on the device. The display will initialize, connect to Wi-Fi, and show the clock and weather.

## Code Structure
- **`display_driver.py`**:
  Initializes the GC9A01 display via SPI, setting up resolution (240x240), color format (RGB565), and pin configurations.
  ```python
  def init_display():
      spi_bus = SPI.Bus(host=_SPI_HOST, mosi=_SPI_MOSI, miso=_SPI_MISO, sck=_SPI_SCK)
      display_bus = lcd_bus.SPIBus(spi_bus=spi_bus, dc=_LCD_DC, cs=_LCD_CS, freq=_LCD_FREQ)
      return gc9a01.GC9A01(data_bus=display_bus, display_width=_WIDTH, display_height=_HEIGHT, ...)
  ```

- **`watch.py`**:
  Manages the LVGL interface, clock updates, weather fetching, and Wi-Fi/NTP functionality.
  - **Clock Update**:
    ```python
    def update_clock():
        current_time = rtc.datetime()
        if seconds != prev_seconds:
            update_second_hand(seconds)
            prev_seconds = seconds
    ```
  - **Weather Update**:
    ```python
    def update_weather(e):
        if connect_wifi():
            weather_data = fetchWeather()
            weather_json = json.loads(weather_data)
            ui_label.set_text(f"{temp}°C")
            ui_label5.set_text(wheather)
    ```
  - **UI Components**:
    ```python
    ui_HourHand = lv.line(ui_Screen1)
    ui_HourHand.set_style_line_color(lv.color_hex(0x800080), lv.PART.MAIN | lv.STATE.DEFAULT)
    ui_HourHand.set_style_line_width(8, lv.PART.MAIN | lv.STATE.DEFAULT)
    ```

## Usage
- **Clock**: The analog clock updates every 100ms, with hands drawn using LVGL lines. The digital time and date are displayed below the clock.
- **Weather**: Weather updates occur every 3 hours or on startup. The display shows the current temperature and weather condition with an icon.
- **NTP Sync**: Press the on-screen button to sync the time via NTP if Wi-Fi is connected.

## Troubleshooting
- **Wi-Fi Connection Fails**: Verify SSID and password in `watch.py`. Ensure the network is in range.
- **Weather Icon Not Found**: Confirm that the `weather_incons` directory contains the correct PNG files.
- **Display Issues**: Check pin connections and ensure the GC9A01 driver is properly initialized.
- **Time Sync Errors**: Ensure internet connectivity and a valid NTP server.

## Contributing
Feel free to fork this project, add features (e.g., new UI elements, power-saving modes), or optimize the code. Submit pull requests or open issues for feedback and improvements.

## License
This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

## Acknowledgments
- **LVGL**: For the powerful and lightweight graphics library.
- **MicroPython**: For enabling Python-based embedded development.
- **Seniverse API**: For providing weather data.
- **ESP32-S3 & GC9A01**: For being the perfect hardware combo for this project.

Happy tinkering, and enjoy your custom smartwatch display!
