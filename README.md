# Использование k8s в IaaS
Егор Баяндин, технический директор S7 Travel Retail

Самолеты зеленые, теперь будут и ракеты запускать.
Black Friday коснулся и S7, началась распродажа.

История про развитие ecommerce в S7. В 18 году перешли на новую версию ИТ платформы, S7 мигрировала в другую систему бронирования, проект миграции занял около года.

Предпосылки:
* до 2012 дедики - месяцы поставки
* 2012 - приватные vmware - дни
* 2014 iaas - часы, миграция заняла пару месяцев, используют имена серверов, а не ip и прошло гладко
* 2018 - контейнеры, появился k8s кластер в облаке

Сформулировали список условий (pre-condition):
* Stateless приложения
* Service/microservices arch
* Репозиторий для контейнеров
* Централизованное логирование и мониторинг
* Единое договоренности по сетевой структуре, появился зоопарк, перед запуском в prod договорились о общих подходах (на самом 100% не договорились, трафик заворачивают через traefik, в том числе и чтобы на лету менять конфиги)
* Автоматизация поставок - желательно, можно и ручками, но для скорости

Варианты:
* baremetal, на физические сервера возвращаться не хотелось
* iaas, ручками настраивать k8s + api vmware, был рабочий вариант, пока не наткнулись на
* CSE (ProtonOS масштабирование из коробки) - продукт от vmware, 

CSE - container server extension, расширяет vcloud api для управления жизненного цикла k8s кластеров, шаблоны

Результаты - все счастливы:
* разработчики - не надо выпрашивать сервера
* админы не надо отвлекаться от важных задач
* бизнес - уменьшение ttm, экономия на ресурсах, вся инфра сейчас 300 серверов

Но это не серебряная пуля:
* БД и персистетные хранилища на выд серверах
* Внешний мониторинг, чтобы 
* Все непросто с критичными по безопасности системами и сервисами , получилось договориться с security, с персональными данными 

Что уже удалось:
* 4 кластера в 2 дц - прод и дев
* Система контроля продаж
* Сервис предоставления мин цены
* Сервис для чатов, backend (веб сокет сервер, клиенты с сайта и с моб приложения)
и + парочка систем на подходе (персональный кабинет, изначально проектировалась под k8s, и портал s7, пока монолит, но планируют распилить)

Вопросы:
* Почему не baremetal?
С железом не сложилось, доп расходы - заказы, эксплуатация. В CSE низкий порог входа, за 40 минут подняли первый контейнер, после чтения документации.
Арендуют ресурсы, а не железки. Есть только одна железка - cisco router.

* А что про pci-dss?
Номера карт не хранят, только данные передают, они сертиф по pci и они тоже в облаке

* Примеры приложений?
Смотри выше, докладчик про них говорил.

* Стек технологий для мониторинга и логов?
ELK, filebeat+logstash+elastic+kibana+grafana
Мониторинг, до k8s жили на nagios, смотрят в сторону prometheus+zabbix, решение пока не нравится, есть над чем поработать.

# Kubernetes для тех кому за 30
Николай Сивко, Okmeter

Докладчик быстро рассказывал, не успел все записать. Слайды тут https://ict.moscow/static/Devops_2.pdf, позже по нему сделаю расшифровку.

Okmeter - это сервис мониторинга, есть агенты и серверная часть:
* 10+ сервисов на go/python
* kafka, cassandra, elastic, postresql
* bare metal + private network, так исторически сложилось и нравится, в облако ехать не хотят
* нет ci/cd
* большая нагрузка и постоянно масштабироваться (раз в два месяца)

Проблемы с ростом:
1 итерация - На каждом сервере запущено все (cassandra, elastic, сервисы), управление через ansible.

2 итерация - вынесли elastic на отдельные машины, нужны ресурсы.

3 итерация - in memory cash метрик, нужны ресурсы и тд.

Проблемы:
* сложно управлять в ansible
* маппинг server-roles сложно 
* сложно делать бесшовый деплой
* полный прогон почти когда не делается, можно состариться, используют теги

Управление ресурсами. Куда пристраивать новый сервис? Смотрели глазами, выбирали вручную делезки, смотрели inventory и в мониторинге. И вроде как в k8s это уже есть, request+limit то что нужно. Ввязываемся, чтобы *управлять ресурсами*.

