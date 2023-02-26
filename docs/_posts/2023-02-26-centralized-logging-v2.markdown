---
layout: post
title:  "Централизованное логирование V2"
date:   2023-02-25 12:00:00 +0000
categories: howto
---

Представим, что вы — системный инженер в небольшой IT-компании. У компании десять серверов; некоторые обслуживают бухгалтерию, некоторые используются для разработки и тестирования ПО. Вы обеспечиваете стабильную работу серверов, анализируете их работу, а в случае сбоев — должны быстро находить причину сбоя.

В поиске причин вам помогут логи, а системы централизованного логирования позволят собрать логи со всех серверов в одно место и сократят время поиска.

В этом занятии мы освежим то, что уже знаем о логах, а также разберемся, как их агрегировать и анализировать. Если мы качественно настроим централизованное логирование, поиск нужных нам записей на 10 и на 100 серверах будет занимать одинаковое время.

### Что такое логи?

Почти все приложения, которые мы запускаем, пишут логи. Логи — это записи о событиях, возникающих во время работы программы.

Программа пишет логи автоматически и сохраняет их в файлы.

Работая с логами, мы обычно говорим про анализ произошедших событий. Изучая логи, мы понимаем:

- Когда и почему возникали системные ошибки (ошибки файловой системы, ошибки выделения памяти, проблемы с сетевыми устройствами и драйверами)
- Когда и кто подключался к серверам — и какие операции выполнял
- Какие запросы идут на наш HTTP-сервер и с каких адресов
- Какие запросы в базу данных выполняются медленнее всего
- Почему приложение не запускается на заданных настройках

Проанализировав события и частоту их возникновения за определенный период, мы оцениваем, является ли наличие этих событий нормой, и если нет - принимаем решение по исправлению ситуации.

### Где их искать?

Системные логи Windows хранятся в `Windows\system32\config`. Их можно просматривать через приложение "Event Viewer" (вызов через командную строку: `eventvwr.msc`). Логи приложений могут храниться в директории приложения или в `AppData\Local` пользователя — информация об этом должна быть в документации приложения.

В Unix-like операционных системах (например, MacOS и семейство Linux) логи текстовые. Для работы с ними достаточно терминала. Как правило, логи стоит искать в `/var/log`. Там же создаются директории для логов приложений.

Например, можем найти там логи авторизации в систему:

{% highlight sh %}
tail /var/log/auth.log

# =>
Jan 23 11:49:37 web-01 systemd-logind[509]: Removed session 1894068.
Jan 23 11:49:42 web-01 sshd[2955520]: Invalid user alex from 124.42.78.202 port 44442
Jan 23 11:49:42 web-01 sshd[2955520]: pam_unix(sshd:auth): check pass; user unknown
Jan 23 11:49:42 web-01 sshd[2955520]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=159.65.34.202
Jan 23 11:49:43 web-01 sshd[2955520]: Failed password for invalid user alex from 124.42.78.202 port 44442 ssh2
Jan 23 11:49:45 web-01 sshd[2955520]: Received disconnect from 124.42.78.202 port 44442:11: Bye Bye [preauth]
Jan 23 11:49:45 web-01 sshd[2955520]: Disconnected from invalid user alex 124.42.78.202 port 44442 [preauth]
Jan 23 11:50:00 web-01 sshd[2955522]: Accepted publickey for ubuntu from 124.42.78.202 port 47368 ssh2: RSA SHA256:#
Jan 23 11:50:00 web-01 sshd[2955522]: pam_unix(sshd:session): session opened for user ubuntu by (uid=0)
Jan 23 11:50:00 web-01 systemd-logind[509]: New session 1894069 of user ubuntu.
{% endhighlight %}

Большую часть системных логов Linux, связанных с функционированием сервисов и приложений, мы найдем в `/var/log/syslog` или в `/var/log/messages`:

{% highlight sh %}
tail /var/log/syslog

# =>
Jan 23 12:07:53 web-01 systemd[1]: Started Session 1905779 of user ubuntu.
Jan 23 12:07:53 web-01 ntpd[545]: Soliciting pool server 91.209.94.10
Jan 23 12:07:54 web-01 ntpd[545]: Soliciting pool server 188.225.9.167
Jan 23 12:07:59 web-01 ntpd[545]: Soliciting pool server 85.254.217.3
Jan 23 12:08:10 web-01 systemd[1]: Started Session 1905780 of user ubuntu.

{% endhighlight %}

### Как их анализировать?

Возьмем пример, для чего эти данные могут пригодиться. Обратимся к `/var/log/auth.log`, записи которого мы просматривали ранее.

