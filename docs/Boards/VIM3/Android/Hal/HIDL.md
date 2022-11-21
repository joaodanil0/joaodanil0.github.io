# Blink Led HIDL

<figure markdown>
  ![imagems](images/hello-world-hidl.jpg){ width="600" }
  <figcaption>
  Image by <a href="https://pixabay.com/users/geralt-9301/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=5556539">Gerd Altmann</a> from <a href="https://pixabay.com//?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=5556539">Pixabay</a>
  </figcaption>
</figure>

> Vale ressaltar que para esse post é necessário já ter um produto buildado e para o ultimo tópico é preciso do display já em funcionamento.

## Introdução

Apesar de já estar depreciado, o HIDL ainda pode ser largamente encontrado no código fonte do AOSP. Caso algum código legado apareça, não custa nada ter esse tutorial pra relembrar com é feito um processo completo de implementação da HIDL.

## Informações úteis

- Versão do AOSP: `android-12.0.0_r4`
- Versão do kernel Android: `android-amlogic-bmeson-5.4`
- Distro para compilaçao: `Ubuntu 22.04.1 LTS`
- Versão do kernel da distro: `5.15.0-53-generic`

## Implementação

Tentarei seguir os mesmos tópicos do post: `Implementando a HIDL`.

### HAL

Na raiz da pasta do AOSP criei o seguinte caminho:

```{.sh}
mkdir -p device/casa/placamae/interfaces/userled
```

Dentro da pasta, criei o seguinte arquivo:

```{.java title=Android.bp}
hidl_package_root {
    name: "placamae.hal.userled",
    path: "device/casa/emulator/interfaces/userled",
}
```

Agora, criei a pasta: 

```{.sh}
mkdir 1.0
```

e dentro dela os seguintes arquivos:

```{.java title=Android.bp}
hidl_interface {
    name: "placamae.hal.userled@1.0",
    root: "placamae.hal.userled", //must be a prefix of placamae.hal.userled@1.0
    gen_java: true,
    product_specific: true,

    srcs: [
        "IUserLed.hal",
    ],
    interfaces: [
        "android.hidl.base@1.0",
    ],

}
```

```{.c title=IUserLed.hal}
package placamae.hal.userled@1.0;

interface IUserLed{
    setMode(string mode) generates (bool result);
};
```

O resultado foi esse:

```
device/casa/placamae/interfaces/userled/
├── 1.0
│   ├── Android.bp
│   └── IUserLed.hal
└── Android.bp
```

Agora é só adicionar ao produto, no meu caso:

```{title=meuproduto.mk}
...
PRODUCT_PACKAGES += \
  placamae.hal.userled@1.0
```

---

Agora vamos gerar os arquivos fonte para a HIDL. Para isso, ainda dentro da pasta `1.0`, digite:

```{.sh}
mm
```

Ele irá gerar as classes necessárias dentro da pasta :

```
out/soong/.intermediates/device/casa/placamae/interfaces/userled/1.0/
├── placamae.hal.userled@1.0
├── placamae.hal.userled@1.0-adapter
├── placamae.hal.userled@1.0-adapter_genc++
├── placamae.hal.userled@1.0-adapter-helper
├── placamae.hal.userled@1.0-adapter-helper_genc++
├── placamae.hal.userled@1.0-adapter-helper_genc++_headers
├── placamae.hal.userled@1.0_genc++
├── placamae.hal.userled@1.0_genc++_headers
├── placamae.hal.userled-V1.0-java
├── placamae.hal.userled-V1.0-java_gen_java
└── placamae.hal.userled-V1.0-java-shallow
```

### Serviço

Criei a pasta `device/casa/placamae/interfaces/userled/1.0/default/`, e dentro dela os arquivos:


