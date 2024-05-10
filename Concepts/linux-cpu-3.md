Механизм initcall
================================================================================

Вступление
--------------------------------------------------------------------------------

Как вы можете понять из названия, в этой части будет рассмотрена интересная и важная концепция ядра Linux, которая называется `initcall`. Мы уже видели такие определения:

```C
early_param("debug", debug_kernel);
```

или

```C
arch_initcall(init_pit_clocksource);
```

в некоторых частях ядра Linux. Прежде чем мы увидим, как этот механизм реализован в ядре Linux, мы должны знать, что это такое и как его использует ядро Linux. Подобные определения представляют собой функцию [обратного вызова](https://en.wikipedia.org/wiki/Callback_%28computer_programming%29), которая будет вызываться сразу после инициализации ядра Linux. Собственно, основной задачей механизма `initcall` является определение правильного порядка инициализации встроенных модулей и подсистем. Например, давайте посмотрим на следующую функцию:

```C
static int __init nmi_warning_debugfs(void)
{
    debugfs_create_u64("nmi_longest_ns", 0644,
                       arch_debugfs_dir, &nmi_longest_ns);
    return 0;
}
```

из файла исходного кода [arch/x86/kernel/nmi.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/nmi.c). Как мы видим, он просто создает файл `nmi_longest_ns` [debugfs](https://en.wikipedia.org/wiki/Debugfs) в каталоге `arch_debugfs_dir`. На самом деле этот файл `debugfs` может быть создан только после того, как будет создан `arch_debugfs_dir`. Создание этого каталога происходит во время инициализации ядра Linux в зависимости от архитектуры. На самом деле этот каталог будет создан функцией `arch_kdebugfs_init` из файла [arch/x86/kernel/kdebugfs.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/kdebugfs. в) файл исходного кода. Обратите внимание, что функция `arch_kdebugfs_init` также помечена как `initcall`:

```C
arch_initcall(arch_kdebugfs_init);
```

Ядро Linux вызывает все специфичные для архитектуры `initcalls` перед `initcalls`, связанными с `fs`. Итак, наш файл `nmi_longest_ns` будет создан только после создания каталога `arch_kdebugfs_dir`. На самом деле ядро Linux предоставляет восемь уровней основных `initcalls` вызовов:

* `early`;
* `core`;
* `postcore`;
* `arch`;
* `subsys`;
* `fs`;
* `device`;
* `late`.

Все их имена представлены массивом `initcall_level_names`, который определен в файле исходного кода [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c):

```C
static char *initcall_level_names[] __initdata = {
	"early",
	"core",
	"postcore",
	"arch",
	"subsys",
	"fs",
	"device",
	"late",
};
```

Все функции, помеченные этими идентификаторами как `initcall`, будут вызываться в том же порядке, либо сначала будут вызываться `ранние initcall`, потом `core initcalls` и т.д. С этого момента мы немного знаем о `initcall` механизме, поэтому мы можем начать погружаться в исходный код ядра Linux, чтобы увидеть, как реализован этот механизм.

Реализация механизма initcall в ядре Linux
--------------------------------------------------------------------------------

Ядро Linux предоставляет набор макросов из заголовочного файла [include/linux/init.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init.h) для маркировки заданного функционировать как `initcall`. Все эти макросы довольно просты:

```C
#define early_initcall(fn)		__define_initcall(fn, early)
#define core_initcall(fn)		__define_initcall(fn, 1)
#define postcore_initcall(fn)		__define_initcall(fn, 2)
#define arch_initcall(fn)		__define_initcall(fn, 3)
#define subsys_initcall(fn)		__define_initcall(fn, 4)
#define fs_initcall(fn)			__define_initcall(fn, 5)
#define device_initcall(fn)		__define_initcall(fn, 6)
#define late_initcall(fn)		__define_initcall(fn, 7)
```

и, как мы видим, эти макросы просто расширяются до вызова макроса `__define_initcall` из того же заголовочного файла. Более того, макрос `__define_initcall` принимает два аргумента:

* `fn` - функция обратного вызова, которая будет вызываться при вызове `initcalls` определенного уровня;
* `id` — идентификатор для идентификации `initcall`, чтобы предотвратить ошибку, когда два одинаковых `initcall` указывают на один и тот же обработчик.

Реализация макроса `__define_initcall` выглядит так:

```C
#define __define_initcall(fn, id) \
	static initcall_t __initcall_##fn##id __used \
	__attribute__((__section__(".initcall" #id ".init"))) = fn; \
	LTO_REFERENCE_INITCALL(__initcall_##fn##id)
```

Чтобы понять макрос `__define_initcall`, прежде всего давайте посмотрим на тип `initcall_t`. Этот тип определен в том же файле [header]() и представляет собой указатель на функцию, которая возвращает указатель на [integer](https://en.wikipedia.org/wiki/Integer), который будет результатом `initcall `:

```C
typedef int (*initcall_t)(void);
```

Теперь вернемся к макросу `_-define_initcall`. [##](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html) обеспечивает возможность объединения двух символов. В нашем случае первая строка макроса `__define_initcall` создает определение данной функции, которая находится в `.initcall id .init` [раздел ELF](http://www.skyfree.org/linux/references/ ELF_Format.pdf) и отмечен следующими атрибутами [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection): `__initcall_function_name_id` и `__used`. Если мы посмотрим в заголовочный файл [include/asm-generic/vmlinux.lds.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/asm-generic/vmlinux.lds.h) который представляет данные для скрипта ядра [linker](https://en.wikipedia.org/wiki/Linker_%28computing%29), мы увидим, что все разделы `initcalls` будут помещены в раздел `.data` :

```C
#define INIT_CALLS					\
		VMLINUX_SYMBOL(__initcall_start) = .;	\
		*(.initcallearly.init)					\
		INIT_CALLS_LEVEL(0)					    \
		INIT_CALLS_LEVEL(1)					    \
		INIT_CALLS_LEVEL(2)					    \
		INIT_CALLS_LEVEL(3)					    \
		INIT_CALLS_LEVEL(4)					    \
		INIT_CALLS_LEVEL(5)					    \
		INIT_CALLS_LEVEL(rootfs)				\
		INIT_CALLS_LEVEL(6)					    \
		INIT_CALLS_LEVEL(7)					    \
		VMLINUX_SYMBOL(__initcall_end) = .;

#define INIT_DATA_SECTION(initsetup_align)	\
	.init.data : AT(ADDR(.init.data) - LOAD_OFFSET) {	   \
        ...                                                \
        INIT_CALLS						                   \
        ...                                                \
	}

```

Второй атрибут — `__used` определен в заголовке [include/linux/compiler-gcc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/compiler-gcc.h). файл и расширяется до определения следующего атрибута `gcc`:

```C
#define __used   __attribute__((__used__))
```

что предотвращает предупреждение `переменная определена, но не используется`. Последняя строка макроса `__define_initcall`:

```C
LTO_REFERENCE_INITCALL(__initcall_##fn##id)
```

зависит от параметра конфигурации ядра `CONFIG_LTO` и просто предоставляет заглушку для компилятора [оптимизация времени компоновки](https://gcc.gnu.org/wiki/LinkTimeOptimization):

```
#ifdef CONFIG_LTO
#define LTO_REFERENCE_INITCALL(x) \
        static __used __exit void *reference_##x(void)  \
        {                                               \
                return &x;                              \
        }
#else
#define LTO_REFERENCE_INITCALL(x)
#endif
```

Чтобы предотвратить проблемы, связанные с отсутствием ссылки на переменную в модуле, она будет перенесена в конец программы. Это все, что касается макроса `__define_initcall`. Таким образом, все макросы `*_initcall` будут расширены во время компиляции ядра Linux, и все `initcalls` будут помещены в свои разделы, и все они будут доступны из раздела `.data`, и ядро Linux будет знать, где найти определенный `initcall`, чтобы вызвать его во время процесса инициализации.

Поскольку `initcalls` может вызываться ядром Linux, давайте посмотрим, как ядро Linux это делает. Этот процесс запускается функцией `do_basic_setup` из файла исходного кода [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c):

```C
static void __init do_basic_setup(void)
{
    ...
    ...
    ...
   	do_initcalls();
    ...
    ...
    ...
}
```

который вызывается во время инициализации ядра Linux, сразу после основных этапов инициализации, таких как инициализация, связанная с диспетчером памяти, подсистема `ЦП` и другие, уже завершенные. Функция `do_initcalls` просто проходит через массив уровней `initcall` и вызывает функцию `do_initcall_level` для каждого уровня:

```C
static void __init do_initcalls(void)
{
	int level;

	for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
		do_initcall_level(level);
}
```

Массив `initcall_levels` определен в том же исходном коде [файл](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) и содержит указатели на разделы, которые были определены в макрос `__define_initcall`:

```C
static initcall_t *initcall_levels[] __initdata = {
	__initcall0_start,
	__initcall1_start,
	__initcall2_start,
	__initcall3_start,
	__initcall4_start,
	__initcall5_start,
	__initcall6_start,
	__initcall7_start,
	__initcall_end,
};
```

Если вам интересно, вы можете найти эти разделы в скрипте компоновщика `arch/x86/kernel/vmlinux.lds`, который генерируется после компиляции ядра Linux:

```
.init.data : AT(ADDR(.init.data) - 0xffffffff80000000) {
    ...
    ...
    ...
    ...
    __initcall_start = .;
    *(.initcallearly.init)
    __initcall0_start = .;
    *(.initcall0.init)
    *(.initcall0s.init)
    __initcall1_start = .;
    ...
    ...
}
```

Если вы не знакомы с этим, вы можете узнать больше о [линкерах](https://en.wikipedia.org/wiki/Linker_%28computing%29) в специальной [части](https://github.com/proninyaroslav/linux-insides-ru/blob/master/Misc/linux-misc-3.md) этой книги.

Как мы только что увидели, функция `do_initcall_level` принимает один параметр - уровень initcall и выполняет две следующие задачи: Во-первых, эта функция анализирует `initcall_command_line`, который является копией обычной [командной строки](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.rst) ядра, которая может содержать параметры для модулей с помощью функции `parse_args` из исходного кода файла [kernel/params.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/params.c) и вызывает функцию `do_on_initcall` для каждого уровня.

```C
for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
		do_one_initcall(*fn);
```

`do_on_initcall` выполняет за нас основную работу. Как мы видим, эта функция принимает один параметр, который представляет функцию обратного вызова `initcall`, и выполняет вызов данного обратного вызова:

```C
int __init_or_module do_one_initcall(initcall_t fn)
{
	int count = preempt_count();
	int ret;
	char msgbuf[64];

	if (initcall_blacklisted(fn))
		return -EPERM;

	if (initcall_debug)
		ret = do_one_initcall_debug(fn);
	else
		ret = fn();

	msgbuf[0] = 0;

	if (preempt_count() != count) {
		sprintf(msgbuf, "preemption imbalance ");
		preempt_count_set(count);
	}
	if (irqs_disabled()) {
		strlcat(msgbuf, "disabled interrupts ", sizeof(msgbuf));
		local_irq_enable();
	}
	WARN(msgbuf[0], "initcall %pF returned with %s\n", fn, msgbuf);

	return ret;
}
```

Давайте попробуем понять, что делает функция `do_on_initcall`. Прежде всего мы увеличиваем счетчик [preemption](https://en.wikipedia.org/wiki/Preemption_%28computing%29), чтобы позже проверить его и убедиться, что он не несбалансирован. После этого шага мы можем увидеть вызов функции `initcall_backlist`, которая проходит по списку `blacklisted_initcalls`, в котором хранятся занесенные в черный список `initcalls`, и освобождает данный `initcall`, если он находится в этом списке:

```C
list_for_each_entry(entry, &blacklisted_initcalls, next) {
	if (!strcmp(fn_name, entry->buf)) {
		pr_debug("initcall %s blacklisted\n", fn_name);
		kfree(fn_name);
		return true;
	}
}
```

Занесенные в черный список `initcalls` хранятся в списке `blacklisted_initcalls`, и этот список заполняется во время ранней инициализации ядра Linux из командной строки ядра Linux.

После того как `initcalls`, занесенные в черный список, будут обработаны, следующая часть кода выполняет непосредственный вызов `initcall`:

```C
if (initcall_debug)
	ret = do_one_initcall_debug(fn);
else
	ret = fn();
```

В зависимости от значения переменной `initcall_debug` функция `do_one_initcall_debug` будет вызывать `initcall` или эта функция сделает это напрямую через `fn()`. Переменная `initcall_debug` определена в [том же](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) файле исходного кода:

```C
bool initcall_debug;
```

и предоставляет возможность распечатать некоторую информацию в ядре [журнала](https://en.wikipedia.org/wiki/Dmesg). Значение переменной можно установить из команд ядра через параметр `initcall_debug`. Как мы можем прочитать из [документации](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.rst) командной строки ядра Linux:

```
initcall_debug	[KNL] Trace initcalls as they are executed.  Useful
                      for working out where the kernel is dying during
                      startup.
```

И это правда. Если мы посмотрим на реализацию функции `do_one_initcall_debug`, мы увидим, что она делает то же самое, что и функция `do_one_initcall` или, то есть, `do_one_initcall` функция `do_one_initcall_debug` вызывает заданный `initcall` и печатает некоторую информацию (например, [pid](https://en.wikipedia.org/wiki/Process_identifier) текущей выполняющейся задачи, продолжительность выполнения `initcall` и т. д.), связанное с выполнением данного `initcall`:

```C
static int __init_or_module do_one_initcall_debug(initcall_t fn)
{
	ktime_t calltime, delta, rettime;
	unsigned long long duration;
	int ret;

	printk(KERN_DEBUG "calling  %pF @ %i\n", fn, task_pid_nr(current));
	calltime = ktime_get();
	ret = fn();
	rettime = ktime_get();
	delta = ktime_sub(rettime, calltime);
	duration = (unsigned long long) ktime_to_ns(delta) >> 10;
	printk(KERN_DEBUG "initcall %pF returned %d after %lld usecs\n",
		 fn, ret, duration);

	return ret;
}
```

Поскольку `initcall` был вызван одной из функций ` do_one_initcall` или `do_one_initcall_debug`, мы можем увидеть две проверки в конце функции `do_one_initcall`. Первый проверяет количество возможностей.
ble `__preempt_count_add` и `__preempt_count_sub` вызывает внутри выполненного initcall, и если это значение не равно предыдущему значению вытесняемого счетчика, мы добавляем строку ``preempt_count_add`` в буфер сообщений и устанавливаем правильное значение вытесняемого счетчика:

```C
if (preempt_count() != count) {
	sprintf(msgbuf, "preemption imbalance ");
	preempt_count_set(count);
}
```

Позже эта строка ошибки будет напечатана. Последние проверяют состояние локальных [IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29), и если они отключены, мы добавляем строки `отключенных прерываний` в наш буфер сообщений и включаем `IRQ` для текущего процессора, чтобы предотвратить состояние, когда `IRQ` были отключены `initcall` и больше не включались:

```C
if (irqs_disabled()) {
	strlcat(msgbuf, "disabled interrupts ", sizeof(msgbuf));
	local_irq_enable();
}
```

Вот и все. Таким образом, ядро Linux выполняет инициализацию многих подсистем в правильном порядке. С этого момента мы знаем, что такое механизм `initcall` в ядре Linux. В этой части мы рассмотрели основную часть механизма `initcall`, но оставили некоторые важные понятия. Давайте кратко рассмотрим эти понятия.

Прежде всего, мы пропустили один уровень `initcalls`, это `rootfs initcalls`. Вы можете найти определение `rootfs_initcall` в заголовочном файле [include/linux/init.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init.h) вместе со всеми аналогичные макросы, которые мы видели в этой части:

```C
#define rootfs_initcall(fn)		__define_initcall(fn, rootfs)
```

Как мы можем понять из названия макроса, его основная цель — хранить обратные вызовы, связанные с [rootfs](https://en.wikipedia.org/wiki/Initramfs). Помимо этой цели, может быть полезно инициализировать другие элементы после инициализации, связанные с уровнем файловых систем, только если элементы, связанные с устройствами, не инициализированы. Например, распаковка [initramfs](https://en.wikipedia.org/wiki/Initramfs), которая произошла в функции `populate_rootfs` из файла [init/initramfs.c](https://github.com /torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/initramfs.c) файл исходного кода:

```C
rootfs_initcall(populate_rootfs);
```

Отсюда мы можем увидеть знакомый вывод:

```
[    0.199960] Unpacking initramfs...
```

Помимо уровня `rootfs_initcall`, существуют дополнительные уровни `console_initcall`, `security_initcall` и другие вторичные уровни `initcall`. Последнее, что мы упустили, это набор уровней `*_initcall_sync`. Почти каждый макрос `*_initcall`, который мы видели в этой части, имеет макрос-компаньон с префиксом `_sync`:

```C
#define core_initcall_sync(fn)		__define_initcall(fn, 1s)
#define postcore_initcall_sync(fn)	__define_initcall(fn, 2s)
#define arch_initcall_sync(fn)		__define_initcall(fn, 3s)
#define subsys_initcall_sync(fn)	__define_initcall(fn, 4s)
#define fs_initcall_sync(fn)		__define_initcall(fn, 5s)
#define device_initcall_sync(fn)	__define_initcall(fn, 6s)
#define late_initcall_sync(fn)		__define_initcall(fn, 7s)
```

Основная цель этих дополнительных уровней — дождаться завершения всех процедур инициализации, связанных с модулем, для определенного уровня.

Вот и все.

Заключение
--------------------------------------------------------------------------------

В этой части мы увидели важный механизм ядра Linux, который позволяет вызывать функцию, которая зависит от текущего состояния ядра Linux во время его инициализации.

Ссылки
--------------------------------------------------------------------------------

* [Функция обратного вызова](https://en.wikipedia.org/wiki/Callback_%28computer_programming%29)
* [debugfs](https://en.wikipedia.org/wiki/Debugfs)
* [Целочисленный тип](https://en.wikipedia.org/wiki/Integer)
* [Конкатенация символов](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html)
* [GCC](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)
* [Оптимизация времени ссылки](https://gcc.gnu.org/wiki/LinkTimeOptimization)
* [Вступление в линкеры](https://github.com/proninyaroslav/linux-insides-ru/blob/master/Misc/linux-misc-3.md)
* [Командная строка ядра Linux](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.rst)
* [Идентификатор процесса](https://en.wikipedia.org/wiki/Process_identifier)
* [IRQs](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [rootfs](https://en.wikipedia.org/wiki/Initramfs)
* [Предыдущая часть](https://github.com/proninyaroslav/linux-insides-ru/blob/master/Concepts/linux-cpu-2.md)
