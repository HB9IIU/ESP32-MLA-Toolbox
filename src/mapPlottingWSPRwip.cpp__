#include <myconfig.h>       // your configuration
#include <WiFi.h>           // Wi-Fi Library
#include <ESPmDNS.h>        // For mDNS (local hostname resolution)
#include <PNGdec.h>         //PNG image decoding library
#include <TFT_eSPI.h>       //TFT graphics library 
#include <HTTPClient.h>
#include "FS.h"
#include "SPIFFS.h"  




//#include <XPT2046_Touchscreen.h>



#define SCREEN_WIDTH 320
#define SCREEN_HEIGHT 240
#define ROW_HEIGHT 30
#define MAX_VISIBLE_ROWS (SCREEN_HEIGHT / ROW_HEIGHT)
#define MAX_ENTRIES 50

PNG png; // create png instance
TFT_eSPI tft = TFT_eSPI(); // create tft instance

int deviceMODE=0;


//#define XPT2046_CS 33
//#define XPT2046_IRQ 36
//SPIClass touchscreenSPI = SPIClass(VSPI);
//XPT2046_Touchscreen touchscreen(XPT2046_CS, XPT2046_IRQ);

struct Spot
{
    String time;
    String callsign;
    String band;
    String distance;
    float lat;
    float lon;
};

Spot spots[MAX_ENTRIES];
int totalEntries = 0;
int scrollIndex = 0;

struct BandMap
{
    int bandValue;
    const char *bandLabel;
};

BandMap bands[] = {
    {3, "80"}, {5, "60"}, {7, "40"}, {10, "30"}, {14, "20"}, {18, "17"}, {21, "15"}, {24, "12"}, {28, "10"}, {50, "6"}};

#define NUM_BANDS (sizeof(bands) / sizeof(bands[0]))

const char *getBandLabel(int bandValue)
{
    for (int i = 0; i < NUM_BANDS; i++)
    {
        if (bands[i].bandValue == bandValue)
            return bands[i].bandLabel;
    }
    return "Unknown";
}
// GOOD INSIDE HERE ---------------------------------------------------------------------------

// Function to connect to Wi-Fi
void connectToWifi() {
    int attemptCount = 0;
    bool connected = false;

    Serial.println("Starting Wi-Fi connection...");

    // Try connecting to the main Wi-Fi network
    while (attemptCount < 5 && !connected) {
        attemptCount++;
        Serial.print("Attempting to connect to Wi-Fi (Attempt ");
        Serial.print(attemptCount);
        Serial.println(")...");
        
        WiFi.begin(mainSSID, mainPassword);

        // Wait for connection
        int waitTime = 0;
        while (WiFi.status() != WL_CONNECTED && waitTime < 30) {
            delay(1000);
            Serial.print(".");
            waitTime++;
        }

        if (WiFi.status() == WL_CONNECTED) {
            Serial.println("\nConnected to the main Wi-Fi network!");
            connected = true;
        } else {
            Serial.println("\nFailed to connect to the main Wi-Fi. Trying alternate Wi-Fi...");
        }
    }

    // If connection to the main Wi-Fi failed, try alternate Wi-Fi
    if (!connected) {
        attemptCount = 0;
        while (attemptCount < 5 && !connected) {
            attemptCount++;
            Serial.print("Attempting to connect to Alternate Wi-Fi (Attempt ");
            Serial.print(attemptCount);
            Serial.println(")...");

            WiFi.begin(alternateSSID, alternatePassword);

            // Wait for connection
            int waitTime = 0;
            while (WiFi.status() != WL_CONNECTED && waitTime < 30) {
                delay(1000);
                Serial.print(".");
                waitTime++;
            }

            if (WiFi.status() == WL_CONNECTED) {
                Serial.println("\nConnected to the alternate Wi-Fi network!");
                connected = true;
            } else {
                Serial.println("\nFailed to connect to the alternate Wi-Fi. Trying again...");
            }
        }
    }

    // If both Wi-Fi connections fail, reboot the ESP32
    if (!connected) {
        Serial.println("\nBoth Wi-Fi connections failed. Rebooting...");
        ESP.restart();
    }

    // Successfully connected, set the hostname for mDNS (local network resolution)
    if (connected) {
        Serial.print("Setting hostname: ");
        Serial.println(deviceHostname);

         // Once connected to Wi-Fi, set up mDNS
    if (!MDNS.begin(deviceHostname)) {
        Serial.println("Error setting up MDNS responder!");
    } else {
        Serial.println("MDNS responder started.");
        // Print the local IP address of the ESP32 (to verify it's connected)
        Serial.print("Device IP Address: ");
        Serial.println(WiFi.localIP());

        // Optional: Print the mDNS hostname
        Serial.print("You can now access your device using '");
        Serial.print(deviceHostname);
        Serial.println(".local' in your network.");
    }

        // Print the assigned IP address
        Serial.print("Or via device IP Address: http://");
        Serial.println(WiFi.localIP());  // Print the local IP address of the ESP32
        Serial.println();
    }
}






// bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
// Function block to display png images stored in flash
fs::File pngFile;// Global File handle (required for PNGdec callbacks)
void *fileOpen(const char *filename, int32_t *size)
{
    String fullPath = "/" + String(filename);
    pngFile = SPIFFS.open(fullPath, "r");
    if (!pngFile)
        return nullptr;
    *size = pngFile.size();
    return (void *)&pngFile;
}

void fileClose(void *handle)
{
    ((fs::File *)handle)->close();
}

int32_t fileRead(PNGFILE *handle, uint8_t *buffer, int32_t length)
{
    return ((fs::File *)handle->fHandle)->read(buffer, length);
}

int32_t fileSeek(PNGFILE *handle, int32_t position)
{
    return ((fs::File *)handle->fHandle)->seek(position);
}

void displayPNGfromSPIFFS(const char *filename, int duration_ms)
{
    if (!SPIFFS.begin(true))
    {
        Serial.println("Failed to mount SPIFFS!");
        return;
    }

    int16_t rc = png.open(filename, fileOpen, fileClose, fileRead, fileSeek, [](PNGDRAW *pDraw)
                          {
        uint16_t lineBuffer[480];  // Adjust to your screen width if needed
        png.getLineAsRGB565(pDraw, lineBuffer, PNG_RGB565_BIG_ENDIAN, 0xFFFFFFFF);
        tft.pushImage(0, pDraw->y, pDraw->iWidth, 1, lineBuffer); });

    if (rc == PNG_SUCCESS)
    {
        Serial.printf("Displaying PNG: %s\n", filename);
        tft.startWrite();
        png.decode(nullptr, 0);
        tft.endWrite();
    }
    else
    {
        Serial.println("PNG decode failed.");
    }

    delay(duration_ms);
}

// bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb

// ---------------------------------------------------------------------------------------------


void printDirectory(fs::FS &fs, const char *dirname, uint8_t levels = 5, String indent = "") {
    fs::File root = fs.open(dirname);
    if (!root || !root.isDirectory()) {
        Serial.printf("%s[!] Failed to open directory: %s\n", indent.c_str(), dirname);
        return;
    }

    Serial.printf("%s📁 %s\n", indent.c_str(), dirname);

    fs::File file = root.openNextFile();
    while (file) {
        if (file.isDirectory()) {
            // Recursive for subfolders
            printDirectory(fs, file.path(), levels - 1, indent + "  ");
        } else {
            Serial.printf("%s  📄 %s (%d bytes)\n", indent.c_str(), file.name(), file.size());
        }
        file = root.openNextFile();
    }
}






void drawTable()
{
    tft.fillScreen(TFT_BLACK);
    tft.setTextColor(TFT_WHITE, TFT_BLACK);

    tft.setFreeFont(&FreeMono9pt7b);

    // Set color for header
    tft.setTextColor(TFT_YELLOW, TFT_BLACK);
    tft.setCursor(1, 20);
    tft.print("Id Time  Call     Band  km");
    // Set color for table rows
    tft.setTextColor(TFT_WHITE, TFT_BLACK);
    for (int i = 0; i < MAX_VISIBLE_ROWS - 1; ++i)
    {
        int entry = scrollIndex + i;
        if (entry >= totalEntries)
            break;
        int y = (i + 2) * ROW_HEIGHT - 5;
        tft.setCursor(1, y);
        tft.printf("%2d %5s %-9s %2s %5s",
                   entry + 1,
                   spots[entry].time.c_str(),
                   spots[entry].callsign.c_str(),
                   spots[entry].band.c_str(),
                   spots[entry].distance.c_str());
    }
}
String urlencode(const String &str)
{
    String encodedStr = "";
    char buffer[4];
    for (unsigned int i = 0; i < str.length(); i++)
    {
        char c = str.charAt(i);
        if (isalnum(c) || c == '-' || c == '_' || c == '.' || c == '~')
        {
            encodedStr += c;
        }
        else
        {
            snprintf(buffer, sizeof(buffer), "%%%02X", c);
            encodedStr += buffer;
        }
    }
    return encodedStr;
}

