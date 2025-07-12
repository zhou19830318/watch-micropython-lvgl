# ESP32-S3 智能手表显示项目

该项目使用 **ESP32-S3** 微控制器和 **GC9A01** 240x240 圆形 TFT 显示屏，打造一个生动且功能强大的智能手表界面。项目基于 **MicroPython** 和 **LVGL (Light and Versatile Graphics Library)** 驱动。

## 功能特点
- **模拟时钟**：通过三角函数计算实现精准定位，时、分、秒针每 100ms 更新一次，动画流畅。
- **数字时间/日期**：以 HH:MM:SS 格式显示时间，YYYY-MM-DD 格式显示日期，并通过 NTP 同步。
- **天气更新**：使用 Seniverse API 获取指定城市（默认：常州）的实时天气数据（温度和状况），每 3 小时更新一次。
- **Wi-Fi 和 NTP 同步**：通过按键连接 Wi-Fi 并与 NTP 服务器同步时间。
- **性能优化**：通过跟踪时间变化减少重绘，确保可穿戴设备的资源高效使用。

## 硬件需求
- **ESP32-S3**：具有 Wi-Fi 功能以及充足内存以支持 LVGL 的微控制器。
- **GC9A01 显示屏**：240x240 圆形 TFT 显示屏，支持 SPI 接口。
- **开发板**：兼容 ESP32-S3 和 MicroPython 固件。
- **Wi-Fi 网络**：用于天气更新和 NTP 时间同步。

## 软件需求
- **MicroPython 固件**：支持 LVGL 和 GC9A01 驱动的版本。
- **必需库**：
  - `lvgl`: 图形用户界面库。
  - `gc9a01`: GC9A01 TFT 的显示驱动。
  - `lcd_bus`: 显示屏的 SPI 总线通信库。
  - `urequests`: 用于向天气 API 发送 HTTP 请求。
  - `ntptime`: 用于 NTP 时间同步。
  - `machine`、`network`、`time`、`json`、`math`: 标准 MicroPython 库。

## 设置指南
1. **准备硬件**：
   - 使用以下引脚将 GC9A01 显示屏连接到 ESP32-S3：
     - SCK: 引脚 8
     - MOSI: 引脚 9
     - MISO: 未使用 (-1)
     - DC: 引脚 11
     - CS: 引脚 12
     - RST: 引脚 10
   - 确保 ESP32-S3 已刷入支持 LVGL 和 GC9A01 的 MicroPython 固件。

2. **配置软件**：
   - 将 `display_driver.py` 和 `watch.py` 文件复制到 ESP32-S3。
   - 在 `watch.py` 中更新 Wi-Fi 凭证和城市名称以进行天气更新：
     ```python
     SSID = "Your_WiFi_SSID"
     PASSWORD = "Your_WiFi_Password"
     city = "Your_City"
     ```
   - 确保天气图标存储在设备的 `weather_incons` 目录中，文件名需与 `weather_dict` 键匹配（例如 "Sunny" 使用 `0@1x.png`）。

3. **安装依赖**：
   - 确保所有必需库已包含在 MicroPython 固件中或上传到设备。

4. **运行项目**：
   - 使用 `ampy` 或 Thonny 将代码上传到 ESP32-S3。
   - 通电后设备将初始化显示屏，连接到 Wi-Fi，并显示时钟和天气。

## 代码结构
- **`display_driver.py`**：
  通过 SPI 初始化 GC9A01 显示屏，设置分辨率（240x240）、颜色格式（RGB565）以及引脚配置。
  ```python
  def init_display():
      spi_bus = SPI.Bus(host=_SPI_HOST, mosi=_SPI_MOSI, miso=_SPI_MISO, sck=_SPI_SCK)
      display_bus = lcd_bus.SPIBus(spi_bus=spi_bus, dc=_LCD_DC, cs=_LCD_CS, freq=_LCD_FREQ)
      return gc9a01.GC9A01(data_bus=display_bus, display_width=_WIDTH, display_height=_HEIGHT, ...)
  ```

- **`watch.py`**：
  管理 LVGL 界面、时钟更新、天气获取及 Wi-Fi/NTP 功能。
  - **时钟更新**：
    ```python
    def update_clock():
        current_time = rtc.datetime()
        if seconds != prev_seconds:
            update_second_hand(seconds)
            prev_seconds = seconds
    ```
  - **天气更新**：
    ```python
    def update_weather(e):
        if connect_wifi():
            weather_data = fetchWeather()
            weather_json = json.loads(weather_data)
            ui_label.set_text(f"{temp}°C")
            ui_label5.set_text(wheather)
    ```
  - **UI 组件**：
    ```python
    ui_HourHand = lv.line(ui_Screen1)
    ui_HourHand.set_style_line_color(lv.color_hex(0x800080), lv.PART.MAIN | lv.STATE.DEFAULT)
    ui_HourHand.set_style_line_width(8, lv.PART.MAIN | lv.STATE.DEFAULT)
    ```

## 使用方法
- **时钟**：模拟时钟每 100ms 更新一次，使用 LVGL 绘制表针。数字时间和日期显示在时钟下方。
- **天气**：天气更新每 3 小时一次或启动时自动更新。显示屏显示当前温度和天气状况，并带有图标。
- **NTP 同步**：按屏幕上的按钮可通过 Wi-Fi 同步时间。

## 故障排除
- **Wi-Fi 连接失败**：验证 `watch.py` 中的 SSID 和密码。确保网络信号强度足够。
- **天气图标未找到**：确认 `weather_incons` 目录中包含正确的 PNG 文件。
- **显示问题**：检查引脚连接并确保 GC9A01 驱动已正确初始化。
- **时间同步错误**：确保网络连接正常并且 NTP 服务器有效。

## 贡献
欢迎 fork 此项目，添加新功能（如新的 UI 元素、节能模式）或优化代码。提交 pull request 或打开 issue 以获得反馈和改进建议。

## 许可
该项目采用 MIT 许可证授权。详见 [LICENSE](LICENSE) 文件。

## 致谢
- **LVGL**：提供强大且轻量的图形库。
- **MicroPython**：支持基于 Python 的嵌入式开发。
- **Seniverse API**：提供天气数据。
- **ESP32-S3 和 GC9A01**：完美的硬件组合。

祝您玩得开心，享受自定义智能手表显示项目！