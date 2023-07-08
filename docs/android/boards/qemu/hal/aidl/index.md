# Implementando a AIDL


<figure markdown>
  ![imagems](images/aidl.jpg){ width="600" }
  <figcaption>
  Image by <a href="https://pixabay.com/users/methodshop-1460919/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=3404892">methodshop</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=3404892">Pixabay</a>
  </figcaption>
</figure>

> Baseado no curso: [*Android 12 Internals material, labs and discussion*](https://8mantech.thinkific.com/courses/android-internals)

## Introdução

A partir do Android 12, novas *Hardware Abstraction Layer* (HAL) passam a ser obrigatoriamente implementadas utilizando *Android Interface Definition Language* (AIDL). Por isso, criei esse post.

## Implementação

Vou tentar dividir em alguns tópicos para facilitar.

### HAL

Na pasta raiz do [AOSP](https://source.android.com/), criei o seguinte caminho:

```{.sh}
mkdir -p device/casa/emulator/interfaces/foo/aidl/
```

Dentro da pasta, criei o seguinte arquivo:

```{.mk title="Android.bp"}
aidl_interface {
    name: "mypackage.mysubpackage.fooAIDL",
    vendor: true,
    srcs: ["mypackage/mysubpackage/fooAIDL/*.aidl"],
    stability: "vintf",
    owner: "CASA",
    backend: {
        cpp: {
            enabled: true,
        },
        java: {
            sdk_version: "module_current",
        },
    },
}
```

ainda dentro da pasta, criei o seguinte caminho:

```{.sh}
mkdir -p mypackage/mysubpackage/fooAIDL/
```

com o seguinte arquivo:

```{.aidl title="ITest.aidl"}
package mypackage.mysubpackage.fooAIDL;

@VintfStability
interface ITest {
    String getTest();
    boolean setTest(in String param);
}
```

O resultado foi esse:

```{.sh}
device/casa/emulator/interfaces/foo/aidl
├── Android.bp
└── mypackage
    └── mysubpackage
        └── fooAIDL
            └── ITest.aidl
```

Agora no arquivo do produto, no meu caso:

```{.mk title="emulator.mk"}
...
PRODUCT_PACKAGES += \
    mypackage.mysubpackage.fooAIDL \
```

Na pasta raiz do AOSP, digite:

```{.sh}
m mypackage.mysubpackage.fooAIDL-update-api
```
Esse comando vai gerar a pasta `aidl_api`:

```{.sh  hl_lines="2"}
device/casa/emulator/interfaces/foo/aidl
├── aidl_api
│   └── mypackage.mysubpackage.fooAIDL
│       └── current
│           └── mypackage
│               └── mysubpackage
│                   └── fooAIDL
│                       └── ITest.aidl
├── Android.bp
└── mypackage
    └── mysubpackage
        └── fooAIDL
            └── ITest.aidl
```

Esse comando faz uma espécie de versionamento da AIDL.

Agora dentro da pasta `device/casa/emulator/interfaces/foo/aidl`, digite o comando:

```{.sh}
mm
```

Ela irá gerar as classes necessárias dentro da pasta :

```{.sh}
out/soong/.intermediates/device/casa/emulator/interfaces/foo/aidl
├── mypackage.mysubpackage.fooAIDL-api
├── mypackage.mysubpackage.fooAIDL-V1-cpp
├── mypackage.mysubpackage.fooAIDL-V1-cpp-source
├── mypackage.mysubpackage.fooAIDL-V1-java
├── mypackage.mysubpackage.fooAIDL-V1-java-source
├── mypackage.mysubpackage.fooAIDL-V1-ndk
├── mypackage.mysubpackage.fooAIDL-V1-ndk_platform
├── mypackage.mysubpackage.fooAIDL-V1-ndk_platform-source
└── mypackage.mysubpackage.fooAIDL-V1-ndk-source
```

### Serviço

Criei a pasta `device/casa/emulator/interfaces/foo/aidl/default/`, e dentro dela os arquivos:

```{.c title="Test.h"}
#pragma once

// Mesmo caminho após:
// mypackage.mysubpackage.fooAIDL-V1-ndk_platform-source/gen/include/
// dentro da pasta : 
// out/soong/.intermediates/device/casa/emulator/interfaces/foo/aidl
#include <aidl/mypackage/mysubpackage/fooAIDL/BnTest.h>

namespace aidl::mypackage::mysubpackage::fooAIDL {

class Test : public BnTest {
  public:
    ndk::ScopedAStatus getTest(std::string* _aidl_return) override;
    ndk::ScopedAStatus setTest(const std::string& in_param, bool* _aidl_return) override;
};

}  // namespace aidl::mypackage::mysubpackage::fooAIDL
```

```{.cpp title="Test.cpp"}
#include "Test.h"

#include <utils/Log.h>
#include <sys/stat.h>

namespace aidl::mypackage::mysubpackage::fooAIDL {

// conservative|powersave|performance|schedutil
static const char SCALING_GOVERNOR[] = \
        "/sys/devices/system/cpu/cpu0/cpufreq/scaling_governor";

static int write_value(const char *file, const char *value)
{
    int to_write, written, ret, fd;

    fd = TEMP_FAILURE_RETRY(open(file, O_WRONLY));
    if (fd < 0) {
        return -1;
    }

    to_write = strlen(value) + 1;
    written = TEMP_FAILURE_RETRY(write(fd, value, to_write));
    if (written == -1) {
        ret = -2;
    } else if (written != to_write) {
        ret = -3;
    } else {
        ret = 0;
    }

    errno = 0;
    close(fd);

    return ret;
}

ndk::ScopedAStatus Test::getTest(std::string* _aidl_return) {
    char str[20];
    int fd;
    ssize_t ret = 0;
    struct stat info;
    void *data = NULL;
    size_t size;

    // If open returns error code EINTR, retry again until error code
    // is not a temporary failure
    fd = TEMP_FAILURE_RETRY(open(SCALING_GOVERNOR, O_RDONLY));
    if (fd < 0) {
        ndk::ScopedAStatus::fromServiceSpecificError(-1);
    }

    fstat(fd, &info);
    size = info.st_size;
    data = malloc(size);
    if (data == NULL) {
        *_aidl_return = "error can't malloc";
        goto exit;
    }

    ret = read(fd, data, size);
    if (ret < 0) {
        *_aidl_return = "error reading fd";
        goto exit;
    }

    snprintf(str, sizeof(str), "%s", (const unsigned char*)data);
    ALOGI("Test AIDL::getTest data=(%s)", str);
    *_aidl_return = str;

exit:
    close(fd);
    free(data);
    return ndk::ScopedAStatus::ok();
}

ndk::ScopedAStatus Test::setTest(const std::string& in_param, bool* _aidl_return) {
    ALOGI("Test AIDL::setTest data=(%s)", in_param.c_str());
    *_aidl_return = write_value(SCALING_GOVERNOR, in_param.c_str()) == 0;
    return ndk::ScopedAStatus::ok();
}

}  // namespace aidl::vendor::eightman::cpu
```

```{.cpp title="service.cpp"}
#define LOG_TAG "mypackage.mysubpackage.fooAIDL-service"

#include <android-base/logging.h>
#include <android/binder_process.h>
#include <binder/ProcessState.h>

#include <android/binder_manager.h>
#include <binder/IServiceManager.h>

#include "Test.h"

using aidl::mypackage::mysubpackage::fooAIDL::Test;
using std::string_literals::operator""s;

int main() {
    // Enable vndbinder to allow vendor-to-vendor binder calls.
    android::ProcessState::initWithDriver("/dev/vndbinder");

    ABinderProcess_startThreadPool();

    LOG(INFO) << "Test aidl service for fooAIDL is starting.,,";

    std::shared_ptr<Test> test = ndk::SharedRefBase::make<Test>();
    const std::string name = Test::descriptor + "/default"s;
    CHECK_EQ(STATUS_OK, AServiceManager_addService(test->asBinder().get(), name.c_str()));

    LOG(INFO) << "Initialized fooAIDL cpu";

    ABinderProcess_joinThreadPool();

    return EXIT_FAILURE;  // should not reach
}
```

```{.bp title="Android.bp"}
cc_binary {
    name: "mypackage.mysubpackage.fooAIDL-service",
    vendor: true,
    relative_install_path: "hw",
    init_rc: ["mypackage.mysubpackage.fooAIDL-service.rc"],
    vintf_fragments: ["mypackage.mysubpackage.fooAIDL-service.xml"],

    srcs: [
        "service.cpp",
        "Test.cpp"
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
        "mypackage.mysubpackage.fooAIDL-V1-ndk_platform",
    ],
}
```

```{.rc title="mypackage.mysubpackage.fooAIDL-service.rc"}
service mypackage.mysubpackage.fooAIDL-service /vendor/bin/hw/mypackage.mysubpackage.fooAIDL-service
    interface aidl mypackage.mysubpackage.fooAIDL-service.ITest/default
    class hal
    user system
    group system
        
on boot
    chown system system /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
    chmod 0660 /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

```{.xml title="mypackage.mysubpackage.fooAIDL-service.xml"}
<manifest version="1.0" type="device">
    <hal format="aidl">
        <name>mypackage.mysubpackage.fooAIDL</name>
        <version>1</version>
        <fqname>ITest/default</fqname>
    </hal>
</manifest>
```

Agora só resta adicionar o serviço no produto:

```{.mk title="emulator.mk" hl_lines="4"}
...
PRODUCT_PACKAGES += \
    mypackage.mysubpackage.fooAIDL \
    mypackage.mysubpackage.fooAIDL-service
```

### Permissões

Rebuildando e checando o `dmesg`:

```{.py }
10-08 07:51:06.674     0     0 I init    : Parsing file /vendor/etc/init/mypackage.mysubpackage.fooAIDL-service.rc...
10-08 07:51:10.482     0     0 E init    : Could not start service 'mypackage.mysubpackage.fooAIDL-service' as part of class 'hal': File /vendor/bin/hw/mypackage.mysubpackage.fooAIDL-service(labeled "u:object_r:vendor_file:s0") has incorrect label or no domain transition from u:r:init:s0 to another SELinux domain defined. Have you configured your service correctly? https://source.android.com/security/selinux/device-policy#label_new_services_and_address_denials. Note: this error shows up even in permissive mode in order to make auditing denials possible.
10-08 07:51:10.560     0     0 I init    : processing action (boot) from (/vendor/etc/init/mypackage.mysubpackage.fooAIDL-service.rc:7)
```

Para resolver esse problema, criei uma pasta:

```{.sh}
mkdir -p /device/casa/emulator/sepolicy
```

e dentro dela os arquivos:

```{ title="file_contexts"}
/vendor/bin/hw/mypackage\.mysubpackage\.fooAIDL-service         u:object_r:hal_fooAIDL_default_exec:s0
```

```{title="hal_fooAIDL_default.te"}
type hal_fooAIDL_default, domain;
type hal_fooAIDL_default_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(hal_fooAIDL_default);
```

Rebuildando e checando o `dmesg`:

```{.py hl_lines="3"}
10-08 08:11:08.392     0     0 I init    : starting service 'mypackage.mysubpackage.fooAIDL-service'...
10-08 08:11:09.375  1932  1932 I mypackage.mysubpackage.fooAIDL-service: Test aidl service for fooAIDL is starting.,,
10-08 08:11:09.369  1932  1932 W mypackage.mysub: type=1400 audit(0.0:20): avc: denied { read } for name="vndbinder" dev="binder" ino=6 scontext=u:r:hal_fooAIDL_default:s0 tcontext=u:object_r:vndbinder_device:s0 tclass=chr_file permissive=0
10-08 08:11:09.369  1932  1932 W Binder:1932_2: type=1400 audit(0.0:21): avc: denied { call } for scontext=u:r:hal_fooAIDL_default:s0 tcontext=u:r:servicemanager:s0 tclass=binder permissive=0
10-08 08:11:09.378  1932  1932 F mypackage.mysubpackage.fooAIDL-service: Check failed: STATUS_OK == AServiceManager_addService(test->asBinder().get(), name.c_str()) (STATUS_OK=0, AServiceManager_addService(test->asBinder().get(), name.c_str())=-129) 
10-08 08:11:09.373  1932  1932 W Binder:1932_2: type=1400 audit(0.0:22): avc: denied { call } for scontext=u:r:hal_fooAIDL_default:s0 tcontext=u:r:servicemanager:s0 tclass=binder permissive=0
10-08 08:11:09.389  1937  1937 F DEBUG   : Cmdline: /vendor/bin/hw/mypackage.mysubpackage.fooAIDL-service
10-08 08:11:09.389  1937  1937 F DEBUG   : pid: 1932, tid: 1932, name: Binder:1932_2  >>> /vendor/bin/hw/mypackage.mysubpackage.fooAIDL-service <<<
10-08 08:11:09.390  1937  1937 F DEBUG   :       #03 pc 00000000000031a9  /vendor/bin/hw/mypackage.mysubpackage.fooAIDL-service (main+889) (BuildId: fdbc2fbc6416778b0057ade844657474)
10-08 08:11:08.424     0     0 I init    : Service 'mypackage.mysubpackage.fooAIDL-service' (pid 1932) received signal 6
10-08 08:11:08.424     0     0 I init    : Sending signal 9 to service 'mypackage.mysubpackage.fooAIDL-service' (pid 1932) process group...
10-08 08:11:08.425     0     0 E init    : process with updatable components 'mypackage.mysubpackage.fooAIDL-service' exited 4 times in 4 minutes
```

Para resolver, criei os seguintes arquivos:

```{ title="service_contexts"}
mypackage.mysubpackage.fooAIDL.ITest/default          u:object_r:fooAIDL_service:s0
```

```{title="service.te"}
type fooAIDL_service, service_manager_type, vendor_service;
```

e adicionando essas configurações no arquivo:

```{title="hal_fooAIDL_default.te" hl_lines="5-6"}
type hal_fooAIDL_default, domain;
type hal_fooAIDL_default_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(hal_fooAIDL_default);
vndbinder_use(hal_fooAIDL_default);
add_service(hal_fooAIDL_default, fooAIDL_service);
```

Rebuildando e checando o `dmesg`:


```{.py hl_lines="4 6"}
10-08 08:32:41.525     0     0 E apexd   : Native process 'mypackage.mysubpackage.fooAIDL-service' is crashing. Attempting a revert
10-08 08:32:46.496     0     0 I init    : starting service 'mypackage.mysubpackage.fooAIDL-service'...
10-08 08:32:46.622  1949  1949 I mypackage.mysubpackage.fooAIDL-service: Test aidl service for fooAIDL is starting.,,
10-08 08:32:46.618  1949  1949 W Binder:1949_2: type=1400 audit(0.0:15): avc: denied { call } for scontext=u:r:hal_fooAIDL_default:s0 tcontext=u:r:servicemanager:s0 tclass=binder permissive=0
10-08 08:32:46.625  1949  1949 F mypackage.mysubpackage.fooAIDL-service: Check failed: STATUS_OK == AServiceManager_addService(test->asBinder().get(), name.c_str()) (STATUS_OK=0, AServiceManager_addService(test->asBinder().get(), name.c_str())=-129) 
10-08 08:32:46.622  1949  1949 W Binder:1949_2: type=1400 audit(0.0:16): avc: denied { call } for scontext=u:r:hal_fooAIDL_default:s0 tcontext=u:r:servicemanager:s0 tclass=binder permissive=0
10-08 08:32:46.638  1954  1954 F DEBUG   : Cmdline: /vendor/bin/hw/mypackage.mysubpackage.fooAIDL-service
10-08 08:32:46.638  1954  1954 F DEBUG   : pid: 1949, tid: 1949, name: Binder:1949_2  >>> /vendor/bin/hw/mypackage.mysubpackage.fooAIDL-service <<<
10-08 08:32:46.639  1954  1954 F DEBUG   :       #03 pc 00000000000031a9  /vendor/bin/hw/mypackage.mysubpackage.fooAIDL-service (main+889) (BuildId: fdbc2fbc6416778b0057ade844657474)
10-08 08:32:46.527     0     0 I init    : Service 'mypackage.mysubpackage.fooAIDL-service' (pid 1949) received signal 6
10-08 08:32:46.527     0     0 I init    : Sending signal 9 to service 'mypackage.mysubpackage.fooAIDL-service' (pid 1949) process group...
10-08 08:32:46.533     0     0 E init    : process with updatable components 'mypackage.mysubpackage.fooAIDL-service' exited 4 times in 4 minutes
```

Para resolver, foi adicionado a seguinte configuração ao arquivo:

```{title="hal_fooAIDL_default.te" hl_lines="7"}
type hal_fooAIDL_default, domain;
type hal_fooAIDL_default_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(hal_fooAIDL_default);
vndbinder_use(hal_fooAIDL_default);
add_service(hal_fooAIDL_default, fooAIDL_service);
allow hal_fooAIDL_service servicemanager:binder { call transfer };
```

Rebuildando e checando o `dmesg`:

```{.py hl_lines="6"}
10-08 08:47:08.591     0     0 I init    : Parsing file /vendor/etc/init/mypackage.mysubpackage.fooAIDL-service.rc...
10-08 08:47:13.008     0     0 I init    : starting service 'mypackage.mysubpackage.fooAIDL-service'...
10-08 08:47:13.054     0     0 I init    : processing action (boot) from (/vendor/etc/init/mypackage.mysubpackage.fooAIDL-service.rc:7)
10-08 08:47:13.049   369   369 I mypackage.mysubpackage.fooAIDL-service: Test aidl service for fooAIDL is starting.,,
10-08 08:47:13.049   170   170 I servicemanager: Found mypackage.mysubpackage.fooAIDL.ITest/default in device VINTF manifest.
10-08 08:47:13.049   369   369 I mypackage.mysubpackage.fooAIDL-service: Initialized fooAIDL cpu
```

Até aqui, tudo certo 😄.

### Testando

Para testar a HAL, criei a pasta:

```{.sh}
mkdir -p /device/casa/emulator/interfaces/foo/aidl/default/halTest
```
e dentro dela os seguintes arquivos:

```{.cpp title="HalTest.cpp"}
#include <aidl/mypackage/mysubpackage/fooAIDL/ITest.h>

#include <iostream>
#include <string>

#include <android/binder_manager.h>

using ::aidl::mypackage::mysubpackage::fooAIDL::ITest;


int main(int argc, char *argv[]) {

    std::shared_ptr<ITest> mHal;
    std::string a;
    bool c;

    if (argc != 2) {
        std::cout << "USAGE ./cpu_client <conservative|powersave|performance|schedutil>\n";
        exit(0);
    }

    AIBinder* binder = AServiceManager_waitForService("mypackage.mysubpackage.fooAIDL.ITest/default");

    if (binder == nullptr){
        std::cout << "Failed to get cpu service\n";
        exit(-1);
    }

    mHal = ITest::fromBinder(ndk::SpAIBinder(binder));


    mHal->getTest(&a);
    std::cout << "getScalingGovernor:" << a << std::endl;

    mHal->setTest(argv[1], &c);
    std::cout << "setScalingGovernor:" << c << std::endl;

    mHal->getTest(&a);
    std::cout << "getScalingGovernor:" << a << std::endl;

    return 0;
}
```

```{.bp title="Android.bp"}
cc_binary {
    name: "foo_AIDL_tester",
    vendor: true,
    relative_install_path: "hw",

    srcs: ["HalTest.cpp"],
    
    shared_libs: [
        "libbase",
        "liblog",
        "libhardware",
        "libbinder_ndk",
        "libbinder",
        "libutils",
        "mypackage.mysubpackage.fooAIDL-V1-ndk_platform"
    ],
}
```

Rebuildando utilizando o comando:

```{.sh}
adb root
adb shell "/vendor/bin/hw/foo_AIDL_tester performance"
```

o programa deve retorna a seguinte mensagem: 

>
getScalingGovernor:schedutil <br>
setScalingGovernor:0 <br>
getScalingGovernor:schedutil <br>

Não houve alteração para `performance`, continuou em `schedutil`. Checando o `dmesg`:

```{.py hl_lines="4"}
10-09 06:38:08.315   370   383 I mypackage.mysubpackage.fooAIDL-service: Test AIDL::getTest data=(schedutil
10-09 06:38:08.315   370   383 I mypackage.mysubpackage.fooAIDL-service: )
10-09 06:38:08.316   370   383 I mypackage.mysubpackage.fooAIDL-service: Test AIDL::setTest data=(performance)
10-09 06:38:08.311   370   370 W Binder:370_1: type=1400 audit(0.0:6): avc: denied { write } for name="scaling_governor" dev="sysfs" ino=19629 scontext=u:r:hal_fooAIDL_default:s0 tcontext=u:object_r:sysfs_devices_system_cpu:s0 tclass=file permissive=0
10-09 06:38:08.317   370   383 I mypackage.mysubpackage.fooAIDL-service: Test AIDL::getTest data=(schedutil
10-09 06:38:08.317   370   383 I mypackage.mysubpackage.fooAIDL-service: )
```

Para resolver isso, devemos adicionar ao arquivo:

```{title="hal_fooAIDL_default.te" hl_lines="8"}
type hal_fooAIDL_default, domain;
type hal_fooAIDL_default_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(hal_fooAIDL_default);
vndbinder_use(hal_fooAIDL_default);
add_service(hal_fooAIDL_default, fooAIDL_service);
allow hal_fooAIDL_service servicemanager:binder { call transfer };
allow hal_fooAIDL_default sysfs_devices_system_cpu:file write;
```

Rebuildando e executando o comando:

```{.sh}
adb root
adb shell "/vendor/bin/hw/foo_AIDL_tester performance"
```

O programa deve retornar a seguinte mensagem:

>
getScalingGovernor:schedutil <br>
setScalingGovernor:1 <br>
getScalingGovernor:performance <br> 

Agora houve a alteração de `schedutil` ➡️ `performance`.

## Conclusão

Implementar a AIDL é relativamente mais simples que a HIDL. Durante a implementação, percebi que os métodos implementados que sobrepõem os métodos virtuais retornam um valor para o `binder`. Por isso, se quisermos ter alguma informação externada, é preciso passar algum ponteiro para dentro do método, adicionar as informações relevantes e depois acessa-lo externamente ao método. Como ocorre no arquivo `HalTest.cpp`, para as variáveis `a` e `c`.