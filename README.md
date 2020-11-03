# Управление процессами

# PID

 Конвейер команд:
```ruby
 $ ls /proc | grep -P ^[0-9] | sort -n | xargs }
```
выведет все папки в /proc, содержищие pid запущенных процессов в системе. Далее нам необходимо обойти все эти папки, заглянуть в вышеописанные файлы и прочитать оттуда сведения о каждом процессе.

# TTY
Смотрим в man, где описаны поля файла /proc/$pid/stat:
```
(7) tty_nr - The controlling terminal of the process. (The minor device number is contained in the combination of bits 31 to 20 and 7 to 0; the major device number is in bits 15 to 8.)
```
Если в этом поле есть какие либо сведения, отличные от 0, то выполняем листинг директории fd просматривая ее на предмет терминала tty или псевдотерминала pts.
```ruby
qq=`ls -l $procpid/fd/ | grep -E '\/dev\/tty|pts' | cut -d\/ -f3,4 | uniq`
Tty=`awk '{ if ($7 == 0) {printf "?"} else { printf "'"$qq"'" }}' /proc/$pid/stat`
```
# STAT

Так же в /proc/$pid/stat нас интересуют следующие сведения, которые мы вытащим с помощью awk (в коде скрипта $3, $6, $8, $19, $20), приводим их к порядку вывода ps (в скобках указан раздел в man, после значение флага):
1. $3 - (3) state; - флаг состояния процесса - D,R,S,T,W,X,Z
2. $19 - (19) nice; флаг приоритета 19 (low priority) to -20 (high priority); flags <,N,
3. if $6==$1 (6) session; флаг, если "начальник" сессии - s
4. $20 (20) num_threads; флаг, если многопоточность, flag - l
5. Флаг блокровки страниц памяти - grep from /proc/$pid/smaps
6. $8 (8) tpgid; процесс выполняется не в фоне, flag - +

В п.5 нам нужно посмотреть внутрь файла /proc/$pid/smaps на предмет флага блокировки.
Собираем все в одну строку с помощью такой конструкции:
```ruby
Stats=`awk '{ printf $3; \
 if ($19<0) {printf "<" } else if ($19>0) {printf "N"}; \
 if ($6 == $1) {printf "s"}; \
 if ($20>1) {printf "l"}}' $procpid/stat; \
 [[ -n $Locked ]] && printf "L"; \
 awk '{ if ($8!=-1) { printf "+" }}' /proc/$pid/stat`
 ```
# TIME
В документации по ps указано, что выводится время жизни процесса состоящее из (14) utime + (15) stime из /proc/$pid/stat, причем значение времени указано в clock ticks. Для того, что бы получить человекочитаемое время, нужно сумму utime и stime разделить на CLK_TCK, которая есть системная переменная getconf CLK_TCK, после чего привести время функцией strftime к виду ЧЧ-ММ.
```ruby
awk -v ticks="$(getconf CLK_TCK)" '{print strftime ("%M:%S", ($14+$15)/ticks)}' /proc/$pid/stat
```
# COMMAND
Ну и наконец, сведения о парметрах запущенного процесса лежат в /proc/$pid/cmdline, man нам говорит:
```This read-only file holds the complete command line for the process, unless the process is a zombie. In the latter case, there is nothing in this file: that is, a read on this file will return 0 characters. The command-line arguments appear in this file as a set of strings separated by null bytes ('\0'), with a further null byte after the last string. ```

```ruby
Cmdline=`awk '{ print $1 }' $procpid/cmdline | sed 's/\x0/ /g'`
[[ -z $Cmdline ]] && Cmdline=`strings -s' ' $procpid/stat | awk '{ printf $2 }' | sed 's/(/[/; s/)/]/'`
```
Не всегда там лежит название процесса, иногда нужно заглянуть в /proc/$pid/stat если в cmdline ничего нет. И побороться с выводом, так как внутри cmdstat разделитель - нулевой байт.

После всех преобразований составляем вывод, близкий к выводу ps. См. файл prc.sh. Файл нужно запустить от рута - sudo chmod +x prc.sh && sudo ./prc.sh.
