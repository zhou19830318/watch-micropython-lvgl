</div>
<div align="center">
  <img src="https://github.com/zhou19830318/watch-micropython-lvgl/blob/main/20250710155039.jpg">
</div>

智能手表项目说明文档
项目概述
本项目是一个基于 MicroPython 和 LVGL（轻量级图形库）的智能手表应用，运行在支持 GC9A01 显示驱动的硬件设备上。该应用实现了以下核心功能：

模拟时钟显示：通过动态更新的时针、分针和秒针显示当前时间，并支持数字时间和日期显示。
天气信息显示：通过 WiFi 连接从心知天气 API 获取实时天气数据，并显示当前温度和天气状况（中文）。
二维码显示：展示一个包含指定 URL 的二维码。
屏幕切换：通过按下 GPIO7 按钮在时间、天气、日期和二维码界面之间切换。
NTP 时间同步：支持通过 WiFi 连接同步网络时间。
天气自动更新：每 3 小时自动更新一次天气数据。

硬件要求

显示屏：240x240 像素的 GC9A01 显示屏。
微控制器：支持 MicroPython 的开发板（如 ESP32）。
WiFi 模块：用于连接网络获取天气数据和同步时间。
GPIO 引脚：
SPI 接口：MOSI (9)、SCK (8)。
显示控制：DC (11)、CS (12)、RST (10)、背光 (13)。
按钮：GPIO7（带上拉电阻，用于屏幕切换）。


存储：支持文件系统，用于存储天气图标（PNG 格式）。

软件依赖

MicroPython：运行时环境。
LVGL 9.2.2：用于界面渲染。
GC9A01 驱动：显示屏驱动。
心知天气 API：用于获取实时天气数据（需提供 API 密钥）。
网络相关库：network、urequests、ntptime 等，用于 WiFi 连接和数据获取。
其他库：machine、time、math、json 等。

文件结构

display_driver.py：初始化 GC9A01 显示屏的驱动程序，配置 SPI 总线和显示参数。
watch.py：主程序文件，包含 LVGL 界面设计、时钟逻辑、天气获取、屏幕切换等功能。

功能说明与示例代码
以下是主要功能的详细说明及其核心代码示例：
1. 时钟显示

功能描述：通过 ui_HourHand、ui_MinuteHand 和 ui_SecondHand 绘制时针、分针和秒针，使用三角函数计算角度实现平滑旋转。数字时间和日期分别通过 ui_Label2 和 ui_Label3 显示。
更新频率：时钟每 100 毫秒更新一次，确保秒针平滑移动。
示例代码（来自 watch.py）：def update_clock():
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
            date_str = "{:04d}-{:02d}-{:02d}".format(current_time[0], current_time[1], current_time[2])
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



2. 天气显示

功能描述：通过心知天气 API 获取指定城市（默认：常州）的实时天气数据，显示温度、天气状况（中文）及对应图标。
更新频率：每 3 小时自动更新一次，启动时立即更新。
示例代码（来自 watch.py）：def update_weather(e):
    if connect_wifi():
        weather_data = fetchWeather()
        if weather_data:
            try:
                weather_json = json.loads(weather_data)
                weather = weather_json["results"][0]["now"]["text"]
                temp = weather_json["results"][0]["now"]["temperature"]
                weather_image_keys = find_keys_by_value(weather_dict, weather)
                weather_image_key = weather_image_keys[0] if weather_image_keys else '99@1x.png'
                weather_Zh_key = weather_Zh.get(weather, '未知')
                
                ui_label.set_text(f"{temp}°C")
                ui_label5.set_text(weather_Zh_key)
                
                try:
                    with open(f'weather_incons/{weather_image_key}', 'rb') as f:
                        png_data = f.read()
                    img_cogwheel = lv.image_dsc_t({'data_size': len(png_data), 'data': png_data})
                    img1.set_src(img_cogwheel)
                except Exception as e:
                    print(f"Could not load weather icon {weather_image_key}: {e}")
                    ui_label.set_text("Icon Load Error")
            except Exception as e:
                print(f"Weather JSON parse error: {e}")
                ui_label.set_text("Weather Parse Error")
        else:
            ui_label.set_text("Weather Fetch Failed")
    else:
        ui_label.set_text("WiFi Not Connected")



