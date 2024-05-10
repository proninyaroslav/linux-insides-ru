Цепочки уведомлений в Ядре Linux
================================================================================

Вступление
--------------------------------------------------------------------------------

Ядро Linux - огромный кусок кода на языке программирования C, который состоит из множества различных подсистем. Каждая подсистема имеет свою собственную цель, независимую от других подсистем. Однако часто одна подсистема хочет узнать что-то из другой подсистемы (или подсистем). В ядре Linux существует специальный механизм, который позволяет частично решить эту проблему. Название этого механизма - `цепочки уведомлений`, и его главная цель - обеспечить способ для различных подсистем подписаться на асинхронные события от других подсистем. Обратите внимание, что этот механизм предназначен только для общения внутри ядра, но существуют и другие механизмы для общения между ядром и пользовательским пространством.

Прежде чем мы рассмотрим [API](https://en.wikipedia.org/wiki/Application_programming_interface) `цепочек уведомлений` и реализацию этого API, давайте посмотрим на механизм `цепочек уведомлений` с теоретической стороны, как мы делали это в других частях этой книги. Все, что связано с механизмом `цепочек уведомлений`, находится в заголовочном файле [include/linux/notifier.h](https://github.com/torvalds/linux/blob/master/include/linux/notifier.h) и файле исходного кода [kernel/notifier.c](https://github.com/torvalds/linux/blob/master/kernel/notifier.c). Итак, давайте откроем их и начнем погружение.

Структуры данных, связанные с цепочками уведомлений.
--------------------------------------------------------------------------------

Давайте начнем рассматривать механизм `цепочек уведомлений` с помощью связанных структур данных. Как я указывал выше, основные структуры данных должны находиться в заголовочном файле [include/linux/notifier.h](https://github.com/torvalds/linux/blob/master/include/linux/notifier.h), поэтому ядро Linux предоставляет общедоступный API, который не зависит от определенной архитектуры. В общем, механизм `цепочек уведомлений` представляет собой список (поэтому он назван `цепочками`) функций [обратного вызова](https://en.wikipedia.org/wiki/Callback_%28computer_programming%29), которые будут выполнены, когда произойдет событие.

Все эти функции обратного вызова представлены как тип `notifier_fn_t` в ядре Linux:

```C
typedef	int (*notifier_fn_t)(struct notifier_block *nb, unsigned long action, void *data);
```

Итак, мы видим, что он принимает три следующих аргумента:

* `nb` - связанный список указателей на функции (сейчас мы его увидим);
* `action` - тип события. Цепочка уведомлений может поддерживать несколько событий, поэтому этот параметр нужен нам, чтобы отличать событие от других событий;
* `data` — хранилище приватной информации. Фактически это позволяет предоставить дополнительные данные о событии.

Кроме того, мы можем увидеть, что `notifier_fn_t` возвращает целочисленное значение. Это целочисленное значение может быть одним из:

* `NOTIFY_DONE` - подписчик не заинтересован в уведомлении;
* `NOTIFY_OK` - уведомление обработано корректно;
* `NOTIFY_BAD` — что-то пошло не так;
* `NOTIFY_STOP` — уведомление выполнено, но для этого события дальнейшие обратные вызовы вызываться не должны.

Все эти результаты определены как макросы в заголовочном файле [include/linux/notifier.h](https://github.com/torvalds/linux/blob/master/include/linux/notifier.h):

```C
#define NOTIFY_DONE		0x0000
#define NOTIFY_OK		0x0001
#define NOTIFY_BAD		(NOTIFY_STOP_MASK|0x0002)
#define NOTIFY_STOP		(NOTIFY_OK|NOTIFY_STOP_MASK)
```

где `NOTIFY_STOP_MASK` представлен как:

```C
#define NOTIFY_STOP_MASK	0x8000
```

макрос и означает, что обратные вызовы не будут вызываться при следующих уведомлениях.

Каждая часть ядра Linux, которая хочет получать уведомления об определенном событии, должна предоставить собственную функцию обратного вызова `notifier_fn_t`. Основная роль механизма `цепочек уведомлений` заключается в вызове определенных обратных вызовов при возникновении асинхронного события.

Основным строительным блоком механизма `цепочек уведомлений` является структура `notifier_block`:

```C
struct notifier_block {
	notifier_fn_t notifier_call;
	struct notifier_block __rcu *next;
	int priority;
};
```
который определен в файле [include/linux/notifier.h](https://github.com/torvalds/linux/blob/master/include/linux/notifier.h). Эта структура содержит указатель на функцию обратного вызова — `notifier_call`, ссылку на следующий обратный вызов уведомления и `приоритет` функции обратного вызова, поскольку функции с более высоким приоритетом выполняются первыми.

Ядро Linux предоставляет цепочки уведомлений четырех следующих типов:

* Блокировка цепочек уведомлений;
* Цепочки уведомлений SRCU;
* Атомарные цепочки уведомлений;
* Необработанные цепочки уведомлений.

Давайте рассмотрим все эти типы цепочек уведомлений по порядку:

В первом случае для `блокирующих цепочек уведомлений` обратные вызовы будут вызываться/выполняться в контексте процесса. Это означает, что вызовы в цепочке уведомлений могут быть заблокированы.

Вторые `цепочки уведомлений SRCU` представляют собой альтернативную форму `блокирующих цепочек уведомлений`. В первом случае блокировка цепочек уведомлений использует примитив синхронизации `rw_semaphore` для защиты звеньев цепочки. Цепочки уведомлений `SRCU` также выполняются в контексте процесса, но используют специальную форму механизма [RCU](https://en.wikipedia.org/wiki/Read-copy-update), который можно блокировать на критической стороне чтения. раздел.

В третьем случае `атомарные цепочки уведомлений` выполняются в прерывании или атомарном контексте и защищены [spinlock](https://github.com/proninyaroslav/linux-insides-ru/blob/master/SyncPrim/linux-sync-1.md) примитив синхронизации. Последние `необработанные цепочки уведомлений` предоставляют особый тип цепочек уведомлений без каких-либо ограничений блокировки обратных вызовов. Это означает, что защита ложится на плечи вызывающей стороны. Это очень полезно, когда мы хотим защитить нашу цепь с помощью специального запорного механизма.

Если мы посмотрим на реализацию структуры `notifier_block`, то увидим, что она содержит указатель на `следующий` элемент из списка цепочки уведомлений, но у нас нет заголовка. На самом деле заголовок такого списка находится в отдельной структуре и зависит от типа цепочки уведомлений. Например, для `блокирующих цепочек уведомлений`:

```C
struct blocking_notifier_head {
	struct rw_semaphore rwsem;
	struct notifier_block __rcu *head;
};
```

или для `atomic notification chains`:

```C
struct atomic_notifier_head {
	spinlock_t lock;
	struct notifier_block __rcu *head;
};
```

Теперь, когда мы немного знаем о механизме `цепочек уведомлений`, давайте рассмотрим реализацию его API.

Цепочки уведомлений
--------------------------------------------------------------------------------

Обычно в механизмах публикации/подписки есть две стороны. Одна сторона, которая хочет получать уведомления, и другая сторона (стороны), которая генерирует эти уведомления. Мы рассмотрим механизм цепочек уведомлений с обеих сторон. В этой части мы рассмотрим `блокировку цепочек уведомлений`, поскольку другие типы цепочек уведомлений аналогичны ей и отличаются в основном механизмами защиты.

Прежде чем производитель уведомлений сможет создать уведомление, прежде всего он должен инициализировать главу цепочки уведомлений. Например, давайте рассмотрим цепочки уведомлений, связанные с ядром [загружаемые модули] (https://en.wikipedia.org/wiki/Loadable_kernel_module). Если мы посмотрим файл исходного кода [kernel/module.c](https://github.com/torvalds/linux/blob/master/kernel/module.c), мы увидим следующее определение:

```C
static BLOCKING_NOTIFIER_HEAD(module_notify_list);
```

который определяет заголовок для загружаемых модулей, блокирующих цепочку уведомлений. Макрос `BLOCKING_NOTIFIER_HEAD` определен в заголовочном файле [include/linux/notifier.h](https://github.com/torvalds/linux/blob/master/include/linux/notifier.h) и расширяется до следующего: код:

```C
#define BLOCKING_INIT_NOTIFIER_HEAD(name) do {	\
		init_rwsem(&(name)->rwsem);	                            \
		(name)->head = NULL;		                            \
	} while (0)
```

Итак, мы можем видеть, что он берет имя или имя главы цепочки блокирующих уведомлений и инициализирует чтение/запись [семафор] (https://github.com/proninyaroslav/linux-insides-ru/blob/master/SyncPrim/linux-sync-3.md) и установите для заголовка значение `NULL`. Помимо макроса `BLOCKING_INIT_NOTIFIER_HEAD`, ядро Linux дополнительно предоставляет макросы `ATOMIC_INIT_NOTIFIER_HEAD`, `RAW_INIT_NOTIFIER_HEAD` и функцию `srcu_init_notifier` для инициализации атомарных и других типов цепочек уведомлений.

После инициализации главы цепочки уведомлений подсистема, которая хочет получать уведомления из данной цепочки уведомлений, должна зарегистрироваться с определенной функцией, которая зависит от типа уведомления. Если вы посмотрите заголовочный файл [include/linux/notifier.h](https://github.com/torvalds/linux/blob/master/include/linux/notifier.h), вы увидите следующие четыре функции для этот:

```C
extern int atomic_notifier_chain_register(struct atomic_notifier_head *nh,
		struct notifier_block *nb);

extern int blocking_notifier_chain_register(struct blocking_notifier_head *nh,
		struct notifier_block *nb);

extern int raw_notifier_chain_register(struct raw_notifier_head *nh,
		struct notifier_block *nb);

extern int srcu_notifier_chain_register(struct srcu_notifier_head *nh,
		struct notifier_block *nb);
```

Как я уже писал выше, в этой части мы рассмотрим только блокировку цепочек уведомлений, поэтому рассмотрим реализацию функции `blocking_notifier_chain_register`. Реализация этой функции находится в файле исходного кода [kernel/notifier.c](https://github.com/torvalds/linux/blob/master/kernel/notifier.c) и, как мы видим, `blocking_notifier_chain_register` принимает два параметра:

* `nh` — начало цепочки уведомлений;
* `nb` — дескриптор уведомления.

Теперь давайте посмотрим на реализацию функции `blocking_notifier_chain_register`:

```C
int raw_notifier_chain_register(struct raw_notifier_head *nh,
		struct notifier_block *n)
{
	return notifier_chain_register(&nh->head, n);
}
```

Как мы видим, она просто возвращает результат функции `notifier_chain_register` из того же файла исходного кода, и, как мы понимаем, эта функция делает всю работу за нас. Определение функции `notifier_chain_register` выглядит так:

```C
int blocking_notifier_chain_register(struct blocking_notifier_head *nh,
		struct notifier_block *n)
{
	int ret;

	if (unlikely(system_state == SYSTEM_BOOTING))
		return notifier_chain_register(&nh->head, n);

	down_write(&nh->rwsem);
	ret = notifier_chain_register(&nh->head, n);
	up_write(&nh->rwsem);
	return ret;
}
```

Как мы видим, реализация `blocking_notifier_chain_register` довольно проста. Прежде всего, есть проверка, которая проверяет текущее состояние системы, и если система находится в состоянии перезагрузки, мы просто вызываем `notifier_chain_register`. Другим способом мы делаем тот же вызов `notifier_chain_register`, но, как вы можете видеть, этот вызов защищен семафорами чтения/записи. Теперь давайте посмотрим на реализацию функции `notifier_chain_register`:

```C
static int notifier_chain_register(struct notifier_block **nl,
		struct notifier_block *n)
{
	while ((*nl) != NULL) {
		if (n->priority > (*nl)->priority)
			break;
		nl = &((*nl)->next);
	}
	n->next = *nl;
	rcu_assign_pointer(*nl, n);
	return 0;
}
```

Эта функция просто вставляет новый `notifier_block` (предоставленный подсистемой, которая хочет получать уведомления) в список цепочки уведомлений. Помимо подписки на событие, подписчик может отказаться от подписки на определенные события с помощью набора функций `отписки`:

```C
extern int atomic_notifier_chain_unregister(struct atomic_notifier_head *nh,
		struct notifier_block *nb);

extern int blocking_notifier_chain_unregister(struct blocking_notifier_head *nh,
		struct notifier_block *nb);

extern int raw_notifier_chain_unregister(struct raw_notifier_head *nh,
		struct notifier_block *nb);

extern int srcu_notifier_chain_unregister(struct srcu_notifier_head *nh,
		struct notifier_block *nb);
```

Когда производитель уведомлений хочет уведомить подписчиков о событии, будет вызвана функция `*.notifier_call_chain`. Как вы уже догадались, каждый тип цепочек уведомлений предоставляет собственную функцию для создания уведомлений:

```C
extern int atomic_notifier_call_chain(struct atomic_notifier_head *nh,
		unsigned long val, void *v);

extern int blocking_notifier_call_chain(struct blocking_notifier_head *nh,
		unsigned long val, void *v);

extern int raw_notifier_call_chain(struct raw_notifier_head *nh,
		unsigned long val, void *v);

extern int srcu_notifier_call_chain(struct srcu_notifier_head *nh,
		unsigned long val, void *v);
```

Рассмотрим реализацию функции `blocking_notifier_call_chain`. Эта функция определена в файле исходного кода [kernel/notifier.c](https://github.com/torvalds/linux/blob/master/kernel/notifier.c):

```C
int blocking_notifier_call_chain(struct blocking_notifier_head *nh,
		unsigned long val, void *v)
{
	return __blocking_notifier_call_chain(nh, val, v, -1, NULL);
}
```

и, как мы видим, он просто возвращает результат функции `__blocking_notifier_call_chain`. Как мы видим, `blocking_notifer_call_chain` принимает три параметра:

* `nh` - заголовок списка цепочек уведомлений;
* `val` - тип уведомления;
* `v` — входной параметр, который может использоваться обработчиками.

А вот функция `__blocking_notifier_call_chain` принимает пять параметров:

```C
int __blocking_notifier_call_chain(struct blocking_notifier_head *nh,
				   unsigned long val, void *v,
				   int nr_to_call, int *nr_calls)
{
    ...
    ...
    ...
}
```

Где `nr_to_call` и `nr_calls` — это количество вызываемых функций уведомления и количество отправленных уведомлений. Как вы можете догадаться, основная цель функции `__blocking_notifer_call_chain` и других функций для других типов уведомлений — вызвать функцию обратного вызова при возникновении события. Реализация `__blocking_notifier_call_chain` довольно проста: она просто вызывает функцию `notifier_call_chain` из того же файла исходного кода, защищенного семафором чтения/записи:

```C
int __blocking_notifier_call_chain(struct blocking_notifier_head *nh,
				   unsigned long val, void *v,
				   int nr_to_call, int *nr_calls)
{
	int ret = NOTIFY_DONE;

	if (rcu_access_pointer(nh->head)) {
		down_read(&nh->rwsem);
		ret = notifier_call_chain(&nh->head, val, v, nr_to_call,
					nr_calls);
		up_read(&nh->rwsem);
	}
	return ret;
}
```

и возвращает свой результат. В этом случае вся работа выполняется функцией `notifier_call_chain`. Основная цель этой функции информирует зарегистрированных уведомителей об асинхронном событии:

```C
static int notifier_call_chain(struct notifier_block **nl,
			       unsigned long val, void *v,
			       int nr_to_call, int *nr_calls)
{
    ...
    ...
    ...
    ret = nb->notifier_call(nb, val, v);
    ...
    ...
    ...
    return ret;
}
```

Вот и все. В целом все выглядит довольно просто.

Теперь давайте рассмотрим простой пример, связанный с [загружаемыми модулями](https://en.wikipedia.org/wiki/Loadable_kernel_module). Если мы посмотрим в [kernel/module.c](https://github.com/torvalds/linux/blob/master/kernel/module.c). Как мы уже видели в этой части, есть:

```C
static BLOCKING_NOTIFIER_HEAD(module_notify_list);
```

определение `module_notify_list` в файле исходного кода [kernel/module.c](https://github.com/torvalds/linux/blob/master/kernel/module.c). Это определение определяет заголовок списка цепочек блокирующих уведомлений, связанных с модулями ядра. Есть как минимум три следующих события:

* MODULE_STATE_LIVE
* MODULE_STATE_COMING
* MODULE_STATE_GOING

в чем возможно интересуются некоторые подсистемы ядра Linux. Например трассировка состояний модулей ядра. Вместо прямого вызова `atomic_notifier_chain_register`, `blocking_notifier_chain_register` и т. д. большинство цепочек уведомлений поставляются с набором оболочек, используемых для регистрации в них. Регистрация на событиях этих модулей происходит с помощью такой обертки:

```C
int register_module_notifier(struct notifier_block *nb)
{
	return blocking_notifier_chain_register(&module_notify_list, nb);
}
```

Если мы посмотрим в файл исходного кода [kernel/tracepoint.c](https://github.com/torvalds/linux/blob/master/kernel/tracepoint.c), то увидим такую регистрацию во время инициализации [tracepoints](https://www.kernel.org/doc/Documentation/trace/tracepoints.txt):

```C
static __init int init_tracepoints(void)
{
	int ret;

	ret = register_module_notifier(&tracepoint_module_nb);
	if (ret)
		pr_warn("Failed to register tracepoint module enter notifier\n");

	return ret;
}
```

Где `tracepoint_module_nb` предоставляет функцию обратного вызова:

```C
static struct notifier_block tracepoint_module_nb = {
	.notifier_call = tracepoint_module_notify,
	.priority = 0,
};
```

Когда произошло одно из событий `MODULE_STATE_LIVE`, `MODULE_STATE_COMING` или `MODULE_STATE_GOING`. Например, уведомления `MODULE_STATE_LIVE` и `MODULE_STATE_COMING` будут отправлены во время выполнения [init_module](http://man7.org/linux/man-pages/man2/init_module.2.html) [системного вызова](https://github.com/proninyaroslav/linux-insides-ru/blob/master/SysCall/linux-syscall-1.md). Или, например, `MODULE_STATE_GOING` будет отправлен во время выполнения `системного вызова` [delete_module](http://man7.org/linux/man-pages/man2/delete_module.2.html):

```C
SYSCALL_DEFINE2(delete_module, const char __user *, name_user,
		unsigned int, flags)
{
    ...
    ...
    ...
    blocking_notifier_call_chain(&module_notify_list,
				     MODULE_STATE_GOING, mod);
    ...
    ...
    ...
}
```

Таким образом, когда один из этих системных вызовов будет вызван из пользовательского пространства, ядро Linux отправит определенное уведомление в зависимости от системного вызова и будет вызвана функция обратного вызова `tracepoint_module_notify`.

Вот и все.

Ссылки
--------------------------------------------------------------------------------

* [Язык программирования С](https://en.wikipedia.org/wiki/C_%28programming_language%29)
* [API](https://en.wikipedia.org/wiki/Application_programming_interface)
* [Функция обратного вызова](https://en.wikipedia.org/wiki/Callback_%28computer_programming%29)
* [RCU](https://en.wikipedia.org/wiki/Read-copy-update)
* [spinlock](https://github.com/proninyaroslav/linux-insides-ru/blob/master/SyncPrim/linux-sync-1.md)
* [Загружаемые модули](https://en.wikipedia.org/wiki/Loadable_kernel_module)
* [semaphore](https://github.com/proninyaroslav/linux-insides-ru/blob/master/SyncPrim/linux-sync-3.md)
* [tracepoints](https://www.kernel.org/doc/Documentation/trace/tracepoints.txt)
* [Системный вызов](https://github.com/proninyaroslav/linux-insides-ru/blob/master/SysCall/linux-syscall-1.md)
* [Системный вызов init_module](http://man7.org/linux/man-pages/man2/init_module.2.html)
* [delete_module](http://man7.org/linux/man-pages/man2/delete_module.2.html)
* [Предыдущая часть](https://github.com/proninyaroslav/linux-insides-ru/blob/master/Concepts/linux-cpu-3.md)
