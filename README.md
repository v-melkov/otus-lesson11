## Стенд для домашнего занятия "Управление процессами"

1. написать свою реализацию ``ps ax`` используя анализ ``/proc``.    Результат ДЗ: рабочий скрипт который можно запустить
2. написать свою реализацию ``lsof``.  
Результат ДЗ: рабочий скрипт который можно запустить
3. дописать обработчики сигналов в прилагаемом скрипте, оттестировать, приложить сам скрипт, инструкции по использованию. Результат ДЗ: рабочий скрипт который можно запустить + инструкция по использованию и лог консоли
4. реализовать 2 конкурирующих процесса по IO. пробовать запустить с разными ``ionice``. Результат ДЗ: скрипт запускающий 2 процесса с разными ``ionice``, замеряющий время выполнения и лог консоли
5. реализовать 2 конкурирующих процесса по CPU. пробовать запустить с разными ``nice``. Результат ДЗ - скрипт запускающий 2 процесса с разными ``nice`` и замеряющий время выполнения и лог консоли.
------
## Выполним задание №1
##### Проанализируем вывод команды ``ps ax``.
В выводе мы увидим несколько столбцов:
- PID
- TTY
- STAT
- TIME
- COMMAND

##### Для теста запустим на соседнем терминале команду
`sudo tail -f /var/log/messages`  
и узнаем PID запущенной команды (5184 в моем случае).  
``ps ax | grep "sudo tail -f"``

##### Посмотрим откуда берутся данные командой
``strace -e open ps ax``

Часть вывода:  

    open("/proc/5184/stat", O_RDONLY)       = 6
    open("/proc/5184/status", O_RDONLY)     = 6
    open("/proc/5184/cmdline", O_RDONLY)    = 6
    open("/proc/tty/drivers", O_RDONLY)  = 6
Посмотрим содержимое файла ``/proc/5184/stat``  

    cat /proc/5184/stat

Вывод команды:

    5184 (sudo) S 4409 5184 4409 34816 5184 1077944576 1420 296 0 0 0 1 0 0 20 0 1 0 537980 144367616 1151 18446744073709551615 1 1 0 0 0 0 0 0 752135 18446744073709551615 0 0 17 0 0 0 0 0 0 0 0 0 0 0 0 0 0

Смотрим мануал по proc: ``man 5 proc``
Из нужного нам это pid (поле 1), state (поле 3), tty_nr (поле 7), utime (поле 14), stime (поле 15)  

-----
Для поля **PID** используем команду ``cat /proc/5184/stat | awk '{ print $1 }'``    
Вывод:   

    [vagrant@lesson11 ~]$ cat /proc/5184/stat | awk '{ print $1 }'  
    5184
Запишем в переменную PIDVAR

     PIDVAR=`cat /proc/5184/stat | awk '{ print $1 }'`
----
Поле **TTY** выясним используя комманду ``sudo ls -l /proc/5184/fd | head -n2 | tail -n1 | sed 's%.*/dev/%%'``  (*вообще нужно использовать поле 7, но я не программист)*)  
Вывод:

    [vagrant@lesson11 ~]$ sudo ls -l /proc/5184/fd | head -n2 | tail -n1 | sed 's%.*/dev/%%'
    pts/0

Добавим несколько проверок на отсутствие TTY и запишем полученное значение в переменную TTYNUMBER (*проще, наверное, на наличие TTY делать*)

    TTYCOMMAND=`sudo ls -l /proc/5184/fd | head -n2 | tail -n1 | sed 's%.*/dev/%%'`
    if [[ $TTYCOMMAND == "total 0" ]] || [[ $TTYCOMMAND == "null" ]] || [[ $TTYCOMMAND == *"socket"* ]]; then
      TTYNUMBER="?"
    else
      TTYNUMBER=$TTYCOMMAND
    fi

----
Для поля **STAT** используем команду ``cat /proc/5184/stat | awk '{ print $3 }'``  
Вывод:   

    [vagrant@lesson11 ~]$ cat /proc/5184/stat | awk '{ print $3 }'  
    S

