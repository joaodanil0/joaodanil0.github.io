# Implementando a HIDL


<figure markdown>
  ![imagems](images/hild_1.jpg){ width="600" }
  <figcaption>
  Image by <a href="https://pixabay.com/users/methodshop-1460919/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=3404892">methodshop</a> from <a href="https://pixabay.com/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=3404892">Pixabay</a>
  </figcaption>
</figure>

> Baseado no curso: [*Android 12 Internals material, labs and discussion*](https://8mantech.thinkific.com/courses/android-internals)
## Introdução

Umas das coisas mais difíceis que eu encontrei durante o processo de entender o sistema Android, foi sobre *Hardware Abstraction Layer* (HAL). Que é a forma de conectar o *framework* do Android com a parte do *hardware*. Atualmente, a forma mais disseminada de fazer essa conexão é utilizando a *HAL Interface Definition Language* (HIDL). Que irei abordar nesse post.

## Definições

Quando um ==aplicativo A== precisa se comunicar com um ==aplicativo B==, existem duas formas ~~grosseiramente~~ de se fazer isso. A primeira é passando diretamente a região de memória do ==aplicativo A== para o ==aplicativo B==. A segunda é utilizando um mecanismo que gerencie essa troca de informação entre o ==aplicativo A== e o ==aplicativo B==. Essa comunicação entre aplicativos é conhecida como *inter-process communication* (IPC), que utilizam *proxy* e *stub* para fazer a comunicação, como uma espécie de cliente-servidor.

No Android, esse mecanismo é o [`Binder`](https://proandroiddev.com/binders-in-android-part-1-e875daeb762f) e o "protocolo" que esse mecanismo utiliza é a HAL, que por sua vez pode ser implementada com a HIDL. Nesse sistema os equivalentes do *proxy* e *stub* são o *Binder Proxy* (Bp) e *Binder Native* (Bn). Dessa forma, as siglas `Bp` e `Bn` irão aparecer constantemente no processo de implementação da HAL do Android.

> Não confundir com a extensão `.Bp` (Blueprint) que faz referência aos arquivos de *build* do Android.

## Implementação

Vou tentar dividir em alguns tópicos para facilitar.

### HAL

Na pasta raiz do [AOSP](https://source.android.com/), criei o seguinte caminho:

```{.c}
mkdir -p device/casa/emulator/interfaces/foo/1.0
```

A pasta `foo/` é o nome da HAL e a pasta `1.0/` seria a versão dessa HAL. Dentro da pasta `1.0/` criei os arquivos:

```{.mk title="Android.bp"}
hidl_interface {
    name: "mypackage.mysubpackage.foo@1.0",
    root: "mypackage.mysubpackage",
    srcs: [
        "ITest.hal",
    ],
    interfaces: [
        "android.hidl.base@1.0",
    ],
    gen_java: true,
    product_specific: true,
}
```

```{.hal title="ITest.hal"}
package mypackage.mysubpackage.foo@1.0;

interface ITest {
    getTest() generates (string res);
    setTest(string param) generates (bool res);
};
```

Algumas coisas que podem causar confusão e geralmente dão problema de compilação são os nomes. Nesse caso, no arquivo ==Android.bp== a tag `root` é um prefixo para a tag `name` (`mypackage.mysubpackage`), que é seguida do nome da HAL (foo) com sua versão (1.0). No arquivo ==ITest.hal==, o `package` deve ser o mesmo nome da tag `name` do ==Android.bp==.

Agora dentro da pasta `interfaces/` (`device/casa/emulator/interfaces`), criei outro arquivo:

```{.mk title="Android.bp"}
hidl_package_root {
    name: "mypackage.mysubpackage",
    path: "device/casa/emulator/interfaces",
}
```

A tag `name` desse arquivo é igual ao da tag `root` do outro arquivo ==Android.bp==. A tag `path` é o caminho relativo da pasta raiz do AOSP até a pasta `interfaces`.

Finalmente, dentro da pasta `device/casa/emulator/interfaces/foo/1.0` o seguinte comando foi usado:

```{.sh}
mm -j8
```

Ele compila a HAL foo. Ao final do processo uma mensagem similar deve aparecer:

>
============================================ <br>
[100% 149/149] Copy: out/target/product/emulator/data/nativetest/product/mypackage.mysubpackage.foo@1.0-adapter/mypackage.mysubpackage.foo@1.0.so.toc<br>
build completed successfully (7 seconds) #### <br>

Os arquivos gerados estão dentro do diretório:

> out/soong/.intermediates/device/casa/emulator/interfaces/foo/1.0

No curso *Android 12 Internals* é dado mais detalhes sobre alguns arquivos dentro dessa pasta.

Por fim, é necessário adicionar a HAL foo ao final do produto, no meu caso, ==emulator.mk== ( `device/casa/emulator/emulator.mk`), da seguinte forma:

```{.mk title="emulator.mk"}
...
PRODUCT_PACKAGES += \
    mypackage.mysubpackage.foo@1.0
```

Agora é só rebuildar o AOSP `m -j8`.

Após o build terminar, na pasta do produto dentro da pasta `out/` é possível ver onde a HAL foi adicionada:

>
./vendor/lib/mypackage.mysubpackage.foo@1.0.so<br>
./vendor/lib/mypackage.mysubpackage.foo@1.0-adapter-helper.so<br>
./vendor/lib64/mypackage.mysubpackage.foo@1.0.so<br>
./vendor/lib64/mypackage.mysubpackage.foo@1.0-adapter-helper.so<br>
>
./product/lib/mypackage.mysubpackage.foo@1.0.so<br>
./product/lib/mypackage.mysubpackage.foo@1.0-adapter-helper.so<br>
./product/lib64/mypackage.mysubpackage.foo@1.0.so<br>
./product/lib64/mypackage.mysubpackage.foo@1.0-adapter-helper.so<br>
./product/framework/mypackage.mysubpackage.foo-V1.0-java.jar<br>
./product/framework/mypackage.mysubpackage.foo-V1.0-java-shallow.jar<br>

Serão adicionados na imagem `vendor.img` e `product.img`.

### Serviço

Antes de criar o serviço, precisamos implementar o que será processado dentro das duas funções que definidas no ==ITest.hal==. Quando compilamos a HAL no passo anterior, o Android criou 2 métodos virtuais dentro do arquivo: 

` 
out/soong/.intermediates/device/casa/emulator/interfaces/foo/1.0/mypackage.mysubpackage.foo@1.0_genc++_headers/gen/mypackage/mysubpackage/foo/1.0/ITest.h
`

Os métodos são:

```{.cpp title="ITest.h"}
/**
* Return callback for getTest
*/
using getTest_cb = std::function<void(const ::android::hardware::hidl_string& res)>;

virtual ::android::hardware::Return<void> getTest(getTest_cb _hidl_cb) = 0;
virtual ::android::hardware::Return<bool> setTest(const ::android::hardware::hidl_string& param) = 0;
```

Para fazer a implementação desses métodos, criei dentro da pasta `device/casa/emulator/interfaces/foo/1.0/default` os arquivos:

```{.cpp title="Test.h"}
#ifndef MYPACKAGE_MYSUBPACKAGE_FOO_V1_0_ITEST_H
#define MYPACKAGE_MYSUBPACKAGE_FOO_V1_0_ITEST_H

// Mesmo caminho após:
// mypackage.mysubpackage.foo@1.0_genc++_headers/gen/
// dentro da pasta out/
#include <mypackage/mysubpackage/foo/1.0/ITest.h>

namespace mypackage {
namespace mysubpackage {
namespace foo {
namespace V1_0 {
namespace implementation {


using ::android::hardware::hidl_string; // const hidl_string
using ::android::hardware::Return; // Return<void>
using ::android::hardware::Void; // return Void();
using ::mypackage::mysubpackage::foo::V1_0::ITest; // public ITest

class Test : public ITest {
    public:
        Return<bool> setTest(const hidl_string& param) override;
        Return<void> getTest(getTest_cb _hidl_c) override;
};

extern "C" ITest* HIDL_FETCH_ITest(const char* name);

} // namespace implementation
} // namespace V1_0 
} // namespace foo
} // namespace mysubpackage
} // namespace mypackage

#endif
``` 

```{.cpp title="Test.cpp"}
#include "Test.h"

#include <utils/Log.h> //ALOGI e ALOGE
#include <sys/stat.h> // struct stat info


namespace mypackage {
namespace mysubpackage {
namespace foo {
namespace V1_0 {
namespace implementation {

// conservative|powersave|performance|schedutil
static const char SCALING_GOVERNOR[] = "/sys/devices/system/cpu/cpu0/cpufreq/scaling_governor";

static int write_value(const char *file, const char *value)
{
    int to_write, written, ret, fd;

    fd = TEMP_FAILURE_RETRY(open(file, O_WRONLY));
    if (fd < 0) {
	ALOGE("write_value failed to open");
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

Return<bool> Test::setTest(const hidl_string& param) {
    ALOGI("Test::setTest data=(%s)", param.c_str());
    return write_value(SCALING_GOVERNOR, param.c_str()) == 0;
}

Return<void> Test::getTest(getTest_cb _hidl_cb) {
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
        _hidl_cb("error can open fd");
        return Void();
    }

    fstat(fd, &info);
    size = info.st_size;
    data = malloc(size);
    if (data == NULL) {
        _hidl_cb("error can't malloc");
        close(fd);
        free(data);
        return Void();
    }

    ret = read(fd, data, size);
    if (ret < 0) {
    	_hidl_cb("error reading fd");
	    close(fd);
        free(data);
        return Void();
    }

    snprintf(str, sizeof(str), "%s", (const unsigned char*)data);
    ALOGI("Test:getTest data=(%s)", str);
    _hidl_cb(str);
}

// Methods from ::android::hidl::base::V1_0::IBase follow.
ITest* HIDL_FETCH_ITest(const char* /* name */) {
    return new Test();
}

} // namespace implementation
} // namespace V1_0 
} // namespace foo
} // namespace mysubpackage
} // namespace mypackage
```

Fiz praticamente o mesmo código do curso, ele é capaz de alterar de checar o escalonador do processador. A ideia aqui não é explicar o código, e sim só implementar alguma coisa nos 2 métodos. Voce pode implementar qualquer outra coisa.

Uma vez que o código foi implementado, precisamos criar o serviço que deixará o mesmo disponível. Dentro da mesma pasta foi criado o arquivo:

```{.cpp title="service.cpp"}

#define LOG_TAG "mypackage.mysubpackage.foo@1.0-service"

#include <mypackage/mysubpackage/foo/1.0/ITest.h>

#include <log/log.h>
#include <hidl/HidlTransportSupport.h>

#include "Test.h"

using android::sp;
using android::status_t;
using android::OK;

// libhwbinder:
using android::hardware::configureRpcThreadpool;
using android::hardware::joinRpcThreadpool;

// Generated HIDL files
using mypackage::mysubpackage::foo::V1_0::ITest;
using mypackage::mysubpackage::foo::V1_0::implementation::Test;

using namespace mypackage;

int main(int /* argc */, char** /* argv */) {
    ALOGI("Foo Service 1.0 for test is starting.");

    // Android Strong Pointer (don't GC until exit)
    sp<ITest> service = new Test();
    if (service == nullptr) {
        ALOGE("Can not create an instance of ITest HAL, exiting.");
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
        ALOGE("Could not register service for ITest HAL (%d), exiting.", status);
        return 1;
    }
    ALOGI("Test Service is ready");

    // system/libhidl/transport/include/hidl/HidlTransportSupport.h
    // Joins a threadpool that you configured earlier
    joinRpcThreadpool();

    // In normal operation, we don't expect the thread pool to exit
    ALOGE("Test Service is shutting down");
    return 1;
}
```

De modo geral, a implementação é a mesma. Só precisa ficar atento ao nome da HAL, nesse caso `Test`.

Agora é necessário criar o arquivo:

```{.mk title="Android.bp"}
cc_binary {
    relative_install_path: "hw",
    name: "mypackage.mysubpackage.foo@1.0-service",
    init_rc: ["mypackage.mysubpackage.foo@1.0-service.rc"],
    srcs: ["service.cpp", "Test.cpp"],
    vendor: true,
    // system_ext_specific: true,
    vintf_fragments: ["mypackage.mysubpackage.foo@1.0-service.xml"],

    shared_libs: [
        "libhidlbase",
        "liblog",
        "libutils",
        "libhardware",
        "mypackage.mysubpackage.foo@1.0",
    ],
}
```

Precisamos agora criar 2 arquivos ==mypackage.mysubpackage.foo@1.0-service.rc== e ==mypackage.mysubpackage.foo@1.0-service.xml==, referente as tags `init_rc` e  `vintf_fragments` respectivamente.

```{.rc title="mypackage.mysubpackage.foo@1.0-service.rc"}
service mysubpackage_foo_hal_service /vendor/bin/hw/mypackage.mysubpackage.foo@1.0-service
        interface mypackage.mysubpackage.foo@1.0::ITest default
        class hal
        user system
        group system
```
 
```{.xml title="mypackage.mysubpackage.foo@1.0-service.xml"}
<manifest version="1.0" type="device">
    <hal format="hidl">
        <name>mypackage.mysubpackage.foo</name>
        <transport>hwbinder</transport>
        <version>1.0</version>
        <interface>
            <name>ITest</name>
            <instance>default</instance>
        </interface>
        <fqname>@1.0::ITest/default</fqname>
    </hal>
</manifest>
```

Agora é só fazer um rebuild `m -j8`.

Podemos checar dentro da pasta `/out/target/product/emulator/vendor` os seguintes arquivos após o build:

> 
./bin/hw/mypackage.mysubpackage.foo@1.0-service<br>
./lib/mypackage.mysubpackage.foo@1.0.so<br>
./lib/mypackage.mysubpackage.foo@1.0-adapter-helper.so<br>
./lib64/mypackage.mysubpackage.foo@1.0.so<br>
./lib64/mypackage.mysubpackage.foo@1.0-adapter-helper.so<br>
./etc/init/mypackage.mysubpackage.foo@1.0-service.rc<br>
./etc/vintf/manifest/mypackage.mysubpackage.foo@1.0-service.xml<br>


### Permissão

Iniciando dispositivo, e capturando o log do `dmesg`, é possível encontrar a seguinte mensagem:

```{.py}
10-02 11:57:23.566     0     0 I init    : Parsing file /vendor/etc/init/mypackage.mysubpackage.foo@1.0-service.rc...
10-02 11:57:27.172     0     0 E init    : Could not start service 'mysubpackage_foo_hal_service' as part of class 'hal': File /vendor/bin/hw/mypackage.mysubpackage.foo@1.0-service(labeled "u:object_r:vendor_file:s0") has incorrect label or no domain transition from u:r:init:s0 to another SELinux domain defined. Have you configured your service correctly? https://source.android.com/security/selinux/device-policy#label_new_services_and_address_denials. Note: this error shows up even in permissive mode in order to make auditing denials possible.
```

Esse erro é referente a falta de permissão que nosso serviço possui. Vamos fazer as devidas configurações.

Criei a pasta ==Sepolicy== dentro do produto:

```{.sh}
mkdir -p device/casa/emulator/sepolicy
```

Dentro da pasta foi criado os arquivos:

```{.c title="file_contexts"}
/vendor/bin/hw/mypackage\.mysubpackage\.foo@1\.0-service u:object_r:hal_mysubpackage_default_exec:s0
```

```{.c title="hal_mysubpackage_default.te"}
type hal_mysubpackage_default, domain;
type hal_mysubpackage_default_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(hal_mysubpackage_default);
```

Além disso, é necessário adicionar a pasta na compilação do AOSP. Isso é feito dentro do ==BoardConfig.mk== do produto, no meu caso `device/casa/emulator/BoardConfig.mk`:

```{.mk title="BoardConfig.mk"}
...
BOARD_SEPOLICY_DIRS += device/casa/emulator/sepolicy
```

Agora é só recompilar o AOSP.

Após o build, capturando o log do `dmesg`, podemos encontrar a seguinte mensagem:

```{.py}
10-02 14:30:53.735   370   370 W mypackage.mysub: type=1400 audit(0.0:1020): avc: denied { read } for name="u:object_r:hwservicemanager_prop:s0" dev="tmpfs" ino=168 scontext=u:r:hal_mysubpackage_default:s0 tcontext=u:object_r:hwservicemanager_prop:s0 tclass=file permissive=0
```

Essa mensagem é referente ao *SELinux audit messages* e para resolve-las é necessario criar as *SELinux allow rules*. Para isso, pode se usar o `policycoreutils`, que pode ser instalado com o comando:

```{.sh}
sudo apt install policycoreutils-python-utils
```

Agora precisamos baixar o arquivo ==policy== do device, para isso é usado o comando:

```{.sh}
adb pull /sys/fs/selinux/policy
```

Na mesma pasta onde foi baixado o arquivo ==policy==, foi usado o seguinte comando:

```{.sh}
adb logcat -b all -d | audit2allow -p policy
```

Foi retornado a seguinte mensagem:

> 
============= hal_mysubpackage_default ============== <br>
allow hal_mysubpackage_default hwservicemanager_prop:file read;

Essa mensagem diz que a HAL precisa de permissão de leitura em `hwservicemanager_prop`. Para fazer isso, adicionamos a seguinte linha no arquivo:

```{.c title="hal_mysubpackage_default.te" hl_lines="6-7"}
type hal_mysubpackage_default, domain;
type hal_mysubpackage_default_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(hal_mysubpackage_default);

# allow hal_mysubpackage_default hwservicemanager_prop:file read;
get_prop(hal_mysubpackage_default, hwservicemanager_prop); 
```

Rebuildando e checando o `dmesg`, agora encontramos as seguinte mensagem:

```{.py}
10-02 14:53:10.769     0     0 I init    : starting service 'mysubpackage_foo_hal_service'...
10-02 14:53:11.162  1855  1855 I mypackage.mysubpackage.foo@1.0-service: Foo Service 1.0 for test is starting.
10-02 14:53:11.163  1855  1855 E HidlServiceManagement: Could not get transport for mypackage.mysubpackage.foo@1.0::ITest/default: Status(EX_TRANSACTION_FAILED): 'FAILED_TRANSACTION: '
10-02 14:53:11.163  1855  1855 E mypackage.mysubpackage.foo@1.0-service: Could not register service for ITest HAL (-2147483648), exiting.
10-02 14:53:11.160  1855  1855 W mypackage.mysub: type=1400 audit(0.0:11): avc: denied { call } for scontext=u:r:hal_mysubpackage_default:s0 tcontext=u:r:hwservicemanager:s0 tclass=binder permissive=0
10-02 14:53:10.783     0     0 I init    : Service 'mysubpackage_foo_hal_service' (pid 1855) exited with status 1
10-02 14:53:10.785     0     0 I init    : Sending signal 9 to service 'mysubpackage_foo_hal_service' (pid 1855) process group...
10-02 14:53:10.786     0     0 E init    : process with updatable components 'mysubpackage_foo_hal_service' exited 4 times in 4 minutes
```

Para resolver, vamos novamente usar os seguintes comandos:

```{.sh}
adb pull /sys/fs/selinux/policy
adb logcat -b all -d | audit2allow -p policy
```

Ele retorna a seguinte mensagem:

>
============= hal_mysubpackage_default ==============<br>
allow hal_mysubpackage_default hwservicemanager:binder call;

Ela informa que é necessário fazer a chamada do `hwservicemanager`. Para isso, adicionamos ao arquivo:

```{.c title="hal_mysubpackage_default.te" hl_lines="9-10"}
type hal_mysubpackage_default, domain;
type hal_mysubpackage_default_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(hal_mysubpackage_default);

# allow hal_mysubpackage_default hwservicemanager_prop:file read;
get_prop(hal_mysubpackage_default, hwservicemanager_prop); 

# allow hal_mysubpackage_default hwservicemanager:binder call;
hwbinder_use(hal_mysubpackage_default);
```

Rebuildando e checando o `dmesg`, temos a seguinte mensagem:

```{.py}
10-02 15:05:28.709     0     0 I init    : starting service 'mysubpackage_foo_hal_service'...
10-02 15:05:29.551  1891  1891 I mypackage.mysubpackage.foo@1.0-service: Foo Service 1.0 for test is starting. 
10-02 15:05:29.551   172   172 E SELinux : avc:  denied  { find } for interface=mypackage.mysubpackage.foo::ITest sid=u:r:hal_mysubpackage_default:s0 pid=1891 scontext=u:r:hal_mysubpackage_default:s0 tcontext=u:object_r:default_android_hwservice:s0 tclass=hwservice_manager permissive=0
10-02 15:05:29.551  1891  1891 E HidlServiceManagement: Service mypackage.mysubpackage.foo@1.0::ITest/default must be in VINTF manifest in order to register/get.
10-02 15:05:29.551  1891  1891 E mypackage.mysubpackage.foo@1.0-service: Could not register service for ITest HAL (-2147483648), exiting.
10-02 15:05:28.722     0     0 I init    : Service 'mysubpackage_foo_hal_service' (pid 1891) exited with status 1
10-02 15:05:28.722     0     0 I init    : Sending signal 9 to service 'mysubpackage_foo_hal_service' (pid 1891) process group...
10-02 15:05:28.723     0     0 E init    : process with updatable components 'mysubpackage_foo_hal_service' exited 4 times in 4 minutes
```

Esse erro ocorre devido ao conflito de regras do `domain`, definido no arquivo ==hal_mysubpackage_default.te==. Para obter mais informações, sugiro checar o curso.

Para resolver, vamos novamente usar os seguintes comandos:

```{.sh}
adb pull /sys/fs/selinux/policy
adb logcat -b all -d | audit2allow -p policy
```

Ele retorna a seguinte mensagem:

>
============= hal_mysubpackage_default ============== <br>
allow hal_mysubpackage_default default_android_hwservice:hwservice_manager find;

Para resolver isso, foi criado dentro da pasta `device/casa/emulator/sepolicy` os arquivos:

```{.c title="hwservice.te"}
type hal_mysubpackage_hwservice, hwservice_manager_type;
```

```{.c title="hwservice_contexts"}
mypackage.mysubpackage.foo::ITest       u:object_r:hal_mysubpackage_hwservice:s0
```

E adicionamos as seguintes linhas ao arquivo:

```{.c title="hal_mysubpackage_default.te" hl_lines="12-13"}
type hal_mysubpackage_default, domain;
type hal_mysubpackage_default_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(hal_mysubpackage_default);

# allow hal_mysubpackage_default hwservicemanager_prop:file read;
get_prop(hal_mysubpackage_default, hwservicemanager_prop); 

# allow hal_mysubpackage_default hwservicemanager:binder call;
hwbinder_use(hal_mysubpackage_default);

# allow hal_mysubpackage_default default_android_hwservice:hwservice_manager find;
add_hwservice(hal_mysubpackage_default, hal_mysubpackage_hwservice);
```


Rebuildando e checando o `dmesg`, temos a seguinte mensagem:

```{.py}
10-02 15:27:27.714     0     0 I init    : Parsing file /vendor/etc/init/mypackage.mysubpackage.foo@1.0-service.rc...
10-02 15:27:35.135     0     0 I init    : starting service 'mysubpackage_foo_hal_service'...
10-02 15:27:35.135   370   370 I mypackage.mysubpackage.foo@1.0-service: Foo Service 1.0 for test is starting.
10-02 15:27:35.141   370   370 I HidlServiceManagement: Registered mypackage.mysubpackage.foo@1.0::ITest/default
10-02 15:27:35.141   370   370 I HidlServiceManagement: Removing namespace from process name mypackage.mysubpackage.foo@1.0-service to foo@1.0-service.
10-02 15:27:35.141   370   370 I mypackage.mysubpackage.foo@1.0-service: Test Service is ready
```
 
Agora nosso serviço está com todas as permissões necessárias. Observe que algumas das mensagens foram implementadas no arquivo ==service.cpp==

### Testando

Para criar um binário para teste, foi criado na pasta `device/casa/emulator/interfaces/foo/1.0/default/halTest` os seguintes arquivos:

```{.cpp title="HalTest.cpp"}
#include <mypackage/mysubpackage/foo/1.0/ITest.h>

#include <hidl/Status.h>
#include <hidl/LegacySupport.h>
#include <utils/misc.h>
#include <hidl/HidlSupport.h>
#include <iostream>
#include <cstdlib>
#include <string>

using ::android::sp;
using ::mypackage::mysubpackage::foo::V1_0::ITest;
using android::hardware::hidl_string;

int main(int argc, char *argv[]) {
    if (argc != 2) {
        std::cout << "USAGE ./cpu_client <conservative|powersave|performance|schedutil>\n";
        exit(0);
    }

    android::sp<ITest> cpuService = ITest::getService();
    if (cpuService == nullptr) {
        std::cout << "Failed to get cpu service\n";
        exit(-1);
    }

    cpuService->getTest([&](hidl_string result) {
        std::cout << "getScalingGovernor:" << result << std::endl;
    });

    bool result = cpuService->setTest(argv[1]);
    std::cout << "setScalingGovernor:" << result << std::endl;

    cpuService->getTest([&](hidl_string result) {
        std::cout << "getScalingGovernor:" << result << std::endl;
    });

    return 0;
}
```
```{.bp title="Android.bp"}
cc_binary {
    relative_install_path: "hw",
    defaults: ["hidl_defaults"],
    name: "subpackage_foo_tester",
    srcs: ["HalTest.cpp"],
    vendor: true,

    shared_libs: [
        "liblog",
        "libhardware",
        "libhidlbase",
        "libutils",
        "mypackage.mysubpackage.foo@1.0",
    ],
}
```

Agora adicionando o binário ao produto, no arquivo:

```{.mk title="emulator.mk" hl_lines="5-6"}
...
PRODUCT_PACKAGES += \
    mypackage.mysubpackage.foo@1.0

PRODUCT_PACKAGES_DEBUG += \
    subpackage_foo_tester
```


Re-buildando o AOSP e executando os seguintes comandos:

```{.sh linenums="1"}
adb root
adb shell
vendor/bin/hw/subpackage_foo_tester performance
```

> O caminho do binário é relativo a `vendor=true` (vendor/), `cc_binary` (bin/) e `relative_install_path` (hw/). O resultado do caminho é `vendor/bin/hw/`.

Checando o `dmesg`, encontramos a seguinte mensagem:

```{.py hl_lines=4}
10-02 21:03:19.861   371   371 I mypackage.mysubpackage.foo@1.0-service: Test:getTest data=(schedutil
10-02 21:03:19.861   371   371 I mypackage.mysubpackage.foo@1.0-service: )
10-02 21:03:19.861   371   371 I mypackage.mysubpackage.foo@1.0-service: Test::setTest data=(performance)
10-02 21:03:19.861   371   371 E mypackage.mysubpackage.foo@1.0-service: write_value failed to open
10-02 21:03:19.861   371   371 I mypackage.mysubpackage.foo@1.0-service: Test:getTest data=(schedutil
10-02 21:03:19.861   371   371 I mypackage.mysubpackage.foo@1.0-service: )
```

Isso se deve porque somente o usuário `root` tem permissão para escrever no caminho `/sys/devices/system/cpu/cpu0/cpufreq/scaling_governor` (definido no arquivo ==Teste.cpp==). Então precisamos dar permissão para o nosso aplicativo, para isso vamos adicionar ao arquivo:

```{.bp title="mypackage.mysubpackage.foo@1.0-service.rc" hl_lines="7-9"}
service mysubpackage_foo_hal_service /vendor/bin/hw/mypackage.mysubpackage.foo@1.0-service
    interface mypackage.mysubpackage.foo@1.0::ITest default
    class hal
    user system
    group system

on boot
    chown system system /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
    chmod 0664 /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

Rebuildando e execuntado o binário de teste, temos a seguinte mensagem :

>
getScalingGovernor:schedutil
>
setScalingGovernor:0<br>
getScalingGovernor:schedutil

Obervar que o `ScalingGovernor` não foi alterado. No log de `dmesg`, temos essa:


```{.py hl_lines="5"}
10-02 21:09:09.854   370   370 I mypackage.mysubpackage.foo@1.0-service: Test:getTest data=(schedutil
10-02 21:09:09.854   370   370 I mypackage.mysubpackage.foo@1.0-service: )
10-02 21:09:09.854   370   370 I mypackage.mysubpackage.foo@1.0-service: Test::setTest data=(performance)
10-02 21:09:09.854   370   370 E mypackage.mysubpackage.foo@1.0-service: write_value failed to open
10-02 21:09:09.853   370   370 W foo@1.0-service: type=1400 audit(0.0:5): avc: denied { write } for name="scaling_governor" dev="sysfs" ino=19629 scontext=u:r:hal_mysubpackage_default:s0 tcontext=u:object_r:sysfs_devices_system_cpu:s0 tclass=file permissive=0
10-02 21:09:09.858   370   370 I mypackage.mysubpackage.foo@1.0-service: Test:getTest data=(schedutil
10-02 21:09:09.858   370   370 I mypackage.mysubpackage.foo@1.0-service: )
```

Como é um problema do `audit`, repetimos os passos:

```{.sh}
adb pull /sys/fs/selinux/policy
adb logcat -b all -d | audit2allow -p policy
```

retornando a mensagem:

>
============= hal_mysubpackage_default ============== <br>
allow hal_mysubpackage_default sysfs_devices_system_cpu:file write;

Foi preciso adicionar ao arquivo:

```{.c title="hal_mysubpackage_default.te" hl_lines="15"}
type hal_mysubpackage_default, domain;
type hal_mysubpackage_default_exec, exec_type, vendor_file_type, file_type;

init_daemon_domain(hal_mysubpackage_default);

# allow hal_mysubpackage_default hwservicemanager_prop:file read;
get_prop(hal_mysubpackage_default, hwservicemanager_prop); 

# allow hal_mysubpackage_default hwservicemanager:binder call;
hwbinder_use(hal_mysubpackage_default);

# allow hal_mysubpackage_default default_android_hwservice:hwservice_manager find;
add_hwservice(hal_mysubpackage_default, hal_mysubpackage_hwservice);

allow hal_mysubpackage_default sysfs_devices_system_cpu:file write;
```

Rebuildando e execuntado o binário de teste, temos a seguinte mensagem :

>
getScalingGovernor:schedutil
>
setScalingGovernor:1<br>
getScalingGovernor:performance

Obervar que o `ScalingGovernor` foi alterado para *performance*. No log de `dmesg`, não encontramos nenhum tipo de erro:

```{.py}
10-02 21:16:46.620   375   375 I mypackage.mysubpackage.foo@1.0-service: Test:getTest data=(schedutil
10-02 21:16:46.620   375   375 I mypackage.mysubpackage.foo@1.0-service: )
10-02 21:16:46.620   375   375 I mypackage.mysubpackage.foo@1.0-service: Test::setTest data=(performance)
10-02 21:16:46.639   375   375 I mypackage.mysubpackage.foo@1.0-service: Test:getTest data=(performance
10-02 21:16:46.639   375   375 I mypackage.mysubpackage.foo@1.0-service: )
```

## Conclusão

Nesse post eu tentei mostrar de forma mais prática, como é implementado uma HIDL. Existe muita teoria envolvida, principalmente na parte de permissões, que são abordadas de uma forma um pouco mais profunda no curso [*Android 12 Internals material, labs and discussion*](https://8mantech.thinkific.com/courses/android-internals). Ao final, fiz um aplicativo para testar a implementação. 

🥇