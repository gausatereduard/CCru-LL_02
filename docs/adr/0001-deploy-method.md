# ADR-0001. Способ деплоя TimeCapsule — скрипт `deploy.sh`

- Статус: принято
- Дата: 2026-09-25

## Контекст

Нужно регулярно выпускать новые версии TimeCapsule (Laravel-вариант, `msu-50014/timecapsule`) на один инстанс EC2 `webserver` (`t3.micro`, Amazon Linux 2023, `eu-central-1b`): `nginx + PHP-FPM 8.3 + PostgreSQL 15`.
Ограничения: один сервер без балансировщика, публичный репозиторий GitHub, секреты в `.env`, на сервере нет credentials для push, деплой должен делать один человек одной командой с ПК по SSH.

## Решение

Выбран Вариант B — скрипт `~/deploy.sh` на сервере:

```bash
cd /var/www/timecapsule
git pull --ff-only
composer install --no-dev --optimize-autoloader --no-interaction
php artisan migrate --force
sudo systemctl reload php-fpm
```

Запуск: локально `ssh -i ~/.ssh/aws-cloudstudent.pem ec2-user@<Public-IP> ./deploy.sh`. Порядок фиксирован: код → библиотеки → миграции → reload.

## Рассмотренные альтернативы

- Вариант A — ручной деплой по шагам. Плюс: наглядно для обучения. Минус: легко пропустить `composer install` или `migrate`, что быстро приведёт к ошибкам и проблемам.
- Полноценный CI/CD (GitHub Actions + webhook). Плюс: автотриггер, тесты, zero-downtime. Минус: избыточно для одного сервера, сложнее в настройке (secrets, runner, хранение истории).
- Копирование файлов по `scp/rsync` без git. Плюс: просто. Минус: нет версий, `git pull --ff-only` и `git log -1 --oneline` теряются, невозможно использовать `git revert`.

## Последствия

- Плюс: Видно какой коммит на сервере, откат — `git revert + push + deploy.sh` (`scr26.png`).
- Плюс: сервер только читает репозиторий по HTTPS, компрометация не даёт переписать код.
- Минус: Во время `git pull` посетитель может получить смесь старых/новых файлов и увидеть ошибку.
- Минус: нет проверок `php -l`, нет тестов и blue-green.