Далее воспользуемся текстовым поиском, чтобы найти записи, которые нас интересуют. Поищем записи с `Failed password`:

{% highlight sh %}
tail -n 1000 /var/log/auth.log | grep "Failed password"

# =>
Jan 23 11:52:53 web-01 sshd[2957865]: Failed password for invalid user user from 159.65.34.202 port 42674 ssh2
Jan 23 11:54:27 web-01 sshd[2958758]: Failed password for invalid user nagios from 159.65.34.202 port 55907 ssh2
Jan 23 11:56:05 web-01 sshd[2960142]: Failed password for invalid user steam from 159.65.34.202 port 40907 ssh2
Jan 23 11:57:47 web-01 sshd[2961552]: Failed password for invalid user frappe from 159.65.34.202 port 54139 ssh2
Jan 23 11:59:24 web-01 sshd[2962650]: Failed password for invalid user developer from 159.65.34.202 port 39139 ssh2
{% endhighlight %}

Проанализировав выдачу поиска, можем сделать вывод, что кто-то пытается подобрать пользователя для подключения к серверу. Следовательно, стоит уделить внимание сетевой безопасности.

### Централизованный сбор логов

Копая логи на серверах по отдельности, можно найти много интересного. Но чем больше парк серверов или их рабочая нагрузка, тем громче начинают говорить о себе проблемы:

