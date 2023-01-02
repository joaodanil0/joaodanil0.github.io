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
    path: "device/casa/placamae/interfaces/userled",
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

Volte para a pasta raiz do AOSP

```
croot
```

Criei a pasta `device/casa/placamae/interfaces/userled/1.0/default/`, e dentro dela os arquivos:


```{.cpp title=UserLed.h}
// Mesmo caminho após:
// placamae.hal.userled@1.0_genc++_headers/gen/placamae/hal/userled/1.0/
// dentro da pasta out/soong/.intermediates/device/casa/placamae/interfaces/userled/1.0/
#include <placamae/hal/userled/1.0/IUserLed.h>

namespace placamae {
namespace hal {
namespace userled {
namespace V1_0 {
namespace implementation {


using ::android::hardware::hidl_string;         // const hidl_string
using ::android::hardware::Return;              // Return<void>
using ::android::hardware::Void;                // return Void();
using ::placamae::hal::userled::V1_0::IUserLed; // public IUserled

class UserLed : public IUserLed {
    public:
      static inline const char RED_LED[] = "/sys/devices/platform/leds/leds/vim3:red/trigger";
    public:
        Return<bool> setMode(const hidl_string& mode) override;
        static int writeValue(const char *file, const char *value);
};

extern "C" IUserLed* HIDL_FETCH_IUserled(const char* name);

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
    return this->writeValue(RED_LED, mode.c_str()) == 0;
}

// Methods from ::android::hidl::base::V1_0::IBase follow.
IUserLed* HIDL_FETCH_IUserled(const char* /* name */) {
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

#include <placamae/hal/userled/1.0/IUserLed.h>

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
using placamae::hal::userled::V1_0::IUserLed;
using placamae::hal::userled::V1_0::implementation::UserLed;

using namespace placamae;

int main(int /* argc */, char** /* argv */) {
    ALOGI("UserLed -> TESSSSSSTE");

    // Android Strong Pointer (don't GC until exit)
    sp<IUserLed> service = new UserLed();
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
        interface placamae.hal.userled@1.0::IUserLed default
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
        <fqname>@1.0::IUserLed/default</fqname>
    </hal>
</manifest>
```

O resultado foi esse:

```
device/casa/placamae/interfaces/userled/
├── 1.0
│   ├── Android.bp
│   ├── default
│   │   ├── Android.bp
│   │   ├── placamae.hal.userled@1.0-service.rc
│   │   ├── placamae.hal.userled@1.0-service.xml
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
  placamae.hal.userled@1.0 \
  placamae.hal.userled@1.0-service
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

### Java Native Interface

A Java Native Interface (JNI) é responsável por fazer a interface entre o código em JAVA e o código em C/C++. Essa abordagem é largamente usada no mundo JAVA, ou seja, não é uma exclusividade do mundo Android.

Criei a pasta:

```
mkdir -p device/casa/placamae/libs/jni
```

dentro dela os arquivos:

```{.cpp title=UserLedJNI.h}
#ifndef USERLEDJNI_H
#define USERLEDJNI_H

#define LOG_TAG "USERLEDJNI.cpp"
#define LOG_NDEBUG 1

#include "jni.h"

#include <android-base/chrono_utils.h>
#include <utils/Log.h>


#include <placamae/hal/userled/1.0/IUserLed.h>

using ::placamae::hal::userled::V1_0::IUserLed; 

using android::hardware::Return;
using android::hardware::Void;
using android::sp;

class UserLedJNI {

    public:
        inline static sp<IUserLed> UserLedHal = nullptr;
        inline static bool UserLedHalExists = true;
        inline static const char *classPathName = "com/interfaces/UserLedJNI"";

    public:
        static bool getUserLedHal();
        
        static void processReturn(const Return<void> &ret, const char* functionName);        
        static int registerNativeMethods(JNIEnv* env, const char* className, JNINativeMethod* gMethods, int numMethods);
        static int registerJniUserLed(JNIEnv* env);

        static void nativeInit(JNIEnv* env, jobject /* obj */);
        static jboolean setMode(JNIEnv*  env, jclass /* clazz */, jstring mode);
};

#endif
```

```{.cpp title=UserLedJNI.cpp}
#include "UserLedJNI.h"

