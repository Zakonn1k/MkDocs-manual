# Прошивка Cisco Catalyst 3850

Просмотр файлов на флешке 

``` bash
sh usbflash0:
```

Смотрим место на Cisco
``` bash
sh flash:
```
Удаляем лишние пакеты 
``` bash
install remove inactive
```
## Вариант с флешкой 
 Подготовка флешки для прошивки 
``` 
diskpart
list disk
select disk <INDEX OF USB DISK>
clean
convert mbr
create part primary size=4000
active
```
### Копируем файл с флешки в память Cisco 
Копируем файл 
``` bash
copy usbflash0:cat3k_caa-universalk9.16.12.12.SPA.bin flash:
```
## Вариант с HTTP-сервером

🔹 Шаг 1. Скачай прошивку на ПК
Пусть файл лежит, например, в папке:

``` ps1
C:\firmware
```
(название файла: cat3k_caa-universalk9.16.12.13.SPA.bin)

🔹 Шаг 2. Запусти HTTP-сервер
В Windows устанавливем Python
запускаем HTTP-сервер в PowerShell:
``` powershell
cd C:\firmware
python -m http.server 8080
```
После этого твой ПК начнёт раздавать файлы по адресу: http://<IP_ТВОЕГО_ПК>:8080/
👉 <IP_ТВОЕГО_ПК> — это IP-адрес твоей машины в сети, где видит Cisco (например 192.168.1.50).

🔹 Шаг 3. На Cisco качаем файл
Подключись к коммутатору и выполни:
``` bash
copy http://192.168.1.50:8080/cat3k_caa-universalk9.16.12.13.SPA.bin flash:
```
🔹 Шаг 4. Проверка
``` bash
dir flash:
```
Убедись, что файл появился полностью.

⚡ Важно: на время загрузки не закрывай PowerShell/оконный терминал, где крутится python -m http.server.

Проверяем контрольную сумму 
``` bash
verify /md5 flash:cat3k_caa-universalk9.16.12.12.SPA.bin
```
Меняем конфиг
``` bash
conf t
boot system flash:packages.conf		
```
или названия прошивки boot system flash:cat3k_caa-universalk9.16.12.12.SPA.bin
если не работает команда install
``` bash
no boot manual
exit
wr
```
Проверяем загрузку 
``` bash
sh boot
install add file flash:cat3k_caa-universalk9.16.12.12.SPA.bin activate commit
wr
```
Eсли не сработало пробуем 
``` bash
request platform software package install switch all file flash:cat3k_caa-universalk9.16.12.13.SPA.bin on-reboot new auto-copy
wr
reload
```
