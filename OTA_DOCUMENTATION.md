# ESP32 OTA (Over-The-Air) Update Documentation

## Overview

This project implements OTA firmware updates for ESP32 using HTTPS downloads from GitHub Releases. The user can trigger an update by pressing the BOOT button (GPIO 0), which downloads new firmware from a specified URL and updates the device.

---

## Main Components

### 1. **Global Variables**

```c
const int software_version = 4;
SemaphoreHandle_t ota_semaphore;
```

- **`software_version`**: Track current firmware version (increment for each new release)
- **`ota_semaphore`**: Binary semaphore to signal when OTA should start (triggered by button press)

---

### 2. **Application Entry Point: `app_main()`**

```c
void app_main(void)
{
  printf("HAY!!! This is a new feature\n");
  ESP_LOGI("SOFTWARE VERSION", "we are running %d", software_version);
  
  // Configure GPIO 0 (BOOT button) as input with interrupt
  gpio_config_t gpioConfig = {
      .pin_bit_mask = 1ULL << GPIO_NUM_0,
      .mode = GPIO_MODE_DEF_INPUT,
      .pull_up_en = GPIO_PULLUP_ENABLE,
      .pull_down_en = GPIO_PULLUP_DISABLE,
      .intr_type = GPIO_INTR_NEGEDGE
  };
  gpio_config(&gpioConfig);
  gpio_install_isr_service(0);
  gpio_isr_handler_add(GPIO_NUM_0, on_button_pushed, NULL);

  // Create semaphore and OTA task
  ota_semaphore = xSemaphoreCreateBinary();
  xTaskCreate(run_ota, "run_ota", 1024 * 8, NULL, 2, NULL);
}
```

**What happens here:**
1. Prints current version to console
2. Configures GPIO 0 (BOOT button) to trigger interrupt on falling edge (button press)
3. Creates a binary semaphore for synchronization
4. Spawns the `run_ota` task that waits for button press

---

### 3. **Button Interrupt Handler: `on_button_pushed()`**

```c
void on_button_pushed(void *params)
{
  xSemaphoreGiveFromISR(ota_semaphore, pdFALSE);
}
```

**What happens:**
- Called when BOOT button is pressed
- Releases the semaphore, signaling the OTA task to proceed

---

### 4. **OTA Update Task: `run_ota()`**

```c
void run_ota(void *params)
{
  while (true)
  {
    // Wait for button press
    xSemaphoreTake(ota_semaphore, portMAX_DELAY);
    ESP_LOGI(TAG, "Invoking OTA");

    // Initialize NVS (required for WiFi)
    ESP_ERROR_CHECK(nvs_flash_init());

    // Initialize network stack
    esp_netif_init();
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    
    // Connect to WiFi (credentials configured via menuconfig)
    ESP_ERROR_CHECK(example_connect());

    // Configure HTTPS client
    esp_http_client_config_t clientConfig = {
        .url = "https://github.com/TapioJarnfors/esp32-ota-demo/releases/download/v1.0.0/my_ota.bin",
        .event_handler = client_event_handler,
        .crt_bundle_attach = esp_crt_bundle_attach,  // Use built-in CA certificates
        .buffer_size = 4096,         // RX buffer size
        .buffer_size_tx = 2048,      // TX buffer size
        .timeout_ms = 30000          // 30 second timeout
    };

    // Configure OTA
    esp_https_ota_config_t ota_config = {
        .http_config = &clientConfig
    };

    // Perform OTA update
    if (esp_https_ota(&ota_config) == ESP_OK)
    {
      ESP_LOGI(TAG, "OTA flash successful for version %d.", software_version);
      printf("restarting in 5 seconds\n");
      vTaskDelay(pdMS_TO_TICKS(5000));
      esp_restart();  // Reboot to run new firmware
    }
    ESP_LOGE(TAG, "Failed to update firmware");
  }
}
```

**Step-by-step process:**

1. **Wait** for button press (blocks on semaphore)
2. **Initialize** NVS flash and network stack
3. **Connect** to WiFi using stored credentials
4. **Configure** HTTPS client with:
   - GitHub release URL
   - CA certificate bundle for SSL/TLS
   - Buffer sizes for download
5. **Download & Flash** new firmware using `esp_https_ota()`
6. **Verify** chip ID and firmware validity
7. **Reboot** if successful, or log error and wait for next trigger