3. 二维码显示

功能描述：显示一个包含固定 URL（https://lvgl.io）的二维码，大小为 150x150 像素，带有浅蓝色边框。
示例代码（来自 watch.py）：bg_color = lv.palette_lighten(lv.PALETTE.LIGHT_BLUE, 5)
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



4. 屏幕切换

功能描述：通过按下 GPIO7 按钮在时间、天气、日期和二维码界面之间循环切换，切换无动画效果。
示例代码（来自 watch.py）：screens = [ui_time, ui_weather, ui_date, ui_qrcode]
current_screen_idx = 0
button = Pin(7, Pin.IN, Pin.PULL_UP)

def switch_screen():
    global current_screen_idx
    current_screen_idx = (current_screen_idx + 1) % len(screens)
    lv.screen_load(screens[current_screen_idx])

last_button_state = button.value()
def check_button(*args):
    global last_button_state
    current_button_state = button.value()
    if last_button_state == 1 and current_button_state == 0:
        switch_screen()
    last_button_state = current_button_state



5. 时间同步

功能描述：通过 ntptime 模块从 NTP 服务器同步时间，同步按钮显示在天气界面。
示例代码（来自 watch.py）：def sync_ntp_time(e):
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



安装与配置

硬件连接：

按照 display_driver.py 中的引脚定义连接 GC9A01 显示屏。
确保 GPIO7 连接按钮并启用上拉电阻。


WiFi 配置：

修改 watch.py 中的 SSID 和 PASSWORD 为您的 WiFi 名称和密码。
修改 city 为您希望查询天气的城市（英文名称，如 shanghai）。
确保已注册心知天气 API 并将 API 密钥填入 weather_url。


天气图标准备：

将天气图标（PNG 格式，文件名如 0@1x.png）存储在设备的 weather_incons 文件夹中。
确保设备支持文件系统（通过 fs_driver 注册）。


上传代码：

将 display_driver.py 和 watch.py 上传到 MicroPython 设备。
确保已安装 MicroPython 和必要的库（如 LVGL、urequests 等）。


运行程序：

启动设备，程序将自动初始化显示屏、连接 WiFi、加载界面并开始运行。



使用方法

启动：

设备启动后，默认显示时间界面（模拟时钟）。
天气数据会在启动时立即获取并显示。


屏幕切换：

按下 GPIO7 按钮可在时间、天气、日期和二维码界面之间循环切换。


时间同步：

在天气界面点击“常州”按钮（ui_Button1）以同步网络时间。


天气更新：

天气数据每 3 小时自动更新，也可通过时间同步按钮触发更新。



注意事项

WiFi 连接：确保网络稳定，否则天气数据和时间同步可能失败。
天气图标：需预先准备好 PNG 格式的天气图标并存储在正确路径。
字体支持：程序尝试加载 myfont_18.bin 字体文件，若不存在则使用默认字体。
性能优化：时钟更新频率为 100ms，天气更新频率为 3 小时，以平衡性能和功耗。
错误处理：
如果天气图标加载失败，界面将显示“Icon Load Error”。
如果 WiFi 连接失败，界面将显示“WiFi Not Connected”。



扩展与改进

动态城市选择：增加界面允许用户输入或选择城市。
更多天气信息：扩展显示风速、湿度等其他天气数据。
触摸支持：如果硬件支持触摸屏，可添加触摸交互。
低功耗优化：在长时间无操作时降低更新频率或关闭显示。

许可证
本项目基于 MIT 许可证，欢迎修改和分享。

