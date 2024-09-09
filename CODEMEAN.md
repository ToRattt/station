# ĐỌC HIỂU CODE
## Giải thích code mẫu Wifi Station


## Khai báo thư viện 
```bash 
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"
#include "esp_system.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "lwip/err.h"
#include "lwip/sys.h"
```
- string.h: Đây là thư viện chuẩn của C cung cấp các hàm để xử lý chuỗi như sao chép, nối và so sánh chuỗi.

- freertos/FreeRTOS.h: Đây là thư viện của FreeRTOS, một hệ điều hành thời gian thực được sử dụng để quản lý đa nhiệm, cung cấp các API để tạo và quản lý task, queue, semaphore, và các đối tượng đồng bộ hóa khác.

- freertos/task.h: Thư viện này cung cấp các hàm cụ thể để làm việc với các task trong FreeRTOS, bao gồm tạo task, xóa task, và chuyển đổi giữa các task.

- freertos/event_groups.h: Thư viện này cho phép bạn sử dụng các nhóm sự kiện (event groups) trong FreeRTOS, hữu ích trong việc đồng bộ hóa giữa các task hoặc giữa task và interrupt.

- esp_system.h: Thư viện này cung cấp các API liên quan đến hệ thống của ESP-IDF, bao gồm khởi động lại hệ thống, lấy thông tin hệ thống, và các thao tác quản lý nguồn.

- esp_wifi.h: Đây là thư viện dành riêng cho ESP32 để quản lý Wi-Fi, bao gồm khởi tạo Wi-Fi, kết nối vào mạng, và xử lý các sự kiện liên quan đến Wi-Fi.

- esp_event.h: Thư viện này cung cấp một hệ thống quản lý sự kiện cho ESP-IDF, cho phép đăng ký và xử lý các sự kiện như Wi-Fi, hệ thống, và sự kiện từ các thành phần khác.

- esp_log.h: Thư viện này cung cấp các hàm để ghi log, giúp theo dõi và gỡ lỗi các ứng dụng trên ESP32.

- nvs_flash.h: Thư viện này cho phép truy cập và quản lý bộ nhớ NVS (Non-Volatile Storage), nơi bạn có thể lưu trữ dữ liệu không mất đi khi mất nguồn.

- lwip/err.h và lwip/sys.h: Đây là các thư viện thuộc phần lwIP (Lightweight IP), một stack giao thức mạng được sử dụng trong ESP32 để xử lý các kết nối mạng TCP/IP. err.h chứa các định nghĩa mã lỗi, và sys.h chứa các hàm và macro cần thiết cho việc tích hợp với hệ điều hành.

## Đặt lại cấu hình Wifi 
```bash
#define EXAMPLE_ESP_WIFI_SSID      CONFIG_ESP_WIFI_SSID
// khai báotên biến gán cho wiffi ssid
#define EXAMPLE_ESP_WIFI_PASS      CONFIG_ESP_WIFI_PASSWORD
// khai báo tên biến gán cho wiffi password
#define EXAMPLE_ESP_MAXIMUM_RETRY  CONFIG_ESP_MAXIMUM_RETRY
// số lần thử kết nối tối đa (thông thường mặc định là 5, có thể tùy chỉnh)
#ESP_WIFI_SAE_MODE: 
// Chế độ xác thực Simultaneous Authentication of Equals (SAE) cho WPA3.
#EXAMPLE_H2E_IDENTIFIER: 
// Chuỗi xác định Home-to-Home (H2H) cho SAE.
#ESP_WIFI_SCAN_AUTH_MODE_THRESHOLD: 
// Ngưỡng xác thực khi quét mạng Wi-Fi.
```