---

### 5. **HTTP Event Handler: `client_event_handler()`**

```c
esp_err_t client_event_handler(esp_http_client_event_t *evt)
{
  return ESP_OK;
}
```

**Purpose:**
- Minimal event handler for HTTP client
- Can be extended to handle events like progress tracking, errors, etc.

---

## How OTA Update Works

### Partition Layout

ESP32 uses a dual-partition scheme for OTA:

```
┌─────────────────┐
│   Bootloader    │  0x1000
├─────────────────┤
│ Partition Table │  0x8000
├─────────────────┤
│   NVS (WiFi)    │  
├─────────────────┤
│   OTA Data      │  (tracks active partition)
├─────────────────┤
│   ota_0         │  ← Running firmware
├─────────────────┤
│   ota_1         │  ← New firmware downloaded here
└─────────────────┘
```

### Update Process

1. **Current state**: Device runs from `ota_0`
2. **Download**: New firmware downloads to `ota_1`
3. **Verify**: Check chip ID, revision, and integrity
4. **Mark active**: Update OTA data partition to boot from `ota_1`
5. **Reboot**: Device restarts and runs new firmware from `ota_1`
6. **Next update**: Downloads to `ota_0` (alternates between partitions)

### Security Features

- **HTTPS with certificate verification**: Uses built-in CA bundle
- **Chip ID verification**: Ensures firmware is built for ESP32
- **Chip revision check**: Verifies minimum revision requirements
- **Image integrity**: Validates firmware header and checksums

---

## Configuration Requirements

### WiFi Credentials (menuconfig)

```bash
idf.py menuconfig
```

Navigate to:
- **Example Connection Configuration**
  - WiFi SSID: Your network name
  - WiFi Password: Your network password

### Partition Table (sdkconfig)

Must use **Two OTA** partition scheme:
```
CONFIG_PARTITION_TABLE_TWO_OTA=y
CONFIG_PARTITION_TABLE_FILENAME="partitions_two_ota.csv"
```

### Certificate Bundle

Enabled by default in ESP-IDF v5.5+:
```
CONFIG_MBEDTLS_CERTIFICATE_BUNDLE=y
CONFIG_MBEDTLS_CERTIFICATE_BUNDLE_DEFAULT_FULL=y
```

---

## Building and Deploying New Firmware

### Step 1: Build Current Firmware

```bash
idf.py build
idf.py flash monitor
```

### Step 2: Create New Version

Modify code (e.g., increment version):
```c
const int software_version = 5;
printf("This is version 5!\n");
```

### Step 3: Build New Firmware

```bash
idf.py build
```

### Step 4: Upload to GitHub

1. Go to https://github.com/TapioJarnfors/esp32-ota-demo/releases
2. Click "Draft a new release"
3. Tag: `v1.0.1` (or next version)
4. Upload: `build/my_ota.bin`
5. Publish release

### Step 5: Update URL (if needed)

If using a new release tag, update the URL in main.c:
```c
.url = "https://github.com/TapioJarnfors/esp32-ota-demo/releases/download/v1.0.1/my_ota.bin"
```

### Step 6: Trigger OTA Update

1. Device boots with old firmware
2. Press **BOOT button** (GPIO 0)
3. Watch serial monitor for progress
4. Device downloads, verifies, flashes, and reboots
5. New firmware is running!

---

## Security Enhancements

### Current Security Features (Already Implemented)

✅ **HTTPS with Certificate Verification**: Uses TLS/SSL with CA certificate bundle  
✅ **Chip ID Verification**: Ensures firmware matches ESP32 hardware  
✅ **Chip Revision Check**: Validates minimum hardware revision  
✅ **Image Header Validation**: Checks firmware header integrity  

### Recommended Security Improvements

#### 1. **Firmware Signing (Highly Recommended)**

Add digital signature verification to prevent malicious firmware:

**Enable in menuconfig:**
```bash
idf.py menuconfig
```
Navigate to:
- **Security features** → **Enable hardware Secure Boot v2**
- **Security features** → **Enable flash encryption**

