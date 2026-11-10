# 🛒 RFID-Based SmartTrolley \| IoT Supermarket Product Locator

> **Find products faster. Scan an RFID card or browse the product menu
> to locate items inside a supermarket.**

An ESP32-based smart trolley prototype that helps customers find the
exact store location of a product. The system uses an **MFRC522 RFID
reader**, **ESP32-WROOM-32**, **0.96-inch OLED display**, four push
buttons and **Firebase Realtime Database**.

Customers can either: - Browse the product list using **UP, DOWN, SELECT
and BACK** buttons. - Scan a registered RFID card to retrieve the
associated product location.

The ESP32 reads the RFID UID, looks up the matching product record in
Firebase over Wi-Fi/HTTPS, and displays the product's **floor, section,
rack and shelf** on the OLED.

------------------------------------------------------------------------

## ✨ Features

-   RFID-based product identification using RC522.
-   Manual product search using four push buttons.
-   OLED interface for menus, status messages and product locations.
-   Firebase Realtime Database for centrally stored product information.
-   Wi-Fi communication between ESP32 and Firebase.
-   Product locations can be updated in Firebase without reprogramming
    the ESP32.
-   Modular Arduino firmware split into `.ino`, `.h` and `.cpp` files.
-   Unknown-card and communication-error handling.

## 🧰 Components Required

  ------------------------------------------------------------------------
  Component                                 Quantity Purpose
  --------------------- ---------------------------- ---------------------
  ESP32-WROOM-32                                   1 Main controller and
  development board                                  Wi-Fi communication

  MFRC522 RFID reader                              1 Reads compatible RFID
  module                                             card UIDs

  Compatible 13.56 MHz                     2 or more Product
  RFID cards                                         identification

  0.96-inch SSD1306                                1 Displays product and
  OLED (128 × 64)                                    location details

  Push buttons                                     4 UP, DOWN, SELECT and
                                                     BACK navigation

  Breadboard / PCB                                 1 Circuit assembly

  Jumper wires                           As required Connections

  USB power supply /                               1 Prototype power
  power bank                                         
  ------------------------------------------------------------------------

------------------------------------------------------------------------

## 🧠 System Block Diagram

``` mermaid
flowchart LR
    A[RFID Card] -->|13.56 MHz RFID| B[RC522 Reader]
    B -->|SPI| C[ESP32-WROOM-32]
    D[4 Push Buttons] -->|GPIO| C
    C -->|I2C| E[0.96 inch OLED]
    C <-->|Wi-Fi / HTTPS| F[Wi-Fi Router]
    F <-->|Internet| G[Firebase Realtime Database]
    G -->|Product JSON| C
```

**System operation:** The ESP32 is the central controller. It receives
input from the RFID reader or push buttons, communicates with Firebase
through Wi-Fi, processes the returned product data and updates the OLED.

------------------------------------------------------------------------

## 🔌 Circuit Diagram and Pin Connections

### 1. RC522 RFID Reader to ESP32

  RC522 Pin     ESP32-WROOM-32 GPIO Description
  ----------- --------------------- -------------------------------
  SDA / SS                   GPIO 5 SPI chip select
  SCK                       GPIO 18 SPI clock
  MOSI                      GPIO 23 ESP32 data to RC522
  MISO                      GPIO 19 RC522 data to ESP32
  RST                       GPIO 22 RFID reader reset
  VCC                          3.3V Power supply
  GND                           GND Common ground
  IRQ                 Not connected Not required by this firmware

> **Important:** The RC522 pin marked `SDA` is used as SPI chip-select
> (SS) in this circuit. It is not the I²C SDA connection. Power the
> RC522 from **3.3V only**; do not connect its VCC to 5V.

### 2. OLED Display to ESP32

  OLED Pin     ESP32-WROOM-32 GPIO
  ---------- ---------------------
  VCC                       3.3V\*
  GND                          GND
  SDA                      GPIO 21
  SCL                      GPIO 17

*The wiring assumes a 3.3V-compatible OLED module. Check the
specifications of your particular display board.*

### 3. Push Buttons to ESP32

Each button connects between its GPIO pin and GND. The firmware uses
`INPUT_PULLUP`.

  Button     ESP32 GPIO Other button terminal
  -------- ------------ -----------------------
  UP            GPIO 25 GND
  DOWN          GPIO 26 GND
  SELECT        GPIO 27 GND
  BACK          GPIO 32 GND

