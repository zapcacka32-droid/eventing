# EDP, techsec-dev

Прод-контур. Неймспейсы из ТПР: `ecpk-knative-eventing`, `ecpk-knative-serving`, `ecpk-knative-kafka`.

Запуск: `applicationset.yaml`. Values читаются из каталога `edp/` репозитория `http://gitea-gl.ecpk.rosneft.ru/core/knative-platform.git`. Это единственный git из списка компонентов. `namespace`, `policies`, `ingress-gateway`, `knative-serving` и `knative-eventing-config` — чарты из `registry-gl.ecpk.rosneft.ru/managed-services`. Перед apply каталог `edp/` должен быть на `main`, и AppProject должен пускать три неймспейса выше.

На кластере с этим ApplicationSet не должно быть второго `KnativeEventing` или `KnativeServing`. На одном кластере они перезаписывают webhook и ClusterRoleBinding.

`metadata.namespace` ApplicationSet сейчас `tech-argocd`, project `ecpk-core`. Если Argo EDP живёт в другом неймспейсе, поменять эти два поля до apply.

Волны: 0 неймспейсы, 1 политика Kafka, 2 Serving и Kafka, 3 Eventing, 4 политики Eventing и Serving. Очередь держится флагом контроллера `applicationsetcontroller.enable.progressive.syncs=true`. Без него волны стартуют вместе, и политика Eventing закрывает webhook раньше, чем control plane поднялся.

## Что ломалось на тесте

### Чарт не находится

`ComparisonError: chart not found` при пине, которого нет в реестре. Локальный `Chart.yaml` ingress-gateway был `4.0.0`, в Harbor этого тега не было, живой тег оказался `11.0.0`. Перед запуском EDP проверить, что в `registry-gl.ecpk.rosneft.ru/managed-services` есть `namespace` `7.0.3`, `knative-serving` `0.1.2`, `knative-eventing-config` `0.3.17`, `policies` `0.6.3`. На тесте эти теги лежали в `harbor.idp.ecpk.test`.

В `repoURL` и в секрете репозитория Argo схемы `oci://` нет. Её пишет сам Argo. В `Chart.yaml` у зависимости схема обязательна: `oci://registry-gl.ecpk.rosneft.ru/managed-services`. Без неё Helm идёт в `index.yaml`.

`503` на `index.yaml` при этом означает, что TLS уже прошёл, а сервис чартов реестра не ответил. Это другая ошибка, чем сертификат.

### Сертификат реестра и OCI-зависимость

`helm dependency build` падал так: `could not download oci://registry-gl.ecpk.rosneft.ru/managed-services/ingress-gateway` версии `9.5.0`, `x509: certificate signed by unknown authority`.

Файл `/app/config/tls/<хост>` из `argocd-tls-certs-cm` использует `helm repo add`. Клиент OCI внутри `dependency build` его не читает. Флаг `insecure: "true"` на секрете репозитория до этой команды тоже не доходит. Выпускающий сертификат нужно дописать в системный набор `argocd-repo-server` и перезапустить под. В тексте ошибки бывает `(cached)`: после починки у Application нужен Hard Refresh.

### Values в Git и локальные разъехались

Argo ставит файл с `main` GitLab, не файл с машины. На тесте в GitLab было `platformManifests.enabled: false`, локально `true`, и раздача манифестов Kafka не встала. Перед чистой установкой `edp/` на `main` должен совпадать с этим каталогом: `platformManifests.enabled: true`, реплики из таблицы ТПР.

Чарты Eventing и Serving неймспейс не создают: в values `controlPlane.create: false`. Неймспейс принадлежит отдельному релизу чарта `namespace`. Второй владелец вместе с `selfHeal` уводит неймспейс в Terminating.

Пакет `knative-eventing-config` собирать из overlay, в котором есть шаблон `kafka-pods-defaulting-webhook`. Каталог `eventing/knative-eventing-main` с тем же номером версии этот шаблон может не содержать.

### Webhook и квота

Webhook не создаёт Secret `kafka-webhook-eventing-certs`, он заполняет уже существующий пустой Secret. Без него под webhook не становится Ready.

`ResourceQuota` требует и requests, и limits. Job миграции в workloads называется `storage-version-migration-eventing-`, с дефисом на конце: оператор матчит `generateName`.

ConfigMap `eventing-kafka-manifests` около 255 КБ. Client-side apply кладёт копию в `last-applied-configuration` и упирается в лимит аннотации. В ApplicationSet стоит `ServerSideApply=true`. В самом ConfigMap манифесты лежат в `binaryData`, иначе Argo режет файл по `---`.

### Kafka: SASL и секрет

Слушатель `plain:9092` — это `SASL_PLAINTEXT` и `SCRAM-SHA-512`, не TLS. Предупреждение клиента про firewall и TLS на этом порту к маршруту не относится.

Пароль в JAAS нельзя собирать в shell: `!` и `$` портятся, а подстановка срезает перевод строки. Брать ключ `sasl.jaas.config` из секрета как есть.

На тесте и секрет, и `kafka-topics` всё равно отвечали `invalid credentials`. User-operator писал, что Secret `knative-kafka-server-super-user` был `MODIFIED`, и сверял его примерно раз в минуту. У секрета пользователя в соседнем чарте стоит метка `syncvault: "true"`: Vault перезаписывает пароль после того, как Strimzi записал SCRAM. Пока секрет и учётные данные в Kafka расходятся, топик через этот пользователь не создаётся. TCP до брокера при этом уже открыт: ошибка на SASL, а не timeout.

### Сеть снаружи и шлюз

С VDI пакета до адреса балансировщика нет. `traceroute` умирает на первом хопе, виртуальном NAT (`mshome.net`). Поднятый Gateway и открытый TCP из пода брокера этот маршрут не создают.

Один TCPRoute на bootstrap не делает внешнего клиента Kafka. Оригинальный чарт анонсирует внутренние имена `*.svc`. Поле `externalListeners` есть в другом чарте, в `kafka-1.0.1-test` его не включают.

NetworkPolicy чарта ingress-gateway открывает 80, 443, 15020, 15021, 15090 и не открывает порт слушателя 9092. Ручной `kubectl patch` при `selfHeal: true` откатывается. Две политики на один и тот же номер порта друг другу не мешают, если селекторы разные: брокеры и поды шлюза.

Чужой VIP на Service этого шлюза не вешать. На тесте `10.193.96.238` уже отвечал из кластера и не был ни Service, ни Gateway в API. Аннотация `metallb.io/loadBalancerIPs` на этот адрес столкнёт два владельца, а с VDI маршрут всё равно не появится.

### Шлюзы из ТПР

| Кластер | Имя в ТПР | FQDN | VIP |
| --- | --- | --- | --- |
| global (DEV-ECPK) | id-dev-igw-knative | *.knative.idp.ecpk.test | 10.193.96.238 |
| techsec-dev | ed-tco-igw-knative | *.knative.edp.ecpk.test | 10.193.98.102 |
| dev-common | ed-dev-igw-knative | *.knative-dev.edp.ecpk.test | 10.193.99.6 |
| test-common | ed-tst-igw-knative | *.knative-test.edp.ecpk.test | 10.193.99.133 |

Порт у всех 443 HTTPS/TCP, маршрутизация `*/*`. В запуске сейчас только techsec-dev: values `ingress-gateway/ecpk-knative-serving.yaml`, адрес `10.193.98.102`. Чарт вставляет в имя сегмент проекта, ресурс будет `ed-tco-igw-ecpk-knative`. Бэкенд таблица не называет, маршрут не создан. Слушатель Passthrough: сертификат для Terminate в ТПР не задан.
