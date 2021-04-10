**KUnit: Introdução ao framework de testes unitários do kernel Linux - parte 1**

**Introdução**

Em certa ocasião em 2020, trabalhei em um projeto que se tratava de um módulo de segurança que
rodava em *kernel land*, escrito em C. Até aí nenhuma novidade. No entanto, ele tinha um requisito
bem peculiar: deveria ser coberto por testes unitários. Nada demais para aplicações em *user land*,
certo?Porém em *kernel land* a realidade é bem diferente. O Linux até dispõe de ferramentas de teste,
como o *kselftest* (framework para testes funcionais), no entanto, ferramentas de teste unitário ainda
era uma lacuna a ser preenchida. Porém, essa realidade já havia mudado em meados de 2019, sem que eu
tivesse tomado conhecimento. Depois de alguma pesquisa, me deparei com um
[artigo](https://sergioprado.org/como-o-kernel-linux-e-testado/) bem interessante do Sérgio Prado que
falava sobre ferramentas de teste utilizadas no desenvolvimento do Linux e lá ele citava um framework
de testes unitários relativamente novo na comunidade, chamado *KUnit*. Era exatamente o que estava
procurando.

Idealizado por Brendan Higgins, Engenheiro de software do Google, o *KUnit* é um framework
de testes unitários fortemente baseado no GTest/GMock (também do Google) que - ainda que não
amplamente utilizado - veio para preencher uma lacuna existente no processo de desenvolvimento do
Linux: ter a capacidade de se testar pequenas unidades do código de maneira rápida e isolada. A
idéia do artigo é compartilhar um pouco da experiência que tive com essa ferramenta e nesta
primeira parte do artigo irei abordar de maneira introdutória o uso do KUnit, cobrindo o setup do
ambiente e a criação de uma suíte de testes simples.

**Arquitetura**

TODO

**Preparando o ambiente**

Um ponto importante a ser ressaltado é que o *KUnit* não é um framework standalone. Ele está
embutido na árvore do Linux, logo precisamos dela localmente:

```bash
$ git clone https://kunit.googlesource.com/linux --branch kunit/release/4.19/0.7
```

> **NOTA**: Neste artigo iremos utilizar como base a última release do KUnit que está disponível no
> fork do Linux mantido pelo Google. Isso porque, até o momento que escrevo este artigo, o KUnit
> se encontra parcialmente integrado na mainline do Linux. Sendo assim algumas funcionalidades que
> iremos exemplificar aqui ainda não foram mergeadas.

Juntamente com a árvore localmente na nossa máquina, precisamos instalar as dependências
necessárias:

```bash
$ apt install flex bison ...
```

### Creating a test module

Before we start writing the test module itself, we must have a module to be tested. Let's take [this module](some_link_here) as example. Basically its only work is to emulate the functionalities of a security module, which checks the sanity of a well-defined user process and issue an alarm if such process was tampered by a malicious user (take a look at the file for more details). The module was implemented taking in mind the OO programming concept, as suggested in the [framework documentation](https://kunit.dev/third_party/stable_kernel/docs/usage.html), which is recommended in order to take the maximum of the KUnit's capabilities. A simple test module has the following format:

```c
#include <test/test.h>
#include <test/mock.h>

static void simple_test_case(struct test *test)
{
    test_info(test, "Hello, I'm a test case!\n");

    EXPECT_EQ(test, 2, 1 + 1);
}

static int test_module_init(struct test *test)
{
    test_info(test, "initializing the test module\n");

    return 0;
}

static struct test_case module_test_cases[] = {
    TEST_CASE(simple_test_case),
    {}
};

static struct test_module module_test_suite = {
    .name = "TEST SUITE NAME",
    .init = test_module_init,
    .test_cases = module_test_cases,
};

module_test(module_test_suite);
```

In our case, the implementation of the test module targeted for our example is available [here](some_link_here). Let's take a look at some parts of it in detail:

```c
#include <test/mock.h>

...

DECLARE_STRUCT_CLASS_MOCK_PREREQS(memory_scan);

DEFINE_STRUCT_CLASS_MOCK(
    METHOD(check_lib),
    CLASS(memory_scan),
    RETURNS(int),
    PARAMS(struct memory_scan *, const char *)
);

DEFINE_STRUCT_CLASS_MOCK(
    METHOD(check_cow),
    CLASS(memory_scan),
    RETURNS(int),
    PARAMS(struct memory_scan *, const char *)
);

...
```

In order to limit the amount of code under test to a single unit, it's highly recommended we mock all components that rely on an external agent to do its work. In our case, the memory scanner and the ILO. The code snippet above illustrates the mock definition for the memory scanner structure and its functions ```check_lib``` and ```check_cow```. By using such macros, KUnit will generate fake implementations for both specified methods, which can be used in place of the real ones. For ILO component, we have the same procedure:

```c
...

DECLARE_STRUCT_CLASS_MOCK_PREREQS(ilo_socket);

DEFINE_STRUCT_CLASS_MOCK(
    METHOD(send),
    CLASS(ilo_socket),
    RETURNS(int),
    PARAMS(struct ilo_socket *, const char *)
);

...
```

After we define the mocks for our data structures, we must to initialize our fake objects:

```c
static int memory_scan_init(struct MOCK(memory_scan) *mock_memory_scan)
{
    struct memory_scan *scan = mock_get_trgt(mock_memory_scan);

    scan->check_lib = check_lib;
    scan->check_cow = check_cow;

    return 0;
}

DEFINE_STRUCT_CLASS_MOCK_INIT(memory_scan, memory_scan_init);

static int ilo_sock_init(struct MOCK(ilo_socket) *mock_ilo_sock)
{
    struct ilo_socket *ilo = mock_get_trgt(mock_ilo_sock);

    ilo->send = send;

    return 0;
}

DEFINE_STRUCT_CLASS_MOCK_INIT(ilo_socket, ilo_sock_init);
```

After defining the mocked objects, we are able to use them in our test cases:

```c
static void main_routine_test(struct test *test)
{
    /* constants of test case */
    const char *protected_process = "foo";
    const char *expected_msg = "alert message\n";
    /* mocks definition */
    struct MOCK(memory_scan) *mock_scan = CONSTRUCT_MOCK(memory_scan, test);
    struct MOCK(ilo_socket) *mock_ilo = CONSTRUCT_MOCK(ilo_socket, test);
    /* fake objects */
    struct memory_scan *scan = mock_get_trgt(mock_scan);
    struct ilo_socket *ilo = mock_get_trgt(mock_ilo);
    /* expectations */
    struct mock_expectation *handle_cow, *handle_lib, *handle_ilo_send;

    /* set the expectations for memory scanner */
    handle_cow = EXPECT_CALL(
        check_cow(mock_get_ctrl(mock_scan),
        streq(test, protected_process))
    );
    handle_cow->action = int_return(test, 1); /* defininig the return value of check_cow function */

    handle_lib = EXPECT_CALL(
        check_lib(mock_get_ctrl(mock_scan),
        streq(test, protected_process))
    );
    handle_lib->action = int_return(test, 1); /* defininig the return value of check_lib function */

    /* set the expectations for ilo */
    handle_ilo_send = EXPECT_CALL(
        send(mock_get_ctrl(mock_ilo),
        streq(test, expected_msg))
    );
    handle_ilo_send->action = int_return(test, 1); /* defininig the return value of send function */

    /* execute the action */
    main_routine(scan, ilo, protected_process);
}
```

This test case illustrates a simple example of how we can use the mocking feature to verify the code behavior under a
specific situation.

### Adding the test module to Linux tree

The first step to run the test module is to make it "compilable" in the Linux compilation process. For that it must be placed into the Linux tree, in our case, we will create a specific folder for our code inside the driver's folder, as shown below:

```bash
$ cd $LINUX_ROOT_DIR
$ mkdir drivers/dime && cd $_
$ git clone some_link_here
```

Now we must to add the entry related to our module into the **drivers** Makefile and Kconfig. For that apply the patches below:

```patch
diff --git a/drivers/Makefile b/drivers/Makefile
index 578f469f72fb..44d032c029ca 100644
--- a/drivers/Makefile
+++ b/drivers/Makefile
@@ -186,3 +186,4 @@ obj-$(CONFIG_MULTIPLEXER)   += mux/
 obj-$(CONFIG_UNISYS_VISORBUS)  += visorbus/
 obj-$(CONFIG_SIOX)             += siox/
 obj-$(CONFIG_GNSS)             += gnss/
+obj-$(CONFIG_DIME)              += dime/
```

```patch
diff --git a/drivers/Kconfig b/drivers/Kconfig
index ab4d43923c4d..1563d772f1ba 100644
--- a/drivers/Kconfig
+++ b/drivers/Kconfig
@@ -219,4 +219,6 @@ source "drivers/siox/Kconfig"

 source "drivers/slimbus/Kconfig"

+source "drivers/dime/Kconfig"
+
 endmenu
```

### Setup kunit

The KUnit requires some user configuration to work. Such configuration is located at **kunitconfig** file, placed at the Linux root directory. It contains some kernel configurations which will be used to configure the testing kernel during its compilation process, which is the first step of the KUnit test execution. For this example, the minimum configuration that must be supplied is shown below:

```
CONFIG_TEST=y # To enable KUnit
CONFIG_DIME=y # To enable the module to be tested
CONFIG_DIME_UNIT_TESTS=y # To enable the module test for DIME module
```

### Running the tests

The tests must be executed by running the **kunit.py** script:

```bash
$ cd linux
$ ./tools/testing/kunit/kunit.py run
```

And the expected result must be something like that:

```
[11:27:21] Building KUnit Kernel ...
[11:27:28] Starting KUnit Kernel ...
[11:27:28] ==============================
[11:27:28] [PASSED] dime:protected_app_with_lib_injection_test
[11:27:28] [PASSED] dime:protected_app_with_cow_test
[11:27:28] [PASSED] dime:protected_app_with_lib_injection_and_cow_test
[11:27:28] [PASSED] dime:protected_app_with_no_injection_test
[11:27:28] ==============================
[11:27:28] Testing complete. 4 tests run. 0 failed. 0 crashed.
[11:27:28] Elapsed time: 7.417s total, 0.001s configuring, 7.164s building, 0.252s running.
```

The logs produced during the suite execution can be visualized in the produced log file
**test.log**, which can contain some useful information:

```
...
kunit dime: initializing dime tests
kunit dime: this is a log message from protected_app_with_lib_injection_test
kunit dime: protected_app_with_lib_injection_test passed
kunit dime: initializing dime tests
kunit dime: this is a log message from protected_app_with_cow_test
kunit dime: protected_app_with_cow_test passed
kunit dime: initializing dime tests
kunit dime: this is a log message from protected_app_with_lib_injection_and_cow_test
kunit dime: protected_app_with_lib_injection_and_cow_test passed
kunit dime: initializing dime tests
kunit dime: this is a log message from protected_app_with_no_injection_test
kunit dime: protected_app_with_no_injection_test passed
kunit dime: initializing dime tests
kunit dime: this is a log message from main_routine_test
kunit dime: all tests passed
...
```

### Useful links

- https://kunit.dev/third_party/stable_kernel/docs