```{.cpp title=UserLed.h}
// Mesmo caminho após:
// placamae.hal.userled@1.0_genc++_headers/gen/placamae/hal/userled/1.0/
// dentro da pasta out/soong/.intermediates/device/casa/placamae/interfaces/userled/1.0/
#include <placamae/hal/userled/1.0/IUserled.h>

namespace placamae {
namespace hal {
namespace userled {
namespace V1_0 {
namespace implementation {


using ::android::hardware::hidl_string;         // const hidl_string
using ::android::hardware::Return;              // Return<void>
using ::android::hardware::Void;                // return Void();
using ::placamae::hal::userled::V1_0::IUserled; // public IUserled

class UserLed : public IUserled {
    public:
      static inline const char RED_LED[] = "/sys/devices/platform/leds/leds/vim3:red/trigger";
    public:
        Return<bool> setMode(const hidl_string& mode) override;
        static int writeValue(const char *file, const char *value);
};

extern "C" IUserled* HIDL_FETCH_IUserled(const char* name);

} // namespace implementation
} // namespace V1_0 
} // namespace userled
} // namespace hal
} // namespace placamae
```

```{.cpp title=UserLed.cpp}
#include "UserLed.h"

#include <log/log.h>
#include <sys/stat.h> // struct stat info

namespace placamae {
namespace hal {
namespace userled {
namespace V1_0 {
namespace implementation {

int UserLed::writeValue(const char *file, const char *value) {

    int fd;
    int str_len = strlen(value) + 1;

    fd = open(file, O_WRONLY);

    if (fd < 0) {
      return -1;
    }

    if(!write(fd, value, str_len)){
      close(fd);
      return -1;
    }  

    close(fd);
    return 0;
}

Return<bool> UserLed::setMode(const hidl_string& mode) {
    ALOGI("UserLed -> setMode data=(%s)", mode.c_str());
    return this->write_value(RED_LED, mode.c_str()) == 0;
}

// Methods from ::android::hidl::base::V1_0::IBase follow.
IUserled* HIDL_FETCH_IUserled(const char* /* name */) {
    return new UserLed(); // Return new instance of this class
}

} // namespace implementation
} // namespace V1_0 
} // namespace userled
} // namespace hal
} // namespace placamae
```

```{.cpp title=service.cpp}
#define LOG_TAG "placamae.hal.userled@1.0-service"

#include <placamae/hal/userled/1.0/IUserled.h>

#include <log/log.h>
#include <hidl/HidlTransportSupport.h>

#include "UserLed.h"

using android::sp;
using android::status_t;
using android::OK;

// libhwbinder:
using android::hardware::configureRpcThreadpool;
using android::hardware::joinRpcThreadpool;

// Generated HIDL files
using placamae::hal::userled::V1_0::IUserled;
using placamae::hal::userled::V1_0::implementation::UserLed;

using namespace placamae;

int main(int /* argc */, char** /* argv */) {
    ALOGI("UserLed -> TESSSSSSTE");

    // Android Strong Pointer (don't GC until exit)
    sp<IUserled> service = new UserLed();
    if (service == nullptr) {
        ALOGE("Can not create an instance of UserLed HAL, exiting.");
        return 1;
    }

    // system/libhidl/transport/include/hidl/HidlTransportSupport.h
    // Configures the threadpool used for handling incoming RPC calls in this process:
    // @param maxThreads maximum number of threads in this process
    // @param callerWillJoin whether the caller will join the threadpool later.
    configureRpcThreadpool(1, true /*callerWillJoin*/);

    // registerAsService calls registerAsServiceInternal in
    // system/libhidl/transport/ServiceManagement.cpp
    // registerAsServiceInternal registers with hwservicemanager
    status_t status = service->registerAsService();
    if (status != OK) {
        ALOGE("Could not register service for UserLed HAL (%d), exiting.", status);
        return 1;
    }
    ALOGI("UserLed Service is ready");

    // system/libhidl/transport/include/hidl/HidlTransportSupport.h
    // Joins a threadpool that you configured earlier
    joinRpcThreadpool();

    // In normal operation, we don't expect the thread pool to exit
    ALOGE("UserLed Service is shutting down");
    return 1;
}
```

```{.java title=Android.bp}
cc_binary {
    name: "placamae.hal.userled@1.0-service",
    init_rc: ["placamae.hal.userled@1.0-service.rc"],
    srcs: ["service.cpp", "UserLed.cpp"],
    vintf_fragments: ["placamae.hal.userled@1.0-service.xml"],
    vendor: true,
    relative_install_path: "hw",

    shared_libs: [
        "libhidlbase",
        "liblog",
        "libutils",
        "libhardware",
        "placamae.hal.userled@1.0",
    ],
}
```

