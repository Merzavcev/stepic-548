###3.4. Файловая система /proc.

Через файловый интерфейс мы можем получить информацию о процессоре, оперативной
памяти, выполняющихся процессах и т.д.
Чтобы понять, почему это возможно, нам нужно вначале научиться отслеживать
системные вызовы. Посмотрим на программу Hello World, с помощью *ldd* мы можем
отследить зависимость от библиотек:
```
    $ ldd hello
	linux-vdso.so.1 =>  (0x00007ffd4f331000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fa03d168000)
	/lib64/ld-linux-x86-64.so.2 (0x0000557b548dd000)
```
  Применим к ней две
утилиты: **ltrace** и **strace**.

```
    $ ltrace ./hello
    __libc_start_main(0x400526, 1, 0x7ffe4302a9c8, 0x400540 <unfinished ...>
    puts("Hello"Hello
    )                                                                                                        = 6
    +++ exited (status 0) +++
```
Эта утилита показывает, какие *библиотечные* вызовы были сделаны программой
в ходе ее выполнения. Точнее, 
> **ltrace** is a program that simply runs the specified command until it exits.  
> It intercepts and records the dynamic library calls which are called by the executed process  and  the
> signals which are received by that process.  It can also intercept and print the system calls executed by the program.

Вторая утилита:
```
    $ strace ./hello
    execve("./hello", ["./hello"], [/* 75 vars */]) = 0
    brk(NULL)                               = 0xe64000
    access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
    mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fc0041a6000
    access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
    open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
    fstat(3, {st_mode=S_IFREG|0644, st_size=163006, ...}) = 0
    mmap(NULL, 163006, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7fc00417e000
    close(3)                                = 0
    access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
    open("/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
    read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0P\t\2\0\0\0\0\0"..., 832) = 832
    fstat(3, {st_mode=S_IFREG|0755, st_size=1864888, ...}) = 0
    mmap(NULL, 3967488, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fc003bba000
    mprotect(0x7fc003d7a000, 2093056, PROT_NONE) = 0
    mmap(0x7fc003f79000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1bf000) = 0x7fc003f79000
    mmap(0x7fc003f7f000, 14848, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7fc003f7f000
    close(3)                                = 0
    mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fc00417d000
    mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fc00417c000
    mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fc00417b000
    arch_prctl(ARCH_SET_FS, 0x7fc00417c700) = 0
    mprotect(0x7fc003f79000, 16384, PROT_READ) = 0
    mprotect(0x600000, 4096, PROT_READ)     = 0
    mprotect(0x7fc0041a8000, 4096, PROT_READ) = 0
    munmap(0x7fc00417e000, 163006)          = 0
    fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 6), ...}) = 0
    brk(NULL)                               = 0xe64000
    brk(0xe85000)                           = 0xe85000
    write(1, "Hello\n", 6Hello
    )                  = 6
    exit_group(0)                           = ?
    +++ exited with 0 +++
```

С помощью утилиты **strace** мы отслеживаем *системные* вызовы, в частности, мы
видим, что внутри библиотечного вызова *puts* скрывается системный вызов
*write*. 
Также мы видим, что запуск приложения осуществлен с помощью *execve*.

Что происходит при попытке чтения или записи файла? В пользовательском
пространстве мы вызываем функцию printf, которая транслируется в 
вызов write. В ядре у нас есть соответствующий вызов, который мы обозначим sys_write.
Он принимает дескриптор файла, в который пойдет запись, что будет записано,
    и сколько байт нужно записать. Файловый дескриптор в данном случае был создан при запуске
    приложения, когда открывался стандартный вывод на консоль. Если бы мы
    писали в файл, то файловый дескриптор был бы создан при вызове функции open
    или fopen (fopen вызовет open и поместит дескриптор в структуру  FILE). 
    Этому соответствует системный вызов sys_open (sys_create).  

Что именно происходит при вызове sys_open? 
На уровне оборудования у ОС есть жесткий диск, который обслуживается драйвером.
Также у нас есть драйвер файловой системы. open использует имя файла,
и обработчик может определить, к какой конкретно ФС это имя относится,
драйвер этой системы умеет читать соотв. разделы диска. В памяти процесса
таким образуется связь между созданным файловым дескриптором и конкретным
inode.


Как использовать ФС для обращения к памяти? Надо написать драйвер специальной
ФС /proc, который может осуществлять доступ к представлению процессов в ядре.
sys_open может ассоциировать файловый дескриптор с конкретным процессом.
Получив его, мы можем читать нужную информацию, передавая этот дескриптор
стандартным пользовательским функциям типа fgets, etc.

Еще одна деталь: драйвер ФС может быть не один, но все имена файлов
организованы в одну иерархию с общим корнем /.
Различные ФС *монтируются* в какие-то каталоги корневой ФС. Информацию о ФС
можно посмотреть утилитой **mount**.

Теперь мы можем заглянуть в каталог /proc, где для каждого процесса есть свой
каталог, имя которого - это id процесса. Например, **cwd** дает текущий каталог
для процесса ($$ будет заменено на pid текущей сессии bash):
```
$ ls -la "/proc/$$/cwd"
lrwxrwxrwx 1 narn narn 0 Nov  7 10:19 /proc/23983/cwd -> /home/narn
```
Также можно посмотреть окружение (environ), различные лимиты, статистику:
```
    narn@narnur-lenovo:/$ cat "/proc/$$/status"
    Name:	bash
    State:	S (sleeping)
    Tgid:	23983
    Ngid:	0
    Pid:	23983
    PPid:	23320
    TracerPid:	0
    Uid:	1000	1000	1000	1000
    Gid:	1000	1000	1000	1000
    FDSize:	256
    Groups:	4 24 27 30 46 115 131 1000 
    NStgid:	23983
    NSpid:	23983
    NSpgid:	23983
    NSsid:	23983
    VmPeak:	   23764 kB
    VmSize:	   23708 kB
    VmLck:	       0 kB
    VmPin:	       0 kB
    VmHWM:	    6144 kB
    VmRSS:	    6144 kB
    VmData:	    2592 kB
    VmStk:	     136 kB
    VmExe:	     976 kB
    VmLib:	    2312 kB
    VmPTE:	      68 kB
    VmPMD:	      12 kB
    VmSwap:	       0 kB
    HugetlbPages:	       0 kB
    Threads:	1
    SigQ:	0/30391
    SigPnd:	0000000000000000
    ShdPnd:	0000000000000000
    SigBlk:	0000000000010000
    SigIgn:	0000000000380004
    SigCgt:	000000004b817efb
    CapInh:	0000000000000000
    CapPrm:	0000000000000000
    CapEff:	0000000000000000
    CapBnd:	0000003fffffffff
    CapAmb:	0000000000000000
    Seccomp:	0
    Cpus_allowed:	ff
    Cpus_allowed_list:	0-7
    Mems_allowed:	00000000,00000001
    Mems_allowed_list:	0
    voluntary_ctxt_switches:	105
    nonvoluntary_ctxt_switches:	2
```

Не вся информация доступна обычному пользователю по соображениям безопасности.
Также через ФС /proc можно получить информацию об оборудовании, например, 
    об имеющихся ядрах процессора:
```
    $ cat /proc/cpuinfo
```

