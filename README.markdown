## GUI示例
</div>
<div align="center">
  <img src="https://github.com/zhou19830318/watch-micropython-lvgl/blob/main/example-GUI.png">
</div>


## 项目概述

本项目是一个基于 MicroPython 和 LVGL（轻量级图形库）的智能手表应用，运行在支持 GC9A01 显示驱动的硬件设备上。该应用实现了以下核心功能：

- **模拟时钟显示**：通过动态更新的时针、分针和秒针显示当前时间，并支持数字时间和日期显示。
- **天气信息显示**：通过 WiFi 连接从心知天气 API 获取实时天气数据，并显示当前温度和天气状况（中文）。
- **二维码显示**：展示一个包含指定 URL 的二维码。
- **屏幕切换**：通过触摸屏手势滑动（左滑/右滑）在时间、天气、日期和二维码界面之间切换。
- **NTP 时间同步**：支持通过 WiFi 连接同步网络时间。
- **天气自动更新**：每 3 小时自动更新一次天气数据。
- **触摸屏支持**：集成 CST816S 触摸控制器，支持多种手势识别和触摸交互。

## 硬件要求

- **显示屏**：240x240 像素的 GC9A01 显示屏。
- **微控制器**：支持 MicroPython 的开发板（如 ESP32-S3）。
- **WiFi 模块**：用于连接网络获取天气数据和同步时间。
- **触摸屏**：CST816S 触摸控制器，支持手势识别。
- **GPIO 引脚**：
  - SPI 接口：MOSI (9)、SCK (8)。
  - 显示控制：DC (11)、CS (12)、RST (10)、背光 (13)。
  - I2C 接口：SCL (6)、SDA (7)（用于触摸屏通信）。
  - 触摸屏重置：GPIO (5)。
- **存储**：支持文件系统，用于存储天气图标（PNG 格式）。

## 软件依赖

- **MicroPython**：运行时环境。
- **LVGL 9.2.2**：用于界面渲染。
- **GC9A01 驱动**：显示屏驱动。
- **CST816S 驱动**：触摸屏驱动（支持手势识别：SWIPE_LEFT、SWIPE_RIGHT 等）。
- **心知天气 API**：用于获取实时天气数据（需提供 API 密钥）。
- **网络相关库**：`network`、`urequests`、`ntptime` 等，用于 WiFi 连接和数据获取。
- **其他库**：`machine`、`time`、`math`、`json`、`gc9a01` 等。

## 文件结构