**Add signature verification to code:**
```c
#include "esp_secure_boot.h"

// Before OTA update
if (!esp_secure_boot_enabled()) {
    ESP_LOGW(TAG, "Secure boot not enabled!");
}

// After download, verify signature
esp_err_t err = esp_secure_boot_verify_signature(
    partition->address, 
    partition->size
);
if (err != ESP_OK) {
    ESP_LOGE(TAG, "Signature verification failed!");
    return;
}
```

#### 2. **Version Rollback Protection**

Prevent downgrade attacks by validating version numbers:

```c
#include "esp_app_desc.h"

// Get running firmware version
const esp_app_desc_t *running_app = esp_app_get_description();
int current_version = running_app->version; // Or parse from version string

// Get new firmware version
esp_app_desc_t new_app_info;
esp_https_ota_get_img_desc(https_ota_handle, &new_app_info);
int new_version = new_app_info.version; // Or custom version field

// Compare versions
if (new_version <= current_version) {
    ESP_LOGE(TAG, "Downgrade not allowed! Current: %d, New: %d", 
             current_version, new_version);
    esp_https_ota_abort(https_ota_handle);
    return;
}

ESP_LOGI(TAG, "Version upgrade: %d -> %d", current_version, new_version);
```

#### 3. **Firmware Validation Before Reboot**

Add custom checks to verify new firmware:

```c
// After OTA download completes
esp_app_desc_t new_app_info;
if (esp_https_ota_get_img_desc(https_ota_handle, &new_app_info) == ESP_OK) {
    // Verify project name
    if (strcmp(new_app_info.project_name, "my_ota") != 0) {
        ESP_LOGE(TAG, "Wrong project! Expected 'my_ota', got '%s'", 
                 new_app_info.project_name);
        esp_https_ota_abort(https_ota_handle);
        return;
    }
    
    // Verify IDF version compatibility
    ESP_LOGI(TAG, "New firmware IDF version: %s", new_app_info.idf_ver);
    
    // Verify compilation time (ensure it's newer)
    ESP_LOGI(TAG, "New firmware built: %s %s", 
             new_app_info.date, new_app_info.time);
}
```

#### 4. **OTA Diagnostic and Rollback**

Automatically rollback if new firmware fails:

```c
#include "esp_ota_ops.h"

void app_main(void)
{
    // Check if this is first boot after OTA
    const esp_partition_t *running = esp_ota_get_running_partition();
    esp_ota_img_states_t ota_state;
    
    if (esp_ota_get_state_partition(running, &ota_state) == ESP_OK) {
        if (ota_state == ESP_OTA_IMG_PENDING_VERIFY) {
            // New firmware is running - perform health checks
            bool is_healthy = perform_health_checks();
            
            if (is_healthy) {
                ESP_LOGI(TAG, "New firmware validated, marking as valid");
                esp_ota_mark_app_valid_cancel_rollback();
            } else {
                ESP_LOGE(TAG, "New firmware failed health check, rolling back!");
                esp_ota_mark_app_invalid_rollback_and_reboot();
            }
        }
    }
    
    // Rest of app_main...
}

bool perform_health_checks(void)
{
    // Add your validation logic:
    // - Test WiFi connectivity
    // - Check sensor readings
    // - Verify critical functionality
    // - etc.
    return true; // Return false to trigger rollback
}
```

**Update OTA code to enable rollback:**
```c
esp_https_ota_config_t ota_config = {
    .http_config = &clientConfig,
    .partial_http_download = false,
    .bulk_flash_erase = false
};

if (esp_https_ota(&ota_config) == ESP_OK) {
    ESP_LOGI(TAG, "OTA successful, will verify on next boot");
    esp_restart();
}
```

#### 5. **Use Specific Certificate Instead of Bundle**

For maximum security, use only your server's certificate:

**Create cert/github.pem** with GitHub's certificate:
```bash
openssl s_client -showcerts -connect github.com:443 </dev/null 2>/dev/null | \
    openssl x509 -outform PEM > cert/github.pem
```

**Update CMakeLists.txt:**
```cmake
idf_component_register(SRCS "main.c"
                    INCLUDE_DIRS "."
                    EMBED_TXTFILES "../cert/github.pem")
```

