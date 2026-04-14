# Сессионный проект — Операционные системы семейства Unix (МИФИ, 2026)

**Студент:** Партанский Илья Михайлович, М25-510, зачётная книжка **M2551063**
**Курс:** Операционные системы семейства Unix, весна 2026
**Дата сдачи:** 14.04.2024

---

## Окружение

В качестве «минимальной установки Fedora 43 без графического интерфейса» использован арендованный VPS:

| Параметр | Значение |
|---|---|
| Провайдер | LED1, Санкт-Петербург |
| Гипервизор | KVM (QEMU, i440FX) |
| CPU / RAM | 1 vCPU / 4096 MB |
| Диск | 1 × NVMe 15 GB (единственный, без LVM) |
| Канал | 200 Mbit (fair-share) |
| IPv4 | 194.87.69.56/23, gw 194.87.68.1 |
| IPv6 | 2a00:b700:5::3:315/118 |
| ОС | Fedora Linux 43 (Server Edition), kernel 6.17.1-300.fc43.x86_64 |

Установка выполнялась с официального `Fedora-Server-dvd-x86_64-43-1.6.iso`. Перед первой загрузкой в systemd на этапе anaconda-installer был **отключён root-аккаунт** (стандартная опция «Root account: disabled»), создан пользователь `geles` c правами `wheel`. Вся дальнейшая работа велась по SSH из-под `geles` с локальной Arch-машины.

## Честные отклонения от буквы ТЗ

Задание рассчитано на изолированный lab-стенд в сети 192.168.1.0/24 с выделенным вторым физическим диском. У арендованного VPS эти предпосылки не выполняются, поэтому два пункта были реализованы **функционально эквивалентно**, но с явным осознанным отличием:

### 1. Dummy-интерфейс `mephi0` вместо смены основного NIC

Если буквально сменить IP основного интерфейса `ens18` на 192.168.1.100/24, SSH-связь с сервером обрывается мгновенно (провайдерский шлюз 194.87.68.1 становится недостижим). Чтобы и выполнить требование ТЗ по п. 1.1, и сохранить доступ к машине, создан виртуальный **dummy-интерфейс** `mephi0` со статической конфигурацией:

```
nmcli connection add type dummy con-name mephi0 ifname mephi0 \
    ipv4.method manual \
    ipv4.addresses "192.168.1.100/24,192.168.1.1/32" \
    ipv4.gateway 192.168.1.1 \
    ipv4.dns 8.8.8.8 \
    ipv4.never-default yes \
    ipv4.route-metric 9999 \
    ipv6.method disabled \
    connection.autoconnect yes
```

Нюанс: NetworkManager **автоматически сбросил** значение `ipv4.gateway`, потому что 192.168.1.1 уже был объявлен локальным адресом (192.168.1.1/32 на том же интерфейсе). Это сознательный компромисс — второй адрес нужен для успешного `ping 192.168.1.1` из п. 1.2, иначе пакеты ICMP ушли бы в пустоту через dummy без ARP-ответа. В `ip route` корректно присутствует:

```
192.168.1.0/24 dev mephi0 proto kernel scope link src 192.168.1.100 metric 9999
```

`ipv4.never-default yes` + высокий metric 9999 гарантируют, что реальный default route остаётся через `ens18` (metric 100). После перезагрузки всё поднимается автоматически благодаря `connection.autoconnect yes` и persistent-модулю `/etc/modules-load.d/dummy.conf`.

### 2. Loop-device `/dev/loop0` вместо физического `/dev/sdb`

У VPS нет возможности добавить второй физический диск (провайдер не предоставляет такой API). Создан sparse-файл `/var/lib/mephi/sdb.img` (2 GB), прицепленный как loop-устройство через systemd-юнит `/etc/systemd/system/mephi-sdb.service`:

