Егор Баяндин технический директор S7 Travel Retail
Использование k8s в IaaS

Самолеты зеленые, теперь будут и ракеты запускать, так как купили платформу.
Black Friday коснулся и S7, началась распродажа.

История про развитие ecommerce в S7
В 18 году перешли на новую версию платформы, S7 мигрировала в другую систему бронирования, проект миграции занял около года.

Предпосылки
до 2012 дедидки - месяцы поставки
2012 - приватные vmware - дни
2014 iaas - часы, миграция заняла пару месяцев, используют имена серверов, а не ip и прошло гладко
2018 - контейнеры появился k8s кластер в облаке

Сформулировали список условий (pre-condition)
stateless приложения
Service/microservices arch
Репозиторий для контейнеров
Централизованное логирование и мониторинг
Единое договоренности по сетевой структуре, появился зоопарк, перед запуском в prod договорились о общих подходах (на самом 100% не договорились, трафик заворачивают через traefik, в том числе и чтобы на лету менять конфиги)
Автоматизация поставок - желательно, можно и ручками, но для скорости

Варианты:
baremetal, на физические сервера возвращаться не хотелось
iaas, ручками настраивать k8s + api vmware, был рабочий вариант, пока не наткнулись на
CSE (ProtonOS масштабирование из коробки) - продукт от vmware, 

cse - container server extension
расширяет vcloud api для управления жизненного цикла k8s кластеров, шаблоны

Результаты - все счастливы
разработчики - не надо выпрашивать сервера
админы не надо отвлекаться от важных задач
бизнес - уменьшение ttm, экономия на ресурсах, вся инфра сейчас 300 серверов

Но это не серебряная пуля
Бд и персистетные хранилища на выд серверах
Внешний мониторинг, чтобы 
Все непросто с критичными по безопасности системами и сервисами , получилось договориться с security, с персональными данными 

Что уже удалось:
4 кластера в 2 дц - прод и дев
Система контроля продаж
Сервис предоставления мин цены
Сервис для чатов, backend (веб сокет сервер, клиенты с сайта и с моб приложения)
и + парочка систем на подходе (персональный кабинет, изначально проектировалась под k8s, и портал s7, пока монолит, но планируют распилить)

Вопросы 
Почему не baremetal?
С железом не сложилось, доп расходы - заказы, эксплуатация. В CSE низкий порог входа, за 40 минут подняли первый контейнер, после чтения документации.
Арендуют ресурсы, а не железки. Есть только одна железка - cisco router.

А что про pci-dss?
Номера карт не хранят, только данные передают, они сертиф по pci и они тоже в облаке

Примеры приложений?
Смотри выше, докладчик про них говорил.

Стек технологий для мониторинга и логов?
ELK, filebeat+logstash+elastic+kibana+grafana
Мониторинг, до k8s жили на nagios, смотрят в сторону prometheus+zabbix, решение пока не нравится, есть над чем поработать.

