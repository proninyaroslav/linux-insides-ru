Контрольные группы
================================================================================

Вступление
--------------------------------------------------------------------------------

Это первая часть новой главы книги [linux insides](https://proninyaroslav.gitbooks.io/linux-insides-ru/content/) и как вы можете предположить по названию - эта часть описывает
[контрольные группы](https://en.wikipedia.org/wiki/Cgroups) или `cgroups` механизм в Linux ядре.

`Cgroups` - это специальный механизм предоставленный Linux ядром, который позволяет нам аллоцировать конкретные ресурсы такие как процессорное время, число процессов каждой группы, количество памяти для каждой контрольной группы или сочетание таких ресурсов для процесса и набора процессов. `Cgroups` - организован в иерархическом порядке и в данном случае механизм аналогичен обычным процессам потому что они тоже организованы в иерархическом порядке и дочерний `cgroups` наследует набор определенных параметров от их родителей. Но они не одинаковые. Основные различия между `cgroups` и обычными процессами заключаются в том, что в одно и то же время может существовать множество различных иерархий управляющих групп, в то время как обычное дерево процессов всегда одно. Это не случайно потому что каждая иерархия контрольной группы прикреплена к набору `subsystems` контрольной группы.

Одна `подсистема контрольной группы` представляет один тип ресурсов, например, время процессора или число [pids](https://en.wikipedia.org/wiki/Process_identifier), т.е. число процессов для `контрольной группы`. Линукс ядро предоставляет поддержку для следующих 12 `подсистем контрольных групп`:

`cpuset` - назначает индивидуальный процессор/процессоры и узлы памяти для задачи/задач в группе;

`cpu` - использует планировщик для предоставления доступа задач cgroup к ресурсам процессора;

`cpuacct` - генерирует отчеты об использовании процессора группой;

`io` - sets limit to read/write from/to block devices;
задает ограничение для чтения/записи от/к [блочным устройствам](https://en.wikipedia.org/wiki/Device_file);

`memory` - задает ограничение на использование памяти задачей(задачами) из группы;

`devices` - открывает доступ к устройствам задаче(задачам) из группы;

`freezer` - позволяет приостановить/возобновить задачу(задачи) из группы;

`net_cls` - позволяет отмечать сетевые пакеты задаче(задач) группы;

`net_prio` - предоставляет путь динамично задать приоритет сетевого трафика для каждого сетевого интерфейсы группы;

`perf_event` - предоставляет доступ к [событиям производительности](https://en.wikipedia.org/wiki/Perf_(Linux)) для группы;

`hugetlb` - активирует поддержку для [объемных страниц](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt) группы;

`pid` - устанавливает ограничение на количество процессов в группе;

Каждая из этих подсистем контрольной группы зависит от соответствующей конфигурационной опции. Например, подсистему `cpuset` следует включить с помощью опции конфигурации ядра `CONFIG_CPUSETS`, `io` подсистему с помощью `CONFIG_BLK_CGROUP` опции конфигурации ядра и т.д. Все эти опции конфигурации ядра могут быть найдены в `General setup -> Control Group support` меню:

![menuconfig](images/menuconfig.png)

Вы можете увидеть включенные контрольные группы на вашем компьютере через файловую систему [proc](https://en.wikipedia.org/wiki/Procfs):

```
$ cat /proc/cgroups 
#subsys_name	hierarchy	num_cgroups	enabled
cpuset	8	1	1
cpu	7	66	1
cpuacct	7	66	1
blkio	11	66	1
memory	9	94	1
devices	6	66	1
freezer	2	1	1
net_cls	4	1	1
perf_event	3	1	1
net_prio	4	1	1
hugetlb	10	1	1
pids	5	69	1
```

или через [sysfs](https://en.wikipedia.org/wiki/Sysfs):

```
$ ls -l /sys/fs/cgroup/
total 0
dr-xr-xr-x 5 root root  0 Dec  2 22:37 blkio
lrwxrwxrwx 1 root root 11 Dec  2 22:37 cpu -> cpu,cpuacct
lrwxrwxrwx 1 root root 11 Dec  2 22:37 cpuacct -> cpu,cpuacct
dr-xr-xr-x 5 root root  0 Dec  2 22:37 cpu,cpuacct
dr-xr-xr-x 2 root root  0 Dec  2 22:37 cpuset
dr-xr-xr-x 5 root root  0 Dec  2 22:37 devices
dr-xr-xr-x 2 root root  0 Dec  2 22:37 freezer
dr-xr-xr-x 2 root root  0 Dec  2 22:37 hugetlb
dr-xr-xr-x 5 root root  0 Dec  2 22:37 memory
lrwxrwxrwx 1 root root 16 Dec  2 22:37 net_cls -> net_cls,net_prio
dr-xr-xr-x 2 root root  0 Dec  2 22:37 net_cls,net_prio
lrwxrwxrwx 1 root root 16 Dec  2 22:37 net_prio -> net_cls,net_prio
dr-xr-xr-x 2 root root  0 Dec  2 22:37 perf_event
dr-xr-xr-x 5 root root  0 Dec  2 22:37 pids
dr-xr-xr-x 5 root root  0 Dec  2 22:37 systemd
```

Как вы можете предположить механизм `контрольных групп` был разработан не напрямую для Linux ядра, а в основном для userspace. Чтобы использовать `контрольную группу`, сначала мы должны ее создать. Мы можем создать ее двумя путями.

Первый способ это создать подкаталог в любой подсистеме `sys/fs/cgroup` и добавить pid задачи в `tasks` файл, который будет создан автоматически сразу после того, как мы создадим подкаталог.

Второй способ - создать/уничтожить/управлять `cgroups` с помощью утилит, такие как `libcgroup` библиотека (`libcgroup-tools` в Fedora).

Давайте рассмотрим простой пример. Данный [bash](https://www.gnu.org/software/bash/) скрипт выводит строку в `/dev/tty` устройство, которое представляет контрольный терминал для текущего процесса:

```shell
#!/bin/bash

while :
do
    echo "print line" > /dev/tty
    sleep 5
done
```

При выполнении данного скрипта мы увидим следующее:

```
$ sudo chmod +x cgroup_test_script.sh
~$ ./cgroup_test_script.sh 
print line
print line
print line
...
...
...
```

Теперь давайте посмотрим где `cgroupfs` монтируется в нашей системе. Как мы только что видели, это `/sys/fs/cgroup` директория, но вы можете смонтировать ее куда угодно.

```
$ cd /sys/fs/cgroup
```
Теперь посмотрим подкаталог `devices`, который представляет собой ресурсы, которые разрешают или отклоняют доступ к устройствам задачами в `cgroup`:

```
# cd /devices
```

и создадим `cgroup_test_group` директорию здесь:

```
# mkdir cgroup_test_group
```

После создания `cgroup_test_group` директории, следующие файлы будут сгенерированы здесь:

```
/sys/fs/cgroup/devices/cgroup_test_group$ ls -l
total 0
-rw-r--r-- 1 root root 0 Dec  3 22:55 cgroup.clone_children
-rw-r--r-- 1 root root 0 Dec  3 22:55 cgroup.procs
--w------- 1 root root 0 Dec  3 22:55 devices.allow
--w------- 1 root root 0 Dec  3 22:55 devices.deny
-r--r--r-- 1 root root 0 Dec  3 22:55 devices.list
-rw-r--r-- 1 root root 0 Dec  3 22:55 notify_on_release
-rw-r--r-- 1 root root 0 Dec  3 22:55 tasks
```

В данный момент нас интересуют файлы `devices.deny` и `tasks`. Первые файлы задач должны содержать pid(s) процессов, которые будут прикреплены к `cgroup_test_group.`
Второй файл `devices.deny` содержит список запрещенных устройств. По умолчанию новая группа не имеет никакие ограничения для доступа устройств. Чтобы запретить устройство (в нашем случае это `/dev/tty`) мы должны написать в `devices.deny` следующую строку:

```
# echo "c 5:0 w" > devices.deny
```

Давайте рассмотрим эту строку подробнее. Первая буква `c` представляет тип устройства. В нашем случае `/dev/tty` это `символьное устройство`. Мы можем проверить это из вывода `ls` команды:

```
~$ ls -l /dev/tty
crw-rw-rw- 1 root tty 5, 0 Dec  3 22:48 /dev/tty
```

Посмотрите первую букву `с` в списке разрешений. Следующая часть `5:0` это старшие и младшие номера устройства. Так же вы можете увидеть эти числа в выводе `ls`. И последняя буква `w` запрещает задачам писать в указанное устройство. Давайте запустим `cgroup_test_script.sh` скрипт:

```
~$ ./cgroup_test_script.sh 
print line
print line
print line
...
...
```

и добавить pid этого процесса в `devices/tasks` файл нашей группы:

```
# echo $(pidof -x cgroup_test_script.sh) > /sys/fs/cgroup/devices/cgroup_test_group/tasks
```

В результате этого действия получится:

```
~$ ./cgroup_test_script.sh 
print line
print line
print line
print line
print line
print line
./cgroup_test_script.sh: line 5: /dev/tty: Operation not permitted
```

Похожой принцип работает и при запуске [docker](https://en.wikipedia.org/wiki/Docker_(software)) контейнеров, например:

```
~$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
fa2d2085cd1c        mariadb:10          "docker-entrypoint..."   12 days ago         Up 4 minutes        0.0.0.0:3306->3306/tcp   mysql-work

~$ cat /sys/fs/cgroup/devices/docker/fa2d2085cd1c8d797002c77387d2061f56fefb470892f140d0dc511bd4d9bb61/tasks | head -3
5501
5584
5585
...
...
...
```

И так, во время выполнения `docker` контейнера, `docker` будет создавать `cgroup` для процессов в этом контейнере:

```
$ docker exec -it mysql-work /bin/bash
$ top
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                   1 mysql     20   0  963996 101268  15744 S   0.0  0.6   0:00.46 mysqld
   71 root      20   0   20248   3028   2732 S   0.0  0.0   0:00.01 bash
   77 root      20   0   21948   2424   2056 R   0.0  0.0   0:00.00 top
```

И мы можем увидеть эту `cgroup` на хост машине:

```C
$ systemd-cgls

Control group /:
-.slice
├─docker
│ └─fa2d2085cd1c8d797002c77387d2061f56fefb470892f140d0dc511bd4d9bb61
│   ├─5501 mysqld
│   └─6404 /bin/bash
```

Теперь мы немного знаем о механизме `контрольных групп`, как использовать его вручную и какова его цель. Давайте погрузимся в глубины исходного кода ядра Linux и начнем изучать работу этого механизма.

Ранняя инициализация контрольных групп
--------------------------------------------------------------------------------

Мы немного познакомились с теорией механизма `контрольных групп` Linux ядра, и теперь мы можем начать погружаться в исходный код Linux ядра, чтобы познакомиться с этим механизмом поближе. Как всегда, мы начнем с инициализации `контрольных групп`.  Инициализация `cgroups` делиться на две части в Linux ядре: (early) ранняя и (late) поздняя. В этом разделе мы рассмотрим только `early`, `late` будет рассмотрена в следующих разделах.

Ранняя инициализация `cgroups` начинается с вызова:

```C
cgroup_init_early();
```

функция находится в [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) выполняет early инициализацию Linux ядра. Эта функция определена в [kernel/cgroup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/cgroup.c) файле и начинается с определения следующих двух локальных переменных:

```C
int __init cgroup_init_early(void)
{
	static struct cgroup_sb_opts __initdata opts;
	struct cgroup_subsys *ss;
    ...
    ...
    ...
}
```

`cgroup_sb_opts` структура определенна в таком же исходном файле и выглядит:

```C
struct cgroup_sb_opts {
	u16 subsys_mask;
	unsigned int flags;
	char *release_agent;
	bool cpuset_clone_children;
	char *name;
	bool none;
};
```

Представляет опции монтирования `cgroupfs`. Например, мы можем создать именованную иерархию cgroup (с именем `my_cgrp`) с `name=` опцией и без каких либо подсистем:

```
$ mount -t cgroup -oname=my_cgrp,none /mnt/cgroups
```

Вторая переменная - `ss` имеет тип - структуру `cgroup_subsys` которая определена в [include/linux/cgroup-defs.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/cgroup-defs.h) заголовочном файле и как вы можете догадаться из имени его типа, представляет собой подсистему `cgroup`. Эта система содержит несколько полей и обратных вызовов функций, например:

```C
struct cgroup_subsys {
    int (*css_online)(struct cgroup_subsys_state *css);
    void (*css_offline)(struct cgroup_subsys_state *css);
    ...
    ...
    ...
    bool early_init:1;
    int id;
    const char *name;
    struct cgroup_root *root;
    ...
    ...
    ...
}
```

Где, например, вызываются обратные вызовы `ccs_online` и `ccs_offline` после успешного завершения всех выделений в cgroup и перед его освобождением соответственно. Флаг `early_init` отмечает подсистемы, которые могут/должны быть инициализированы рано. Поля `id` и `name` представляют собой уникальный идентификатор в массиве зарегистрированных подсистем для cgroup и `name` подсистемы соответственно. Последнее - поле `root` представляет собой указатель на корень иерархии cgroup.

Конечно структура `cgroup_subsys` больше, и содержит другие поля, но пока этого достаточно. Теперь когда мы познакомились с важными структурами связанными с механизмом `cgroups`. Давайте вернемся к `cgroup_init_early` функции. Основная цель этой функции - делать первоначальную инициализацию некоторых подсистем. Как вы могли предположить, эти `early` подсистемы должны иметь `cgroup_subsys->early_init = 1`. Давайте посмотрим какие подсистемы могут быть инициализированы первоначально.

После определения двух локальных переменных мы можем увидеть следующие строки кода:

```C
init_cgroup_root(&cgrp_dfl_root, &opts);
cgrp_dfl_root.cgrp.self.flags |= CSS_NO_REF;
```

Здесь мы можем увидеть вызов `init_cgroup_root` функции, которая выполнит инициализацию стандартной объединенной иерархии и после этого мы задаем `CSS_NO_REF` флаг в состоянии этой `cgroup` по умолчанию, чтобы отключить подсчет ссылок для этого css. `cgrp_dfl_root` определен в том же исходном файле:

```C
struct cgroup_root cgrp_dfl_root;
```

Это поле `cgrp` представленное структурой `cgroup`, которое представляет `cgroup` (как вы уже могли догадаться), и определено в файле [include/linux/cgroup-defs.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/cgroup-defs.h). Мы уже знаем, что процесс, который представлен `task_struct` в Linux ядре. `task_struct` не содержит прямую ссылку на `cgroup` куда эта задача прикреплена. Но это можно будет достичь через `ccs_set` поле `task_struct`. Это `ccs_set` структура содержит указатель на массив состояний подсистем:

```C
struct css_set {
    ...
    ...
    ....
    struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
    ...
    ...
    ...
}
```

И через `cgroup_subsys_state`, процесс может сообщить `cgroup` что этот процесс прикреплен к:

```C
struct cgroup_subsys_state {
    ...
    ...
    ...
    struct cgroup *cgroup;
    ...
    ...
    ...
}
```

И так, общая картина `cgroups` структура данных связанна так:

```                                                 
+-------------+         +---------------------+    +------------->+---------------------+          +----------------+
| task_struct |         |       css_set       |    |              | cgroup_subsys_state |          |     cgroup     |
+-------------+         |                     |    |              +---------------------+          +----------------+
|             |         |                     |    |              |                     |          |     flags      |
|             |         |                     |    |              +---------------------+          |  cgroup.procs  |
|             |         |                     |    |              |        cgroup       |--------->|       id       |
|             |         |                     |    |              +---------------------+          |      ....      |
|-------------+         |---------------------+----+                                               +----------------+
|   cgroups   | ------> | cgroup_subsys_state | array of cgroup_subsys_state
|-------------+         +---------------------+------------------>+---------------------+          +----------------+
|             |         |                     |                   | cgroup_subsys_state |          |      cgroup    |
+-------------+         +---------------------+                   +---------------------+          +----------------+
                                                                  |                     |          |      flags     |
                                                                  +---------------------+          |   cgroup.procs |
                                                                  |        cgroup       |--------->|        id      |
                                                                  +---------------------+          |       ....     |
                                                                  |    cgroup_subsys    |          +----------------+
                                                                  +---------------------+
                                                                             |
                                                                             |
                                                                             ↓
                                                                  +---------------------+
                                                                  |    cgroup_subsys    |
                                                                  +---------------------+
                                                                  |         id          |
                                                                  |        name         |
                                                                  |      css_online     |
                                                                  |      css_ofline     |
                                                                  |        attach       |
                                                                  |         ....        |
                                                                  +---------------------+
```

`init_cgroup_root` заполняет `cgrp_dfl_root` со значениями по умолчанию. Следующий этап это первоначальное назначение `ccs_set` к `init_task` который представляет первый процесс в системе:

```C
RCU_INIT_POINTER(init_task.cgroups, &init_css_set);
```

И последним важным моментом в функции `cgroup_init_early` является инициализация `early groups`. Здесь мы проходимся по всем зарегистрированным подсистемам, присваиваем им уникальный идентификационный номер, имя подсистемы и вызываем функцию `cgroup_init_subsys` для тех подсистем, которые помечены как early.

```C
for_each_subsys(ss, i) {
        ss->id = i;
        ss->name = cgroup_subsys_name[i];

        if (ss->early_init)
            cgroup_init_subsys(ss, true);
}
```

Макрос `for_each_subsys` здесь определен в файле исходного кода [kernel/cgroup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/cgroup.c) и просто разворачивается в цикл `for` по массиву `cgroup_subsys`. Определение этого массива может быть найдено в том же самом файле, и оно выглядит немножко необычно:

```C
#define SUBSYS(_x) [_x ## _cgrp_id] = &_x ## _cgrp_subsys,
    static struct cgroup_subsys *cgroup_subsys[] = {
        #include <linux/cgroup_subsys.h>
};
#undef SUBSYS
```

Это определено как макрос `SUBSYS`, который принимает один аргумент (имя подсистемы) и определяет `cgroup_subsys` массив подсистем cgroup. И так же мы можем увидеть что массив инициализирован с содержимым [linux/cgroup_subsys.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/cgroup_subsys.h) файла. Если мы посмотрим содержание этого заголовочного файла мы снова увидим набор макросов `SUBSYS` с полученными именами подсистем:

```C
#if IS_ENABLED(CONFIG_CPUSETS)
SUBSYS(cpuset)
#endif

#if IS_ENABLED(CONFIG_CGROUP_SCHED)
SUBSYS(cpu)
#endif
...
...
...
```

Это работает благодаря `#undef` состоянию после первого определения макроса `SUBSYS`. Посмотрите `&_x ## _cgrp_subsys` выражение. `##` оператор соединяет правое и левое выражение в макросе `C`. Итак, когда мы передаем `cpuset`, `cpu` и т. д., в макрос `SUBSYS`, где-то должны быть определены `cpuset_cgrp_subsys`, `cp_cgrp_subsys`. Так оно и есть. Если вы посмотрите в [kernel/cpuset.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/cpuset.c) файл, вы увидите это определение:

```C
struct cgroup_subsys cpuset_cgrp_subsys = {
    ...
    ...
    ...
    .early_init = true,
};
```

Последний этап - это функция инициализация `cgroup_init_early` подсистем early с вызовом `cgroup_init_subsys` функции. Следующие early подсистемы будут инициализированы:

* `cpuset`;
* `cpu`;
* `cpuacct`.

Функция `cgroup_init_subsys` выполняет инициализацию указанной подсистемы со значениями по умолчанию. Например, устанавливает корень иерархии, выделяет пространство для указанной подсистемы с вызовом функции обратного вызова `css_alloc`, связывает подсистему с родительской, если она существует, добавляет выделенную подсистему к начальному процессу и так далее.

Это все. С этого момента early подсистемы инициализированы.

Заключение
--------------------------------------------------------------------------------

Это завершение первой части которая описывает введение в `Контрольные группы` механизм в Linux ядре. Мы изучили некоторую теорию и первые шаги то, что связано с механизмом `контрольных групп`. В следующей части мы продолжим погружаться в более практические аспекты `контрольных групп`.

**От переводчика: пожалуйста, имейте в виду, что английский - не мой родной язык, и я очень извиняюсь за возможные неудобства. Если вы найдёте какие-либо ошибки или неточности в переводе, пожалуйста, пришлите pull request в [linux-insides-ru](https://github.com/proninyaroslav/linux-insides-ru)**.

Links
--------------------------------------------------------------------------------

* [Контрольные группы](https://en.wikipedia.org/wiki/Cgroups)
* [PID](https://en.wikipedia.org/wiki/Process_identifier)
* [cpuset](http://man7.org/linux/man-pages/man7/cpuset.7.html)
* [Блочные устройства](https://en.wikipedia.org/wiki/Device_file)
* [Объемные страницы](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt)
* [sysfs](https://en.wikipedia.org/wiki/Sysfs)
* [proc](https://en.wikipedia.org/wiki/Procfs)
* [cgroups kernel документация](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)
* [cgroups v2](https://www.kernel.org/doc/Documentation/cgroup-v2.txt)
* [bash](https://www.gnu.org/software/bash/)
* [docker](https://en.wikipedia.org/wiki/Docker_(software))
* [perf events](https://en.wikipedia.org/wiki/Perf_(Linux))
* [Прошлая глава](https://proninyaroslav.gitbooks.io/linux-insides-ru/content/MM/linux-mm-1.html)
