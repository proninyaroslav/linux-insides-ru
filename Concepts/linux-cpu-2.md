Маски процессора
================================================================================

Вступление
--------------------------------------------------------------------------------

`Cpumasks` - это особый способ, предоставляемый ядром Linux для хранения информации о процессорах (CPUs) в системе. Соответствующий исходный код и заголовочные файлы, которые содержат API для манипуляций `Cpumasks`:

* [include/linux/cpumask.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/cpumask.h)
* [lib/cpumask.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/lib/cpumask.c)
* [kernel/cpu.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/cpu.c)

Как написанно в комментарии из [include/linux/cpumask.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/cpumask.h): CPU маски предоставляют набор битов подходящих для набора процессоров в системе, по одной битовой позиции на номер процессора. Мы уже видели cpumask бит в `boot_cpu_init` функции из [Точка входа в ядро](https://github.com/proninyaroslav/linux-insides-ru/blob/master/Initialization/linux-initialization-4.md) части. Эта функция делает первый загрузочный процессор онлайн, активным и т. д.:

```C
set_cpu_online(cpu, true);
set_cpu_active(cpu, true);
set_cpu_present(cpu, true);
set_cpu_possible(cpu, true);
```

Прежде чем мы рассмотрим реализацию этих функций, давайте рассмотрим все эти маски.

`cpu_possible` - это набор идентификаторов процессоров, которые могут быть подключены в любое время во время загрузки системы или, другими словами, маска возможных ЦП содержит максимальное количество процессоров, которые могут быть установлены в системе. Она будет равна значению `NR_CPUS`, которое устанавливается статически через опцию конфигурации ядра `CONFIG_NR_CPUS`.

Маска `cpu_present` представляет, какие процессоры в данный момент подключены.

Маска `cpu_online` представляет собой подмножество `cpu_present` и указывает на процессоры, которые доступны для планирования, или другими словами, бит из этой маски сообщает ядру, что процессор может быть использован ядром Linux.

Последняя маска - `cpu_active`. Биты этой маски сообщают ядру Linux, что задача может быть перемещена на определенный процессор.

Все эти маски зависят от опции конфигурации `CONFIG_HOTPLUG_CPU`, и если эта опция отключена, `possible == present` и `active == online`. Реализации всех этих функций очень похожи. Каждая функция проверяет второй параметр. Если он `истинный`, она вызывает `cpumask_set_cpu`, в противном случае вызывает `cpumask_clear_cpu`.

Существует два способа создания `маски` процессоров. Первый - использовать `cpumask_t`. Он определен как:

```C
typedef struct cpumask { DECLARE_BITMAP(bits, NR_CPUS); } cpumask_t;
```

Он оборачивает структуру `cpumask`, которая содержит одно поле bitmask `bits`. Макрос `DECLARE_BITMAP` получает два параметра:

* Имя bitmap;
* Количество бит.

и создает массив `unsigned long` с указанным именем. Реализация этого довольно проста:

```C
#define DECLARE_BITMAP(name,bits) \
        unsigned long name[BITS_TO_LONGS(bits)]
```

где `BITS_TO_LONGS`:

```C
#define BITS_TO_LONGS(nr)       DIV_ROUND_UP(nr, BITS_PER_BYTE * sizeof(long))
#define DIV_ROUND_UP(n,d) (((n) + (d) - 1) / (d))
```

Поскольку мы сосредотачиваемся на архитектуре `x86_64`, `unsigned long` имеет размер 8 байт, и наш массив будет содержать только один элемент:

```
(((8) + (8) - 1) / (8)) = 1
```

Макрос `NR_CPUS` представляет собой количество процессоров в системе и зависит от макроса `CONFIG_NR_CPUS`, который определен в [include/linux/threads.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/threads.h) и имеет следующий вид:

```C
#ifndef CONFIG_NR_CPUS
        #define CONFIG_NR_CPUS  1
#endif

#define NR_CPUS         CONFIG_NR_CPUS
```

Второй способ определения cpumask - использовать макрос `DECLARE_BITMAP` напрямую и макрос `to_cpumask`, который преобразует данный bitmap в `struct cpumask *`:

```C
#define to_cpumask(bitmap)                                              \
        ((struct cpumask *)(1 ? (bitmap)                                \
                            : (void *)sizeof(__check_is_bitmap(bitmap))))
```

Здесь мы видим тернарный оператор, который всегда возвращает `true`. Функция inline `__check_is_bitmap` определена следующим образом:

```C
static inline int __check_is_bitmap(const unsigned long *bitmap)
{
        return 1;
}
```

И возвращает `1` каждый раз. Нам нужно это здесь только для одной цели: на этапе компиляции он проверяет, является ли переданный `bitmap` bitmap, или, другими словами, он проверяет, имеет ли `bitmap` тип - `unsigned long *`. Поэтому мы просто передаем `cpu_possible_bits` в макрос `to_cpumask` для преобразования массива `unsigned long` в `struct cpumask *`.

cpumask API
--------------------------------------------------------------------------------

Таким образом, мы можем определить маску процессора с помощью одного из методов. Ядро Linux предоставляет API для манипулирования маской процессора. Рассмотрим одну из функций, которая была представлена выше. Например, `set_cpu_online`. Эта функция принимает два параметра:

* Количество процессоров;
* Статус процессора;

Реализация этой функции выглядит следующим образом:

```C
void set_cpu_online(unsigned int cpu, bool online)
{
	if (online) {
		cpumask_set_cpu(cpu, to_cpumask(cpu_online_bits));
		cpumask_set_cpu(cpu, to_cpumask(cpu_active_bits));
	} else {
		cpumask_clear_cpu(cpu, to_cpumask(cpu_online_bits));
	}
}
```

Сначала проверяется второй параметр `state` и в зависимости от этого вызывается функция `cpumask_set_cpu` или `cpumask_clear_cpu`. Здесь мы видим приведение к типу `struct cpumask *` второго параметра в `cpumask_set_cpu`. В нашем случае это `cpu_online_bits`, который является bitmap и определен как:

```C
static DECLARE_BITMAP(cpu_online_bits, CONFIG_NR_CPUS) __read_mostly;
```

Функция `cpumask_set_cpu` делает всего один вызов функции `set_bit`:

```C
static inline void cpumask_set_cpu(unsigned int cpu, struct cpumask *dstp)
{
        set_bit(cpumask_check(cpu), cpumask_bits(dstp));
}
```

Функция `set_bit` также принимает два параметра и устанавливает указанный бит (первый параметр) в памяти (второй параметр или bitmap `cpu_online_bits`). Здесь мы видим, что до вызова `set_bit` его два параметра будут переданы в

* cpumask_check;
* cpumask_bits.

Давайте рассмотрим эти два макроса. Сначала, если `cpumask_check` в нашем случае ничего не делает и просто возвращает заданный параметр. Второй макрос `cpumask_bits` просто возвращает поле `bits` из заданной структуры `struct cpumask *`:

```C
#define cpumask_bits(maskp) ((maskp)->bits)
```

Теперь давайте посмотрим на реализацию `set_bit`:

```C
 static __always_inline void
 set_bit(long nr, volatile unsigned long *addr)
 {
         if (IS_IMMEDIATE(nr)) {
                asm volatile(LOCK_PREFIX "orb %1,%0"
                        : CONST_MASK_ADDR(nr, addr)
                        : "iq" ((u8)CONST_MASK(nr))
                        : "memory");
        } else {
                asm volatile(LOCK_PREFIX "bts %1,%0"
                        : BITOP_ADDR(addr) : "Ir" (nr) : "memory");
        }
 }
```

Эта функция выглядит устрашающе, но она не так сложна, как кажется. Прежде всего, он передает `nr` или номер бита макросу `IS_IMMEDIATE`, который просто вызывает внутреннюю функцию GCC `__builtin_constant_p`:

```C
#define IS_IMMEDIATE(nr)    (__builtin_constant_p(nr))
```
`__builtin_constant_p` проверяет, что данный параметр является известной константой во время компиляции. Поскольку наш `процессор` не является константой времени компиляции, будет выполнено предложение `else`:

```C
asm volatile(LOCK_PREFIX "bts %1,%0" : BITOP_ADDR(addr) : "Ir" (nr) : "memory");
```

Давайте попробуем понять, как это работает, шаг за шагом:

`LOCK_PREFIX` — это инструкция `блокировки` x86. Эта инструкция сообщает процессору, что он должен занять системную шину, пока будут выполняться инструкции. Это позволяет ЦП синхронизировать доступ к памяти, предотвращая одновременный доступ нескольких процессоров (или устройств — например, контроллера DMA) к одной ячейке памяти.

`BITOP_ADDR` преобразует заданный параметр в `(*( Летучий длинный *)` и добавляет ограничения `+m`. `+` означает, что этот операнд одновременно читается и записывается инструкцией. `m` показывает, что это операнд памяти. `BITOP_ADDR` определяется как:

```C
#define BITOP_ADDR(x) "+m" (*(volatile long *) (x))
```

Далее идет метка `памяти` clobber. Она сообщает компилятору, что ассемблерный код выполняет чтение или запись в память элементов, отличных от перечисленных во входных и выходных операндах (например, доступ к памяти, на которую указывает один из входных параметров).

`Ir` - немедленный регистровый операнд.

Инструкция `bts` устанавливает указанный бит в строке битов и сохраняет значение указанного бита в флаге `CF`. Таким образом, мы передали номер процессора, который в нашем случае равен нулю, и после выполнения `set_bit` устанавливается нулевой бит в cpumask `cpu_online_bits`. Это означает, что первый процессор в настоящее время находится в онлайн-режиме.

Помимо API `set_cpu_*`, cpumask, конечно, предоставляет другие API для манипулирования битовыми масками процессоров. Давайте кратко рассмотрим это.

Дополнительный API cpumask
--------------------------------------------------------------------------------

cpumask предоставляет набор макросов для получения номеров процессоров в различных состояниях. Например:

```C
#define num_online_cpus()	cpumask_weight(cpu_online_mask)
```

Этот макрос возвращает количество `онлайн` процессоров. Он вызывает функцию `cpumask_weight` с bitmap `cpu_online_mask` (прочтите о ней). Функция `cpumask_weight` выполняет один вызов функции `bitmap_weight` с двумя параметрами:

* cpumask bitmap;
* `nr_cpumask_bits` — в нашем случае это `NR_CPUS`.

```C
static inline unsigned int cpumask_weight(const struct cpumask *srcp)
{
	return bitmap_weight(cpumask_bits(srcp), nr_cpumask_bits);
}
```

и вычисляет количество битов в заданной bitmap. Кроме `num_online_cpus`, cpumask предоставляет макросы для всех состояний ЦП:

* num_possible_cpus;
* num_active_cpus;
* cpu_online;
* cpu_possible.

и многое другое.

Кроме того, ядро Linux предоставляет следующий API для манипулирования `cpumask`:

- `for_each_cpu` - перебирает каждый CPU в маске;
- `for_each_cpu_not` - перебирает каждый CPU в дополненной маске;
- `cpumask_clear_cpu` - сбрасывает CPU в cpumask;
- `cpumask_test_cpu` - проверяет CPU в маске;
- `cpumask_setall` - устанавливает все CPU в маске;
- `cpumask_size` - возвращает размер для выделения памяти для 'struct cpumask' в байтах;

и многое многое другое.

Ссылки
--------------------------------------------------------------------------------

* [документация cpumask](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)
