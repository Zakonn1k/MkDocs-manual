# Установка Debian 12.10.0 (Netinst, XFCE)

**ISO-файл:**  
[debian-12.10.0-amd64-netinst.iso](https://cdimage.debian.org/mirror/cdimage/archive/12.10.0/amd64/iso-cd/debian-12.10.0-amd64-netinst.iso)

---

## УСТАНОВКА

- **Hostname:** `audio`
- **Domain name:** пропускаем
- **Root password:** пропускаем
- **Full name for the new user:** `debian`
- **Username for new account:** `debian`
- **Пароль:** `testtest`
- **Разметка диска:** использовать весь диск
- **Схема разметки:** без разницы
- **Установка программного обеспечения:**  
  Убираем GNOME, выбираем **XFCE**

---

## ПОСЛЕ УСТАНОВКИ

1. Скопировать `full.sh` на рабочий стол
2. Открыть терминал
3. Выполнить команду:
   ```sh
   sh ~/Desktop/full.sh
   ```
4. Ввести пароль `testtest`
5. После окончания скрипта компьютер **выключится**

---

## ВОЗМОЖНЫЕ ОШИБКИ

### 🔁 Зависание в самом начале установки

При загрузке из меню GRUB нужно после строки:

```
linux /install.amd/vmlinuz ...
```

дописать:

```
acpi=off
```

---

### 🛠 Ручная установка загрузчика GRUB

В терминале выбрать `Execute shell`, затем выполнить:

```sh
mount /dev/sda2 /mnt              # Монтируем корневой раздел
mkdir -p /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi     # Монтируем EFI раздел

mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys

chroot /mnt

grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=debian --recheck
update-grub
exit

umount /mnt/dev
umount /mnt/proc
umount /mnt/sys
umount /mnt/boot/efi
umount /mnt
```

---

> ✅ После всех шагов Debian с XFCE будет готов к работе.
