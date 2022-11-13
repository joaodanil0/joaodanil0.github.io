# Blink Led AIDL

<figure markdown>
  ![imagems](images/hello-world.jpg){ width="600" }
  <figcaption>
  Image by <a href="https://pixabay.com/users/wnantes-463928/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1333103">Wilson Nantes</a> from <a href="https://pixabay.com//?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=1333103">Pixabay</a>
  </figcaption>
</figure>

> Vale ressaltar que para esse post é necessário já ter um produto buildado e para o ultimo tópico é preciso do display já em funcionamento.

## Introdução

Depois de fazer o curso [*Android 12 Internals material, labs and discussion*](https://8mantech.thinkific.com/courses/android-internals), resolvi implementar um *blink led* utilizando a AIDL, e cá estou registrando os meus passos aqui :alien:.

## Informações úteis

- Versão do AOSP: `android-12.0.0_r4`
- Versão do kernel Android: `android-amlogic-bmeson-5.4`
- Distro para compilaçao: `Linux Mint 20.3`
- Versão do kernel da distro: `5.15.0-52-generic`

## Implementação

Tentarei seguir os mesmos tópicos do post: ==Implementando a AIDL==.

### HAL

Na raiz da pasta do AOSP criei o seguinte caminho:

```{.sh}
mkdir -p device/casa/placamae/interfaces/userled/aidl/
```

Dentro da pasta, criei o seguinte arquivo:

```{.c title="Android.bp"}
aidl_interface {
  name: "placamae.hal.userled",
  vendor: true,
  srcs: ["placamae/hal/userled/*.aidl"],
  stability: "vintf",
  owner: "Jao",
  backend: {
    cpp: {
      enabled: false, //enabled by default
    },
    java: {
      sdk_version: "module_current",
    },
    ndk: {
      //ndk_platform is generated too, but it will deprecated on Android 13.
      //just ndk will be available.
      enabled: true, //enabled by default, just exposing
    }
  },
}
```

ainda dentro da pasta, criei o seguinte caminho:

```{.sh}
mkdir -p placamae/hal/userled
```
com o seguinte arquivo:

```{.java title="IUserLed.aidl"}
package placamae.hal.userled;

@VintfStability
interface IUserLed{
    boolean setMode(in String mode);
}
```
O resultado foi esse:

```{.sh}
device/casa/placamae/interfaces/userled/aidl
├── Android.bp
└── placamae
    └── hal
        └── userled
            └── IUserLed.aidl
```

Agora é só adicionar ao produto, no meu caso:

```{.mk title="meuproduto.mk"}
...
PRODUCT_PACKAGES += \
  placamae.hal.userled

```

---

Agora vamos gerar os arquivos fonte para a AIDL. Para isso, na pasta raiz do AOSP, digite:

```{.sh}
m placamae.hal.userled-update-api
```
Esse comando vai gerar a pasta `aidl_api`:

```{.sh  hl_lines="2"}
device/casa/placamae/interfaces/userled/aidl
├── aidl_api
│   └── placamae.hal.userled
│       └── current
│           └── placamae
│               └── hal
│                   └── userled
│                       └── IUserLed.aidl
├── Android.bp
└── placamae
    └── hal
        └── userled
            └── IUserLed.aidl
```

Agora dentro da pasta `device/casa/placamae/interfaces/userled/aidl`, digite o comando:

```{.sh}
mm
```

Ela irá gerar as classes necessárias dentro da pasta :

```{.sh}
out_placamae/soong/.intermediates/device/casa/placamae/interfaces/userled/aidl
├── placamae.hal.userled-api
├── placamae.hal.userled-V1-java
├── placamae.hal.userled-V1-java-source
├── placamae.hal.userled-V1-ndk
├── placamae.hal.userled-V1-ndk_platform
├── placamae.hal.userled-V1-ndk_platform-source
└── placamae.hal.userled-V1-ndk-source
```

### Serviço

Criei a pasta `device/casa/placamae/interfaces/userled/aidl/default/`, e dentro dela os arquivos:

```{.cpp title="UserLed.h"}
// This library is available on:
// out/soong/.intermediates/device/casa/placamae/interfaces/userled/aidl/
// inside: 
// placamae.hal.userled-V1-ndk_platform-source/gen/include/aidl/placamae/hal/userled
#include <aidl/placamae/hal/userled/BnUserLed.h>

namespace aidl::placamae::hal::userled {
  class UserLed : public BnUserLed {
    public:
      static inline const char RED_LED[] = "/sys/devices/platform/leds/leds/vim3:red/trigger";

    public:
      ndk::ScopedAStatus setMode(const std::string &in_mode, bool *_aidl_return) override;
      static int writeValue(const char *file, const char *value);
  };
}
```

```{.cpp title="UserLed.cpp"}
#include "UserLed.h"

#include <utils/Log.h>
#include <android-base/logging.h>
#include <sys/stat.h>

namespace aidl::placamae::hal::userled {

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

  ndk::ScopedAStatus UserLed::setMode(const std::string &in_mode, bool *_aidl_return) {
    LOG(INFO) << "UserLed -> setMode data=(" << in_mode.c_str() << ")";
    *_aidl_return = this->writeValue(RED_LED, in_mode.c_str()) == 0;
    return ndk::ScopedAStatus::ok();
  }
}
```

```{.cpp title="service.cpp"}
#define LOG_TAG "placamae.hal.userled"
#include <android-base/logging.h>
#include <android/binder_process.h>
#include <binder/ProcessState.h>

#include <android/binder_manager.h>
#include <binder/IServiceManager.h>

#include "UserLed.h"

using aidl::placamae::hal::userled::UserLed;
using std::string_literals::operator""s;

int main(){
    LOG(INFO) << "UserLed -> TESSSSSSTE";
    const std::string name = UserLed::descriptor + "/default"s;

    android::ProcessState::initWithDriver("/dev/vndbinder");

    ABinderProcess_startThreadPool();

    LOG(INFO) << "UserLed -> Service is starting...";

    std::shared_ptr<UserLed> userLed = ndk::SharedRefBase::make<UserLed>();

    CHECK_EQ(STATUS_OK, AServiceManager_addService(userLed->asBinder().get(), name.c_str()));

    LOG(INFO) << "UserLed -> started...";

    ABinderProcess_joinThreadPool();

    return EXIT_FAILURE;  // should not reach
}
```

```{.c title="Android.bp"}
cc_binary {
  name: "placamae.hal.userled-service",
  vendor: true,
  relative_install_path: "hw",
  init_rc: ["placamae.hal.userled-service.rc"],
  vintf_fragments: ["placamae.hal.userled-service.xml"],

  srcs: [
    "service.cpp",
    "UserLed.cpp"
  ],

  cflags: [
    "-Wall",
    "-Werror",
  ],

  shared_libs: [
    "libbase",
    "liblog",
    "libhardware",
    "libbinder_ndk",
    "libbinder",
    "libutils",
    //ndk_platform will be deprecated on Android 13. 
    "placamae.hal.userled-V1-ndk_platform",
  ],
}
``` 

```{.rc title="placamae.hal.userled-service.rc"}
service placamae.hal.userled-service /vendor/bin/hw/placamae.hal.userled-service
    interface aidl placamae.hal.userled-service.IUserLed/default
    class hal
    user system
    group system

on boot
    chown system system /sys/devices/platform/leds/leds/vim3:red/trigger
    chmod 0660 /sys/devices/platform/leds/leds/vim3:red/trigger
```

```{.xml title="placamae.hal.userled-service.xml"}
<manifest version="1.0" type="device">
    <hal format="aidl">
        <name>placamae.hal.userled</name>
        <version>1</version>
        <fqname>IUserLed/default</fqname>
    </hal>
</manifest>
```

Agora só resta adicionar o serviço ao produto:

```{.mk title="meuproduto.mk" hl_lines="4"}
...
PRODUCT_PACKAGES += \
  placamae.hal.userled \
  placamae.hal.userled-service
```
### Permissões

Para facilitar a implementação, foi definido a permissão ==permissiva==, mas ainda é necessário subir o serviço. Essa etapa é feita via sepolicy.

Para isso, criei a pasta:

```{.sh}
mkdir -p device/casa/placamae/sepolicy
```

Dentro dela criei os arquivos:

```{.c title="file_contexts"}
/vendor/bin/hw/placamae\.hal\.userled-service         u:object_r:hal_userled_default_exec:s0
```

```{.c title="hal_userled_default.te"}
type hal_userled_default, domain;
type hal_userled_default_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(hal_userled_default);
```

Por fim, adicionei a pasta sepolicy no: 

```{.c title="BoardConfig.mk"}
...
BOARD_SEPOLICY_DIRS += device/casa/placamae/sepolicy
```

### Testando

Para testar a HAL, criei a pasta:

```{.sh}
mkdir -p device/casa/placamae/interfaces/userled/aidl/default/LedTest
```
e dentro dela os seguintes arquivos:


```{.cpp title="LedTest.cpp"}
#include <aidl/placamae/hal/userled/IUserLed.h>
#include <android/binder_manager.h>

#include <iostream>
#include <string>

using ::aidl::placamae::hal::userled::IUserLed;


int main(int argc, char *argv[]) {

    std::shared_ptr<IUserLed> mHal;
    std::string a;
    bool c;

    if (argc != 2) {
        std::cout << "USAGE ./LedTest <none|heartbeat|default-on>\n";
        exit(0);
    }

    AIBinder* binder = AServiceManager_waitForService("placamae.hal.userled.IUserLed/default");

    if (binder == nullptr){
        std::cout << "Failed to get cpu service\n";
        exit(-1);
    }

    mHal = IUserLed::fromBinder(ndk::SpAIBinder(binder));

    mHal->setMode(argv[1], &c);
    std::cout << "setScalingGovernor:" << c << std::endl;


    return 0;
}
```

```{.c title="Android.bp"}
cc_binary {
    name: "LedTest",
    vendor: true,
    relative_install_path: "hw",

    srcs: ["LedTest.cpp"],

    shared_libs: [
        "libbase",
        "liblog",
        "libhardware",
        "libbinder_ndk",
        "libbinder",
        "libutils",
        //ndk_platform will be deprecated on Android 13. 
        "placamae.hal.userled-V1-ndk_platform",
    ],
}
```

Na pasta raiz do AOSP: 

```{.sh}
m -j16
```

Depois de terminar a build, flashei os binários e esperei o Android subir para ter acesso ao ADB. Para testar o serviço, utilizei os comandos: 

```{.sh}
adb root
adb shell
cd /vendor/bin/hw
./LedTest heartbeat
```

Após esses comandos o LED vermelho da placa começará a piscar.

### Criando uma aplicação

Para ter uma interação melhor com o serviço, criei um app simples em java. Dessa forma, é possível alterar o estado do LED apenas apertando os botões da aplicação.

Primeiro, dentro da pasta raiz do AOSP, criei a pasta:

```{.sh}
mkdir -p device/casa/placamae/app/UserLedApp
```

Dentro dela crie os arquivos:

```{.xml title=AndroidManifest.xml}
<?xml version="1.0" encoding="utf-8"?>
<manifest 
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.example.userledapp">

    <application 
        android:name=".UserLedServiceApp"
        android:label="UserLedApp">

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
    system_ext_specific: true,
    platform_apis: true,    
    
    static_libs: [
        "placamae.hal.userled-V1-java",
        "androidx-constraintlayout_constraintlayout",
        "androidx-constraintlayout_constraintlayout-solver",
    ],

    // package native libs in the APK
    use_embedded_native_libs: true,

    resource_dirs: ["res"],

    srcs: [
        "java/**/*.java",
    ],
}
```

--- 

Depois criei a seguinte pasta:

```{.sh}
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

        try {
            UserLedServiceApp.getLed().setMode(setValue);
        }
        catch (android.os.RemoteException e) {
            Log.e("userled", "AIDL Java proxy returned error", e);
        }
    }
}
```

```{.java  title=UserLedBroadcastReceiver.java}
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
            try {
                if(UserLedServiceApp.getLed().setMode(mode)) {
                    Log.d(TAG, "Succesfuly setMode to (" + mode + ")");
                }
                else {
                    Log.e(TAG, "Failed calling setMode to (" + mode + ")");
                }
            }
            catch (android.os.RemoteException e) {
                Log.e(TAG, "IUserLed error", e);

            }
        }
    }
}
```

```{.java title=UserLedServiceApp.java}
package com.example.userledapp;

import android.app.Application;
import android.content.IntentFilter;
import android.os.IBinder;
import android.os.ServiceManager;
import android.util.Log;


public class UserLedServiceApp extends Application {

    private static final String TAG = "userledServiceApp";
    private static final String INTERFACE = "placamae.hal.userled.IUserLed/default";

    UserLedBroadcastReceiver broadcast = new UserLedBroadcastReceiver();

    private static placamae.hal.userled.IUserLed Led; // AIDL Java Proxy

    @Override
    public void onCreate() {
        super.onCreate();

        IBinder binder = ServiceManager.getService(INTERFACE);

        if (binder == null) {
            Log.e(TAG, "Getting " + INTERFACE + " service daemon binder failed");
        }
        else {
            Led = placamae.hal.userled.IUserLed.Stub.asInterface(binder);

            if (Led == null) {
                Log.e(TAG, "Getting ICpu Aidl daemon interface failed");
            }
        }

        IntentFilter filter = new IntentFilter("com.fooHIDL.fooHIDL");
        registerReceiver(broadcast, filter);
    }

    public void onTerminate() {
        super.onTerminate();
        Log.d(TAG, "Terminated");
    }

    // AIDL Java Proxy
    public static placamae.hal.userled.IUserLed getLed() {
        return Led;
    }
}
```

---

Agora criei a pasta:

```{.sh}
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

---

A estrutura final ficou assim: 

```{.sh}
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

Por fim, é só rebuildar o AOSP, flashar as imagens e procurar pelo app `UserLedApp`.

---

Esse é o resultado do funcionamento do aplicativo ativando e desativando o LED vermelho da placa.

<figure markdown>
  ![imagems](images/app_demo.gif){ width="600" }
</figure>

## Conclusão

Apesar de posuir muitos passos, implementar uma HAL utilizando AIDL é mais simples que uma HILD. Não foi preciso implementar uma JNI, pois o AIDL já gera o *backend* em java. Isso e outras coisas acabam facilidando a implementação.

O Led vermleho já vem habilitado por padrão pelo kernel que foi utilizado. Um próximo passo seria utilizar outra porta GPIO, por padrão, elas não vem ativadas e esse seria mais um desafio na parte de kernel, do que de uma HAL.

Por fim, vale relembar que que o `ndk_platform` ira ser depreciado a partir do Android 13, sendo substituído pelo `ndk`, apenas. Para tirar algumas dúvidas de implementação, costumo consultar a HAL de *Power*. Foi nela que encontrei essa informação ([ndk_platform depreciado](https://cs.android.com/android/_/android/platform/hardware/interfaces/+/e0f2d29b7f3045cf5d4abfa382f1761dbe7e3c09)).


