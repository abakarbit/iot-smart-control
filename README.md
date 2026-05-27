# Smart Control — IoT Platform

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](license.txt)
[![PHP Version](https://img.shields.io/badge/PHP-%3E%3D7.4-purple.svg)](https://php.net)
[![CodeIgniter](https://img.shields.io/badge/CodeIgniter-3.x-orange.svg)](https://codeigniter.com)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-blue.svg)](https://www.mysql.com)
[![MQTT](https://img.shields.io/badge/Protocol-MQTT-green.svg)](https://mqtt.org)

A web-based IoT device management and monitoring platform developed by **Hardy Industries**. Smart Control enables users to register, configure, and remotely control MQTT-enabled IoT boards through a centralized dashboard — supporting real-time sensor telemetry for 18+ sensor types, subscription-based access control, and a public company CMS.

---

## Table of Contents

- [About the Project](#about-the-project)
- [Built With](#built-with)
- [System Architecture](#system-architecture)
- [Database Schema](#database-schema)
- [Features](#features)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
  - [Configuration](#configuration)
  - [Database Setup](#database-setup)
- [Usage](#usage)
  - [Admin Panel](#admin-panel)
  - [Member Dashboard](#member-dashboard)
  - [MQTT Device Integration](#mqtt-device-integration)
  - [Sensor Data API Endpoint](#sensor-data-api-endpoint)
- [Environment Variables & Configuration](#environment-variables--configuration)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)
- [Contact](#contact)

---

## About the Project

Smart Control is a multi-tenant IoT platform built on the **CodeIgniter 3 MVC framework**. The platform solves the operational challenge of managing distributed IoT nodes — such as microcontroller-based sensor boards and actuator controllers — from a single, unified web interface.

### Core Problem Solved

Organizations deploying multiple IoT devices (e.g., environmental sensors, smart switches, power meters) across facilities require a centralized platform that can:

1. **Register and authenticate** physical boards using unique Board IDs and MAC addresses.
2. **Map MQTT publish/subscribe topics** per device and per sensor category without modifying firmware.
3. **Record time-series sensor data** with millisecond-level timestamp precision for historical analysis.
4. **Gate access** via a subscription/activation model — ensuring only active, paying users can interact with connected devices.
5. **Visualize telemetry** in real time through interactive charts and filterable history tables.

### Architectural Decision Rationale

| Decision | Rationale |
|---|---|
| **CodeIgniter 3** | Lightweight MVC with minimal abstraction overhead; suitable for rapid deployment on shared PHP hosting |
| **MQTT over HTTP polling** | Event-driven pub/sub reduces server load and achieves near-real-time device responsiveness |
| **MySQL 8.0 with MyISAM (sensor tables)** | MyISAM provides faster `INSERT` throughput for high-frequency time-series writes from IoT nodes |
| **Session-based authentication** | Stateful sessions reduce token management complexity for a controlled SaaS environment |
| **jQuery + DataTables + Chart.js** | Proven client-side stack with minimal bundle size for dashboard interactivity |

---

## Built With

### Backend

| Technology | Version | Role |
|---|---|---|
| PHP | ≥ 7.4 | Server-side runtime |
| CodeIgniter | 3.x | MVC Framework |
| MySQL | 8.0 | Relational database |
| MQTT Protocol | 3.1.1 / 5.0 | IoT messaging protocol |

### Frontend

| Technology | Role |
|---|---|
| Bootstrap 4 | Responsive UI framework |
| Chart.js | Real-time sensor telemetry charts |
| DataTables | Server-side sortable/filterable tables |
| jQuery | AJAX, DOM manipulation |
| DateRangePicker | History time-range filtering |
| CKEditor | Rich-text CMS content editor |
| Moment.js | Date/time formatting |

### Libraries (Composer / Built-in)

| Library | Purpose |
|---|---|
| `FPDF` | PDF report generation |
| `CI Datatables` | Server-side DataTables integration |
| `CI Template` | Layered view rendering (admin / member / public templates) |

### IoT Hardware (Client-Side)

Compatible with any MQTT-capable embedded device, including:

- ESP8266 / ESP32 microcontrollers
- Arduino with Ethernet/Wi-Fi shield
- Raspberry Pi with Paho MQTT
- Any device publishing to a configurable MQTT broker

---

## System Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                        IoT Devices                            │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐   │
│  │ ESP8266/ESP32│   │ Arduino +    │   │ Raspberry Pi     │   │
│  │ Temperature, │   │ Power Meter  │   │ Multi-Sensor Node│   │
│  │ Humidity,    │   │ Ammeter,     │   │ CO2, PM2.5, UV,  │   │
│  │ Air Pressure │   │ Voltmeter    │   │ Wind, Rain       │   │
│  └──────┬───────┘   └──────┬───────┘   └────────┬─────────┘   │
│         │                  │                    │             │
└─────────┼──────────────────┼────────────────────┼─────────────┘
          │   MQTT Publish   │                    │
          ▼                  ▼                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                     MQTT Broker                                 │
│              (Mosquitto / EMQX / HiveMQ)                        │
│        topic: {namespace}/{boardId}/{sensorType}                │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Subscribe (Server-side consumer)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Smart Control Web Platform                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              CodeIgniter 3 MVC Application               │   │
│  │                                                          │   │
│  │  Controllers          		            Views             │   │
│  │  ├── Login.php       			       ├── template/      │   │
│  │  ├── Index.php       				   ├── control/       │   │
│  │  └── control/                         └── login.php      │   │
│  │      ├── Control.php                                     │   │
│  │      └── Login.php                                       │   │
│  │                                                          │   │
│  │  Libraries: Template | Datatables | FPDF | Upload        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                           │                                     │
│              MySQL 8.0 Database                                 │
│   (sensor | mqtt_board | mqtt_pub_sub | boardkey | usercontrol) │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
                 Web Browser (Admin / Member)
          Chart.js | DataTables | jQuery AJAX
```

### Data Flow — Sensor Telemetry

```
Device → MQTT Broker → [Consumer Script] → INSERT sensor table
                                              (boardkey, sensor_type, data, timestamp)
                                                        │
                                                        ▼
                                         Control.php → grafik() / history()
                                                        │
                                                        ▼
                                         Chart.js renders time-series graph
```

### Data Flow — Device Control (Actuator)

```
User clicks ON/OFF → jQuery AJAX → Control.php endpoint
    → INSERT mqtt_history (command log)
    → Publish to topic_pub (via MQTT client)
        → Device receives command on topic_sub
            → Actuator executes (relay ON/OFF)
```

---

## Database Schema

### Core Tables

```
usercontrol          boardkey              mqtt_board
─────────────        ──────────────        ──────────────────
iduser  (PK)         id       (PK)         idboard  (PK, char)
username             boardkey (FK)         tool     (sensor list)
email                status               macaddress
password (bcrypt)    tool                 status
gender               label
level                alias
status               iduser   (FK)
img                  dashboard
endaktifasi
```

```
mqtt_pub_sub         mqtt_user             sensor
──────────────       ──────────────        ──────────────
id      (PK)         idboard  (FK)         idsensor (PK)
idboard (FK)         iduser   (FK)         tgl
category             nameboard             boardkey (FK)
topic_pub            status                sensor
topic_sub            category              date (timestamp)
idicon  (FK)                               data (float)
```

```
mqtt_history         mqtt_icon             mqtt_category_img
──────────────       ──────────────        ──────────────────
idhistory (PK)       idicon    (PK)        id       (PK)
date (timestamp)     nameicon              iduser
idboard  (FK)        iconbefore            idboard
data                 iconafter             category
ket                                        img
```

### CMS Tables

```
about | banner | contact | news | product | daftarsensor | log
```

### Sensor Type Catalog (`daftarsensor`)

| ID | Sensor | Polling Interval |
|---|---|---|
| 1 | Temperature | 60,000 ms |
| 2 | Humidity | 60,000 ms |
| 3 | Air_Pressure | 240,000 ms |
| 4 | CO2 | 60,000 ms |
| 5–7 | PM_1 / PM_2.5 / PM_10 | 60,000 ms |
| 8–10 | Light_Intensity / UV / Solar_Radiation | 60,000 ms |
| 11–13 | Rain_Gauge / Wind_Speed / Wind_Direction | 60,000 ms |
| 14–15 | Soil_Moisture / Water_Level | 60,000 ms |
| 16–18 | Voltmeter / Ammeter / Powermeter | 60,000 ms |

---

## Features

### Authentication & Access Control

- Session-based login with `password_hash` (bcrypt, cost factor 10) + pepper salt
- Role levels: **Admin** and **Member**
- Account activation/expiry date enforcement (`endaktifasi`)
- Flash message feedback for all auth events

### Device & Board Management

- Register MQTT boards with unique Board ID, MAC address, and multi-sensor tool list
- Edit device alias names inline (AJAX, no page reload)
- Pin a primary device to the member dashboard widget
- Full CRUD for board registry with duplicate MAC address validation

### MQTT Topic Management

- Map per-board publish and subscribe topics per sensor category
- Icon state pairs (before/after) for visual device status representation
- Category-based device grouping with custom room/area images

### Sensor Monitoring & History

- Time-series telemetry stored per board, per sensor type
- Interactive line charts (Chart.js) with real-time data rendering
- Filterable history table with DateRangePicker
- Search history across all boards
- Paginated activity logs (on/off command history)

### Subscription System

- Admin manages user activation periods with start/end date enforcement
- Activation log (`log` table) for audit trail

### Company CMS (Public Portal)

- About, Services, Products (with marketplace links: Tokopedia, Shopee, WhatsApp)
- News/Blog with categories, tags, and slug-based URLs
- Dynamic banner management (On/Off toggle per page)
- Contact information and Google Maps embed

---

## Getting Started

### Prerequisites

| Requirement | Minimum Version | Notes |
|---|---|---|
| PHP | 7.4 | Extensions: `mysqli`, `mbstring`, `json`, `gd`, `session` |
| MySQL / MariaDB | 8.0 / 10.4 | UTF8MB4 collation required |
| Apache / Nginx | 2.4+ | `mod_rewrite` enabled (Apache) |
| Composer | 2.x | For dependency management |
| MQTT Broker | Any | Mosquitto, EMQX, or HiveMQ |

#### Apache Configuration

Ensure `mod_rewrite` is enabled and `AllowOverride All` is set for the project directory:

```bash
sudo a2enmod rewrite
sudo systemctl restart apache2
```

#### Nginx Configuration (alternative)

```nginx
location / {
    try_files $uri $uri/ /index.php?$query_string;
}
```

---

### Installation

**1. Clone the repository**

```bash
git clone https://github.com/abakarbit/iot-smart-control.git
cd iot-smart-control
```

**2. Install PHP dependencies**

```bash
composer install
```

**3. Set directory permissions**

```bash
chmod -R 755 application/
chmod -R 777 application/cache/
chmod -R 777 application/logs/
chmod -R 777 galery/
```

**4. Configure the web server document root**

Point your virtual host or server root to the project's root directory (where `index.php` resides).

```apache
<VirtualHost *:80>
    ServerName smartcontrol.local
    DocumentRoot /var/www/html/iot-smart-control
    <Directory /var/www/html/iot-smart-control>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

---

### Configuration

#### Database Connection

Edit `application/config/database.php`:

```php
$db['default'] = array(
    'dsn'      => '',
    'hostname' => 'localhost',       // Database host
    'username' => 'your_db_user',   // Database username
    'password' => 'your_db_pass',   // Database password
    'database' => 'smart_control',  // Database name
    'dbdriver' => 'mysqli',
    'dbprefix' => '',
    'pconnect' => FALSE,
    'db_debug' => (ENVIRONMENT !== 'production'),
    'cache_on' => FALSE,
    'cachedir' => '',
    'char_set' => 'utf8mb4',
    'dbcollat' => 'utf8mb4_general_ci',
);
```

> **Security Note:** Set `db_debug` to `FALSE` in production to prevent database error disclosure.

#### Base URL

Edit `application/config/config.php`. For local development:

```php
$config['base_url'] = 'http://localhost/iot-smart-control/';
```

For production, the base URL is auto-detected dynamically:

```php
$http = 'http' . ((isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] == 'on') ? 's' : '') . '://';
$newurl = str_replace("index.php", "", $_SERVER['SCRIPT_NAME']);
$config['base_url'] = "$http" . $_SERVER['SERVER_NAME'] . $newurl;
```

#### Session & Security

```php
$config['encryption_key']  = 'YOUR_RANDOM_32_CHAR_KEY_HERE';  // REQUIRED — generate a strong key
$config['sess_driver']     = 'files';
$config['sess_expiration'] = 7200;  // 2 hours
```

Generate a secure encryption key:

```bash
php -r "echo bin2hex(random_bytes(16));"
```

---

### Database Setup

**1. Create the database**

```sql
CREATE DATABASE smart_control CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

**2. Import the SQL dump**

```bash
mysql -u your_db_user -p smart_control < smart.sql
```

**3. Verify the import**

```sql
USE smart_control;
SHOW TABLES;
```

Expected output should include: `about`, `banner`, `boardkey`, `contact`, `daftarsensor`, `log`, `mqtt_board`, `mqtt_category_img`, `mqtt_history`, `mqtt_icon`, `mqtt_pub_sub`, `mqtt_user`, `news`, `product`, `sensor`, `usercontrol`.

**4. Create initial admin account**

```sql
INSERT INTO usercontrol (username, email, gender, level, password, status, endaktifasi)
VALUES (
    'admin',
    'admin@example.com',
    'male',
    'admin',
    '$2y$10$...', -- replace with hash generated below
    1,
    '2099-12-31'
);
```

> Generate a proper hash via PHP CLI:
> ```bash
> php -r "echo password_hash('yourpassword@ql153Rd@dU', PASSWORD_DEFAULT, ['cost' => 10]);"
> ```

---

## Usage

### Admin Panel

Access the admin dashboard at `http://your-domain/dashadmin` after logging in with an admin-level account.

**Key admin routes:**

| Route | Description |
|---|---|
| `GET /dashadmin` | Main admin dashboard with device overview |
| `GET /akun` | User account management (CRUD) |
| `GET /boardkey` | MQTT board registry management |
| `GET /boardtool` | MQTT topic pub/sub configuration |
| `GET /icon` | Device state icon pair management |
| `GET /log` | Subscription activation audit log |
| `GET /grafik` | Sensor telemetry chart viewer |
| `GET /history` | MQTT command event history |
| `GET /searchhistory` | Cross-board history search |
| `GET /settingdevice` | User device alias and dashboard pin settings |

---

### Member Dashboard

After login, members access their assigned IoT devices:

| View | Description |
|---|---|
| `control/home` | Dashboard with pinned device widget |
| `control/device` | All registered boards for this user |
| `control/history` | Personal device event history |

---

### MQTT Device Integration

To connect an IoT board to the platform:

**1. Register the board** in the Admin Panel under **Board Key** with:
- A unique `Board ID` (alphanumeric, e.g., `SE003`)
- The device's `MAC Address`
- Comma-separated sensor/tool list (e.g., `Temperature,Humidity,Powermeter`)

**2. Configure topics** under **Board Tool** by mapping:
- `topic_pub`: Topic the server **publishes to** (device subscribes — for actuator commands)
- `topic_sub`: Topic the server **subscribes to** (device publishes — for sensor data)
- Assign a sensor `category` and UI icon pair

**3. Flash your device firmware** to publish sensor data to the configured `topic_sub`:

```cpp
// Example Arduino/ESP8266 sketch (pseudo-code)
const char* topic_sub = "hardy/sensor/SE003/temperature";
float temperature = dht.readTemperature();

// Publish float value as string payload
mqttClient.publish(topic_sub, String(temperature).c_str());
```

**4. Assign the board to a user** via **Member Management** with a room/area category.

---

### Sensor Data API Endpoint

The platform stores telemetry via direct database inserts. For MQTT broker bridge scripts, insert data in this format:

```sql
INSERT INTO sensor (tgl, boardkey, sensor, date, data)
VALUES (
    CURDATE(),      -- date partition key
    'SE003',        -- board ID (matches boardkey.boardkey)
    'temperature',  -- sensor type, lowercase
    NOW(),          -- timestamp
    28.5            -- float reading
);
```

---

## Environment Variables & Configuration

| File | Key | Description | Example |
|---|---|---|---|
| `config/database.php` | `hostname` | DB host | `localhost` |
| `config/database.php` | `username` | DB user | `smart_user` |
| `config/database.php` | `password` | DB password | *(secure value)* |
| `config/database.php` | `database` | DB name | `smart_control` |
| `config/config.php` | `base_url` | Application base URL | `https://smartcontrol.tech/` |
| `config/config.php` | `encryption_key` | Session encryption key | 32-char random hex |
| `config/config.php` | `sess_expiration` | Session lifetime (seconds) | `7200` |
| `config/config.php` | `index_page` | Set blank to remove `index.php` from URLs | `''` |

---


## Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature-name`
3. Commit changes following [Conventional Commits](https://www.conventionalcommits.org/): `git commit -m "feat: add WebSocket push for sensor dashboard"`
4. Push to the branch: `git push origin feature/your-feature-name`
5. Open a Pull Request against `main`

Please read [contributing.md](contributing.md) for full contribution guidelines.

---

## License

Distributed under the **MIT License**. See [license.txt](license.txt) for full terms.

---

## Contact

**Hardy Industries** — Robotics, IoT, AI & Instrumentation

| Channel | Detail |
|---|---|
| Website | [hardyindustries.tech](https://hardyindustries.tech) |
| Email | office@hardyindustries.tech |
| Phone / WhatsApp | +62 857 1436 9716 |
| YouTube | [youtube.com/hardyindustries](https://youtube.com) |
| Instagram | [@hardyindustries](https://instagram.com) |
| Address | Jl. Situsela No.01, Kaumpandak, Karadenan, Cibinong, Kabupaten Bogor 16913 |

---

*Built with precision by Hardy Industries — Engineering Solutions for a Connected World.*