void printSpotData()
{
    // Print table headers
    Serial.println("Time   | Callsign    | Band    | Distance (km) | Latitude  | Longitude");
    Serial.println("---------------------------------------------------------------");

    // Loop through the spots array and print each spot in a formatted way
    for (int i = 0; i < totalEntries; i++)
    {
        String timeStr = spots[i].time;

        // Ensure the time is exactly 5 characters long (HH:MM format)
        if (timeStr.length() == 4)
        {
            timeStr = "0" + timeStr; // Add leading zero for single-digit hours
        }

        String callsign = spots[i].callsign;
        String band = spots[i].band;
        String distance = spots[i].distance;
        String lat = String(spots[i].lat, 6); // Print latitude with 6 decimals
        String lon = String(spots[i].lon, 6); // Print longitude with 6 decimals

        // Print formatted data with fixed width for each column
        Serial.print(timeStr); // Time
        Serial.print(" | ");
        Serial.print(callsign); // Callsign
        Serial.print(" | ");
        Serial.print(band); // Band
        Serial.print(" | ");
        Serial.print(distance); // Distance
        Serial.print("   | ");
        Serial.print(lat); // Latitude
        Serial.print(" | ");
        Serial.println(lon); // Longitude
    }
}

void fetchData()
{
    String uncodedQuery = String("WITH RankedRecords AS (\n") +
                          String("  SELECT\n") +
                          String("    band,\n") +
                          String("    rx_sign,\n") +
                          String("    time,\n") +
                          String("    rx_lat,\n") +
                          String("    rx_lon,\n") +
                          String("    distance,\n") +
                          String("    ROW_NUMBER() OVER (PARTITION BY rx_sign ORDER BY time DESC) AS rn\n") +
                          String("  FROM wspr.rx\n") +
                          String("  WHERE time > subtractHours(now(), ") + HOURS_TO_BE_SUBTRACTED_IN_QUERY + String(")\n") +
                          String("    AND tx_sign = '") + CALLSIGN + String("'\n") +
                          String(")\n") +
                          String("SELECT\n") +
                          String("  band,\n") +
                          String("  rx_sign,\n") +
                          String("  time,\n") +
                          String("  rx_lat,\n") +
                          String("  rx_lon,\n") +
                          String("  distance\n") +
                          String("FROM RankedRecords\n") +
                          String("WHERE rn = 1\n") +
                          String("ORDER BY distance DESC\n") +
                          String("LIMIT 50;");
    String encodedQuery = urlencode(uncodedQuery);

    HTTPClient http;
    http.begin("https://db1.wspr.live:443/?query=" + encodedQuery);
    int httpCode = http.GET();

    if (httpCode > 0)
    {
        String payload = http.getString();

        int lineStart = 0;
        totalEntries = 0;
        while (lineStart < payload.length() && totalEntries < MAX_ENTRIES)
        {
            int lineEnd = payload.indexOf('\n', lineStart);
            if (lineEnd == -1)
                lineEnd = payload.length();

            String line = payload.substring(lineStart, lineEnd);
            lineStart = lineEnd + 1;
            if (line.length() == 0)
                continue;

            String fields[6];
            int fieldIndex = 0, lastPos = 0;
            while (fieldIndex < 6)
            {
                int tabPos = line.indexOf('\t', lastPos);
                if (tabPos == -1)
                    tabPos = line.length();
                fields[fieldIndex] = line.substring(lastPos, tabPos);
                lastPos = tabPos + 1;
                fieldIndex++;
            }

            String timestamp = fields[2];
            int hour = timestamp.substring(11, 13).toInt() + TZ_OFFSET_HOURS;
            if (hour >= 24)
                hour -= 24;
            if (hour < 0)
                hour += 24;
            String timeStr = String(hour) + ":" + timestamp.substring(14, 16);

            int bandNumeric = fields[0].toInt();
            const char *bandLabel = getBandLabel(bandNumeric);

            String callsign = fields[1];
            if (callsign.length() > 9)
            {
                callsign = callsign.substring(0, 8) + "."; // Truncate to 8 chars and add dot
            }
            else
            {
                while (callsign.length() < 9)
                    callsign += " "; // Pad to fixed width
            }

            spots[totalEntries].time = timeStr;
            spots[totalEntries].callsign = callsign;
            spots[totalEntries].band = String(bandLabel);
            spots[totalEntries].distance = fields[5].substring(0, fields[5].indexOf('.')); // Remove decimals and 'km'
            spots[totalEntries].lat = fields[3].toFloat();
            spots[totalEntries].lon = fields[4].toFloat();

            totalEntries++;
        }
    }
    http.end();
    printSpotData();
}