**Button logic:** - Not pressed → `HIGH` - Pressed → `LOW`

### Wiring Overview

``` text
                 ESP32-WROOM-32
              +-------------------+
 RC522 SS  ---| GPIO 5             |
 RC522 SCK ---| GPIO 18            |
 RC522 MOSI --| GPIO 23            |
 RC522 MISO --| GPIO 19            |
 RC522 RST ---| GPIO 22            |
              |                   |
 OLED SDA ----| GPIO 21            |
 OLED SCL ----| GPIO 17            |
              |                   |
 UP button ---| GPIO 25            |
 DOWN button -| GPIO 26            |
 SELECT ------| GPIO 27            |
 BACK --------| GPIO 32            |
              |                   |
              | 3.3V          GND |---- Common GND
              +-------------------+

RC522 VCC  -> 3.3V
OLED VCC   -> 3.3V (if supported by module)
All module grounds -> ESP32 GND
```

All modules must share a common ground. Use a stable regulated supply
capable of handling ESP32 Wi-Fi current peaks.

------------------------------------------------------------------------

## ☁️ Firebase Realtime Database

Firebase is used as the cloud database to store the product-to-location
mapping.

### Why Firebase?

-   **Cloud-hosted:** Product data is available to the trolley through
    the Internet.
-   **JSON structure:** Product information is naturally represented as
    fields such as name, floor, rack and shelf.
-   **REST API:** ESP32 can retrieve records using HTTPS GET requests.
-   **Central updates:** Store staff can change a product location in
    Firebase without modifying the ESP32 firmware.
-   **Authentication and rules:** Access can be restricted using
    Firebase Authentication and Realtime Database Security Rules.
-   **Expandable:** New product records can be added without changing
    the basic UID lookup method.

Firebase was selected for this prototype because it avoids the need to
build and host a separate backend server. Other databases or a local
server could also be used.

### Database Structure

The RFID UID (converted to uppercase hexadecimal without colons) is used
as the product record key.

``` json
{
  "products": {
    "00E1495C": {
      "name": "Good Day Biscuits",
      "category": "Biscuits",
      "floor": "F0",
      "section": "A1",
      "rack": "R3",
      "shelf": "005",
      "location": "F0-A1-R3-005",
      "availability": true
    },
    "DCDA0002": {
      "name": "Dairy Milk",
      "category": "Chocolate",
      "floor": "F0",
      "section": "A2",
      "rack": "R1",
      "shelf": "002",
      "location": "F0-A2-R1-002",
      "availability": true
    }
  }
}
```

### Prototype Product Mapping

  Product             RFID UID read by RC522   Firebase key   Store location
  ------------------- ------------------------ -------------- ----------------
  Good Day Biscuits   `00:E1:49:5C`            `00E1495C`     `F0-A1-R3-005`
  Dairy Milk          `DC:DA:00:02`            `DCDA0002`     `F0-A2-R1-002`

**Location code:** `F0-A1-R3-005` means Floor 0 → Section A1 → Rack 3 →
Shelf 005. These are sample prototype locations; update them to match
the actual store layout.

------------------------------------------------------------------------

## 🔄 Working Principle and Workflow

### A. Manual Product Search

``` mermaid
flowchart TD
    A[Power ON] --> B[Initialize ESP32, OLED, RC522 and Buttons]
    B --> C[Connect to Wi-Fi]
    C --> D[Authenticate with Firebase]
    D --> E[Display Main Menu]
    E --> F[User selects Search Product]
    F --> G[Retrieve Product List from Firebase]
    G --> H[Display Product Names]
    H --> I[UP / DOWN to Navigate]
    I --> J[Press SELECT]
    J --> K[Request Selected Product Record]
    K --> L[Parse JSON Response]
    L --> M[Display Floor, Section, Rack and Shelf]
    M --> N[Press BACK]
    N --> E
```

### B. RFID Product Search

``` mermaid
flowchart TD
    A[User selects Scan RFID] --> B[OLED prompts: Scan Card]
    B --> C[RC522 detects RFID card]
    C --> D[Read Card UID]
    D --> E[Convert UID to Firebase key]
    E --> F[ESP32 sends authenticated HTTPS GET]
    F --> G[Firebase searches product record]
    G --> H{Product found?}
    H -->|Yes| I[Firebase returns product JSON]
    I --> J[ESP32 parses product fields]
    J --> K[OLED displays product location]
    H -->|No| L[Display Unknown Card / Not Registered]
    K --> M[Press BACK]
    L --> M
    M --> A
```