JNINativeMethod methods[] = {
    { "nativeInit", "()V", (void*) UserLedJNI::nativeInit },
    { "nativeSetMode", "(Ljava/lang/String;)Z", (void*) UserLedJNI::setMode },
};

std::mutex UserLedHalMutex;

bool UserLedJNI::getUserLedHal() {
    
    if (UserLedHalExists && UserLedHal == nullptr) {

        UserLedHal = IUserLed::getService(); // Proxy to the service

        if (UserLedHal != nullptr) {
            ALOGI("Loaded foo HAL service");
        } 
        else {
            ALOGI("Couldn't load foo HAL service");
            UserLedHalExists = false;
        }
    }
    return UserLedHal != nullptr;
}

void UserLedJNI::nativeInit(JNIEnv* env, jobject /* obj */) {
    UserLedHalMutex.lock();
    getUserLedHal();
    UserLedHalMutex.unlock();
}

jboolean UserLedJNI::setMode(JNIEnv*  env, jclass /* clazz */, jstring mode) {

    ALOGD("nativeSetMode");

    std::lock_guard<std::mutex> lock(UserLedHalMutex);

    bool ret = false;
    if (UserLedJNI::getUserLedHal()) {
        android::base::Timer t;

        const char* cMode = env->GetStringUTFChars(mode, NULL);
        ret = UserLedJNI::UserLedHal->setMode(cMode);
        env->ReleaseStringUTFChars(mode, cMode);

        if (t.duration() > 20ms) {
            ALOGW("Excessive delay in setScalingGovernor");
        }
    }

    return ret;
}

int UserLedJNI::registerNativeMethods(JNIEnv* env, const char* className, JNINativeMethod* gMethods, int numMethods) {
    
    jclass clazz;

    ALOGE("registerNativeMethods '%s'", className);

    clazz = env->FindClass(className);

    if (clazz == NULL) {
        ALOGE("Native registration unable to find class '%s'", className);
        return JNI_FALSE;
    }

    if (env->RegisterNatives(clazz, gMethods, numMethods) < 0) {
        ALOGE("RegisterNatives failed for '%s'", className);
        return JNI_FALSE;
    }

    return JNI_TRUE;
}

int UserLedJNI::registerJniUserLed(JNIEnv* env) {
    if (!registerNativeMethods(env, UserLedJNI::classPathName, methods, sizeof(methods) / sizeof(methods[0])))
        return JNI_FALSE;
    return JNI_TRUE;
}
```

```{.cpp title=onload.cpp}
#include "jni.h"
#include "utils/Log.h"
#include "utils/misc.h"


namespace UserLedJNI {
     int registerJniUserLed(JNIEnv* env);
};

using namespace UserLedJNI;