Требования:
* Отказоустойчивость и контроль
* Не готовы инвестировать много времени команды, команда маленькая, полгода в рисерч долго, сделать прямо сейчас

Страх 1
* k8s сложный и непонятный, много строчек кода и постоянно управляет продом
* Ansible простой и понятно как работает

Страх 1 На самом деле
* Маппинг сервисов на ноды станет динамическим, k8s постоянно приводит к нужному стейту
* Как сделать так чтобы k8s стал похож на ansible?

Страх 2 Service discovery
* не было 
* сложно
* дополнительный слой (dns, etcd)

Страх 2 Компенсация
Продумали liveness/readiness пробы для проверки сервисов
Работа с отказами на балансировщиках, часто меняется список апстримов, все ретраи и таймауты должны быть продуманы. Есть доклад про балансировку, там тема раскрыта.
Graceful shutdown, логи trace id.

Страх 3 Сеть
Было 10 компьютеров и l2 и понятная адресация, а сейчас плагины которые сделают новую сеть поверх моей сети.
и трафик пойдет через iptables и kube-proxy
pod network 20+ плагинов (bgp имплементация и прочее вместо плоской сети в linux)
Меняем шило не на мыло, а на мыльный завод.

Страх 3 pod network
* не хотим дополнительную инкапсуляцию, так как сложно траблшутить
* пробовали sr-iov, работает но требования к жлезеу + VFs limit (8/64)
* flannel host-gw своя /24 на каждой ноде + стат маршруты на всех нодах

Страх 3 services
Выбрали headless services

Окей, идем в k8s, что нужно сделать
* Подготовить приложения для работы в k8s, уже были в docker
* Как будем деплоить?
* Протестировать отказы? Выдернуть мастера, днс, серверы
* Подготовить ха для production

Уже был docker, конфиги монтировались с хоста
конфиги в configmap и монтировать, но нельзя сделать reconfig
Услшали про helm, но хотелось же ресурсы, проблемы в том, что нужны immutable configmap и их нет в k8s issue 22368

Configs
true way - environment variables, сделали костыль в env положили в yml, пример для golang app

Без helm, используем ansible, выкатываем руками, ansible шаблонизирует спеки, потом apply

Мониторинг теперь не в терминах машины, а в потребление сервисов в пределах их квот. С ограничением памяти все просто, но oom киллер может придти за всеми сразу, сделали разные лимиты по памяти через несколько deployments, в итоге oom прибивает в разное время.

проблема с частыми рестартами есть exp back-off (5 минут)

k8s services vs k8s headless service

selector+probes -> живые endpoints
сервис дает виртуальный ip
proxy userspace, iptables, ipvs

Картинка как работает k8s service
Есть задержка, не ясно какая и не ясно что будет если связь с apiserver оборвется

в качестве балансера envoy, научились его готовить, устраивает по фичам, деплоят как sidecar container, rolling update/rollback

Пример envoy cluster

Service mesh не используем из-за сложности, подробнее на слайдах. Ingress controller также не используем.

Идея доклада - откинут все лишнее и взять только то что нужно

HA cluster - есть много способов, есть официальная инструкция, есть kubespray, нужно разбираться как оно работает, в итоге написали свой playbook

DNS - CoreDNS, DaemonSet+host net

k8s ноды 3 мастера + n нод
на мастерах убрали noschedule taint
stateful также на эти машины в systemd и лимиты на cpu  и памяти
Резервируем ресурсы в kubelet

Картинка как выглядит pod network
На этапе когда k8s и старая инфраструктура использовали статические маршруты добавляя вручную

Итого:
* Внедрили за 1 месяц, так как срезали углы
* Упрощали везде где можно, сеть и прочее
* Обратили внимание на design, обошли много граблей
* Местами костыли, подойдет не всем, но работает.

# Кит против кашалота: великий соблазн SOA
Alex von Rosen, первый ЦУПИС

Рассказ про SOA, как из монолита сделать SOA докладчик не расскажет, а расскажет про полезные вещи.

