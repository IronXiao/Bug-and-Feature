# Realtek 129x 红外驱动调试小结

## 使遥控器设备拥有vendor_id 和product_id

```c
diff --git a/buildscripts/OpenWrt-ImageBuilder-rtd1295-mnas_emmc.Linux-x86_64/linux-4.1.7/drivers/soc/realtek/rtd129x/irda/ir_input.c b/buildscripts/OpenWrt-ImageBuilder-rtd1295-mnas_emmc.Linux-x86_64/linux-4.1.7/drivers/soc/realtek/rtd129x/irda/ir_input.c
index 8e4119e83b..9c46255a31 100644
--- a/buildscripts/OpenWrt-ImageBuilder-rtd1295-mnas_emmc.Linux-x86_64/linux-4.1.7/drivers/soc/realtek/rtd129x/irda/ir_input.c
+++ b/buildscripts/OpenWrt-ImageBuilder-rtd1295-mnas_emmc.Linux-x86_64/linux-4.1.7/drivers/soc/realtek/rtd129x/irda/ir_input.c
@@ -61,6 +61,9 @@ int venus_ir_input_init(void)
        data->input_dev->evbit[0] = BIT_MASK(EV_KEY) |BIT(EV_REL);
        data->input_dev->name = "venus_IR_input";
        data->input_dev->phys = "venus/input0";
+       data->input_dev->id.vendor  = 0x0001;
+       data->input_dev->id.product = 0x0001;
+       data->input_dev->id.version = 0x0100;
        
        data->input_dev->setkeycode = _venus_ir_setkeycode;
        data->input_dev->getkeycode = _venus_ir_getkeycode;
```

## Vendor_xxxx_Product_xxxx.kl 不生效

主要原因在于复制Generic.kl 修改为Vendor_xxxx_Product_xxxx.kl 时，Vendor_xxxx_Product_xxxx.kl 包含部分无法Map的Key，需加如下补丁找出：

```cpp
diff --git a/buildscripts/android/frameworks/native/libs/input/KeyLayoutMap.cpp b/buildscripts/android/frameworks/native/libs/input/KeyLayoutMap.cpp
index 2b2f13e4c7..21fb21cab9 100644
--- a/buildscripts/android/frameworks/native/libs/input/KeyLayoutMap.cpp
+++ b/buildscripts/android/frameworks/native/libs/input/KeyLayoutMap.cpp
@@ -13,7 +13,7 @@
  * See the License for the specific language governing permissions and
  * limitations under the License.
  */
-
+#define LOG_NDEBUG 0
 #define LOG_TAG "KeyLayoutMap"
 
 #include <stdlib.h>
@@ -34,8 +34,10 @@
 #define DEBUG_PARSER_PERFORMANCE 0
 
 // Enables debug output for mapping.
-#define DEBUG_MAPPING 0
+#define DEBUG_MAPPING 1
 
+// CallStack debuging
+#include <utils/CallStack.h>
 
 namespace android {
 
@@ -85,6 +87,7 @@ status_t KeyLayoutMap::load(const String8& filename, sp<KeyLayoutMap>* outMap) {
 status_t KeyLayoutMap::mapKey(int32_t scanCode, int32_t usageCode,
         int32_t* outKeyCode, uint32_t* outFlags) const {
     const Key* key = getKey(scanCode, usageCode);
+    CallStack stack(LOG_TAG);
     if (!key) {
 #if DEBUG_MAPPING
         ALOGD("mapKey: scanCode=%d, usageCode=0x%08x ~ Failed.", scanCode, usageCode);
```

编译生成相关组件抓取开机日志，关注如下代码处日志：

```cpp
status_t KeyLayoutMap::mapKey(int32_t scanCode, int32_t usageCode,
        int32_t* outKeyCode, uint32_t* outFlags) const {
    const Key* key = getKey(scanCode, usageCode);
    if (!key) {
#if DEBUG_MAPPING
        ALOGD("mapKey: scanCode=%d, usageCode=0x%08x ~ Failed.", scanCode, usageCode);
#endif
        *outKeyCode = AKEYCODE_UNKNOWN;
        *outFlags = 0;
        return NAME_NOT_FOUND;
    }

    *outKeyCode = key->keyCode;
    *outFlags = key->flags;

#if DEBUG_MAPPING
    ALOGD("mapKey: scanCode=%d, usageCode=0x%08x ~ Result keyCode=%d, outFlags=0x%08x.",
            scanCode, usageCode, *outKeyCode, *outFlags);
#endif
    return NO_ERROR;
}

```

即：

```shell
logcat -s KeyLayoutMap|grep Failed
```

## kl 加载说明

input 设备选择kl 有两大函数：

```cpp
String8 getInputDeviceConfigurationFilePathByDeviceIdentifier(
        const InputDeviceIdentifier& deviceIdentifier,
        InputDeviceConfigurationFileType type) {
    if (deviceIdentifier.vendor !=0 && deviceIdentifier.product != 0) {
        if (deviceIdentifier.version != 0) {
            // Try vendor product version.
            String8 versionPath(getInputDeviceConfigurationFilePathByName(
                    String8::format("Vendor_%04x_Product_%04x_Version_%04x",
                            deviceIdentifier.vendor, deviceIdentifier.product,
                            deviceIdentifier.version),
                    type));
            if (!versionPath.isEmpty()) {
                return versionPath;
            }
        }

        // Try vendor product.
        String8 productPath(getInputDeviceConfigurationFilePathByName(
                String8::format("Vendor_%04x_Product_%04x",
                        deviceIdentifier.vendor, deviceIdentifier.product),
                type));
        if (!productPath.isEmpty()) {
            return productPath;
        }
    }

    // Try device name.
    return getInputDeviceConfigurationFilePathByName(deviceIdentifier.name, type);
}
```

```cpp
String8 getInputDeviceConfigurationFilePathByName(
        const String8& name, InputDeviceConfigurationFileType type) {
    // Search system repository.
    String8 path;

    // Treblized input device config files will be located /odm/usr or /vendor/usr.
    const char *rootsForPartition[] {"/odm", "/vendor", getenv("ANDROID_ROOT")};
    for (size_t i = 0; i < size(rootsForPartition); i++) {
        path.setTo(rootsForPartition[i]);
        path.append("/usr/");
        appendInputDeviceConfigurationFileRelativePath(path, name, type);
#if DEBUG_PROBE
        ALOGD("Probing for system provided input device configuration file: path='%s'",
              path.string());
#endif
        if (!access(path.string(), R_OK)) {
#if DEBUG_PROBE
            ALOGD("Found");
#endif
            return path;
        }
    }

    // Search user repository.
    // TODO Should only look here if not in safe mode.
    path.setTo(getenv("ANDROID_DATA"));
    path.append("/system/devices/");
    appendInputDeviceConfigurationFileRelativePath(path, name, type);
#if DEBUG_PROBE
    ALOGD("Probing for system user input device configuration file: path='%s'", path.string());
#endif
    if (!access(path.string(), R_OK)) {
#if DEBUG_PROBE
        ALOGD("Found");
#endif
        return path;
    }

    // Not found.
#if DEBUG_PROBE
    ALOGD("Probe failed to find input device configuration file: name='%s', type=%d",
            name.string(), type);
#endif
    return String8();
}
```

即以我第一个补丁为例优先级如下：

```cpp
Vendor_0001_Product_0001_Version_0100.kl //Vendor_%04x_Product_%04x_Version_%04x
Vendor_0001_Product_0001.kl //Vendor_%04x_Product_%04x
venus_IR_input.kl     //byName
Generic.kl //fall back
```