void drawGreatCircleO(float lat1, float lon1, float lat2, float lon2, uint16_t color = TFT_YELLOW)
{
    const int segments = 200; // Increase for smoother curves
    lat1 = radians(lat1);
    lon1 = radians(lon1);
    lat2 = radians(lat2);
    lon2 = radians(lon2);

    for (int i = 0; i < segments; i++)
    {
        float f1 = (float)i / segments;
        float f2 = (float)(i + 1) / segments;

        // Interpolate along the great circle
        float A = sin((1 - f1) * acos(sin(lat1) * sin(lat2) + cos(lat1) * cos(lat2) * cos(lon2 - lon1))) / sin(acos(sin(lat1) * sin(lat2) + cos(lat1) * cos(lat2) * cos(lon2 - lon1)));
        float B = sin(f1 * acos(sin(lat1) * sin(lat2) + cos(lat1) * cos(lat2) * cos(lon2 - lon1))) / sin(acos(sin(lat1) * sin(lat2) + cos(lat1) * cos(lat2) * cos(lon2 - lon1)));

        float x = A * cos(lat1) * cos(lon1) + B * cos(lat2) * cos(lon2);
        float y = A * cos(lat1) * sin(lon1) + B * cos(lat2) * sin(lon2);
        float z = A * sin(lat1) + B * sin(lat2);

        float lat = atan2(z, sqrt(x * x + y * y));
        float lon = atan2(y, x);

        int px1 = map(degrees(lon) + 180, 0, 360, 0, SCREEN_WIDTH);
        int py1 = map(90 - degrees(lat), 0, 180, 0, SCREEN_HEIGHT);

        // Next point
        A = sin((1 - f2) * acos(sin(lat1) * sin(lat2) + cos(lat1) * cos(lat2) * cos(lon2 - lon1))) / sin(acos(sin(lat1) * sin(lat2) + cos(lat1) * cos(lat2) * cos(lon2 - lon1)));
        B = sin(f2 * acos(sin(lat1) * sin(lat2) + cos(lat1) * cos(lat2) * cos(lon2 - lon1))) / sin(acos(sin(lat1) * sin(lat2) + cos(lat1) * cos(lat2) * cos(lon2 - lon1)));

        x = A * cos(lat1) * cos(lon1) + B * cos(lat2) * cos(lon2);
        y = A * cos(lat1) * sin(lon1) + B * cos(lat2) * sin(lon2);
        z = A * sin(lat1) + B * sin(lat2);

        lat = atan2(z, sqrt(x * x + y * y));
        lon = atan2(y, x);

        int px2 = map(degrees(lon) + 180, 0, 360, 0, SCREEN_WIDTH);
        int py2 = map(90 - degrees(lat), 0, 180, 0, SCREEN_HEIGHT);

        // Draw segment
        tft.drawLine(px1, py1, px2, py2, color);
    }
}