Архитектуру не следует рассматривать как узкую или техническую часть, она должна учитывать все - политику, кадры, пиар, технологию, место размещения, личные требования. Если не подумать обо всем в целом, то будут проблемы. Пример плохой и хорошей архитектуры на слайдах.

SOA паттерн якобы уже мало кому интересен, хороший и плохие практики известны, но по опыту докладчика мало где видели хорошую SOA. В докладе будут перечислены ошибки, которые не стоит совершать.

Проектирование начинается в голове, у нас есть паттерны, для Алекса SOA это пример конвейера, как у Форда - есть воркеры и есть данные, которые идут по конвейеру. Разделяй и влавствуй. Примеры SOA много где можно увидеть в индустриях.

На практике сложно объяснить слабую и сильную связность. Обычно на уровне интуиции.

Как программисты понимают SOA? 

Если с точки зрения функциональных единиц, то это плохой подход.
Есть подводные камни, а есть подводные грабли.
На практике в SOA есть скрытые связи или зависимости, о соа нужно думать какой предмет общения, как сервисы общаются.

Правильно спроектирования СОА подразумевает автономность сервисов, но на практике это похоже на спрут - запутанная модель данных, транспорт, сеть. Это неухоженная СОА. 


Второй антипример - монолит. Сильная внутренняя связность, транспорт зашитвый в приложение, функ единицы чувствительны к транспорту, без ретраев и лейтенси тайм.


Третий антипример - монолит и спрут, спрутолот, имеет недостатки соа и монолита.


3 простых шага, о которых нужно думать при проектирование СОА архитектуры: 
* Data & Metadata, модель данных и метаданных, команды, приказы, логи. как будет расти, как меняться, какая будет модель. Если есть четкие ответы, то первую стадию прошли.
* Flow, флоу по которому данные двигаются, конвейерная лента, потоки данные, с заделом на будущее, пример, время реакции, можно предвидеть на основе опыта
* Actions, думать о действиях, какие действия происходят. Карта схем связи между сервисами. В будущем даст картину сетевых взаимодействий, требования к оркестровки и прочее.

Если пройти эти три шага, то спрутолота не будет, как и спрута и кашалота.

Принцип Оккама. Делать предельно просто, топорно и управляема. По мнению докладчика это конвейер.

Пример про высоконагруженный финтех - без облаков, данные платежных карт, требования регуляторов, персональные данные, интеграции. Задачи сложные. Система антифрод, второго поколения. Когда сформулировали требования, то оказалось что это отдельная платформа, большая, со стейтами, должна быть сервис ориентированная, чтобы управлять и передавать другим командам.
Из чего исходили, писали на python, так как были разработчики. Сразу был соблазн - сделать архитектуру в виде конвейера, единая шина или транспорт. Но проект нужно было сделать за 1.5 месяца, фунциональные требования не очень понятны, команда маленькая. Поэтому сделали монолит с шиной, выбрали rabbitmq. Поэтому если важно быстро запуститься, то советует начинаем с монолита, но модель и флоу проектировать сразу. Это поможет потом перейти к SOA. 

Картинка про носорога Дюрера.

# Эволюция слонов. Развитие инфраструктуры баз данных на примере растущей компании 
Антон Маркелов, инженер United Traders

Доклад про боль при росте инфраструктуры.

United Traders это около 30 разработчиков, порядка 100 серверов, 2 инженера эксплуатации и поддержка от айтисуммы.

Сначала была плохая инфраструктура, стали строить правильную.

Кратко о стеке - java,kotlin, микросервисная архитектура, pgsql база с каждым сервисом, прикрыто nginx, деплоили с помощью ansible, унифицированные плейбуки, 4-5 ролей, мониторили сначала заббиксом, но было больно, api не стали использовать, бд узкое место, поэтому перешли на prometheus, приложения на spring, поэтому есть интеграция. К остальным прикрутили jmx exporters (nginx, pgsql, kafka). 10-к самописных экспортетов на python. Визуализация в Grafana. Логи Graylog (logback+beats). Используют Sentry для записи exceptions, разработчики после деплоя смотрят в нее, чтобы понять все ли ок.

Боль 1. 