```
[Unit]
Description=Attach MEPHI virtual disk (simulated /dev/sdb) at /dev/loop0
DefaultDependencies=no
After=local-fs-pre.target
Before=local-fs.target
Conflicts=shutdown.target
Before=shutdown.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=/sbin/modprobe loop
ExecStart=/usr/sbin/losetup -P /dev/loop0 /var/lib/mephi/sdb.img
ExecStop=/usr/sbin/losetup -d /dev/loop0

[Install]
WantedBy=local-fs.target
```

На полученном блочном устройстве `/dev/loop0` создан GPT + один раздел `/dev/loop0p1`, отформатированный в ext4 с меткой `MEPHI_DATA`. Монтирование — через `/etc/fstab` по LABEL с явной зависимостью от сервиса:

```
LABEL=MEPHI_DATA /data/mephi-web ext4 defaults,nofail,x-systemd.requires=mephi-sdb.service 0 2
```

После `reboot` loop-сервис поднимается до `local-fs.target`, `/dev/loop0p1` появляется до парсинга fstab, монтирование проходит штатно. Это проверено вручную после `sudo reboot` — см. запись `journalctl` и вывод `findmnt /data/mephi-web` в `project_history.txt`.

## Ход выполнения по разделам

### Раздел 1. Сеть
- **1.1** `hostnamectl set-hostname mephi-2026.domain.local` — persistent-запись в `/etc/hostname`. Статический IP — на dummy-интерфейсе (см. выше).
- **1.2** `ping -c 4` до 192.168.1.1 и 8.8.8.8 — оба 4/4, см. `network_check.txt`.

### Раздел 2. Пакеты
- **2.1** `sudo dnf install -y nginx tcpdump libcap-ng-utils`. Плюс `policycoreutils-python-utils` — для `semanage`.
- **2.2** `sudo dnf download tcpdump --destdir=/tmp`, затем `sudo rpm -Uvh --replacepkgs /tmp/tcpdump-4.99.6-2.fc43.x86_64.rpm`. Пакет скачан с официального обновляемого репозитория (fedora-updates), сигнатура Fedora GPG (Key ID 829b…5531) проверена `rpm -qi`. Сам RPM-файл — в корне репозитория.

### Раздел 3. ФС и сервисы
- **3.1** Loop-«sdb» + ext4 `MEPHI_DATA` + fstab по LABEL (подробно выше). `stat` → `permissions.txt`, fstab → `fstab.txt`.
- **3.2** `sudo systemctl enable --now nginx`, затем `journalctl -u nginx --since "5 minutes ago"` → `nginx_recent_logs.txt`. Для генерации записей в journal выполнен `systemctl reload nginx`.

### Раздел 4. Управление доступом
- **4.1** DAC:
  - `groupadd mephi-devs`
  - `useradd -m -G mephi-devs mephi-admin`
  - пароль `P@ssw0rd2026` через `passwd mephi-admin`
  - `chown mephi-admin:mephi-devs /data/mephi-web`
  - `chmod 2775 /data/mephi-web` — setgid-бит, видно по `drwxrwsr-x` в `permissions.txt`.
- **4.2** SELinux + capabilities:
  - `semanage fcontext -a -t httpd_sys_content_t '/data/mephi-web(/.*)?'` (persistent fcontext-запись),
  - `restorecon -Rv /data/mephi-web` — применение, контекст виден в `file_contexts.txt`.
  - `chmod u-s /usr/sbin/tcpdump` — снятие SUID (он не стоял по дефолту, команда для идемпотентности).
  - `setcap cap_net_raw,cap_net_admin+ep /usr/sbin/tcpdump`. Результат — `tcpdump_capabilities.txt`.
  - Проверка: `sudo -u mephi-admin tcpdump --help` — работает без `Operation not permitted`.