```{.rc title=placamae.hal.userled@1.0-service.rc}
service placamae.hal.userled-service /vendor/bin/hw/placamae.hal.userled@1.0-service
        interface placamae.hal.userled@1.0::IUserled default
        class hal
        user system
        group system

on boot
    chown system system /sys/devices/platform/leds/leds/vim3:red/trigger
    chmod 0660 /sys/devices/platform/leds/leds/vim3:red/trigger
```

```{.xml title=placamae.hal.userled@1.0-service.xml}
<manifest version="1.0" type="device">
    <hal format="hidl">
        <name>placamae.hal.userled</name>
        <transport>hwbinder</transport>
        <version>1.0</version>
        <interface>
            <name>IUserled</name>
            <instance>default</instance>
        </interface>
        <fqname>@1.0::IUserled/default</fqname>
    </hal>
</manifest>
```

O resultado foi esse:

```
device/casa/placamae/interfaces/userled/
├── 1.0
│   ├── Android.bp
│   ├── default
│   │   ├── service.cpp
│   │   ├── UserLed.cpp
│   │   └── UserLed.h
│   └── IUserLed.hal
└── Android.bp
```

Agora só resta adicionar o serviço ao produto:

```{.mk title=meuproduto.mk}
...
PRODUCT_PACKAGES += \
  placamae.hal.userled@1.0 \
  placamae.hal.userled@1.0-service
```

### Permissões

Para facilitar a implementação, foi definido a permissão ==permissiva==, mas ainda é necessário subir o serviço. Essa etapa é feita via sepolicy.

Para isso, criei a pasta:

```
mkdir -p device/casa/placamae/sepolicy
```

Dentro dela criei os arquivos:

```{title=file_contexts}
/vendor/bin/hw/placamae\.hal\.userled@1\.0-service      u:object_r:hal_userled_default_exec:s0
```

```{title=hal_userled_default.te}
type hal_userled_default, domain;
type hal_userled_default_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(hal_userled_default);
```

Por fim, adicionei a pasta sepolicy no:

```{title=BoardConfig.mk}
...
BOARD_SEPOLICY_DIRS += device/casa/placamae/sepolicy
```

### Testando

Para testar a HAL, criei a pasta:

```
mkdir -p device/casa/placamae/interfaces/userled/1.0/default/LedTest
```

e dentro dela os seguintes arquivos:

```{.cpp title=LedTest.cpp}
#include <placamae/hal/userled/1.0/IUserled.h>

#include <hidl/Status.h>
#include <hidl/LegacySupport.h>
#include <utils/misc.h>
#include <hidl/HidlSupport.h>
#include <iostream>
#include <cstdlib>
#include <string>

using ::android::sp;
using ::placamae::hal::userled::V1_0::IUserled;
using android::hardware::hidl_string;

int main(int argc, char *argv[]) {
    iif (argc != 2) {
        std::cout << "USAGE ./LedTest <none|heartbeat|default-on>\n";
        exit(0);
    }

    android::sp<IUserled> mHal = IUserled::getService();
    if (mHal == nullptr) {
        std::cout << "Failed to get cpu service\n";
        exit(-1);
    }

    bool result = mHal->setMode(argv[1]);
    std::cout << "setScalingGovernor: (" << result << ") " << argv[1] << std::endl;

    return 0;
}
```

```{.java title=Android.bp}
cc_binary {
    name: "LedTest",
    srcs: ["LedTest.cpp"],
    defaults: ["hidl_defaults"],
    vendor: true,
    relative_install_path: "hw",

    shared_libs: [
        "liblog",
        "libhardware",
        "libhidlbase",
        "libutils",
        "placamae.hal.userled@1.0",
    ],
}
```

Agora só resta adicionar o serviço ao produto:

```{.mk title=meuproduto.mk}
...
PRODUCT_PACKAGES += \
  placamae.hal.userled \
  placamae.hal.userled-service \
  LedTest
```

Na pasta raiz do AOSP:

```
m 
```

Depois de terminar a build, flashei os binários e esperei o Android subir para ter acesso ao ADB. Para testar o serviço, utilizei os comandos:

```
adb root
adb shell
cd /vendor/bin/hw
./LedTest heartbeat
```

### JNI

### Aplicação