Все умешалось, мин инфра, потом началась фаза роста. Новые микросервисы, фичи в новых микросервисах. Ресурсов стало не хватать. Хостились в DO, увеличение ресурсов там медленоое. Быстрее было взять новый сервер, но работа была полуручная. 
Что придумали - возьмем вместо 10 маленьких, 3 больших сервера. Тогда сервисы двигать будем редко. На этом и живем сейчас.

Боль 2. 

Завились аналитики. Появились хотелки. Тяжелые запросы по бд, красивые дашборды, аналитика, кроссбазные запросы.
Придумали - слейв, репликация по нескольким базам. Реализация на pgsql 9.x, смотрели на pglogical, presto, slony, bucardo но остановились на streaming replication + postgres_fdw (для подключения баз друг к другу). Обозвали superslave. Разнесли постгресы по разным портам и настроили репликацию с мастеров на этот superslave. Создали виртуальную бд - crossdatabase. И стало возможно одним запросом получать данные из разных баз. Взяли redash для запуска запросов и дашбордов. Еще позволяет удобно управлять правами, пускают и аналитиков и разработчиков.

Инфраструктура развивалась, появилась kafka, потом nginx как единая точка входа, появился clickhouse.
Посмотрели как устроено HA в kafka и clickhouse и захотели такую же и для pgsql.

Составили требования:
* Отказоустойчивость
* в приложении ничего не менять, только конфииги
* балансировка нагрузка
* написано на языке где есть экспертиза
* легко развернуть и обслуживать
* мин слоев абстракции

Посмотрели:
* Стандартную потоковую репликацию (repmgr, patroni, stolon)
* Репликация на основе триггерах (londiste, slone)
* Репликация запросов в среднем слое (pgpool-2)
* Ассинхронная репликация (bucardo)

Выбрали stolon. Схема на слайде. Конфиг постгреса теперь хранится в etcd.

На что обратить внимание:
* stolon не умеет zookepeer, только consul/etcd/k8s
* etcd чувствителен к io
* даже на быстрых ssd таймауты etcd
* max_connections маленькое по дефолту
* Все коннекты идут на мастер, слейвы недоступны для коннекта
* 3 etcd переживут смерть всего 1 сервера

Написали свою утилиту Stolon-Haproxy, скрипт на python. Генерирует конфиг для haproxy. Можно было взять pgbouncer, но решили не плодить сущностей.

Итого как выглядит сейчас - nginx на входе, на серверах java+nginx, внизу шина на kafka и базы.

После переезда на stolon быстродействие не изменилось.

Аварии. Была только одна, было 3 сервера с etcd, отключилось 2 сервера из 3, кворум пропал. Stolon перестал отвечать. Решили переводом etcd в single node и сделали вывод 5 etcd для 3 серверов. 

По Kafka. Schema registry + Rest-proxy. Burrow для мониторинга, jmx exporters для внутренних метрик, увеличили retention для топиков и оффсетов (1 день по дефолту). Для аналитики есть kafka * sink connector.

По  ClickHouse. Ограничили пул соеденений. Взяли go migrate для миграций, так как flyway не поддерживается.

Движение к версии 3.0:
* Откат БД на произвольный момент времени 
* API gw
* Автоперенос приложений, тест k8s

Ссылки на последнем слайде для доп инфы.

# Любовь со... второго взгляда: наш опыт с Kubernetes

Евгений Потапов, Сергей Спорышев, генеральный директор, начальник отдела высоконагруженных проектов, ITSumma

Несколько кейсов для k8s от Жени:
* enterprise кейс, пространство для разработчиков;
* Уже есть разработка в docker окружении и это удобная система оркестрации;
* Балансировка и управление ресурсами.

Доклад про то, как в ITSumma меняли идеалогию, с какими проблемами стокнулись и полюбили k8s.
Большой слайд со списком клиентов.
Саппортят половину российского интернета. Начинали как компания по разработке и продолжают.

Типичный проект в 2000 - LAMP.

Проекты стали сложнее и сейчас много фронтендов, бекендов и бд. Разные языки, очереди, шины, микросервисы. Усложняется инфраструктура - логирование, мониторинг. Ребяты видели много комбинаций таких архитектур.