- **display_driver.py**：初始化 GC9A01 显示屏的驱动程序，配置 SPI 总线和显示参数。
- **cst816s.py**：CST816S 触摸屏驱动，实现触摸检测和手势识别功能。
- **watch_touch.py**：主程序文件，包含 LVGL 界面设计、时钟逻辑、天气获取、手势控制、屏幕切换等功能。
- **task_handler.py**：事件任务处理器，用于管理定期执行的回调函数。
- **fs_driver.py**：文件系统驱动，用于支持文件读取（天气图标等）。
- **boot.py**：启动脚本。
- **textarea_keyboard_test.py**：文本输入和键盘测试脚本。
- **weather_incons/**：天气图标存储目录（PNG 格式）。

## 功能说明与示例代码

以下是主要功能的详细说明及其核心代码示例：

### 1. 时钟显示
- **功能描述**：通过 `ui_HourHand`、`ui_MinuteHand` 和 `ui_SecondHand` 绘制时针、分针和秒针，使用三角函数计算角度实现平滑旋转。数字时间和日期分别通过标签（Label）显示。
- **更新频率**：时钟每 100 毫秒更新一次，确保秒针平滑移动。
- **示例代码**（来自 `watch_touch.py`）：
  ```python
  def update_clock():
      global prev_seconds, prev_minutes, prev_hours
      current_time = rtc.datetime()
      hours = current_time[4]
      minutes = current_time[5]
      seconds = current_time[6]
      
      if seconds != prev_seconds:
          update_second_hand(seconds)
          prev_seconds = seconds
          time_changed = True
      
      if minutes != prev_minutes:
          update_minute_hand(minutes, seconds)
          prev_minutes = minutes
          time_changed = True
      
      if hours != prev_hours:
          update_hour_hand(hours, minutes)
          prev_hours = hours
          time_changed = True
      
      if time_changed:
          time_str = "{:02d}:{:02d}:{:02d}".format(hours, minutes, seconds)
          ui_Label2.set_text(time_str)
          if seconds == 0:
              date_str = "{:02d}/{:02d}/{:02d}".format(current_time[0], current_time[1], current_time[2])
              ui_Label3.set_text(date_str)

  def update_second_hand(seconds):
      angle_deg = seconds * 6  # 6 degrees per second
      angle_rad = math.radians(angle_deg)
      clock_radius = 100
      center_x, center_y = 120, 120
      start_radius = clock_radius - 15
      start_x = center_x + start_radius * math.sin(angle_rad)
      start_y = center_y - start_radius * math.cos(angle_rad)
      end_x = center_x + clock_radius * math.sin(angle_rad)
      end_y = center_y - clock_radius * math.cos(angle_rad)
      ui_SecondHand.set_points([{"x": int(start_x), "y": int(start_y)}, {"x": int(end_x), "y": int(end_y)}], 2)
  ```

### 2. 天气显示
- **功能描述**：通过心知天气 API 获取指定城市（默认：常州）的实时天气数据，显示温度、天气状况（中文）及对应图标。
- **更新频率**：每 3 小时自动更新一次，启动时立即更新。
- **示例代码**（来自 `watch_touch.py`）：
  ```python
  def update_weather(e):
      if connect_wifi():
          weather_data = fetch_weather()
          if weather_data:
              try:
                  weather_json = json.loads(weather_data)
                  weather = weather_json["results"][0]["now"]["text"]
                  temp = weather_json["results"][0]["now"]["temperature"]
                  weather_image_keys = f"{weather_json['results'][0]['now']['code']}@1x.png"
                  
                  ui_label.set_text(f"{temp}°C")
                  ui_label5.set_text(weather)
                  
                  try:
                      with open(f'weather_incons/{weather_image_keys}', 'rb') as f:
                          png_data = f.read()
                      img_cogwheel = lv.image_dsc_t({'data_size': len(png_data), 'data': png_data})
                      img1.set_src(img_cogwheel)
                  except Exception as e:
                      print(f"Could not load weather icon {weather_image_keys}: {e}")
                      ui_label.set_text("Icon Load Error")
              except Exception as e:
                  print(f"Weather JSON parse error: {e}")
                  ui_label.set_text("Weather Parse Error")
          else:
              ui_label.set_text("Weather Fetch Failed")
      else:
          ui_label.set_text("WiFi Not Connected")
  ```

### 3. 二维码显示
- **功能描述**：显示一个包含固定 URL（`https://lvgl.io`）的二维码，大小为 150x150 像素，带有浅蓝色边框。
- **示例代码**（来自 `watch_touch.py`）：
  ```python
  bg_color = lv.palette_lighten(lv.PALETTE.LIGHT_BLUE, 5)
  fg_color = lv.palette_darken(lv.PALETTE.BLUE, 4)
  qr = lv.qrcode(ui_qrcode)
  qr.set_size(150)
  qr.set_dark_color(fg_color)
  qr.set_light_color(bg_color)
  data = "https://lvgl.io"
  qr.update(data, len(data))
  qr.center()
  qr.set_style_border_color(bg_color, 0)
  qr.set_style_border_width(5, 0)
  ```

### 4. 屏幕切换（手势控制）
- **功能描述**：通过触摸屏手势识别在时间、天气、日期和二维码界面之间切换。支持左滑和右滑手势，具有 500ms 的滑动冷却时间（防止误触）。
- **支持的手势**：
  - **右滑**：切换到下一个屏幕
  - **左滑**：切换到上一个屏幕
- **示例代码**（来自 `watch_touch.py`）：
  ```python
  def switch_screen(direction):
      global current_screen_idx
      if direction == "next":
          current_screen_idx = (current_screen_idx + 1) % len(screens)
      elif direction == "prev":
          current_screen_idx = (current_screen_idx - 1) % len(screens)
      lv.screen_load(screens[current_screen_idx])

  def check_gesture(*args):
      global last_swipe_time
      coords = touch._get_coords()
      now = time.ticks_ms()
      if coords is not None:
          gesture = touch.get_gesture()
          if gesture == GESTURE_SWIPE_RIGHT and time.ticks_diff(now, last_swipe_time) > swipe_cooldown:
              switch_screen("next")
              last_swipe_time = now
          elif gesture == GESTURE_SWIPE_LEFT and time.ticks_diff(now, last_swipe_time) > swipe_cooldown:
              switch_screen("prev")
              last_swipe_time = now
  ```

### 5. 时间同步
- **功能描述**：通过 `ntptime` 模块从 NTP 服务器同步时间，同步按钮显示在天气界面。
- **示例代码**（来自 `watch_touch.py`）：
  ```python
  def sync_ntp_time(e):
      if not sta_if.isconnected():
          ui_Label4.set_text("WiFi not connected")
          return
      ui_Label4.set_text("Syncing...")
      lv.task_handler()
      try:
          ntptime.settime()
          ui_Label4.set_text("Sync OK!")
          global prev_seconds, prev_minutes, prev_hours
          prev_seconds = prev_minutes = prev_hours = -1
      except Exception as e:
          ui_Label4.set_text("Sync Failed")
          print("NTP Error:", e)
  ```

### 6. 触摸屏驱动（CST816S）
- **功能描述**：初始化并管理 CST816S 触摸控制器，支持多种手势识别（滑动、点击等）。
- **初始化代码**（来自 `watch_touch.py`）：
  ```python
  i2c = machine.I2C(0, scl=machine.Pin(6), sda=machine.Pin(7), freq=400000)
  touch = CST816S(i2c, reset_pin=machine.Pin(5, machine.Pin.OUT))
  touch.auto_sleep = False
  ```

## 注意事项

- **WiFi 连接**：确保网络稳定，否则天气数据和时间同步可能失败。
- **天气图标**：需预先准备好 PNG 格式的天气图标并存储在正确路径。
- **字体支持**：程序加载 `myfont_18.bin` 字体文件，若不存在则使用默认字体。
- **触摸屏校准**：首次使用触摸屏时，可能需要进行校准以确保手势识别准确。
- **手势冷却时间**：设置为 500ms，防止快速连续滑动导致的误触，可根据需要调整 `swipe_cooldown` 参数。
- **性能优化**：
  - 时钟更新频率为 100ms，平衡秒针流畅度和功耗。
  - 天气更新频率为 3 小时，减少 API 调用。
  - 手势检测频率为 50ms，确保及时响应。
- **错误处理**：
  - 如果天气图标加载失败，界面将显示"Icon Load Error"。
  - 如果 WiFi 连接失败，界面将显示"WiFi Not Connected"。
  - 如果触摸屏初始化失败，请检查 I2C 连接和引脚配置。

## 扩展与改进

- **动态城市选择**：增加触摸界面允许用户输入或选择城市。
- **更多天气信息**：扩展显示风速、湿度、体感温度等其他天气数据。
- **多语言支持**：添加语言切换功能。
- **低功耗优化**：在长时间无操作时降低更新频率或关闭显示。
- **更多手势支持**：扩展支持点击、长按、双击等其他触摸交互。
- **个性化主题**：添加可切换的配色方案和界面主题。
