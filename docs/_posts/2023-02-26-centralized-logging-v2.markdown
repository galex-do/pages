---
layout: post
title:  "Централизованное логирование V2"
date:   2023-02-26 12:00:00 +0300
categories: howto
---

Представим, что вы — системный инженер в небольшой IT-компании. У компании десять серверов; некоторые обслуживают бухгалтерию, некоторые используются для разработки и тестирования ПО. Для доступа к ресурсам снаружи закрытой сети используется прокси-сервер. Вы обеспечиваете стабильную работу серверов, анализируете их работу, а в случае сбоев — должны быстро находить причину сбоя.

![Исходная схема сети](https://galex-do.github.io/pages/assets/images/logging_scheme1.png "Исходная схема сети")

В поиске причин вам помогут логи, а системы централизованного логирования позволят собрать логи со всех серверов в одно место и сократят время поиска.

В этом занятии мы освежим то, что уже знаем о логах, а также разберемся, как их агрегировать и анализировать. Если мы качественно настроим централизованное логирование, поиск нужных нам записей на 10 и на 100 серверах будет занимать одинаковое время.

### Что такое логи?

Почти все приложения, которые мы запускаем, автоматически пишут логи. Логи — это записи о событиях, возникающих во время работы программы.

![Логи](https://galex-do.github.io/pages/assets/images/logging_ss.png "Логи")

Работая с логами, мы обычно говорим про анализ произошедших событий. Изучая логи, мы понимаем:

- Когда и почему возникали системные ошибки — ошибки файловой системы, ошибки выделения памяти, проблемы с сетевыми устройствами и драйверами
- Когда и кто подключался к серверам — и какие операции выполнял
- Какие запросы идут на наш HTTP-сервер и с каких адресов
- Какие запросы в базу данных выполняются медленнее всего
- Почему приложение не запускается на заданных настройках

Проанализировав события и частоту их возникновения за определенный период, мы оцениваем, является ли наличие этих событий нормой, и если нет — принимаем решение по исправлению ситуации.

### Проблемы поиска логов

Поиск логов на отдельных серверах в целом является решаемой задачей. Главная проблема возникает, когда в инфраструктуре появляется несколько серверов, параллельно решающих одну и ту же задачу. Возьмем для примера схему, описанную в начале, и представим, что для бесперебойного доступа к нашей сети используется три прокси вместо одного, и балансировщик перед ними.

![Продвинутая схема сети](https://galex-do.github.io/pages/assets/images/logging_scheme2.png "Продвинутая схема сети")

### Централизованный сбор логов

Но чем больше парк серверов или их рабочая нагрузка, тем громче начинают говорить о себе проблемы:

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