### Раздел 5. Аутентификация и проверка
- **5.1** PAM + sshd:
  - Создан файл `/etc/ssh/denied_users` с одной строкой `root`.
  - В `/etc/pam.d/sshd` и `/etc/pam.d/login` сразу после шапки `#%PAM-1.0` добавлено:
    ```
    auth       required     pam_listfile.so item=user sense=deny file=/etc/ssh/denied_users onerr=succeed
    ```
    Флаг `onerr=succeed` — защита от self-lockout (если файл случайно удалён или недоступен, PAM не блокирует всех).
  - Дополнительно в `/etc/ssh/sshd_config` проставлено `PermitRootLogin no` — отказ на уровне SSH-протокола, **до** PAM, без запроса пароля.
  - Проверено: попытка `ssh root@194.87.69.56` возвращает `Permission denied` сразу без prompt'а.
- **5.2**
  - Вход: `sudo su - mephi-admin` (login-shell, как того требует формулировка «войдите под пользователем»).
  - Создание страницы: `echo "Hello from Student: M2551063" > /data/mephi-web/index.html`. Благодаря setgid-биту на директории файл получил группу `mephi-devs`. SELinux-контекст унаследован от родителя (`httpd_sys_content_t`) через persistent fcontext-regex.
  - Document root nginx в `/etc/nginx/nginx.conf` изменён на `/data/mephi-web`, `sudo nginx -t && sudo systemctl reload nginx`.
  - `curl http://localhost/` и `curl http://192.168.1.100/` оба возвращают `Hello from Student: M2551063` — см. `curl_output.txt` и скриншот `mephi-nginx-screenshot.png`.

## Проверка persistence

После всего конфигурирования выполнен `sudo reboot`. После переподключения по SSH повторно проверены:

| Что проверяется | Команда | Результат |
|---|---|---|
| hostname | `hostnamectl` | `mephi-2026.domain.local` |
| dummy-интерфейс | `ip -brief addr show mephi0` | `192.168.1.100/24 192.168.1.1/32` |
| loop-диск | `findmnt /data/mephi-web` | `/dev/loop0p1 ext4` |
| ключевые сервисы | `systemctl is-active mephi-sdb nginx sshd` | все `active`, все `enabled` |
| SELinux | `getenforce` | `Enforcing` |
| capabilities | `getcap /usr/sbin/tcpdump` | `cap_net_admin,cap_net_raw=ep` |
| контекст ФС | `ls -ldZ /data/mephi-web` | `httpd_sys_content_t` |
| nginx-раздача | `curl http://localhost/` | `Hello from Student: M2551063` |

Полный лог виден в `project_history.txt` — команды **после** `sudo reboot` начинаются строго с этих проверок.

## Состав репозитория

| Файл | Раздел ТЗ | Содержание |
|---|---|---|
| `project_history.txt` | все | полная `history` bash-сессии (203 строки) |
| `network_check.txt` | 1.2 | `ping -c 4` до 192.168.1.1 и 8.8.8.8, оба 4/4 |
| `fstab.txt` | 3.1 | копия `/etc/fstab` с записью по LABEL |
| `nginx_recent_logs.txt` | 3.2 | `journalctl -u nginx --since "5 minutes ago"` |
| `permissions.txt` | 4.1 | `stat /data/mephi-web`, 2775/drwxrwsr-x |
| `users_groups.txt` | 4.1 | `id mephi-admin` + `getent group mephi-devs` |
| `selinux_status.txt` | 4.2 | `getenforce` → Enforcing |
| `file_contexts.txt` | 4.2 | `ls -Zd /data/mephi-web` с `httpd_sys_content_t` |
| `tcpdump_capabilities.txt` | 4.2 | `getcap /usr/sbin/tcpdump` с cap_net_raw+cap_net_admin=ep |
| `index.html` | 5.2 | web-страница с `Hello from Student: M2551063` |
| `curl_output.txt` | 5.2 | вывод `curl -s http://192.168.1.100` |
| `mephi-nginx-screenshot.png` | 5.2 | скриншот терминала с `curl` |
| `tcpdump-4.99.6-2.fc43.x86_64.rpm` | 2.2 | бинарный RPM, скачанный `dnf download` |
| `README.md` | — | этот файл |