Запишем в переменную STATVAR

       STATVAR=`cat /proc/5184/stat | awk '{ print $3 }'`

----

Для поля **TIME** нужно сложить поля **UTIME** (Количество времени, которые данный процесс  провел в режиме пользователя) и **STIME** (Количество времени, которые данный процесс  провел в режиме ядра) и поделить их на значение переменной ядра **CLK_TCK**. Переменную CLK_TCK можно узнать командой ``getconf CLK_TCK``. Запишем результат в переменную TIMEVAR.

    UTIMEVAR=`cat /proc/5184/stat | awk '{ print $14 }'`
    STIMEVAR=`cat /proc/5184/stat | awk '{ print $15 }'`
    CLKTCK=`getconf CLK_TCK`
    FULLTIME=$((UTIMEVAR+STIMEVAR))
    CPUTIME=$((FULLTIME/CLKTCK))
    TIMEVAR=`date -u -d @${CPUTIME} +"%T"`

----

Поле **COMMAND** можно узнать командой ``cat /proc/5184/cmdline | strings -n 1 | tr '\n' ' '``
Вывод:

    [vagrant@lesson11 ~]$ cat /proc/5184/cmdline | strings -n 1 | tr '\n' ' '
    sudo tail -f /var/log/messages [vagrant@lesson11 ~]$

В некоторых случаях поле может быть пустым, тогда возьмем команду из файла /proc/5184/stat (поле 2)

Запишем в переменную COMMANDVAR

       COMMANDVAR=`cat /proc/5184/cmdline | strings -n 1 | tr '\n' ' '`
       if [[ -z $COMMANDVAR]]; then COMMANDVAR=`cat /proc/5184/stat | awk '{ print $2 }'
----

Команда для вывода всех пид процессов с директории ``/proc``: ``ls -l /proc | awk '{ print $9 }' | grep -Eo '[0-9]{1,4}'| sort -n``  
Из вывода ``ls -l`` отбираем только 9 поле с наименованием и оттуда отбираем только наименования из цифр.

----

Итоговый скрипт:

    #!/bin/bash
    echo "PID     TTY     STAT     TIME     COMMAND" # выведем заголовок
    for ITEM in `ls -l /proc | awk '{ print $9 }' | grep -Eo '[0-9]{1,4}'| sort -n | uniq`
    do
    if [ -d /proc/$ITEM/ ]; then  # допусловие проверки существования процесса
      PIDVAR=`cat /proc/$ITEM/stat | awk '{ print $1 }'`
      TTYCOMMAND=`sudo ls -l /proc/$ITEM/fd | head -n2 | tail -n1 | sed 's%.*/dev/%%'`
        if [[ $TTYCOMMAND == "total 0" ]] || [[ $TTYCOMMAND == "null" ]] || [[ $TTYCOMMAND == *"socket"* ]]; then
          TTYNUMBER="?"
        else
          TTYNUMBER=$TTYCOMMAND
        fi

      STATVAR=`cat /proc/$ITEM/stat | awk '{ print $3 }'`
      UTIMEVAR=`cat /proc/$ITEM/stat | awk '{ print $14 }'`
      STIMEVAR=`cat /proc/$ITEM/stat | awk '{ print $15 }'`
      CLKTCK=`getconf CLK_TCK`
      FULLTIME=$((UTIMEVAR+STIMEVAR))
      CPUTIME=$((FULLTIME/CLKTCK))
      TIMEVAR=`date -u -d @${CPUTIME} +"%T"`

      COMMANDVAR=`cat /proc/$ITEM/cmdline | strings -n 1 | tr '\n' ' '`
      if [[ -z $COMMANDVAR ]]; then COMMANDVAR=`cat /proc/$ITEM/stat | awk '{ print $2 }'`; fi

      echo "$PIDVAR     $TTYNUMBER     $STATVAR     $TIMEVAR     $COMMANDVAR"
    fi
    done


----
### Спасибо за проверку!
