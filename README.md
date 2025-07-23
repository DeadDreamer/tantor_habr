# Как мониторить сотни инстансов PostgreSQL и не сойти с ума

## Введение и цели статьи (для кого она будет полезна и что даст читателю)
Читать 12 минут.

Если вы инженер (?) в крупной компании (особенно если ваша органищация поставляет свои услуги в виде SaaS решений), то вам так или иначе надо будет решать задачу мониторинга работы всех ваших PostgreSQL. Очень часто бывает так что на Postgre SQL завязан важный с т.з. финансовых рисков функционал в вашей организации, поэтому крайне желательно не только иметь монитринг, но и получать уведомления когда что-то идет не так (или предиктивно получать уведомления заранее, если что-то пойдет не так в самом ближайшем будущем).
В рамках статьи мы рассмотрим какими способами это можно организовать:
- Все самому с использованием уже ставшего стандартом стека Prometheus + Grafana (на котором можно будет в последующем строить мониторинг любых других ваших сервисов);
- Подключить opensorce сторонние специализированные решения для мониторинга PostgreSQL;
- Использовать специализированные платные решения;
По каждому варианту поймем все плюсы и минусы, чтобы вы могли объективно выбрать именно ваш путь.

## Все самому с помощью Prometheus + Grafana
На связке Prometheus с Grafana строят свои системы мониторинга большинство кампаний. Здесь основное преимущество в том, что есть готовые экспортеры в Prometheus для разных систем и можно быстро развивать и настраивать систему мониторинга подходящую под практически любые нужды. У вас не будет зоопарка разных инструментов, все консистентно и единообразно.
![Мониторинг PosgreSQL с Prometheus и Grafana](/images/prometheus.drawio.png)

Для того чтобы это настроить вам понадобится:
### 1. Установить и настроить postgresql_exporter на сервере PostgreSQL.
https://github.com/prometheus-community/postgres_exporter/  
По сути это агент, который будет собирать метрики PostgreSQL и отдавать их в Prometheus по локальному endpoint (далее Prometheus опрашивает этот адрес каждые n секунд и складывать в свою time series БД).

---
#### Принцип работы postgres_exporter
##### Подключение к PG:
postgres_exporter подключается к PostgreSQL через стандартное подключение (обычно TCP-соединение через порт 5432) и гоняет к нему стандантные SQL запросы за метриками.
По правам он использует специально созданного PG пользователя с правами на чтение информации о состоянии сервера (например, членство в роли pg_monitor).
##### Сбор метрик:
postgres_exporter выполняет SQL-запросы к системным представлениям PostgreSQL (например, pg_stat_database, pg_stat_activity, pg_stat_replication, pg_stat_bgwriter, и т.д.).
Метрики, которые собираются, конвертируются в формат Prometheus.  
Здесь важно иметь в виду, что если не устанавливать в PG ряд дополнительных расширений, то набор метрик будет весьма скудным. Для сбора расширенной информации по метрикам, например по раздутию таблиц, работе индексов и пр. надо будет доставлять еще ряд расширений и дополнительно конфигурировать postgres_exporter. В частности полезно поставить следующие расширения:
- pg_stat_statements стандартное расширение для анализа производительности запросов и статистики;
- pg_buffercache для анализа буферов PostgreSQL;  

Надо иметь в виду, что установка дополнительных расширений будет, в определенных ситуациях, отжирать у вас ресурсы самого PostgreSQL, важно следить за этим.
Плюс простая установка этих расширений не приведет к тому что все необходимое начнет собирать сам postgres_exporter. Надо будет добавлять необходимые запросы в его конфигурации, что может отнять у вас дополнительное время.

##### Экспорт метрик:
Сервис postgres_exporter предоставляет HTTP-endpoint (/metrics), с которого Prometheus регулярно собирает метрики (scrape - т.н. скраппинг постоянны через n секунд).

> Похожим образом работают все другие виды экспортеров для prometheus. У Prometheus есть масса различных экспортеров для разных систем, в частности mysql_exporter, mongodb_exporter, redis_exporter и пр.

---  


> **Для сбора метрик самого хоста вам придется дополнительно установить и настроить node_exporter, который работает схожим образом**

Установив postgres_exporter надо настроить пользака, под которым будет происходить сбор метрик мониторинга:
```
CREATE USER postgres_exporter WITH PASSWORD '****';
ALTER USER postgres_exporter SET SEARCH_PATH TO postgres_exporter,pg_catalog;

-- Права доступа для пользователя мониторинга:
GRANT CONNECT ON DATABASE postgres TO postgres_exporter;
GRANT pg_monitor TO postgres_exporter;

```
Далее необходимо настроить запуск postgres_exporter в виде сервиса (systemd)

### 2. Настройка запуска postgres_exporter в виде сервиса (systemd):
Создаем файлик /etc/systemd/system/postgres_exporter.service:
```
[Unit]
Description=Prometheus PostgreSQL Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=postgres
Group=postgres
Type=simple
ExecStart=/usr/local/bin/postgres_exporter \
  --web.listen-address=0.0.0.0:9187 \
  --web.telemetry-path=/metrics \
  --log.level=info \
  --extend.query-path=/etc/postgres_exporter/queries.yaml \
  DATA_SOURCE_NAME="postgresql://postgres_exporter:exporter_password@localhost:5432/postgres?sslmode=disable"

Restart=always

[Install]
WantedBy=multi-user.target

```
Запустите и добавьте в автозагрузку:

```
systemctl daemon-reload
systemctl start postgres_exporter
systemctl enable postgres_exporter
```
Проверьте доступность метрик по адресу:
http://<IP_postgres_exporter>:9187/metrics


## Готовые opensource решения
Краткий обзор что есть, какие у них минусы и плюсы
bla bla bla

## Платформа Tantor
Все в одном и не только мониторинг
bla bla bla