**Update main.c:**
```c
extern const uint8_t github_cert_pem_start[] asm("_binary_github_pem_start");
extern const uint8_t github_cert_pem_end[]   asm("_binary_github_pem_end");

esp_http_client_config_t clientConfig = {
    .url = "https://github.com/...",
    .event_handler = client_event_handler,
    .cert_pem = (char *)github_cert_pem_start,  // Use specific cert
    .buffer_size = 4096,
    .buffer_size_tx = 2048,
    .timeout_ms = 30000
};
```

#### 6. **Add Authentication Token**

Protect your firmware downloads with authentication:

**GitHub Personal Access Token:**
```c
esp_http_client_config_t clientConfig = {
    .url = "https://github.com/...",
    .event_handler = client_event_handler,
    .crt_bundle_attach = esp_crt_bundle_attach,
    .auth_type = HTTP_AUTH_TYPE_BEARER,
    .username = "token",  // or your GitHub username
    .password = "ghp_YOUR_GITHUB_TOKEN_HERE",
    .buffer_size = 4096,
    .buffer_size_tx = 2048,
    .timeout_ms = 30000
};
```

**Or use custom header:**
```c
esp_http_client_set_header(https_ota_handle->http_client, 
                          "Authorization", 
                          "Bearer YOUR_TOKEN");
```

#### 7. **Enable Flash Encryption**

Protect firmware in flash memory:

**menuconfig:**
- **Security features** → **Enable flash encryption on boot**
- Choose **Release mode** for production

**Note**: Once enabled, flash can only be updated with encrypted images via OTA.

#### 8. **Implement Rate Limiting**

Prevent DoS attacks on your OTA system:

```c
#include "esp_timer.h"

static int64_t last_ota_attempt = 0;
const int64_t MIN_OTA_INTERVAL = 300 * 1000000; // 5 minutes in microseconds

void on_button_pushed(void *params)
{
    int64_t now = esp_timer_get_time();
    
    if ((now - last_ota_attempt) < MIN_OTA_INTERVAL) {
        ESP_LOGW(TAG, "OTA rate limited, try again later");
        return;
    }
    
    last_ota_attempt = now;
    xSemaphoreGiveFromISR(ota_semaphore, pdFALSE);
}
```

#### 9. **Check Firmware Hash/Checksum**

Add integrity verification:

```c
#include "mbedtls/sha256.h"

// After downloading, calculate SHA256
void verify_firmware_hash(const uint8_t *firmware, size_t size) {
    uint8_t calculated_hash[32];
    mbedtls_sha256(firmware, size, calculated_hash, 0);
    
    // Compare with expected hash (store in code or download separately)
    const uint8_t expected_hash[32] = { /* your hash */ };
    
    if (memcmp(calculated_hash, expected_hash, 32) != 0) {
        ESP_LOGE(TAG, "Firmware hash mismatch!");
        return;
    }
    ESP_LOGI(TAG, "Firmware hash verified");
}
```

### Protecting WiFi Credentials and Sensitive Data

**Important**: WiFi passwords should NEVER be hardcoded in firmware binaries!

#### Current Setup (Using menuconfig)

When you configure WiFi via `idf.py menuconfig`, credentials are:
- ❌ Compiled into the binary (visible in .bin file!)
- ❌ Same password for all devices
- ❌ Exposed if firmware is downloaded

#### Solution 1: NVS Storage with Flash Encryption (Recommended)

Store WiFi credentials in encrypted NVS instead of code:

**Step 1: Enable NVS encryption**

```bash
idf.py menuconfig
```
- **Component config** → **NVS** → **Enable NVS encryption**
- **Security features** → **Enable flash encryption**

**Step 2: Provision WiFi at manufacturing/first boot**

```c
#include "nvs_flash.h"
#include "nvs.h"

// One-time provisioning (e.g., via Bluetooth or serial)
void provision_wifi_credentials(const char* ssid, const char* password)
{
    nvs_handle_t nvs_handle;
    esp_err_t err = nvs_open("wifi", NVS_READWRITE, &nvs_handle);
    if (err == ESP_OK) {
        nvs_set_str(nvs_handle, "ssid", ssid);
        nvs_set_str(nvs_handle, "password", password);
        nvs_commit(nvs_handle);
        nvs_close(nvs_handle);
        ESP_LOGI(TAG, "WiFi credentials stored securely");
    }
}

// Read credentials from NVS
esp_err_t get_wifi_credentials(char* ssid, char* password)
{
    nvs_handle_t nvs_handle;
    esp_err_t err = nvs_open("wifi", NVS_READONLY, &nvs_handle);
    if (err != ESP_OK) return err;
    
    size_t ssid_len = 32;
    size_t pass_len = 64;
    
    err = nvs_get_str(nvs_handle, "ssid", ssid, &ssid_len);
    if (err != ESP_OK) goto cleanup;
    
    err = nvs_get_str(nvs_handle, "password", password, &pass_len);
    
cleanup:
    nvs_close(nvs_handle);
    return err;
}
```

