Процессорные переменные
================================================================================

Процессорные переменные - это одна из функций ядра. Вы можете понять значение этой функции, прочитав ее название. Мы можем создать переменную, и каждое ядро процессора будет иметь свою копию этой переменной. В этой главе мы подробнее рассмотрим эту функцию и попытаемся понять, как она реализована и как работает.

Ядро предоставляет API для создания процессорных переменных - `DEFINE_PER_CPU` macro:

```C
#define DEFINE_PER_CPU(type, name) \
        DEFINE_PER_CPU_SECTION(type, name, "")
```

Этот макрос определен в [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/percpu-defs.h) как и многие другие макросы для работы с процессорными переменными. Теперь посмотрим, как эта функция реализована.

Взглянем на `DEFINE_PER_CPU` определение. Мы видим, что оно принимает 2 параметра: `type` и `name`, поэтому мы можем использовать это для создания процессорных переменных, например вот так:

```C
DEFINE_PER_CPU(int, per_cpu_n)
```

Мы передаем тип и имя нашей переменной. `DEFINE_PER_CPU` вызывает макрос `DEFINE_PER_CPU_SECTION` и передает ему те же два параметра и пустую строку. Взглянем на определение `DEFINE_PER_CPU_SECTION`:

```C
#define DEFINE_PER_CPU_SECTION(type, name, sec)    \
         __PCPU_ATTRS(sec) PER_CPU_DEF_ATTRIBUTES  \
         __typeof__(type) name
```

```C
#define __PCPU_ATTRS(sec)                                                \
         __percpu __attribute__((section(PER_CPU_BASE_SECTION sec)))     \
         PER_CPU_ATTRIBUTES
```

где `section`:

```C
#define PER_CPU_BASE_SECTION ".data..percpu"
```

После раскрытия всех макросов мы получим глобальную процессорную переменную:

```C
__attribute__((section(".data..percpu"))) int per_cpu_n
```

Это означает, что у нас будет переменная `per_cpu_n` в разделе `.data..percpu.` Мы можем найти этот раздел в `vmlinux`:

```
.data..percpu 00013a58  0000000000000000  0000000001a5c000  00e00000  2**12
              CONTENTS, ALLOC, LOAD, DATA
```

Хорошо, теперь мы знаем, что когда мы используем макрос `DEFINE_PER_CPU`, в разделе `.data..percpu` будет создана процессорная переменная. Когда ядро инициализируется, оно вызывает функцию `setup_per_cpu_areas`, которая загружает раздел `.data..percpu` несколько раз, по одному раздела на каждый процессор (CPU).

Давайте посмотрим на процесс инициализации областей каждого процессора. Он начинается в [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) с вызова функции `setup_per_cpu_areas`, которая определена в [arch/x86/kernel/setup_percpu.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup_percpu.c).

```C
pr_info("NR_CPUS:%d nr_cpumask_bits:%d nr_cpu_ids:%d nr_node_ids:%d\n",
        NR_CPUS, nr_cpumask_bits, nr_cpu_ids, nr_node_ids);
```

`setup_per_cpu_areas` начинается с вывода информации о максимальном количестве процессоров, установленном во время конфигурации ядра с помощью параметра конфигурации `CONFIG_NR_CPUS`, фактического количества процессоров, `nr_cpumask_bits` совпадает с битом `NR_CPUS` для новых операторов `cpumask` и количество узлов `NUMA`.

Мы можем увидеть этот вывод в dmesg:

```
$ dmesg | grep percpu
[    0.000000] setup_percpu: NR_CPUS:8 nr_cpumask_bits:8 nr_cpu_ids:8 nr_node_ids:1
```

В следующем шаге мы проверяем аллокатор первого блока percpu. Все области процессора (percpu) аллоцированы блоками/частями. Первый блок используется для статических пемеренных процессора. Ядро Linux имеет параметры командной строки `percpu_alloc`, которые определяют тип первого аллокатора первого блока. Мы можем прочитать об этом в документации ядра:

