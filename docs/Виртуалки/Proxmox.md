# Установка Proxmox VE на голый on Debian 12

 Шаг 1: Обновление системы

???+ example "Команда" 

     apt update && apt upgrade -y

Шаг 2: Добавление репозиториев Proxmox

* Импортируем ключ Proxmox

!!! abstract "Команда" 

     wget https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg -O /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg
    chmod +r /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg

* Добавим репозиторий no-subscription (для бесплатной версии)

!!! info "Команда" 

     echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-install-repo.list

* Обновим список пакетов

Обновим список пакетов

!!! tip "Команда" 

     apt update

Шаг 3: Делаем перезагрузку 

!!! success "Команда" 

     reboot

Шаг 4: Установка Proxmox VE

!!! quote "Команда" 

     apt install proxmox-ve postfix open-iscsi

* В процессе установки будет предложено настроить Postfix — выбирайте "No Config", если не нужна почта.

Шаг 5: Удаление os-prober (опционально, чтобы избежать предупреждений при обновлениях)

!!! question "Команда" 

     apt remove os-prober

Шаг 6: Перезагрузка и доступ к веб-интерфейсу

* Перезагрузите сервер

!!! note "Команда" 

     reboot

* После перезагрузки вы можете войти в веб-интерфейс Proxmox по адресу:

!!! note "Команда" 

     https://IP-сервера:8006