### Example: Scanning Good Day Biscuits

1.  RC522 reads the card UID: `00:E1:49:5C`.
2.  ESP32 converts it to the database key: `00E1495C`.
3.  ESP32 connects to Firebase using Wi-Fi and an authenticated HTTPS
    request.
4.  ESP32 requests the record at `/products/00E1495C.json`.
5.  Firebase returns the product details in JSON format.
6.  ESP32 parses the JSON response using ArduinoJson.
7.  OLED displays:

``` text
GOOD DAY BISCUITS

Floor   : F0
Section : A1
Rack    : R3
Shelf   : 005

BACK: Return
```

------------------------------------------------------------------------

## 🌐 How ESP32 Retrieves Data from the Web

The ESP32 does not open a normal website like a browser. It communicates
with Firebase through its HTTPS REST API.

1.  **Wi-Fi connection:** ESP32 joins the configured Wi-Fi network.
2.  **Authentication:** ESP32 sends the Firebase Web API key, registered
    email and password to the Firebase Authentication sign-in endpoint
    over HTTPS.
3.  **ID token:** Firebase returns an ID token after successful
    authentication.
4.  **RFID identification:** RC522 reads the card UID and ESP32 converts
    it to the database key.
5.  **HTTPS GET request:** ESP32 requests the product record from the
    Realtime Database and includes the ID token.
6.  **JSON response:** Firebase returns the product data.
7.  **JSON parsing:** ArduinoJson extracts the product name, category,
    floor, section, rack, shelf and availability.
8.  **OLED update:** ESP32 displays the retrieved product location.

Example database path:

``` text
/products/00E1495C.json
```

The actual request uses the Realtime Database host shown in the Firebase
Console and includes the authentication token.

------------------------------------------------------------------------

## 🖥️ OLED User Interface

### Main Menu

``` text
     SMART TROLLEY

 > Search Product

   Scan RFID Card
```

### Product Selection

``` text
    SELECT PRODUCT

 > Good Day Biscuits

   Dairy Milk
```

### RFID Scan Screen

``` text
     SCAN RFID CARD

 Place card near
 the reader...

 BACK: Main menu
```

### Product Location Screen

``` text
GOOD DAY BISCUITS

Floor   : F0
Section : A1
Rack    : R3
Shelf   : 005

BACK: Return
```

------------------------------------------------------------------------

## 💻 Software and Libraries

### Development Environment

-   Arduino IDE
-   ESP32 Arduino Core
-   Firebase Realtime Database
-   Firebase Authentication

### Arduino Libraries

  Library            Purpose
  ------------------ ---------------------------------------
  MFRC522            RC522 RFID reader control
  SPI                Communication between ESP32 and RC522
  Wire               I²C communication
  Adafruit SSD1306   OLED display driver
  Adafruit GFX       OLED graphics and text
  WiFi               ESP32 Wi-Fi connection
  WiFiClientSecure   HTTPS/TLS client
  HTTPClient         HTTP/HTTPS requests
  ArduinoJson        JSON creation and parsing

------------------------------------------------------------------------

## 📁 Project File Structure

``` text
SmartTrolley/
│
├── SmartTrolley.ino
├── Config.h
├── Product.h
│
├── FirebaseClient.h
├── FirebaseClient.cpp
│
├── DisplayUI.h
├── DisplayUI.cpp
│
└── README.md
```

  -----------------------------------------------------------------------
  File                                Responsibility
  ----------------------------------- -----------------------------------
  `SmartTrolley.ino`                  Main program, menu navigation, RFID
                                      handling and system integration

  `Config.h`                          Wi-Fi, Firebase credentials and pin
                                      definitions

  `Product.h`                         Product data structure

  `FirebaseClient.h`                  Firebase function declarations

  `FirebaseClient.cpp`                Firebase authentication and
                                      database requests

  `DisplayUI.h`                       OLED display function declarations

  `DisplayUI.cpp`                     OLED menu and product display
                                      functions
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## ⚙️ Setup and Installation

### 1. Install Arduino IDE

Install Arduino IDE and configure ESP32 board support.