**Step 3: Replace example_connect() with custom WiFi connection**

```c
#include "esp_wifi.h"

void connect_wifi_from_nvs(void)
{
    char ssid[32];
    char password[64];
    
    // Read from encrypted NVS
    if (get_wifi_credentials(ssid, password) != ESP_OK) {
        ESP_LOGE(TAG, "No WiFi credentials found!");
        // Trigger provisioning mode
        start_provisioning();
        return;
    }
    
    // Initialize WiFi
    esp_netif_create_default_wifi_sta();
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
    
    // Configure WiFi
    wifi_config_t wifi_config = {
        .sta = {
            .threshold.authmode = WIFI_AUTH_WPA2_PSK,
        },
    };
    
    strcpy((char*)wifi_config.sta.ssid, ssid);
    strcpy((char*)wifi_config.sta.password, password);
    
    // Clear password from memory immediately
    memset(password, 0, sizeof(password));
    
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());
    
    ESP_LOGI(TAG, "Connecting to WiFi...");
}
```

#### Solution 2: WiFi Provisioning via BLE/SoftAP

Use ESP-IDF's unified provisioning for secure credential setup:

```c
#include "wifi_provisioning/manager.h"
#include "wifi_provisioning/scheme_ble.h"

void start_provisioning(void)
{
    // Initialize provisioning manager
    wifi_prov_mgr_config_t config = {
        .scheme = wifi_prov_scheme_ble,
        .scheme_event_handler = WIFI_PROV_SCHEME_BLE_EVENT_HANDLER_FREE_BTDM
    };
    
    ESP_ERROR_CHECK(wifi_prov_mgr_init(config));
    
    bool provisioned = false;
    ESP_ERROR_CHECK(wifi_prov_mgr_is_provisioned(&provisioned));
    
    if (!provisioned) {
        ESP_LOGI(TAG, "Starting WiFi provisioning via BLE");
        
        // Start provisioning with device name
        ESP_ERROR_CHECK(wifi_prov_mgr_start_provisioning(
            WIFI_PROV_SECURITY_1, 
            "PROV_12345",  // Device name
            NULL           // Service key (NULL for no popup)
        ));
        
        // User configures WiFi via smartphone app
        // (ESP BLE Prov app for Android/iOS)
    } else {
        ESP_LOGI(TAG, "Already provisioned, connecting...");
        wifi_prov_mgr_deinit();
        // Connect using stored credentials
    }
}
```

**User provision workflow:**
1. Device boots → No WiFi credentials
2. Starts BLE provisioning mode
3. User connects via "ESP BLE Prov" app
4. Enters WiFi SSID/password in app
5. Credentials stored encrypted in NVS
6. Future boots use stored credentials

#### Solution 3: Per-Device Unique Credentials

For production deployments, pre-provision each device:

**Manufacturing process:**
```bash
# Generate unique credentials per device
esptool.py --port COM8 write_flash 0x9000 nvs_partition_device_001.bin

# Each device gets unique NVS partition with credentials
```

#### Solution 4: Remove WiFi Config from Binary

**Remove from menuconfig:**
```bash
idf.py menuconfig
# Go to Example Connection Configuration
# Clear SSID and Password fields
```

**Verify binary doesn't contain credentials:**
```bash
strings build/my_ota.bin | grep "your_password"
# Should return nothing!
```

#### Best Practices for Sensitive Data

**DO:**
- ✅ Store credentials in **encrypted NVS**
- ✅ Use **provisioning** for initial setup
- ✅ Enable **flash encryption** for protection at rest
- ✅ Clear sensitive data from memory after use
- ✅ Use **secure boot** to prevent firmware tampering
- ✅ Unique credentials per device