```
percpu_alloc=	Select which percpu first chunk allocator to use.
		Currently supported values are "embed" and "page".
		Archs may support subset or none of the	selections.
		See comments in mm/percpu.c for details on each
		allocator.  This parameter is primarily	for debugging
		and performance comparison.
```

Файл [mm/percpu.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/mm/percpu.c) содержит обработчик этой опции командной строки:

```C
early_param("percpu_alloc", percpu_alloc_setup);
```

Где `percpu_alloc_setup` функция задает переменную `pcpu_chosen_fc` в зависимости от значения параметра `percpu_alloc`. По умолчанию, первый блок аллокатора - `auto`:

```C
enum pcpu_fc pcpu_chosen_fc __initdata = PCPU_FC_AUTO;
```

Если `percpu_alloc` не указан в командной строке ядра, будет использоваться аллокатор `embed`, который встраивает первую область percpu в bootmem с помощью [memblock](https://github.com/proninyaroslav/linux-insides-ru/blob/master/MM/linux-mm-1.md). Последний аллокатор — это аллокатор `страниц` первого фрагмента, который сопоставляет первый фрагмент со страницами `PAGE_SIZE`.

Как я писал выше, прежде всего, мы проверяем первый тип аллокатора области в `setup_per_cpu_areas`. Проверяем, что первый аллокатора областей не является страницей: 

```C
if (pcpu_chosen_fc != PCPU_FC_PAGE) {
    ...
    ...
    ...
}
```

Если это не `PCPU_FC_PAGE`, мы будем использовать аллокатор `embed` и аллоцировать пространство для первой области с помощью функции `pcpu_embed_first_chunk`:

```C
rc = pcpu_embed_first_chunk(PERCPU_FIRST_CHUNK_RESERVE,
					    dyn_size, atom_size,
					    pcpu_cpu_distance,
					    pcpu_fc_alloc, pcpu_fc_free);
```

Как показано выше, функция `pcpu_embed_first_chunk` встраивает первая область percpu в bootmem, после чего мы передаем пару параметров в `pcup_embed_first_chunk`. Они заключаются в следующем:

* `PERCPU_FIRST_CHUNK_RESERVE` - размер зарезервированного пространства для статических переменных `percpu`;
* `dyn_size` - минимальный свободный размер для динамического выделения в байтах; 
* `atom_size` - все выделения являются целыми кратными этому и выравниваются по этому параметру;
* `pcpu_cpu_distance` - обратный вызов для определения расстояния между процессорами;
* `pcpu_fc_alloc` - функция для выделения страницы `percpu`
* `pcpu_fc_free` - функция для освобождения страницы `percpu`.

Мы вычисляем все эти параметры перед вызовом `pcpu_embed_first_chunk`:

```C
const size_t dyn_size = PERCPU_MODULE_RESERVE + PERCPU_DYNAMIC_RESERVE - PERCPU_FIRST_CHUNK_RESERVE;
size_t atom_size;
#ifdef CONFIG_X86_64
		atom_size = PMD_SIZE;
#else
		atom_size = PAGE_SIZE;
#endif
```

Если первым аллокатором областей является `PCPU_FC_PAGE`, мы будем использовать `pcpu_page_first_chunk` вместо `pcpu_embed_first_chunk`. После этого мы устанавливаем смещение `percpu` и его сегмент для каждого процессора с помощью функции `setup_percpu_segment` (только для систем `x86`) и перемещаем некоторые ранние данные из массивов в переменные `percpu` (`x86_cpu_to_apicid`, `irq_stack_ptr` и т. д.). После того, как ядро завершит процесс инициализации, мы загрузим N разделов `.data..percpu`, где N — количество процессоров, а раздел, используемый загрузочным процессором, будет содержать неинициализированную переменную, созданную с помощью макроса `DEFINE_PER_CPU`. .

Ядро предоставляет API для управления переменными каждого процессора:

* get_cpu_var(var)
* put_cpu_var(var)

Давайте посмотрим на реализацию `get_cpu_var`:

```C
#define get_cpu_var(var)     \
(*({                         \
         preempt_disable();  \
         this_cpu_ptr(&var); \
}))
```

Ядро Linux является вытесняемым, и для доступа к переменной каждого процессора нам необходимо знать, на каком процессоре работает ядро. Таким образом, текущий код не должен быть вытеснен и перемещен на другой процессор при доступе к переменной для каждого процессора. Поэтому в первую очередь мы видим вызов функции `preempt_disable`, а затем вызов макроса `this_cpu_ptr`, который выглядит так:

```C
#define this_cpu_ptr(ptr) raw_cpu_ptr(ptr)
```

и

```C
#define raw_cpu_ptr(ptr)        per_cpu_ptr(ptr, 0)
```

где `per_cpu_ptr` возвращает указатель на переменную для каждого процессора для данного процессора (второй параметр). После того, как мы создали переменную для каждого процессора и внесли в нее изменения, мы должны вызвать макрос `put_cpu_var`, который включает вытеснение с помощью вызова функции `preempt_enable`. Таким образом, типичное использование переменной для каждого процессора выглядит следующим образом:

```C
get_cpu_var(var);
...
//Do something with the 'var'
...
put_cpu_var(var);
```

Давайте взглянем на `per_cpu_ptr` макрос:

```C
#define per_cpu_ptr(ptr, cpu)                             \
({                                                        \
        __verify_pcpu_ptr(ptr);                           \
         SHIFT_PERCPU_PTR((ptr), per_cpu_offset((cpu)));  \
})
```

Как я писал выше, этот макрос возвращает per-cpu переменную для данного процессора. Прежде всего он вызывает `__verify_pcpu_ptr`:

```C
#define __verify_pcpu_ptr(ptr)
do {
	const void __percpu *__vpp_verify = (typeof((ptr) + 0))NULL;
	(void)__vpp_verify;
} while (0)
```

что делает данный тип `ptr` `const void __percpu *`

После этого мы можем увидеть вызов макроса `SHIFT_PERCPU_PTR` с двумя параметрами. Первый параметр мы передаем наш ptr и второй параметр номер cpu в макрос `per_cpu_offset`:

```C
#define per_cpu_offset(x) (__per_cpu_offset[x])
```

который расширяется до получения элемента `x` из массива `__per_cpu_offset`:

```C
extern unsigned long __per_cpu_offset[NR_CPUS];
```

где `NR_CPUS` — количество процессоров. Массив `__per_cpu_offset` заполняется расстояниями между копиями переменных процессора. Например, все данные каждого процессора имеют размер X байт, поэтому, если мы получим доступ к `__per_cpu_offset[Y]`, будет доступен `X*Y`. Давайте посмотрим на реализацию `SHIFT_PERCPU_PTR`:

```C
#define SHIFT_PERCPU_PTR(__p, __offset)                                 \
         RELOC_HIDE((typeof(*(__p)) __kernel __force *)(__p), (__offset))
```

`RELOC_HIDE` просто возвращает смещение `(typeof(ptr)) (__ptr + (off))` и возвращает указатель на переменную.

Вот и все! Конечно это не полный API, а общий обзор. Начать с этого может быть сложно, но чтобы понять переменные для каждого процессора, вам в основном нужно понимать [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include /linux/percpu-defs.h) магия.

Давайте еще раз посмотрим на алгоритм получения указателя на процессорную переменную:

* Ядро создает несколько разделов `.data..percpu` (по одному на каждый процессор) во время процесса инициализации;
* Все переменные, созданные с помощью макроса `DEFINE_PER_CPU`, будут перенесены в первый раздел или для CPU0;
* Массив `__per_cpu_offset`, заполненный расстоянием (`BOOT_PERCPU_OFFSET`) между секциями `.data..percpu`;
* Когда вызывается `per_cpu_ptr`, например, для получения указателя на определенную переменную для каждого процессора для третьего процессора, будет доступен массив `__per_cpu_offset`, где каждый индекс указывает на требуемый процессор.

Вот и все.