### 2. Install Required Libraries

Install the libraries listed in the Software and Libraries section using
Arduino IDE Library Manager.

### 3. Configure Firebase

1.  Create a Firebase project.
2.  Enable Realtime Database.
3.  Enable Email/Password authentication.
4.  Create a Firebase user.
5.  Import the product JSON data.
6.  Configure authenticated read access for `/products` and deny client
    writes for the trolley.
7.  Copy the Web API key and exact Realtime Database URL.

### 4. Configure `Config.h`

Enter your own values locally:

``` cpp
#define WIFI_SSID "YOUR_WIFI_NAME"
#define WIFI_PASSWORD "YOUR_WIFI_PASSWORD"

#define FIREBASE_API_KEY "YOUR_FIREBASE_WEB_API_KEY"
#define FIREBASE_DATABASE_URL "YOUR_REALTIME_DATABASE_URL"

#define FIREBASE_USER_EMAIL "YOUR_FIREBASE_EMAIL"
#define FIREBASE_USER_PASSWORD "YOUR_FIREBASE_PASSWORD"
```

> **Security:** Never commit your real Wi-Fi password, Firebase account
> password, ID token or other private credentials to a public GitHub
> repository. Keep your personal `Config.h` out of version control and
> provide a `Config.example.h` with placeholders instead. The Firebase
> Web API key identifies the project; it is not a password, but it
> should still have appropriate API restrictions.

### 5. Upload the Firmware

-   Board: **ESP32 Dev Module**
-   Select the correct COM port.
-   Open `SmartTrolley.ino`.
-   Compile and upload.
-   Open Serial Monitor at **115200 baud**.

### 6. Test the System

  Test                      Expected result
  ------------------------- ----------------------------------
  Power ON                  SmartTrolley starts
  Wi-Fi connection          Connected status
  Firebase authentication   Authentication successful
  Database loading          Two products loaded
  Manual search             Product list appears
  Select Good Day           Location `F0-A1-R3-005` appears
  Scan Card 1               Good Day Biscuits appears
  Scan Card 2               Dairy Milk appears
  Scan unknown card         Unknown / not registered message
  Press BACK                Returns to previous screen

------------------------------------------------------------------------

## ⚠️ Limitations and Safety Notes

-   The current prototype requires Wi-Fi and Internet access for cloud
    product lookups; offline caching is not implemented.
-   The RC522 reads compatible 13.56 MHz cards using supported protocols
    (commonly ISO/IEC 14443A). A card's frequency alone does not
    guarantee compatibility.
-   RFID UIDs are identifiers, not secure authentication credentials.
    Some card UIDs can be cloned or emulated; this prototype is for
    product-location assistance, not payment or access-control security.
-   The current development firmware uses `setInsecure()` for HTTPS
    testing. This skips server-certificate verification. Replace it with
    proper CA certificate validation before real deployment.
-   Keep the RC522 supply at 3.3 V.
-   Keep all module grounds connected together.
-   Maintain accurate product locations in Firebase when supermarket
    products are moved.

------------------------------------------------------------------------

## 🚀 Future Improvements

-   Add offline product caching using ESP32 LittleFS.
-   Develop an admin webpage to add products and update locations.
-   Expand the database to hundreds or thousands of products.
-   Add product categories and text-based search.
-   Add battery monitoring and low-battery alerts.
-   Add multiple trolley identification and usage analytics.
-   Improve network recovery and HTTPS certificate validation.
-   Add a larger display or voice guidance for accessibility.

------------------------------------------------------------------------

## 🎯 Project Outcome

The RFID-Based SmartTrolley demonstrates how embedded systems, RFID and
cloud databases can be combined to improve the supermarket shopping
experience. Customers can search for products manually or scan an RFID
card to retrieve its location. Firebase provides centralized product
data, while the ESP32 processes the information and displays the result
on the OLED.

**Built with ESP32 • RFID • OLED • Firebase • Arduino IDE**

------------------------------------------------------------------------

## 👨‍💻 Project Links

-   **GitHub:** [PanduTechLab](https://github.com/PanduTechLab/)
-   **YouTube:** [PandusTechLab](https://www.youtube.com/@PandusTechLab)
-   **Instagram:**
    [@pandutechlab](https://www.instagram.com/pandutechlab)

> *Learn Like an Engineer, Be Like an Engineer!*