**DON'T:**
- ❌ Hardcode passwords in source code
- ❌ Configure passwords via menuconfig for production
- ❌ Share same WiFi password across all devices
- ❌ Store passwords in plain text NVS
- ❌ Leave debug prints with passwords

#### Quick Fix for Current Code

**Remove menuconfig WiFi and add runtime provisioning:**

Update `run_ota()`:
```c
void run_ota(void *params)
{
  while (true)
  {
    xSemaphoreTake(ota_semaphore, portMAX_DELAY);
    ESP_LOGI(TAG, "Invoking OTA");

    ESP_ERROR_CHECK(nvs_flash_init());
    esp_netif_init();
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    
    // Replace example_connect() with:
    connect_wifi_from_nvs();  // Load credentials from NVS
    
    // Rest of OTA code...
  }
}
```

#### Recommended Implementation Order

1. **Immediate**: Remove WiFi credentials from menuconfig
2. **Next**: Implement NVS credential storage
3. **Then**: Add BLE/SoftAP provisioning
4. **Finally**: Enable flash encryption + secure boot

---

### Security Checklist

**Before Production Deployment:**

**Credential & Data Protection:**
- [ ] **Remove WiFi credentials** from menuconfig/source code
- [ ] Store WiFi credentials in **encrypted NVS**
- [ ] Implement **WiFi provisioning** (BLE/SoftAP)
- [ ] Enable **NVS encryption** for sensitive data
- [ ] Clear sensitive data from memory after use
- [ ] No hardcoded API keys or tokens in code

**Firmware Security:**
- [ ] Enable **Secure Boot** (prevents unauthorized firmware)
- [ ] Enable **Flash Encryption** (protects firmware in flash)
- [ ] Implement **version rollback protection**
- [ ] Add **firmware signing** and verification
- [ ] Add **OTA diagnostic** and automatic rollback
- [ ] Implement **health checks** after OTA
- [ ] Test **rollback mechanism** thoroughly

**Communication Security:**
- [ ] Use **specific certificate** instead of bundle
- [ ] Add **authentication** to firmware downloads
- [ ] Verify **HTTPS/TLS** is enforced (no HTTP fallback)
- [ ] Pin certificates for critical connections

**Operational Security:**
- [ ] Enable **rate limiting** on OTA triggers
- [ ] Implement **logging** of OTA attempts and results
- [ ] Use unique credentials per device
- [ ] Verify firmware **before** distributing to devices
- [ ] Have **factory reset** mechanism for recovery

**Testing:**
- [ ] Test OTA with invalid firmware (should reject)
- [ ] Test version downgrade (should block)
- [ ] Test rollback on firmware failure
- [ ] Test with expired/invalid certificates
- [ ] Verify credentials not visible in binary with `strings build/my_ota.bin`

**Monitoring:**
- Log all OTA attempts (success/failure)
- Track firmware versions in use
- Monitor rollback events
- Alert on suspicious activity
- Track failed OTA attempts per device

---

## Troubleshooting

### Error: "Mismatch chip id, expected 0, found 27757"
**Cause**: Receiving HTML instead of binary firmware  
**Fix**: Verify GitHub release URL is correct and file is publicly accessible

### Error: "No option for server verification is enabled"
**Cause**: Missing certificate configuration  
**Fix**: Add `.crt_bundle_attach = esp_crt_bundle_attach` to HTTP config

### Error: "Given staging partition or another suitable Passive OTA partition could not be found"
**Cause**: Partition table not configured for OTA  
**Fix**: Change to TWO_OTA partition table in sdkconfig

### Error: "Connection timed out before data was ready"
**Cause**: Slow connection or small timeout  
**Fix**: Increase `.timeout_ms` value (e.g., 30000 for 30 seconds)

### Error: "Buffer length is small to fit all the headers"
**Cause**: HTTP headers too large for buffer  
**Fix**: Increase `.buffer_size` (e.g., 4096)

---

## Key Files

- **main.c**: OTA application code
- **CMakeLists.txt**: Component configuration
- **sdkconfig**: Project configuration (partition table, WiFi, certificates)
- **build/my_ota.bin**: Compiled firmware binary for OTA updates

---

## References

- [ESP-IDF OTA Documentation](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/ota.html)
- [ESP HTTPS OTA API](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/esp_https_ota.html)
- [ESP-IDF Certificate Bundle](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/protocols/esp_crt_bundle.html)