Проблемы:
* Организация процесса разработки
* Введение новых разработчиков в проект
* Организация деплоя (сильно связанные компоненты)
* Поддержка инфраструктры (24x7)
* Органищация работы команды (много разработчиков и админов)

Кейс про Ops и Dev. Нагрузка выросла, нужен редирект, версия не та. Пожар. Сверху еще и бизнес, которые теряет деньги. Ему нужно быстро выкатывать фичи.

2016 год. Docker, k8s. первый взгляд. По иницативе админов, решит все проблемы.
Первое правило инженера - работает, не трогай. Админы изучили инструмент, нашли баги. В итоге на тот момент, оставили все как есть.

Второй взгляд уже по инициативе разработчиков. Попробовали для локальной разработки.
* Проект - 2 бекенда, фронт angular, сервисы на rust для уведомлений;
* Vagrant -> Docker Compose, написали много конфигураций;
* Minikube для pre-stage.

Выводы:
* Это удобно, много проблем ушли
* Это управляемо, ушли проблемы с зависимостями
* Сокращает время, новые разработчики быстрее входят в курс дела

Кейс 1. Монолит->k8s, новостной паблик (Republic). Что было:
* Монолит symfony + vanila js + angular
* 50 крон тасок
* Кастомный сервис на с + nginx для работы с web socket
* Кастомный сервис на D lang
* php нужна была скомпилированная статика
* Жесткое версионирование (фронт бек)
* PostgreSQL, Redis, Elasticsearch
* Большой объем загружаемых данных - картинки, медиа, видео

Решение:
* Docker multi stage build
* fpm-base контейнер с библиотеками и зависимостями, это базовый образ. Из него соб fpm образ - from node, from base

* Deployment, web
  * 2 контейнера в поде (nginx+fpm)
  * Entrypoints - nginx + cache
  * Rolling update
  * L/R пробы
* Deploy, Admin, аналог + replicaset 1
* Маршрутизация web/admin ingress
* Imageproxy отд прил с соб конфиг и проеки из nginx
* D-lang app (bin в доке)
* External Services - postgres, redis, elastic - проксируют на внешние сервисы вне кластера.
* k8s cronjobs (fpm-based). Первый инцидент - кроны не успевались пройти и после определенного времени k8s падал. Переработали cron таски и стало хорошо.
* 2 PV модуля:
  * PV для статики
  * PV для файлов - sitemap, rss

Итого общая схема на слайде.

Кейс 2, что было:
* symfony-based framework
* 3 проекта, один бекенд
* сбор статики с помощью php
* идентичные конфиги
* жесткое версионирование
* медиа - внешее хранилище
* Отдельный микросервиc elastic
* Много очередей

Взяли аналогичную схема с билдом контейнеров, но другой порядок. Базовый fpm, nginx образ кастомный, с модулями. Из 2 уже собирали основной nginx.

Схема деплоя такая же:
* Файлы конфигов общие на 3 приложения
* Сборка из одних исходников
* External сервисы pg redis
* Fpm based контейнеры для работы с очередями

Выводы:
* Dev понимают, чем заняты OPS
* OPS понимают, как больно DEv
* Dev и Ops делают одно дело
* Бизнес доволен

Еще выводы:
* С инструментами стало работать удобнее
* Безопасно деплоить, без пробы трафик не пойдет
* Инфраструктура как код
* Ввод новых разработчиков и админов быстрее.
* Ввеедение новых фич интересно быстро и увлекательно

Минусы
* Цитата "А теперь мы решаем проблемы, которые раньше не возникали, инструментами, которые для этого не предназначены"


# Тернии контейнеризованных приложений и микросервисов 
Иван Круглов, Principal Developer, Booking.com

Booking.com - итак все знают, что внутри:

С 97 года, устоявшийся стек - железные сервера, serverdb, perl/java, puppet, storage, mysql, zookeeper, kafka, graphite, alerts/predictions.

Пришел бизнес и сказал хотим faster time to market - много идей и фич быстро выводить на рынок.

Раньше чтобы сделать что-то новое, нужна новая server role - паппет, порты, сд, квм, базы, лб, деплой. Все ручное, занимало дни и недели, взаимодействие по Jira.