extern "C" jint JNI_OnLoad(JavaVM* vm, void* /* reserved */)
{
    JNIEnv* env = NULL;
    jint result = -1;

    if (vm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {
        ALOGE("GetEnv failed!");
        return result;
    }
    ALOG_ASSERT(env, "Could not retrieve the env!");

    UserLedJNI::registerJniUserLed(env);
    return JNI_VERSION_1_4;
}
```

```{title=Android.bp}
cc_library_shared {
    name: "libUserLedJNI", //lib prefix is required
    vendor: true,

    defaults: ["libUserLedJNI-libs"],

    cflags: [
        "-Wall",
        "-Werror",
        "-Wno-unused-parameter",
    ],

    srcs: [
        "onload.cpp",
        "UserLedJNI.cpp"
    ],
}

cc_defaults {
    name: "libUserLedJNI-libs",

    shared_libs: [
        "libbase",
        "libcutils",
        "liblog",
        "libhardware",
        "libhidlbase",
        "libutils",
        "placamae.hal.userled@1.0"
    ],

    header_libs: [
        "jni_headers",
    ],
}
```

Agora é preciso adicionar o JNI ao produto:

```{title=meuproduto.mk}

#...
PRODUCT_PACKAGES += \
  libUserLedJNI
```

#### Complementando a JNI

Para que seja possível utilizar essa JNI com outros aplicativos, uma das possibilidades é separar o cliente nativo e criar uma interface para acessar os seus métodos. Para isso, a partir da pasta raiz do AOSP, criei o seguinte diretório:

```
mkdir -p device/casa/placamae/libs/java
```

Dentro dessa pasta, criei esse novo caminho:

```
mkdir -p com/interfaces
```

Dentro da pasta adicionei o seguinte arquivo:

```{.java title=UserLedJNI.java}

package com.interfaces;
import android.util.Log;

public class UserLedJNI {
    static final String TAG = "com.interfaces.UserLedJNI";
    static final boolean DEBUG = false;

    private native void nativeInit();
    private static native boolean nativeSetMode(String mode);

    public UserLedJNI() {
        Log.d(TAG, "UserLedJNI onCreate()");
        synchronized (this) {
            nativeInit();
        }
    }

    public boolean setMode(String mode) {
        return nativeSetMode(mode);
    }

    static {
        Log.d(TAG, "UserLedJNI static");
        System.loadLibrary("UserLedJNI");
    }
}
```

Voltando para a pasta `device/casa/placamae/libs/java`, criei esse arquivo:

```{.java title=Android.bp}
java_sdk_library {
    name: "com.interfaces",
    vendor: true,

    srcs: [
        "com/**/*.java",
    ],

    sdk_version: "current",
    compile_dex: true,
    api_packages: ["com.interfaces"],
}
```

Os arquivos ficam organizados dessa forma:

```
device/casa/placamae/libs/java
├── Android.bp
└── com
    └── interfaces
        └── UserLedJNI.java
```

Agora, na pasta raiz do AOSP, é necessário usar o seguinte comando:

```
build/soong/scripts/gen-java-current-api-files.sh "device/casa/placamae/libs/java/api"  system- test- && m update-api
```

Um erro ocorrerá informando, que é necessário mapear alguns *filegroups*:

```
com.interfaces.api.public.latest
com.interfaces-removed.api.public.latest
com.interfaces-incompatibilities.api.public.latest
com.interfaces.api.system.latest
com.interfaces-removed.api.system.latest
com.interfaces-incompatibilities.api.system.latest
```

> Observe também que uma pasta com o nome `api` foi criada.

Os *filegroups* são adicionados em:

```{.java title=device/casa/placamae/libs/java/libs/java/Android.bp}
java_sdk_library {
    name: "com.interfaces",
    vendor: true,

    srcs: [
        "com/**/*.java",
    ],

    sdk_version: "current",
    compile_dex: true,
    api_packages: ["com.interfaces"],
}

filegroup {
    name: "com.interfaces.api.public.latest",
    srcs: ["api/current.txt"]
}

filegroup {
    name: "com.interfaces-removed.api.public.latest",
    srcs: ["api/removed.txt"]
}

filegroup {
    name: "com.interfaces-incompatibilities.api.public.latest",
    srcs: ["api/incompatibilities.txt"]
}

filegroup {
    name: "com.interfaces.api.system.latest",
    srcs: ["api/system-current.txt"]
}

filegroup {
    name: "com.interfaces-removed.api.system.latest",
    srcs: ["api/system-removed.txt"]
}

filegroup {
    name: "com.interfaces-incompatibilities.api.system.latest",
    srcs: ["api/system-incompatibilities.txt"]
}
```

Ainda resta criar 2 arquivos vazios dentro da pasta `api`:

```
touch device/casa/placamae/libs/java/api/incompatibilities.txt
touch device/casa/placamae/libs/java/api/system-incompatibilities.txt
```

A estrutura ficou assim:

```
device/casa/placamae/libs/java
├── Android.bp
├── api
│   ├── current.txt
│   ├── incompatibilities.txt
│   ├── removed.txt
│   ├── system-current.txt
│   ├── system-incompatibilities.txt
│   ├── system-removed.txt
│   ├── test-current.txt
│   └── test-removed.txt
└── com
    └── interfaces
        └── UserLedJNI.java

```
Executando novamente o comando:

```
build/soong/scripts/gen-java-current-api-files.sh "device/casa/placamae/libs/java/api"  system- test- && m update-api
```

A build deve resultar em sucesso.


### Aplicação

Para ter uma interação melhor com as camadas que foram criadas, criei um app simples em java. Dessa forma, é possível alterar o estado do LED apenas apertando os botões da aplicação.

Primeiro, dentro da pasta raiz do AOSP, criei a pasta:

```
mkdir -p device/casa/placamae/app/UserLedApp
```

Dentro dela crie os arquivos:

```{.xml title=AndroidManifest.xml}
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
  package="com.example.userledapp">

  <application 
      android:name=".UserLedServiceApp"
      android:label="UserLedApp"
      
      android:requiredForAllUsers="true"
      android:persistent="true">
      <uses-library android:name="com.interfaces" />

      <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

```{.c title=Android.bp}
android_app {
    name: "UserLedApp",
    certificate: "platform", // to be a persistent app
    vendor: true,
    sdk_version: "current",

    static_libs: [
        "androidx-constraintlayout_constraintlayout",
        "androidx-constraintlayout_constraintlayout-solver",
    ],

    resource_dirs: ["res"],

    srcs: ["java/**/*.java"],

    defaults: ["hidl_defaults"],
    jni_libs: ["libUserLedJNI"],
    libs: ["com.interfaces"],
 }
```

Depois criei a seguinte pasta:

```
mkdir -p device/casa/placamae/app/UserLedApp/java/com/example/userledapp/
```

e dentro delas as seguintes classes:

```{.java title=MainActivity.java}
package com.example.userledapp;

import android.app.Activity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

public class MainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    @Override
    protected void onResume() {
        super.onResume();
    }

    public void onClick(View view) {
        String setValue = ((Button)view).getText().toString();
        UserLedServiceApp.getLed().setMode(setValue);
    }
}
```

```{.java title=UserLedBroadcastReceiver.java}
package com.example.userledapp;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

public class UserLedBroadcastReceiver extends BroadcastReceiver {
    private static final String TAG = "userledAppBroadcast";

    @Override
    public void onReceive(Context context, Intent intent) {
        String mode = intent.getStringExtra("setMode");
        if(mode != null) {
            if(UserLedServiceApp.getLed().setMode(mode)) {
                Log.d(TAG, "Succesfuly setMode to (" + mode + ")");
            }
            else {
                Log.e(TAG, "Failed calling setMode to (" + mode + ")");
            }
        }
    }
}
```

```{.java title=UserLedServiceApp.java}
package com.example.userledapp;

import android.app.Application;
import android.content.IntentFilter;
import android.util.Log;
import com.interfaces.UserLedJNI;


public class UserLedServiceApp extends Application {
    private static final String TAG = "userledServiceApp";

    UserLedBroadcastReceiver broadcast = new UserLedBroadcastReceiver();
    private static UserLedJNI userledjni; // JAVA -> JNI -> HIDL

    public void onCreate() {
        super.onCreate();

        Log.d(TAG, "onCreate()");
        userledjni = new UserLedJNI();

        Log.d(TAG, "setMode(default-on) => " + userledjni.setMode("default-on"));

        IntentFilter filter = new IntentFilter("com.fooHIDL.fooHIDL");
        registerReceiver(broadcast, filter);
    }

    public void onTerminate() {
        super.onTerminate();
        Log.d(TAG, "Terminated");
    }

    public static UserLedJNI getLed() {
        return userledjni;
    }
}
```

Agora criei a pasta:

```
mkdir -p device/casa/placamae/app/UserLedApp/res/layout
```

com o seguinte arquivo:

```{.xml title=activity_main.xml}
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.constraintlayout.widget.Guideline
        android:id="@+id/middleGuideLine"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintGuide_percent="1"
        app:layout_constraintStart_toStartOf="parent" />

    <Button
        android:id="@+id/onButton"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginStart="200dp"
        android:layout_marginTop="72dp"
        android:onClick="onClick"
        android:text="default-on"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <Button
        android:id="@+id/offButton"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginStart="440dp"
        android:layout_marginTop="76dp"
        android:onClick="onClick"
        android:text="none"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <Button
        android:id="@+id/blinkButton"
        android:layout_width="329dp"
        android:layout_height="46dp"
        android:layout_marginTop="212dp"
        android:onClick="onClick"
        android:text="heartbeat"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

Por fim, é só adicionar o app ao produto:

```{title=meuproduto.mk}
PRODUCT_PACKAGES += \
  libUserLedJNI \
  com.interfaces \
  UserLedApp
```

A estrutura final ficou assim:

```
device/casa/placamae/app/UserLedApp
├── Android.bp
├── AndroidManifest.xml
├── java
│   └── com
│       └── example
│           └── userledapp
│               ├── MainActivity.java
│               ├── UserLedBroadcastReceiver.java
│               └── UserLedServiceApp.java
└── res
    └── layout
        └── activity_main.xml
```

Agora é só rebuildar o AOSP, flashar as imagens e procurar pelo app UserLedApp

<figure markdown>
  ![imagems](images/app_demo.gif){ width="600" }
</figure>

#### Cliente em Java

Existe a possibilidade de não a JNI e usar a HIDL direto com o próprio JAVA. Para isso basta fazer algumas alteracões nos arquivos já criados:

```{.xml title=AndroidManifest.xml}
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
  package="com.example.userledapp">

  <application 
      android:name=".UserLedServiceApp"
      android:label="UserLedApp"
      
      android:requiredForAllUsers="true"
      android:persistent="true">

      <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

```{.c title=Android.bp}
android_app {
    name: "UserLedApp",
    certificate: "platform", // to be a persistent app
    system_ext_specific: true, //https://source.android.com/docs/core/architecture/bootloader/partitions/product-interfaces#java-interfaces
    platform_apis: true,

    resource_dirs: ["res"],

    srcs: ["java/**/*.java"],

    static_libs: [
        "android.hidl.base-V1.0-java",
        "placamae.hal.userled-V1.0-java",
        "androidx-constraintlayout_constraintlayout",
        "androidx-constraintlayout_constraintlayout-solver",
    ],
 }
```

```{.java title=MainActivity.java}
package com.example.userledapp;

import android.app.Activity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

public class MainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    @Override
    protected void onResume() {
        super.onResume();
    }

    public void onClick(View view) {
        String setValue = ((Button)view).getText().toString();

        try {
            UserLedServiceApp.getLed().setMode(setValue);
        } 
        catch (android.os.RemoteException e) {
            Log.e("MainActiviry", "user led HIDL Java proxy returned error", e);
        }
    }
}
```

```{.java title=UserLedBroadcastReceiver.java}
package com.example.userledapp;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

public class UserLedBroadcastReceiver extends BroadcastReceiver {
    private static final String TAG = "userledAppBroadcast";

    @Override
    public void onReceive(Context context, Intent intent) {
        String mode = intent.getStringExtra("setMode");
        try {
            if(mode != null) {
                if(UserLedServiceApp.getLed().setMode(mode)) {
                    Log.d(TAG, "Succesfuly setMode to (" + mode + ")");
                }
                else {
                    Log.e(TAG, "Failed calling setMode to (" + mode + ")");
                }
            }
        }
        catch (android.os.RemoteException e) {
                Log.e(TAG, "IUserLed error", e);
        }
    }
}
```

Agora é só rebuildar e testar.

```{.java title=UserLedServiceApp.java}
package com.example.userledapp;

import android.app.Application;
import android.content.IntentFilter;
import android.util.Log;
import placamae.hal.userled.V1_0.IUserLed;


public class UserLedServiceApp extends Application {
    private static final String TAG = "userledServiceApp";

    UserLedBroadcastReceiver broadcast = new UserLedBroadcastReceiver();
    private static IUserLed userledJava; // HIDL Java Proxy

    public void onCreate() {
        super.onCreate();

        Log.d(TAG, "onCreate()");
        try {
            userledJava = IUserLed.getService(true);
            Log.d(TAG, "HIDL-Java setMode(default-on) => " + userledJava.setMode("default-on"));
        } 
        catch (android.os.RemoteException e) {
            Log.e(TAG, "IUserLed error", e);
        }


        IntentFilter filter = new IntentFilter("com.fooHIDL.fooHIDL");
        registerReceiver(broadcast, filter);
    }

    public void onTerminate() {
        super.onTerminate();
        Log.d(TAG, "Terminated");
    }

    // HIDL Java Proxy
    public static IUserLed getLed() {
        return userledJava;
    }
}
```