void drawGreatCircle(float lat1, float lon1, float lat2, float lon2, uint16_t color = TFT_YELLOW) {
    const int segments = 200;  // Number of segments for the curve

    // Convert degrees to radians
    lat1 = radians(lat1);
    lon1 = radians(lon1);
    lat2 = radians(lat2);
    lon2 = radians(lon2);

    // Calculate the angle between the two points on the unit sphere
    float deltaLon = lon2 - lon1;
    float centralAngle = acos(sin(lat1) * sin(lat2) + cos(lat1) * cos(lat2) * cos(deltaLon));

    // Interpolate along the great circle
    for (int i = 0; i < segments; i++) {
        float t = (float)i / (segments - 1);  // t goes from 0 to 1

        // Spherical interpolation (SLERP)
        float A = sin((1 - t) * centralAngle) / sin(centralAngle);
        float B = sin(t * centralAngle) / sin(centralAngle);

        // Calculate the coordinates of the intermediate point
        float x = A * cos(lat1) * cos(lon1) + B * cos(lat2) * cos(lon2);
        float y = A * cos(lat1) * sin(lon1) + B * cos(lat2) * sin(lon2);
        float z = A * sin(lat1) + B * sin(lat2);

        float lat = atan2(z, sqrt(x * x + y * y));
        float lon = atan2(y, x);

        // Convert back to degrees
        int px1 = map(degrees(lon) + 180, 0, 360, 0, SCREEN_WIDTH);
        int py1 = map(90 - degrees(lat), 0, 180, 0, SCREEN_HEIGHT);

        // Next point
        A = sin((1 - t) * centralAngle) / sin(centralAngle);
        B = sin(t * centralAngle) / sin(centralAngle);

        x = A * cos(lat1) * cos(lon1) + B * cos(lat2) * cos(lon2);
        y = A * cos(lat1) * sin(lon1) + B * cos(lat2) * sin(lon2);
        z = A * sin(lat1) + B * sin(lat2);

        lat = atan2(z, sqrt(x * x + y * y));
        lon = atan2(y, x);

        int px2 = map(degrees(lon) + 180, 0, 360, 0, SCREEN_WIDTH);
        int py2 = map(90 - degrees(lat), 0, 180, 0, SCREEN_HEIGHT);

        // Draw segment
        tft.drawLine(px1, py1, px2, py2, color);
    }
}

void drawGreatCircleEUR(float lat1, float lon1, float lat2, float lon2, uint16_t color = TFT_YELLOW) {
    const int segments = 200;  // Number of segments for the curve

    // Define the longitude and latitude boundaries for Europe
    float left_lon = -12.71;
    float right_lon = 52.8;
    float bottom_lat = 34.8;
    float top_lat = 71.7;

    // Convert degrees to radians for spherical interpolation
    lat1 = radians(lat1);
    lon1 = radians(lon1);
    lat2 = radians(lat2);
    lon2 = radians(lon2);

    // Calculate the central angle between the two points
    float deltaLon = lon2 - lon1;
    float centralAngle = acos(sin(lat1) * sin(lat2) + cos(lat1) * cos(lat2) * cos(deltaLon));

    // Interpolate along the great circle and draw dots
    for (int i = 0; i < segments; i++) {
        float t = (float)i / (segments - 1);  // t goes from 0 to 1

        // Spherical interpolation (SLERP)
        float A = sin((1 - t) * centralAngle) / sin(centralAngle);
        float B = sin(t * centralAngle) / sin(centralAngle);

        // Calculate the coordinates of the intermediate point
        float x = A * cos(lat1) * cos(lon1) + B * cos(lat2) * cos(lon2);
        float y = A * cos(lat1) * sin(lon1) + B * cos(lat2) * sin(lon2);
        float z = A * sin(lat1) + B * sin(lat2);

        float lat = atan2(z, sqrt(x * x + y * y));
        float lon = atan2(y, x);

        // Convert the intermediate point back to degrees
        lat = degrees(lat);
        lon = degrees(lon);

        // **Map** the intermediate lat/lon to Europe map coordinates
        int px = map((lon - left_lon), 0, (right_lon - left_lon), 0, SCREEN_WIDTH);  // x-coordinate
        int py = map((top_lat - lat), 0, (top_lat - bottom_lat), 0, SCREEN_HEIGHT);    // y-coordinate

        // **Draw a dot at this point**
        tft.fillCircle(px, py, 1, color);
    }
}