Доклад про быстрое выведение продуктов. Чтобы не повторять ошибок. Путь в облака был тяжелый и было 3 итерации - марафон, опеншифт и k8s. Как профакапили 2.5 платформы, 1 и 2 точно профакапили, третья с лета и вроде все работает, но уверенности нет. И через год возможно booking.com расскажет как профакапили и 3 платформу.

Первый подход - Marathon. Родился в Хакатоне в середине 2016 года, но все еще не закончился. Набор скриптов. 2 шаренных кластера. Десятки проектов. RBAC не было, сложности с fw, поэтому отказались, но не убрали.

Мысль 1. Не стоит недооценивать способность кода выживать. Сейчас поддерживают все 3 платформы. RIP Marathon - 12.2018.

Второй подход - OpenShift.
OpenShift включает k8s, ansible, open vswitch (network level isolation, pci, динамические дырки в fw, сильно полагались, думали решит все проблемы).

В итоге попали на новый инструмент - ansible, сложный k8s:
* Непонятное поведение, как коробочный проект не взлетело, нужно разбираться что внутри - примеры, пропадали правила в open vswitch, iptables под тысячу строк, дебаг вручную. Рестарт всех контейнеровна ноде. Одна нода решила взять нагрузку в кластере - 356 подов, 1.5к конейтеров.

Мысль 2. Весы Cool vs Boring. Слишком круто не сработало, надо было выбрать Boring.

Проблемы:
* Техобслуживание и поддержка захлебнулась. 
* Мониторинг был, но в итоге подход пользователи как мониторинг. 
* Нет нормальных квот, из-за опечатки все ложилось
* Capacity planning тоже был не ок

Было 2 фундаметальные проблемы

Проблема 1. Хотели простой и удобный интерфейс, аля хероку

Получили:
* отличный опыт первые разы
* непонимание что происходит, вот кнопка и все, когда кнопка не работает никто не понимал
* дебаг  приложений лег на команду
* недовольство опытных пользователей, тяжело пробится через абстракции

Мысль 3. Стоит сформулироват ожидания польз от системы и стсемы от пользователей

Проблема 2. Интеграция в техстек - DLB, service discovery, service mesh, events, graphite, storage, mysql, cassandra, redis, kafka, ssl, identity key maangement, secrets.

Мысль 4 - Kubernetes это хорошо, но его интеграция в существующую экосистему компании лучше. Если вы не начинаете с нуля.

Итого получили:
* Утоплены в поддержке
* с юзерами ожидающими магию
* без понимания инфраструктуры
* полный беклог
* беда с тех обслуживанием

В итоге вышли на сцену и сказали что нужно время на починку и составили план:
* минимум клевых штук, убрали ансибл и всвитч
* мониторинг инфраструктуры, мы сообщаем что у нас проблемы
* четкие ожидания, команда и пользователи, изучите k8s, вы должны обеспечить ваш сервис поддержкой, 10 чел не могут обеспечить 1500 разработчиков
* инвестиции в знания - тренинги, онлайн курсы, книги и документация

k8s, начало 2018, 3 итерация:
* несколько изолированных кластеров k8s, защита от самих себя и своих действий
* компоненты вне кластера k8s
* мониторинг вне кластера k8s
* плоская сеть, сделали так что и поды и сервисы в одной подсети что и все железо в компании, с лаптопа можно сделать пинг на каждый под
* функциональные тесты для k8s и интеграций, cron джобы которые каждую минуту проверяют функционал k8s, хотят выдать в open source

Структура k8s на отедельном слайде.

Достижения:
* стабильная система
* 250 уник серв, стабильный рост
* понимание происход
* отлаженное взаимодействие
* самообслужива
* система обучения, 500 человек обучилось, формируется понимание

Заключение:
* Управление ожиданиями - ключ к успеху! 
* K8s хорош, но его интеграция в экосистему лучше!
* K8s целая экосистема, она требует понимания, динамично развивающаяся среда. Использовать можно, но осторожно.
* Вложитесь в обучение.

Вопросы.

Как обучали:
* Внутренний тренинг, 2 дня, дважды в месяц, на 3 месяца вперед все занято
* Udemy курсы по k8s
* закупили одну книжку про концепции k8s