## Biến và hằng
```bash
#s_wifi_event_group: 
// Được khai báo dưới dạng biến tĩnh (static), đại diện cho một nhóm sự kiện trong FreeRTOS. Nhóm này được sử dụng để đồng bộ hóa tác vụ và báo cáo sự kiện xảy ra trong hệ thống, đặc biệt là quá trình kết nối wiffi -> nhằm theo dõi trạng thái kết nối wiffi và báo hiệu sự kiện quan trọng
#WIFI_CONNECTED_BIT: 
// Bit báo hiệu kết nối thành công.
#WIFI_FAIL_BIT: 
// Bit báo hiệu kết nối thất bại.
#static const char *TAG = "wifi station": 
// Khai báo như trên giúp biến TAG có phạm vi tồn tại trong toàn bộ file, giá trị biến TAG không thể thay đổi sau khi khởi tạo, và biến TAG là một con trỏ đến chuỗi kí tự char.
// "wifi station" là chuỗi ký tự mà biến TAG đang trỏ tới. Nó sẽ được sử dụng làm nhãn để phân biệt các thông báo log đến từ các phần khác nhau của chương trình.
// Nhãn cho log giúp:
+ Phân loại thông tin: Khi chương trình chạy, rất nhiều thông báo log có thể được in ra. Việc sử dụng nhãn giúp ta dễ dàng phân biệt các thông báo đến từ phần nào của chương trình. Ví dụ, nếu bạn có nhiều module khác nhau trong chương trình, mỗi module có thể có một nhãn riêng để phân biệt các thông báo của nó. 
+ Dễ dàng tìm kiếm: Khi bạn muốn tìm kiếm một thông báo log cụ thể, bạn chỉ cần tìm kiếm theo nhãn của nó.
+ Debug: Nhãn giúp bạn xác định chính xác vị trí xảy ra lỗi trong chương trình khi bạn đang gỡ lỗi.
#s_retry_num: 
// Số lần thử kết nối hiện tại.

```
## Hàm xử lý sự kiện
```bash 
#event_handler(): 
// Hàm này xử lý các sự kiện liên quan đến Wi-Fi và IP. Nó kiểm tra trạng thái kết nối và thực hiện các hành động tương ứng.
// even_base: Xác định nhóm sự kiện
// even_id: Xác định cụ thể từng sự kiện trong nhóm
// Nghe các sự kiện: Hàm này liên tục "nghe" các sự kiện được gửi đến, bao gồm các sự kiện liên quan đến việc khởi động quá trình kết nối, ngắt kết nối, và nhận được địa chỉ IP.
// Phân loại sự kiện: Khi một sự kiện xảy ra, hàm sẽ kiểm tra loại sự kiện đó và thực hiện các hành động tương ứng.
// Xử lý sự kiện: Tùy thuộc vào loại sự kiện, hàm sẽ:
// Khởi động lại quá trình kết nối: Nếu thiết bị bị ngắt kết nối và chưa đạt đến số lần thử tối đa.
// Báo hiệu kết nối thành công: Nếu thiết bị đã nhận được địa chỉ IP.
// Báo hiệu kết nối thất bại: Nếu thiết bị đã thử kết nối quá nhiều lần mà không thành công.
// Cập nhật trạng thái: Hàm này sẽ cập nhật trạng thái kết nối của thiết bị bằng cách đặt các cờ (bits) trong nhóm sự kiện "s_wifi_event_group".
// Các sự kiện chính được xử lý bởi hàm event_handler():
+ WIFI_EVENT_STA_START: Sự kiện này xảy ra khi thiết bị bắt đầu quá trình kết nối đến mạng Wi-Fi.
+ WIFI_EVENT_STA_DISCONNECTED: Sự kiện này xảy ra khi thiết bị bị ngắt kết nối khỏi mạng Wi-Fi.
+ IP_EVENT_STA_GOT_IP: Sự kiện này xảy ra khi thiết bị đã nhận được địa chỉ IP và kết nối thành công.
```

## Khởi tạo Wi-Fi
```bash
#wifi_init_sta(): 
// Hàm này khởi tạo Wi-Fi trong chế độ Station. Nó thực hiện các bước sau:
// Tạo một nhóm sự kiện.
// Khởi tạo giao diện mạng.
// Khởi tạo vòng lặp sự kiện.
// Tạo giao diện Wi-Fi mặc định.
// Khởi tạo Wi-Fi.
// Đăng ký xử lý sự kiện.
// Thiết lập cấu hình Wi-Fi.
// Bắt đầu kết nối Wi-Fi.
// Chờ kết quả kết nối.
```
## Hàm chính
```bash
#app_main(): 
// Hàm chính của chương trình. Nó thực hiện các bước sau:
// Khởi tạo lưu trữ không bay hơi (NVS).
// Khởi tạo Wi-Fi bằng hàm wifi_init_sta().
``` 
## Tóm tắt
- Đoạn code này thực hiện các bước sau để kết nối thiết bị ESP32 đến một mạng Wi-Fi:
- Khởi tạo các thư viện và biến cần thiết.
- Thiết lập cấu hình Wi-Fi.
-Khởi tạo Wi-Fi và đăng ký xử lý sự kiện.
- Bắt đầu kết nối Wi-Fi.
- Chờ kết quả kết nối và xử lý các sự kiện liên quan.
- Đoạn code này cung cấp một ví dụ cơ bản về cách kết nối Wi-Fi trên ESP32 và có thể được mở rộng
và tùy chỉnh để phù hợp với các ứng dụng khác nhau.