void drawGreatCircleXXXX(float lat1, float lon1, float lat2, float lon2, uint16_t color = TFT_YELLOW) {
    const int segments = 500;  // Number of segments for the curve

    // Define the longitude and latitude boundaries for Europe
    float left_lon = -12.71;
    float right_lon = 52.8;
    float bottom_lat = 34.8;
    float top_lat = 71.7;

    // Convert degrees to radians for spherical interpolation
    lat1 = radians(lat1);
    lon1 = radians(lon1);
    lat2 = radians(lat2);
    lon2 = radians(lon2);

    // Calculate the central angle between the two points
    float deltaLon = lon2 - lon1;
    float centralAngle = acos(sin(lat1) * sin(lat2) + cos(lat1) * cos(lat2) * cos(deltaLon));

    // Interpolate along the great circle and draw dots
    for (int i = 0; i < segments; i++) {
        float t = (float)i / (segments - 1);  // t goes from 0 to 1

        // Spherical interpolation (SLERP)
        float A = sin((1 - t) * centralAngle) / sin(centralAngle);
        float B = sin(t * centralAngle) / sin(centralAngle);

        // Calculate the coordinates of the intermediate point
        float x = A * cos(lat1) * cos(lon1) + B * cos(lat2) * cos(lon2);
        float y = A * cos(lat1) * sin(lon1) + B * cos(lat2) * sin(lon2);
        float z = A * sin(lat1) + B * sin(lat2);

        float lat = atan2(z, sqrt(x * x + y * y));
        float lon = atan2(y, x);

        // Convert the intermediate point back to degrees
        lat = degrees(lat);
        lon = degrees(lon);

        // **Map** the intermediate lat/lon to Europe map coordinates
        int px = map((lon - left_lon), 0, (right_lon - left_lon), 0, SCREEN_WIDTH);  // x-coordinate
        int py = map((top_lat - lat), 0, (top_lat - bottom_lat), 0, SCREEN_HEIGHT);    // y-coordinate

        // **Draw a dot at this point**
        tft.fillCircle(px, py, 1, color);
    }
}

void drawMapWithSpotsEUR() {
    displayPNGfromSPIFFS("eur.png", 0);  // Display the Europe map without delay
    
    // Define the longitude and latitude boundaries for Europe
    float left_lon = -12.71;
    float right_lon = 52.8;
    float bottom_lat = 34.8;
    float top_lat = 71.7;

    // Transform home location to the Europe map coordinates
    int home_x = map((HOME_LON - left_lon), 0, (right_lon - left_lon), 0, SCREEN_WIDTH);
    int home_y = map((top_lat - HOME_LAT), 0, (top_lat - bottom_lat), 0, SCREEN_HEIGHT);
    
    // Draw the home location on the map as a reference (optional)
    tft.fillCircle(home_x, home_y, 5, TFT_BLUE);  // Blue dot for home location

    // Loop through all spots and place them on the map
    for (int i = 0; i < totalEntries; ++i) {
        // Map the longitude to the x-coordinate for the Europe map
        int x = map((spots[i].lon - left_lon), 0, (right_lon - left_lon), 0, SCREEN_WIDTH);
        
        // Map the latitude to the y-coordinate for the Europe map
        int y = map((top_lat - spots[i].lat), 0, (top_lat - bottom_lat), 0, SCREEN_HEIGHT);




        // Optional: Boundary check to ensure the spot is within the map
       // if (x >= 0 && x < SCREEN_WIDTH && y >= 0 && y < SCREEN_HEIGHT) {
            tft.fillCircle(x, y, 2, TFT_RED);  // Small red dot for each spot
            
            // Draw a great circle between the home location and the reception spot
            tft.drawLine(home_x, home_y, x, y, TFT_YELLOW);  // Draw a yellow line

       // }
    }
}



// ####################################################################################################

void drawMapWithSpots()
{
    displayPNGfromSPIFFS("world.png", 0); // Display the map without delay

    for (int i = 0; i < totalEntries; ++i)
    {
        int x = map((spots[i].lon + 180), 0, 360, 0, SCREEN_WIDTH);
        int y = map((90 - spots[i].lat), 0, 180, 0, SCREEN_HEIGHT);
        Serial.print(x);
        Serial.print("    ");
        Serial.println(y);

        // Optional: Boundary check
        if (x >= 0 && x < SCREEN_WIDTH && y >= 0 && y < SCREEN_HEIGHT)
        {
            tft.fillCircle(x, y, 2, TFT_RED); // Small red dot for each spot
            drawGreatCircleO(HOME_LAT, HOME_LON, spots[i].lat, spots[i].lon);
        }
    }
}