- Нужно знать, где искать (и в разных операционных системах это бывают разные места)
- Логи на серверах не хранятся слишком долго, чтобы не занимать ценное дисковое пространство. Горизонт поиска ограничен сроком [ротации логов](https://www.oslogic.ru/knowledge/431/rotatsiya-logov-logrotate/)
- Гигабайтные файлы логов высоконагруженных приложений сложно анализировать текстовыми фильтрами
- Нет целостной картины происходящего на серверах
- Сложно связать взаимозависимые события на разных машинах

Бонусом к этому, нам было бы полезно видеть динамику записи логов, чтобы отследить пики нагрузки и подозрительные активности.

Чтобы решить вышеописанные проблемы, используется **централизованное логирование**. Его идея в том, что мы собираем логи со всех наших серверов в единую базу, где можем работать с ними, используя расширенные возможности поисковых движков и инструментов визуализации.

#### Как это работает?

На каждом сервере ставится приложение-**агент** (он же shipper, он же exporter), которое сканирует новые строки в указанных ему лог-файлах. Агент передает собранные логи по сети к агрегатору логов. **агрегатор** принимает логи, обрабатывает их, структурирует и пишет в базу. **Визуализатор** отправляет запросы к базе, чтобы достать нужные логи и показать их в удобном интерфейсе.

![Централизованное логирование](https://galex-do.github.io/pages/assets/images/centralized_logging.png "Централизованное логирование")

Это обобщенная концепция, но в целом она описывает, как происходит сбор и анализ логов в распределенных системах. Похожим образом агенты собирают логи с контейнеров Docker и с подов в Kubernetes.

Агент помечает каждую запись лога тегами. С помощью тегов мы можем отфильтровать извлекаемые из базы записи и получить, к примеру, логи авторизации всех серверов, записанные в прошедшем месяце. агрегатор индексирует данные по тегам — в результате физически такой запрос выполняется значительно быстрее, чем если бы мы собрали все лог-файлы в один и попытались отфильтровать данные текстовым поиском.

#### Решения

Поскольку решения централизованного логирования всегда состоят из нескольких компонентов (агент, агрегатор, визуализатор), описывая их, мы говорим о понятии стека. Стек — несколько программных продуктов, решающих в совокупности задачи сбора, упорядочивания, хранения и анализа логов. Рассмотрим наиболее часто встречающиеся стеки, затем попробуем настроить один из них.

[**ELK-стек**](https://www.elastic.co/elastic-stack/) расшифровывается как Elasticsearch + Logstash + Kibana. Все три продукта, а также многочисленные агенты Beats поддерживает и развивает компания Elasticsearch. У всех продуктов Elasticsearch единая линейка версионирования — зная версию Elasticsearch, вы можете быть уверены, что найдете Logstash, Kibana и Beats той же версии, и они будут полностью совместимы.

Также Elasticsearch поддерживает полностью бесплатную OSS-версию стека (с урезанной функциональностью, но вполне достаточную для основных задач логирования).

ELK весьма популярен как интегрированная платформа для сбора логов в облачных сервисах (Amazon EC2, Elastic Cloud в Google Cloud Platform, Yandex Managed Elasticsearch). ELK можно развернуть на своих сравнительно небольших мощностях; также ELK хорошо умеет масштабироваться горизонтально и годится для логирования крупных высоконагруженных проектов.

Агентами, собирающими логи на местах, в случае ELK выступают Beats (Filebeat, Metricbeat, Auditbeat, Winlogbeat для сбора Event-логов Windows, etc). Logstash принимает "сырые" логи от Beats и с других направлений, фильтрует их, индексирует и складывает в Elasticsearch. Elasticsearch предоставляет базу данных и поисковой движок к ней. А Kibana предоставляет Web-интерфейс, позволяющий визуализировать данные, хранимые в Elasticsearch  

![ELK](https://galex-do.github.io/pages/assets/images/logging_blek.png "ELK")

**EFK-стек** расшифровывается как Elasticsearch + Fluentd + Kibana. [Fluentd](https://docs.fluentbit.io/manual/about/fluentd-and-fluent-bit) заменяет собой связку Beats + Logstash из предыдущей схемы.

Fluentd лучше адаптирован к контейнерной инфраструктуре и чаще применяется в решениях на базе Docker и Kubernetes. В семейство Fluentd также входит легковесный Fluent-bit, потребляющий значительно меньше ресурсов, чем аналогичное решение на базе Beats + Logstash.

![EFK](https://galex-do.github.io/pages/assets/images/logging_fek.png "EFK")

**PLG-стек** расшифровывается как Prometheus + Loki + Grafana, но в контексте централизованного логирования нас интересуют последние два. Стек продуктов с открытым исходным кодом развивает и поддерживает команда [Grafana Labs](https://github.com/grafana). PLG-стек преимущественно написан на языке Go, что делает его компоненты более быстрыми и менее ресурсозатратными, чем компоненты ELK-стека.

В PLG-стеке мы размещаем на серверах агент Promtail, который собирает логи на месте, парсит их, фильтрует и пересылает в Loki. Loki формирует индексы по переданным ему лейблам и складывает данные в Timeseries DB. Grafana использует язык запросов LogQL, чтобы получить от Loki необходимые данные.

![PLG](https://galex-do.github.io/pages/assets/images/logging_plg.png "PLG")

Подход Loki к хранению и индексированию логов принципиально отличается от Elasticsearch. Elasticsearch поддерживает быстрый полнотекстовый поиск, что заставляет его создавать индексы по каждому уникальному полю, встреченному в логах. Loki строит индексы только по определенным полям, а для всего остального использует текстовый поиск и regexp. В совокупности это значит, что:

* (**+**) База Loki занимает меньше места
* (**+**) Loki занимает меньше памяти для хранения индексов
* (**+**) Loki тратит меньше времени на индексацию и более производителен на приеме логов
* (**-**) агрегация и поиск в Loki ограничены назначенными ему лейблами
* (**-**) На больших объемах данных скорость выполнения запросов в Loki проигрывает Elasticsearch

Как итог, стек PLG удобен на небольших и средних проектах. Loki умеет экспортировать логи в объектное хранилище (Amazon S3, GCS) для длительного хранения, что позволяет сэкономить на хранении данных, особенно при использовании облачных платформ.

Если сведем обзор по решениям в табличку, получим примерно такое разделение ответственностей между компонентами

| Стек    | Shipper  | Collector | Database & Search | Visualizer |
|---------|----------|-----------|-------------------|------------|
| ELK     | Beats    | Logstash  | Elasticsearch     | Kibana     |
| EFK     | Fluentd  | Fluentd   | Elasticsearch     | Kibana     |
| Loki    | Promtail | Loki      | Loki              | Grafana    |

И это только самые частые комбинации. В реальности компоненты стеков постоянно развиваются и становятся взаимозаменяемыми. Например, мы [можем использовать Grafana](https://grafana.com/docs/grafana/latest/datasources/elasticsearch/) вместо Kibana, чтобы визуализировать данные Elasticsearch. Или можем заменить в PLG агент Promtail на Fluent-bit — в нем есть готовые парсеры для самых разных логов, а также Fluent-bit хорошо справляется с multiline-логами, что важно, если мы хотим адекватно получать Traceback ошибок с бэкенда на Python или на Go.

При выборе конечного решения стоит учитывать:

- Требования к сбору и обработке логов. Наличие необходимых плагинов у компонентов стека
- Объем генерируемых логов
- Знакомство с инструментами и удобство пользования

### Настроим сбор логов

Теперь попробуем развернуть PLG стек и воспользоваться его возможностями для работы с логами. В примере разворачивать будем на сервере Ubuntu Linux. Компоненты PLG-стека просты в развертывании — достаточно скачать архивы подходящей версии для нужной ОС, распаковать их, поправить конфиги и запустить.

Будем использовать для установки компонентов директорию `/opt`

#### Ставим Loki и Promtail

Заглянем в [Github Loki](https://github.com/grafana/loki/releases) и взглянем на последнюю выпущенную версию приложения.

В "Assets" каждого релиза мы можем найти архивы Loki и Promtail для разных ОС. Скопируем ссылки на подходящие архивы и закачаем их на наш сервер логирования, используя `wget`. Для Ubuntu 20, например, подойдет сборка linux-amd64:

{% highlight sh %}
wget https://github.com/grafana/loki/releases/download/v2.7.1/loki-linux-amd64.zip -P /opt
wget https://github.com/grafana/loki/releases/download/v2.7.1/promtail-linux-amd64.zip -P /opt
{% endhighlight %}

В случае успеха для каждого `wget` увидим что-то подобное:

{% highlight sh %}

# =>
--2023-01-25 10:35:03--  https://github.com/grafana/loki/releases/download/v2.7.1/loki-linux-amd64.zip
Resolving github.com (github.com)... 140.82.121.4
Connecting to github.com (github.com)|140.82.121.4|:443... connected.
HTTP request sent, awaiting response... 302 Found
...
Resolving objects.githubusercontent.com (objects.githubusercontent.com)... 185.199.108.133, 185.199.111.133, 185.199.110.133, ...
Connecting to objects.githubusercontent.com (objects.githubusercontent.com)|185.199.108.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 18050163 (17M) [application/octet-stream]
Saving to: ‘/opt/loki-linux-amd64.zip’

loki-linux-amd64.zip                               100%[===============================================================================================================>]  17.21M  37.9MB/s    in 0.5s    

2023-01-25 10:35:04 (37.9 MB/s) - ‘/opt/loki-linux-amd64.zip’ saved [18050163/18050163]

{% endhighlight %}

Теперь у нас есть zip-архивы в `/opt`.

{% highlight sh %}
ls /opt
loki-linux-amd64.zip  promtail-linux-amd64.zip
{% endhighlight %}

Чтобы распаковать их, на Ubuntu нам понадобится утилита unzip. Если её нет — она легко ставится через пакетный менеджер apt:

{% highlight sh %}
sudo apt update && sudo apt install unzip

# =>
...
Suggested packages:
  zip
The following NEW packages will be installed:
  unzip
0 upgraded, 1 newly installed, 0 to remove and 183 not upgraded.
Need to get 168 kB of archives.
After this operation, 593 kB of additional disk space will be used.
Get:1 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 unzip amd64 6.0-25ubuntu1.1 [168 kB]
Fetched 168 kB in 0s (7612 kB/s)
Selecting previously unselected package unzip.
(Reading database ... 65530 files and directories currently installed.)
Preparing to unpack .../unzip_6.0-25ubuntu1.1_amd64.deb ...
Unpacking unzip (6.0-25ubuntu1.1) ...
Setting up unzip (6.0-25ubuntu1.1) ...
Processing triggers for mime-support (3.64ubuntu1) ...
Processing triggers for man-db (2.9.1-1) ...
{% endhighlight %}

Распакуем архивы в директорию logging:

{% highlight sh %}
unzip loki-linux-amd64.zip -d /opt/logging
unzip promtail-linux-amd64.zip -d /opt/logging
{% endhighlight %}

Посмотрим, что получилось:

{% highlight sh %}
ls /opt/logging/
loki-linux-amd64  promtail-linux-amd64
{% endhighlight %}

Мы скачали и распаковали исполняемые файлы Loki и Promtail. Осталось их настроить и запустить.

Базовый конфиг Loki можем взять из [официального github](https://github.com/grafana/loki/blob/main/cmd/loki/loki-local-config.yaml) — его raw-версии будет достаточно для тестовых целей. Пойдем в директорию `/opt/logging` и скачаем его:

{% highlight sh %}
cd /opt/logging && wget https://raw.githubusercontent.com/grafana/loki/main/cmd/loki/loki-local-config.yaml
{% endhighlight %}

На данный момент в конфиге нам интересен только параметр http_listen_port — это порт, на который promtail будет присылать логи.

Запустим Loki и посмотрим, что он скажет:

{% highlight sh %}
./loki-linux-amd64 -config.file=loki-local-config.yaml

# =>
level=info ts=2023-01-25T11:15:28.818425004Z caller=main.go:103 msg="Starting Loki" version="(version=HEAD-e0af1cc, branch=HEAD, revision=e0af1cc8a)"
level=info ts=2023-01-25T11:15:28.818858122Z caller=server.go:323 http=[::]:3100 grpc=[::]:9096 msg="server listening on addresses"
level=warn ts=2023-01-25T11:15:28.819434198Z caller=cache.go:114 msg="fifocache config is deprecated. use embedded-cache instead"
level=warn ts=2023-01-25T11:15:28.819453316Z caller=experimental.go:20 msg="experimental feature in use" feature="In-memory (FIFO) cache - chunksembedded-cache"
level=info ts=2023-01-25T11:15:28.819769758Z caller=table_manager.go:262 msg="query readiness setup completed" duration=1.232µs distinct_users_len=0
level=info ts=2023-01-25T11:15:28.81979252Z caller=shipper.go:127 msg="starting index shipper in RW mode"
level=info ts=2023-01-25T11:15:28.819844879Z caller=shipper_index_client.go:78 msg="starting boltdb shipper in RW mode"
...
level=info ts=2023-01-25T11:15:29.015743752Z caller=loki.go:402 msg="Loki started"
level=info ts=2023-01-25T11:15:32.015764027Z caller=scheduler.go:682 msg="this scheduler is in the ReplicationSet, will now accept requests."
level=info ts=2023-01-25T11:15:32.015797446Z caller=worker.go:209 msg="adding connection" addr=127.0.0.1:9096
level=info ts=2023-01-25T11:15:33.940775675Z caller=compactor.go:407 msg="this instance has been chosen to run the compactor, starting compactor"
level=info ts=2023-01-25T11:15:33.940883466Z caller=compactor.go:436 msg="waiting 10m0s for ring to stay stable and previous compactions to finish before starting compactor"
level=info ts=2023-01-25T11:15:39.016133708Z caller=frontend_scheduler_worker.go:101 msg="adding connection to scheduler" addr=127.0.0.1:9096
{% endhighlight %}

Оставим Loki работать и откроем новую сессию терминала. Теперь займемся Promtail. Его дефолтный конфиг мы [тоже можем найти на Github](https://github.com/grafana/loki/blob/main/clients/cmd/promtail/promtail-local-config.yaml). Скачаем его:

{% highlight sh %}
cd /opt/logging && wget https://raw.githubusercontent.com/grafana/loki/main/clients/cmd/promtail/promtail-local-config.yaml
{% endhighlight %}

И немного доработаем в редакторе.

В clients мы видим адрес агрегатора Loki, на который Promtail будет транслировать логи. В scrape_configs описаны задачи, выполняемые агентом на сервере.

Условно, каждый отдельный job в Promtail это:

- Список целей — откуда Promtail получает логи
- Правила, как он обрабатывает полученные логи перед отправкой (pipeline_stages).

Блок pipeline_stages необязателен — Promtail может никак не обрабатывать логи, а просто передавать их Loki. Но мы попробуем добавить фильтры, чтобы получить представление о том, как он работает.

Уберем из файла стандартный job `system`, который просто транслирует Loki все, что найдет в `/var/log`, оканчивающееся на `log`. Добавим туда свои job для syslog и логов авторизации.

{% highlight yaml %}
static_configs:

- job_name: syslog
  static_configs:
  - targets:
      - localhost
    labels:
      job: syslog
      __path__: /var/log/syslog
  pipeline_stages:
    - regex:
        expression: '(?P<time>\S* \S* \S*) (?P<host>\S*) (?P<service>\S*)\[.*\]: (?P<msg>.*)'
    - labels:
        host:
        service:

- job_name: auth
  static_configs:
  - targets:
      - localhost
    labels:
      job: auth
      __path__: /var/log/auth.log
  pipeline_stages:
    - match:
        selector: '{ filename="/var/log/auth.log" } !~ "Failed password"'
        action: drop
{% endhighlight %}

Теперь разберемся, что происходит в новых job.

В `syslog` мы собираем события только из файла `/var/log/syslog`. Каждую строку мы парсим с помощью регулярного выражения. Для полей `host` и `service` Promtail создает отдельные теги, по которым мы сможем выполнять поиск в Loki.

Promtail, увидев, например, строку лога:

{% highlight sh %}
Jan 25 11:38:23 node-1 sshd[1905]: Disconnected from authenticating user root 81.68.93.197 port 36058 [preauth]
{% endhighlight %}

разберет её с помощью regex следующим образом:

- *time*: `Jan 25 11:38:23`
- *host*: `node-1`
- *service*: `sshd`
- *msg*: `Disconnected from authenticating user root 81.68.93.197 port 36058 [preauth]``

В `host` и `service` у логов часто будут повторяющиеся значения, поэтому будет удобно добавить теги (и индексы) по ним, чтобы ускорить поиск.

Далее, в `auth` мы собираем события только из `/var/log/auth.log`. При этом все записи, которые не содержат строку "Failed password" — по-умолчанию отбрасываем и не отправляем в Loki. Можно сказать, что мы затачиваем этот job строго под сценарий отслеживания подбора пароля к серверам. Такой подход в логировании позволяет отсеять большую часть рутинных записей логов и оставить в хранилище информацию только о тех событиях, которые нам интересны. И это довольно выгодно с точки зрения оптимизации объема базы с логами!

Итак, запустим Promtail с доработанной конфигурацией:

{% highlight sh %}
./promtail-linux-amd64 -config.file=promtail-local-config.yaml

# =>
level=info ts=2023-01-25T13:34:08.707415107Z caller=promtail.go:123 msg="Reloading configuration file" md5sum=88b3a612a80529e6876a275d9a7e3c44
level=info ts=2023-01-25T13:34:08.708475807Z caller=server.go:323 http=[::]:9080 grpc=[::]:42285 msg="server listening on addresses"
level=info ts=2023-01-25T13:34:08.709451383Z caller=main.go:171 msg="Starting Promtail" version="(version=HEAD-e0af1cc, branch=HEAD, revision=e0af1cc8a)"
level=warn ts=2023-01-25T13:34:08.70957955Z caller=promtail.go:220 msg="enable watchConfig"
level=info ts=2023-01-25T13:34:13.710350779Z caller=filetargetmanager.go:352 msg="Adding target" key="/var/log/syslog:{job=\"syslog\"}"
level=info ts=2023-01-25T13:34:13.71042411Z caller=filetargetmanager.go:352 msg="Adding target" key="/var/log/auth.log:{job=\"auth\"}"
level=info ts=2023-01-25T13:34:13.710472081Z caller=filetarget.go:282 msg="watching new directory" directory=/var/log
level=info ts=2023-01-25T13:34:13.71060101Z caller=filetarget.go:282 msg="watching new directory" directory=/var/log
ts=2023-01-25T13:34:13.710669889Z caller=log.go:168 level=info msg="Seeked /var/log/auth.log - &{Offset:75645 Whence:0}"
level=info ts=2023-01-25T13:34:13.710726299Z caller=tailer.go:143 component=tailer msg="tail routine: started" path=/var/log/auth.log
ts=2023-01-25T13:34:13.710778611Z caller=log.go:168 level=info msg="Seeked /var/log/syslog - &{Offset:124303 Whence:0}"
level=info ts=2023-01-25T13:34:13.710823151Z caller=tailer.go:143 component=tailer msg="tail routine: started" path=/var/log/syslog
{% endhighlight %}

Видим, что Promtail начал извлекать логи по указанным нами путям. В то же время, если заглянем в лог Loki, то увидим, что он начал складывать получаемые данные в таблицы

{% highlight sh %}
level=info ts=2023-01-25T13:34:40.897589776Z caller=table_manager.go:166 msg="handing over indexes to shipper"
level=info ts=2023-01-25T13:34:40.897619921Z caller=table_manager.go:134 msg="uploading tables"
level=info ts=2023-01-25T13:35:40.898170816Z caller=table_manager.go:134 msg="uploading tables"
level=info ts=2023-01-25T13:35:40.898161387Z caller=table_manager.go:166 msg="handing over indexes to shipper"
level=info ts=2023-01-25T13:36:40.897893225Z caller=table_manager.go:166 msg="handing over indexes to shipper"
level=info ts=2023-01-25T13:36:40.897899193Z caller=table_manager.go:134 msg="uploading tables"
{% endhighlight %}

То есть, мы уже собираем логи, фильтруем их и сохраняем в базу. Осталось разобраться, как ходить к этой базе и доставать из неё то, что нам нужно.

#### Ставим Grafana

Откроем новую сессию терминала. Пойдем на [сайт Grafana](https://grafana.com/grafana/download?edition=oss) с опубликованными OSS релизами и скачаем подходящий для нашей ОС архив на сервер:

{% highlight sh %}
wget https://dl.grafana.com/oss/release/grafana-9.3.4.linux-amd64.tar.gz -P /opt
{% endhighlight %}

Перейдем в `/opt` и распакуем скачанный архив:

{% highlight sh %}
cd /opt && tar -xzvf grafana-9.3.4.linux-amd64.tar.gz

# =>
grafana-9.3.4/LICENSE
grafana-9.3.4/README.md
grafana-9.3.4/NOTICE.md
grafana-9.3.4/VERSION
grafana-9.3.4/bin
grafana-9.3.4/bin/grafana-cli
grafana-9.3.4/bin/grafana-cli.md5
grafana-9.3.4/bin/grafana-server
...
{% endhighlight %}

После распаковки можем видеть в `/opt` папку grafana выбранной версии.

{% highlight sh %}
ls /opt
grafana-9.3.4  grafana-9.3.4.linux-amd64.tar.gz  logging  loki-linux-amd64.zip  promtail-linux-amd64.zip
ls /opt/grafana-9.3.4
bin  conf  LICENSE  NOTICE.md  plugins-bundled  public  README.md  scripts  VERSION
{% endhighlight %}

Все самое интересное находится в директории `conf`, но плюсом Grafana является то, что она поставляется готовой к употреблению. Нам нужно только запустить её, найдя исполняемый файл в `bin`:

{% highlight sh %}
cd /opt/grafana-9.3.4/bin && ./grafana-server

# =>
Grafana server is running with elevated privileges. This is not recommended
INFO [01-25|13:56:34] Starting Grafana                         logger=settings version=9.3.4 commit=ecf4e45659 branch=HEAD compiled=2023-01-24T14:03:52Z
INFO [01-25|13:56:34] Config loaded from                       logger=settings file=/opt/grafana-9.3.4/conf/defaults.ini
INFO [01-25|13:56:34] Path Home                                logger=settings path=/opt/grafana-9.3.4
INFO [01-25|13:56:34] Path Data                                logger=settings path=/opt/grafana-9.3.4/data
INFO [01-25|13:56:34] Path Logs                                logger=settings path=/opt/grafana-9.3.4/data/log
INFO [01-25|13:56:34] Path Plugins                             logger=settings path=/opt/grafana-9.3.4/data/plugins
INFO [01-25|13:56:34] Path Provisioning                        logger=settings path=/opt/grafana-9.3.4/conf/provisioning
INFO [01-25|13:56:34] App mode production                      logger=settings
...
{% endhighlight %}

Теперь, в зависимости от того, где мы настраиваем наше централизованное логирование, мы должны обнаружить Grafana в браузере по адресу:

* http://localhost:3000, если устанавливаем сервисы локально
* http://$SERVER_IP:3000, если устанавливаем на удаленном сервере

[![Grafana Auth](https://galex-do.github.io/pages/assets/images/grafana_hello.png "Grafana Auth")](https://galex-do.github.io/pages/assets/images/grafana_hello.png)

#### Связываем Grafana и Loki

Авторизуемся в Grafana (при первом входе: *admin:admin*), находим раздел Configuration и страницу Data sources в нем.

[![Grafana DS](https://galex-do.github.io/pages/assets/images/grafana_datasources.png "Grafana DS")](https://galex-do.github.io/pages/assets/images/grafana_datasources.png)

Нажимаем на кнопку "Add Data Source" и выбираем из списка предложенных вариантов Loki.

При настройке нового Data source нам нужно только прописать в поле URL то, что Grafana подсказывает: `http://localhost:3100`. Loki работает на том же сервере, что и Grafana, поэтому Grafana найдет его на указанном порту.

[![Grafana DS](https://galex-do.github.io/pages/assets/images/grafana_ds_loki.png "Grafana DS")](https://galex-do.github.io/pages/assets/images/grafana_ds_loki.png)

Нажав внизу "Save & Exit", мы должны увидеть сообщение: `Data source connected and labels found.`

Пришло время отправиться в раздел Explore и посмотреть на логи, которые сохранились в базу.

[![Grafana explore](https://galex-do.github.io/pages/assets/images/grafana_explore.png "Grafana explore")](https://galex-do.github.io/pages/assets/images/grafana_explore.png)

Grafana использует язык запросов LogQL, чтобы обращаться к данным Loki и получать нужные записи. В последних версиях Grafana в Explore появился конструктор, который позволяет не углубляться в LogQL и получать нужные данные, заполнив несколько форм.

[![Grafana explore](https://galex-do.github.io/pages/assets/images/grafana_explore_labels.png "Grafana explore")](https://galex-do.github.io/pages/assets/images/grafana_explore_labels.png)

Для начала посмотрим, какие логи Promtail прислал нам с меткой `job=auth` за последние 15 минут:

[![Grafana explore](https://galex-do.github.io/pages/assets/images/grafana_explore_auth.png "Grafana explore")](https://galex-do.github.io/pages/assets/images/grafana_explore_auth.png)

Как видим, в результатах действительно только записи, содержащие "Failed password", и мы даже можем наблюдать интенсивность таких событий на временном срезе. Grafana располагает встроенным механизмом алертов, и технически мы можем добавить алерт на слишком частое появление события с Failed password, который будет присылать нам уведомление об этом на почту или в Telegram.

Посмотрим, что нам предлагается по тегу service, который Promtail извлек с помощью regexp:

[![Grafana explore](https://galex-do.github.io/pages/assets/images/grafana_explore_services.png "Grafana explore")](https://galex-do.github.io/pages/assets/images/grafana_explore_services.png)

Promtail нашел все уникальные значения сервисов в `/var/log/syslog` и добавил их в лейбл service. Теперь их можно использовать для поиска по сервису в Loki. Если идти дальше в настройке Promtail, мы можем расширить pipeline_stage и мониторить только некоторые сервисы из системного лога и дропать, например, все, что не принадлежит семейству systemd.

Давайте достанем логи c `service=systemd`, в которых упоминается `Network`.

[![Grafana explore](https://galex-do.github.io/pages/assets/images/grafana_explore_systemd.png "Grafana explore")](https://galex-do.github.io/pages/assets/images/grafana_explore_systemd.png)

Мы можем агрегировать записи по лейблам. Возьмем системные логи с `host=node-1` (единственным в нашем случае), а далее посчитаем сколько в среднем событий событий на интервале минуты генерирует каждый сервис.

[![Grafana explore](https://galex-do.github.io/pages/assets/images/grafana_explore_aggregate.png "Grafana explore")](https://galex-do.github.io/pages/assets/images/grafana_explore_aggregate.png)

Сейчас мы скорее всего увидим один большой пик - это момент, когда Promtail запустился и передал в Loki все записи, которые нашел в указанных ему логах. На более долгом интервале пользования такая статистика станет более наглядной.

Конструктор (если вы им пользуетесь) при сборе любого запроса формирует также raw LogQL query. Для последнего примера запрос выглядит так: `sum by(service) (rate({host="node-1"} [1m]))`. Удобство LogQL в том, что Grafana везде воспринимает его однозначно. Мы можем:

- Взять этот запрос
- Создать дашборд Grafana
- Создать панель
- Вставить в Code запроса к Loki наш запрос
- Получить ту же картинку и мониторить частоту возникновения событий в динамике

[![Grafana dashboard](https://galex-do.github.io/pages/assets/images/grafana_copyquery.png "Grafana dashboard")](https://galex-do.github.io/pages/assets/images/grafana_copyquery.png)

На этом можно сказать, что наш централизованный мониторинг готов к работе. Мы получили логи с условного удаленного хоста в удобном для нас формате и смогли искать нужные нам данные через веб-интерфейс Grafana. Дальше только ставить Promtail на новые серверы, настраивать pipelines, добавлять дашборды и алерты.

### В итоге

Итак, на занятии мы:

1. Вспомнили, зачем нужны логи и где они хранятся
2. Разобрались с концепцией централизованного логирования и зачем оно нужно:
    * Собираем логи в единую базу
    * Используем удобные инструменты поиска
    * Видим картину логов в целом и можем связать события на разных серверах
    * Видим динамику событий в логах и отслеживаем аномалии
3. Познакомились с популярными решениями и инструментами централизованного логирования:
    * ELK, если хотим легко и быстро искать по огромному количеству логов, готовы немного покопаться и не стесняемся в средствах
    * Loki + Grafana, если нужно быстро поднять и не тратить на поддержку много ресурсов
    * Fluentd/Fluent-bit, если хотим много плагинов для парсинга логов или думаем о сборе логов в контейнерной инфраструктуре
    * Другие комбинации, если хотим взять сильные стороны из разных решений
4. Настроили централизованное логирование с помощью Grafana и Loki
5. Познакомились с языком запросов LogQL

Знание о том, как организовать централизованное логирование, позволит вам упростить контроль событий в больших кластерах серверов, а также контейнерной инфраструктуре, такой как Docker и Kubernetes.

### Ссылки

* [Сравнение Fluentd и Logstash](https://signoz.io/blog/fluentd-vs-logstash/)
* [Сравнение ELK и PLG стеков](https://habr.com/ru/company/southbridge/blog/510822/)
* [Сравнение Elasticsearch и Loki](https://habr.com/ru/company/badoo/blog/507718/)
* [Документация LogQL](https://grafana.com/docs/loki/latest/logql/)
