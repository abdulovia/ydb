# Топология кластера {{ ydb-short-name }}

Кластер {{ ydb-short-name }} состоит из [узлов хранения](glossary.md#storage-node) и [узлов баз данных](glossary.md#database-node). Работоспособность обоих типов узлов важна для обеспечения доступности баз данных {{ ydb-short-name }}: узлы баз данных реализуют логику управления данными, а узлы хранения обеспечивают их сохранность. При этом наибольшее влияние на отказоустойчивость кластера и его способность обеспечивать надёжное хранение данных оказывает подсистема [распределённого хранилища](glossary.md#distributed-storage), состоящая из набора узлов хранения. При развёртывании кластера необходимо выбрать [режим работы](#cluster-config) распределённого хранилища в соответствии с ожидаемой нагрузкой и требованиями к [доступности баз данных](#database-availability).

## Режимы работы кластера {#cluster-config}

Топология кластера строится в соответствии с его режимом работы, который должен быть выбран на основе требований к уровню отказоустойчивости. Модель отказа, используемая в {{ ydb-short-name }}, основана на концепциях [домена отказа](glossary.md#fail-domain) и [области отказа](glossary.md#fail-realm). {{ ydb-short-name }} предоставляет следующие режимы работы распределённого хранилища:

- `none`. Избыточность отсутствует. Любой отказ приводит к временной недоступности или потере всех или части хранимых данных. Этот режим рекомендуется использовать только для разработки приложений или проведения функциональных тестов.
- `block-4-2`. Избыточность по схеме [стирающего кода](https://ru.wikipedia.org/wiki/Стирающий_код). На каждые 4 блока исходных данных формируются 2 дополнительных блока с кодами избыточности. Узлы хранения размещаются как минимум в 8 доменах отказа (обычно серверных стойках). [Пул хранения](glossary.md#storage-pool) остаётся полностью доступным при недоступности двух любых доменов отказа, продолжая записывать все 6 частей данных в оставшихся доменах. Этот режим рекомендуется для кластеров, размещённых в одном дата-центре или зоне доступности.
- `mirror-3-dc`. Данные реплицируются между 3 зонами доступности (обычно разными дата-центрами) с использованием как минимум 3 доменов отказа (обычно серверных стоек) в каждой зоне доступности. Кластер {{ ydb-short-name }} остаётся доступным при выходе из строя любой зоны доступности; кроме того, дополнительно может выйти из строя ещё один домен отказа в любой из 2 работоспособных зон доступности без прекращения работы кластера. Этот режим рекомендуется для кластеров с высокими требованиями к отказоустойчивости, размещённых в нескольких дата-центрах.
- `mirror-3-dc-3-nodes`. Упрощённый вариант режима `mirror-3-dc`. В этом режиме требуется не менее 3 серверов, каждый из которых должен содержать как минимум 3 диска для хранения данных. Для обеспечения гарантий отказоустойчивости каждый сервер должен размещаться в своём независимом дата-центре. Доступность кластера обеспечивается при условии отказа не более чем одного сервера. Этот режим рекомендуется для функционального тестирования или прототипирования.

{% note info %}

Под выходом из строя сервера подразумевается как полная, так и частичная его недоступность, например выход из строя одного диска на сервере.

{% endnote %}

Приведённая ниже таблица описывает требования и обеспечиваемый уровень устойчивости к отказам для различных режимов работы кластера:

| Режим | Множитель<br/>объёма<br/>хранения | Минимальное<br/>количество<br/>узлов | Домен<br/>отказа | Область<br/>отказа | Кол-во<br/>дата-центров | Кол-во<br/>серверных<br/>стоек |
| --- | --- | --- | --- | --- | --- | --- |
| `none`, избыточность отсутствует | 1 | 1 | Узел | Узел | 1 | 1 |
| `block-4-2`, переживает отказ 2 стоек | 1.5 | 8 (10 рекомендовано) | Стойка | Дата-центр | 1 | 8 |
| `mirror-3-dc`, переживает отказ дата-центра и ещё 1 стойки в оставшихся дата-центрах | 3 | 9 (12 рекомендовано) | Стойка | Дата-центр | 3 | 3 в каждом дата-центре |
| `block-4-2` *(упрощённый)*, переживает отказ 1 стойки | 1.5 | 10 | ½ стойки | Дата-центр | 1 | 5 |
| `mirror-3-dc` *(упрощённый)*, переживает отказ дата-центра и ещё 1 сервера в оставшихся дата-центрах | 3 | 12 | ½ стойки | Дата-центр | 3 | 6 |
| `mirror-3-dc` *(3 узла)*, переживает отказ 1 узла или 1 дата-центра | 3 | 3 | Сервер | Дата-центр | 3 | Не важно |

{% note info %}

Приведённый выше множитель объёма хранения относится только к фактору обеспечения отказоустойчивости. Для планирования размера хранилища необходимо учитывать и другие влияющие на него факторы, такие как фрагментация и гранулярность [слотов](glossary.md#slot).

{% endnote %}

Базовой единицей выделения ресурсов хранения данных в кластере {{ ydb-short-name }} является [группа хранения](glossary.md#storage-group). При создании группы хранения {{ ydb-short-name }} её части — [VDisk-и](glossary.md#vdisk) — размещаются на [физических дисках](glossary.md#pdisk), принадлежащих разным доменам отказа. Для режима `block-4-2` каждая группа хранения распределена между 8 доменами отказа, а в режиме `mirror-3-dc` группа хранения распределяется между 3 областями отказа, в каждой из которых используются 3 домена отказа.

О том, как задать топологию кластера {{ ydb-short-name }}, читайте в разделе [{#T}](../reference/configuration/index.md#domains-blob).

### Упрощённые конфигурации {#reduced}

В случаях, когда невозможно использовать [рекомендованное количество](#cluster-config) оборудования, можно разделить серверы одной стойки на 2 фиктивных домена отказа. В такой конфигурации отказ одной стойки будет означать отказ не одного, а сразу двух доменов. При использовании таких упрощённых конфигураций {{ ydb-short-name }} сохраняет работоспособность при отказе сразу двух доменов. Минимальное количество стоек в кластере для режима `block-4-2` составляет 5, а для `mirror-3-dc` — по 2 в каждом дата-центре (т.е. суммарно 6 стоек).

В минимальной отказоустойчивой конфигурации {{ ydb-short-name }} используется режим `mirror-3-dc-3-nodes`, и кластер состоит из 3 серверов. В такой конфигурации каждый сервер одновременно является доменом отказа и областью отказа, и кластер может выдержать сбой только одного сервера.

## Восстановление избыточности {#rebuild}

В случае отказа физического диска {{ ydb-short-name }} может автоматически реконфигурировать связанные с ним группы хранения. При этом не важно, связана ли недоступность диска с выходом из строя сервера целиком или нет. Автоматическая реконфигурация групп хранения снижает вероятность потери данных при возникновении последовательности множественных отказов, при условии, что между отказами проходит достаточно времени для завершения восстановления избыточности. По умолчанию реконфигурация начинается через 1 час после того, как {{ ydb-short-name }} выявляет отказ.

Реконфигурация дисковой группы заменяет VDisk, размещённый на отказавшем оборудовании, новым VDisk, который система старается разместить на работоспособном оборудовании. Действуют те же правила, что и при создании новой группы хранения:

* новый VDisk создаётся в домене отказа, отличающемся от доменов отказа всех остальных VDisk в группе;
* в режиме `mirror-3-dc` новый VDisk размещается в той же области отказа, что и отказавший VDisk.

Для того чтобы реконфигурация была возможной, в кластере должны быть свободные слоты для создания VDisk в разных доменах отказа. При расчёте количества оставляемых свободными слотов следует учитывать вероятность отказа оборудования, длительность репликации, а также время, необходимое на замену отказавшего оборудования.

Процесс реконфигурации дисковой группы создаёт повышенную нагрузку на остальные VDisk группы и на сеть. Чтобы уменьшить влияние восстановления избыточности на производительность системы, суммарная скорость репликации данных ограничивается как на стороне VDisk-источника, так и на стороне VDisk-получателя.

Время восстановления избыточности зависит от объёма данных и производительности оборудования. Например, репликация на быстрых NVMe SSD может завершиться за час, а на больших HDD репликация может длиться более суток.

Реконфигурация дисковых групп может быть ограничена или оказаться полностью невозможной, если оборудование кластера состоит из минимально необходимого количества доменов отказа:

* при отказе всего домена реконфигурация теряет смысл, так как новый VDisk может быть размещён только в уже отказавшем домене отказа;
* при отказе части домена реконфигурация возможна, но нагрузка, ранее обрабатываемая отказавшим оборудованием, будет перераспределена только по оставшемуся оборудованию в том же домене отказа.

Если в кластере доступен хотя бы на один домен отказа больше, чем минимально необходимо для создания групп хранения (9 доменов для `block-4-2` и по 4 домена в каждой области отказа для `mirror-3-dc`), то при отказе части оборудования нагрузка может быть перераспределена по всему оставшемуся в строю оборудованию.

## Доступный объём ресурсов и производительность {#capacity}

Система может работать с доменами отказа любого размера, однако при малом количестве доменов и неодинаковом количестве дисков в разных доменах количество групп хранения, которые можно будет создать, окажется ограничено. В таких условиях часть оборудования в слишком больших доменах отказа может оказаться недоиспользованной. В случае полной утилизации оборудования значительный перекос в размерах доменов может сделать невозможной реконфигурацию.

Например, в кластере с режимом отказоустойчивости `block-4-2` имеется 15 стоек. В первой из них расположено 20 серверов, а в остальных 14 стойках — по 10 серверов. Для полноценной утилизации всех 20 серверов из первой стойки {{ ydb-short-name }} будет создавать группы так, что в каждой из них будет участвовать 1 диск из этого самого большого домена отказа. В результате, при выходе из строя оборудования в любом другом домене отказа нагрузка не сможет быть распределена на оборудование в первой стойке.

{{ ydb-short-name }} может объединять в группу хранения диски разных производителей, с различной ёмкостью и скоростью. Результирующие характеристики всей группы будут ограничены наихудшими характеристиками оборудования, которое в неё входит. Обычно лучших результатов удаётся добиться при использовании однотипного оборудования.

{% note info %}

При создании больших кластеров следует учитывать, что оборудование из одной партии с большей вероятностью может иметь одинаковый дефект и отказать одновременно.

{% endnote %}

Таким образом, в качестве оптимальных аппаратных конфигураций кластеров {{ ydb-short-name }} для промышленного применения рекомендуются следующие:

* **Кластер в одной зоне доступности**: использует режим отказоустойчивости `block-4-2` и состоит из 9 или более стоек с равным количеством одинаковых серверов в каждой.
* **Кластер в трёх зонах доступности**: использует режим отказоустойчивости `mirror-3-dc` и расположен в трёх дата-центрах с четырьмя или более стойками в каждом. Стойки укомплектованы равным количеством одинаковых серверов.

## Обеспечение доступности баз данных {#database-availability}

[База данных](glossary.md#database) в кластере {{ ydb-short-name }} доступна, если её хранилище и вычислительные ресурсы находятся в рабочем состоянии:

- Все выделенные базе данных [группы хранения](glossary.md#storage-group) должны быть доступны, что означает соблюдение допустимого уровня отказов для каждой группы.
- Вычислительные ресурсы доступных в данный момент [узлов базы данных](glossary.md#database-node) (в первую очередь объём оперативной памяти) должны быть достаточны для запуска всех [таблеток](glossary.md#tablet), управляющих пользовательскими объектами, такими как [таблицы](glossary.md#table) или [топики](glossary.md#topic), и для обработки пользовательских сессий.

Чтобы база данных могла пережить отказ одного из дата-центров в кластере, использующем режим работы `mirror-3-dc`, должны быть выполнены следующие условия:

- [Узлы хранения](glossary.md#storage-node) должны обеспечивать как минимум двойной запас пропускной способности ввода-вывода и дисковой ёмкости по сравнению с необходимыми для работы в обычных условиях. В худшем случае нагрузка на сохранившиеся узлы при длительном отключении одного из дата-центров может утроиться, но лишь на ограниченный период времени — до завершения восстановления ставших недоступными дисков в оставшихся дата-центрах.
- [Узлы базы данных](glossary.md#database-node) должны быть равномерно распределены между всеми тремя дата-центрами и содержать достаточно ресурсов для обработки полной нагрузки при сохранении двух из трёх дата-центров. Это означает наличие как минимум 35% запаса по ресурсам CPU и оперативной памяти в обычных условиях, то есть при отсутствии каких-либо отказов. Если узлы базы данных обычно нагружены более чем на 65%, следует рассмотреть возможность добавления дополнительных узлов либо увеличения вычислительной мощности каждого узла.

## Дополнительная информация

* [Документация для DevOps-инженеров](../devops/index.md)
* [Примеры конфигурационных файлов кластера](https://github.com/ydb-platform/ydb/tree/main/ydb/deploy/yaml_config_examples/)