void drawMapWithSpotsLigneDroite() {
    displayPNGfromSPIFFS("eur.png", 0);  // Display the Europe map without delay
    
    // Define the longitude and latitude boundaries for Europe
    float left_lon = -12.71;
    float right_lon = 52.8;
    float bottom_lat = 34.8;
    float top_lat = 71.7;

    // Transform home location (for example, your home in Switzerland)
    int home_x = map((HOME_LON - left_lon), 0, (right_lon - left_lon), 0, SCREEN_WIDTH);
    int home_y = map((top_lat - HOME_LAT), 0, (top_lat - bottom_lat), 0, SCREEN_HEIGHT);
    
    // Draw the home location on the map as a reference (optional)
    tft.fillCircle(home_x, home_y, 5, TFT_BLUE);  // Blue dot for home location

    // Loop through all spots and place them on the map
    for (int i = 0; i < totalEntries; ++i) {
        // Map the longitude to the x-coordinate
        int x = map((spots[i].lon - left_lon), 0, (right_lon - left_lon), 0, SCREEN_WIDTH);
        
        // Map the latitude to the y-coordinate
        int y = map((top_lat - spots[i].lat), 0, (top_lat - bottom_lat), 0, SCREEN_HEIGHT);

        // Optional: Boundary check to ensure the spot is within the map
       // if (x >= 0 && x < SCREEN_WIDTH && y >= 0 && y < SCREEN_HEIGHT) {
            tft.fillCircle(x, y, 2, TFT_RED);  // Small red dot for each spot
            
            // Draw a line from home location to this spot (simplified line for now)
            tft.drawLine(home_x, home_y, x, y, TFT_YELLOW);  // Draw a yellow line
        //}
    }
}





void setup()
{
    Serial.begin(115200);
    delay(3000);
    tft.begin();
    tft.setRotation(1);
    tft.fillScreen(TFT_BLACK); // Fill the screen with black color
    if (!SPIFFS.begin(true)) {
        Serial.println("SPIFFS Mount Failed");
        return;
    }
    printDirectory(SPIFFS, "/");


    //touchscreenSPI.begin(25, 39, 32, 33);
    //touchscreen.begin(touchscreenSPI);
    //touchscreen.setRotation(1);

    connectToWifi();


    // displayPNGfromSPIFFS("map.png", 0); //this is a Equirectangular projection map of the world 320x240

    fetchData();
    // drawTable();

    //drawMapWithSpots();        // Draw map + overlay reception spots
    drawMapWithSpotsEUR();


delay(4000);
drawMapWithSpots();  
delay(4000);


tft.fillScreen(TFT_BLACK); // Fill the screen with black color

drawTable();

    //drawMapWithSpotsLigneDroite();
}

void loop()
{
    
    
    /*
    if (touchscreen.touched()) {
      TS_Point p = touchscreen.getPoint();

      // Map raw touch coordinates to screen coordinates
      int x = map(p.x, 200, 3700, 0, SCREEN_WIDTH);
      int y = map(p.y, 240, 3800, 0, SCREEN_HEIGHT);
      int z = p.z;

      if (z > 0) {  // Touch detected
        Serial.printf("Touch detected at X: %d, Y: %d\n", x, y);

        static unsigned long lastTouch = 0;
        if (millis() - lastTouch > 300) {  // Debounce touch

          if (y < SCREEN_HEIGHT / 3) {
            // Upper third of the screen → Scroll Up
            if (scrollIndex > 0) {
              scrollIndex--;
              Serial.printf("Scrolling UP to index: %d\n", scrollIndex);
              drawTable();
            }
          } else if (y > 2 * SCREEN_HEIGHT / 3) {
            // Lower third of the screen → Scroll Down
            if (scrollIndex < totalEntries - (MAX_VISIBLE_ROWS - 1)) {
              scrollIndex++;
              Serial.printf("Scrolling DOWN to index: %d\n", scrollIndex);
              drawTable();
            }
          } else {
            // Middle third → No Action (or you can define one)
            Serial.println("Touched center area - No action.");
          }

          lastTouch = millis();
        }
      }
    }
      */
}
