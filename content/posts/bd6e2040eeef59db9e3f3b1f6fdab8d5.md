---
title: Как «воспитать ламу» и ускорить ML-эксперименты / Хабр
date: 2023-10-27
src_link: https://www.notion.so/ML-7092cd8f900b481aacf6cb8d861d622f
src_date: '2023-10-27 19:01:00'
gold_link: https://habr.com/ru/companies/selectel/articles/767076/
gold_link_hash: bd6e2040eeef59db9e3f3b1f6fdab8d5
tags:
- '#host_habr_com'
---

![https://image.mel.fm/i/1/1Ud7AReU87/1210.jpg](https://habrastorage.org/r/w1560/getpro/habr/post_images/5c6/40e/d62/5c640ed621d2bc9f12bc41a07ae6d273.png)  

Часто проведение ML-экспериментов сводится к долгому поиску и загрузке нужных датасетов и моделей, скрупулезной настройке гиперпараметров с целью проверки гипотез. Но что делать, когда времени мало, а за ночь нужно зафайнтюнить ламу? Давайте это и узнаем.  

  


> Статья написана по мотивам доклада Ефима Головина, MLOps-инженера в отделе [Data- и ML-продуктов Selectel](https://slc.tl/s26vs).

  

  

**Используйте навигацию, если не хотите читать текст целиком:**  

  

→ [Типичная ML-задача: когда нужны ML-эксперименты](#1)  

→ [Работа с датасетами](#2)  

→ [Работа с моделями](#3)  

→ [Деплой модели](#4)  

→ [Как организовывать эксперименты без навыков MLOps](#5)  

  

Типичная ML-задача: когда нужны ML-эксперименты
-----------------------------------------------

  

Любая ML-задача может быть описана с помощью схемы CRISP-ML, которая объединяет три крупных этапа:  

  

* **Понимание данных.** Вам нужно разобраться в том, с какими данными вы работаете, какие в них есть недостатки, нужно ли (и можно ли) их «обогатить». А после — собрать датасет.
* **Построение модели.** Набор данных готов, есть задача — вы начинаете ее решать: проверять гипотезы по различным классам моделей, гиперпараметрам, а также разнообразным «эвристикам», которые можно применить при построении модели.
* **Доведение модели до потребителя.** Все гипотезы отработаны, модели проверены, вы нашли устраивающий вас вариант. Далее нужно довести модель до конечного потребителя — сделать нечто, что временами называют «продуктивизацией модели».

  

![](https://habrastorage.org/r/w1560/getpro/habr/post_images/4d4/52b/4f1/4d452b4f159a2e0014887f00d263457b.png)  

На каждом из этапов вы можете столкнуться в ворохом процессных и технических аспектов, которые могут как ускорить, так и замедлить выполнение ML-экспериментов. Что это за аспекты и с чем их едят — разберемся на примере LLM.  

  

### Тюнинг ламы на тикетах

  

В общих чертах рассмотрим кейс с проведением ML-экспериментов на примере языковой модели LLama 2:  

  

![](https://habrastorage.org/r/w1560/getpro/habr/post_images/b50/970/f4a/b50970f4ac679841de5c61a19078bb40.png)  

Модель берем с платформы [Hugging Face](https://huggingface.co), а предварительно подготовленный датасет с тикетами — из ClearML. Далее в Jupyter пишем прототип (код эксперимента) и после отладки — запускаем его в ClearML. Поскольку гиперпараметры были предварительно закреплены за кодом с помощью ClearML SDK, мы можем удобно с ними экспериментировать — изменять количество эпох, learning rate или batch size и сравнивать результаты.  

  

На выходе мы получаем несколько версий дообученной LLama 2 и лучшую отправляем в KServe — serving engine (виртуальную среду, в которой модель будет сервиться). Готово!  

  

Кажется, что все просто, но на практике могут возникнуть проблемы. Рассмотрим каждый этап эксперимента подробнее.  

  

[![](https://habrastorage.org/r/w1560/webt/ze/72/oj/ze72ojvtzli_zycrneh_pkyslpu.png)](https://selectel.ru/services/cloud/mlops/?utm_source=habr.com&utm_medium=referral&utm_campaign=cloud_article_llama_121023_banner_045_ord)  

Работа с датасетами
-------------------

  

**Проблема** в работе с данными нередко заключается в том, что вы не знаете:  

  

* *общих фактов по датасету* — например, автора или объема данных,
* *базовых статистик* по признакам, целевой переменной и другим метрикам,
* *какие трансформации были проведены с датасетом*,
* *как работать с датасетом* — как при помощи сущности, описывающей датасет, выгрузить данные с внешнего источника (например, с S3 или SFS) и перевести их в нужный формат для работы с PyTorch, TensorFlow или Transformers,
* *откуда появился датасет* — например, из каких данных был собран, где и кем использовался,
* *как менялся датасет*.

  

Все это вносит неопределенность при работе с данными: может элементарно отсутствовать понимание, какие параметры действительно важны, а какие нет. На подобные разбирательства, как правило, времени уходит много.  

  

### Как ускорить работу с датасетами

  

Рассмотрим в качестве примеров два сервиса, с помощью которых можно ускорить работу с датасетами.  

  

#### Пример 1: Hugging Face

  

Разработчики Hugging Face позаботились о том, чтобы авторы могли указывать при загрузке данных всю необходимую информацию.  

  

![](https://habrastorage.org/r/w1560/getpro/habr/post_images/e42/95a/551/e4295a55191d6682b2f78625bb7a0113.png)  

В результате любой пользователь может изучить библиотеку датасетов, найти необходимый тип данных для своей задачи, предварительно оценить его по объему. Кроме того, в Hugging Face есть понятные примеры с кодом — бери и используй — и версионирование. Последнее [напоминает GitHub](https://huggingface.co/docs/hub/repositories-getting-started): вы можете открыть один из старых коммитов и склонировать именно его.  

  


> Мы уже писали статью о том, как автоматизировать сбор, анализ и сравнение датасетов из Hugging Face — читайте по [ссылке](https://habr.com/ru/companies/selectel/articles/761948/).

  

#### Пример 2: ClearML

  

ClearML также существенно упрощает работу с датасетами и сводит ее к графическому интерфейсу. Например, внутри отдельной вкладки с датасетами вы можете группировать их по папкам:  

  

![](https://habrastorage.org/r/w1560/getpro/habr/post_images/831/369/438/83136943868860a458e1460ae097bb56.png)  

Как и в Hugging Face, пользователь ClearML отображает метаинформацию о датасетах, позволяет настроить разделение данных по типу, задаче и любой другой категории. А также — изучить базовую информацию о составе загруженных датасетов.  

  

![](https://habrastorage.org/r/w1560/getpro/habr/post_images/eea/fc6/694/eeafc66948105323f34f3d5f3e13e402.png)  


> Любой датасет в ClearML (как любой эксперимент или пайплайн) создается, изменяется и используется в рамках «[задачи](https://github.com/allegroai/clearml/blob/master/clearml/backend_interface/task/task.py%23L63)» определенного [типа](https://github.com/allegroai/clearml/blob/master/clearml/backend_interface/task/task.py%23L89) — за подробностями рекомендую покопаться в [коде](https://github.com/allegroai/clearml/blob/master/clearml/datasets/dataset.py%23L210). То есть вся функциональность по работе с экспериментами доступна и для датасетов.

  

Рассмотрим пример. Вы загрузили в ClearML датасет, подчистили его с помощью регулярных выражений и передали коллеге. Он загружает данные и видит статистики по трансформациям: какие выражения к каким записям были применены и т. д. Результат: человек «понимает датасет», у него есть необходимая информация для работы с ним, а вы сэкономили время обеих сторон.  

  

![](https://habrastorage.org/r/w1560/getpro/habr/post_images/3e6/ed6/90d/3e6ed690d12962c54576d3f21257c836.png)  

#### Пример 3: кэширование данных

  

Еще один способ ускорить работу с датасетами — настроить кэширование в файловое хранилище. Загрузив данные один раз, вам не придется с нуля подгружать датасеты и веса при каждом запуске эксперимента. Это особенно актуально, если вы работаете с большими моделями вроде LLama 2, веса которой занимают свыше 100 ГБ памяти (файлы модели *Llama-2-70b-chat-hf* суммарно [занимают](https://huggingface.co/meta-llama/Llama-2-70b-chat-hf/tree/main) ~138 ГБ).  

  


> Эта фича уже имплементирована в [ML-платформу Selectel](https://slc.tl/s26vs). Вы можете кэшировать датасеты и веса, чтобы сократить время на подгрузке данных. Вся инфраструктура уже настроена за вас.

  

### Аспекты ускорения

  

Примеры рассмотрели — какой можно сделать вывод? Вернемся к техническим и процессным аспектам: зарезюмируем, какие в работе с датасетами через ClearML с кэшированием данных.  

  

#### Технические аспекты

  

* Все собрано в одной унифицированной платформе.
* Можно повторно использовать код для создания новых версий датасета.
* Не нужно каждый раз тратить время на загрузку данных, все подгружается из кэша.

  

#### Процессные аспекты

* Описания датасетов приведены к единому формату.
* Данные сопровождаются подробными инструкциями.

  

  

Работа с моделями
-----------------

  

**Проблема** в работе с моделями напоминает ситуацию с датасетами. Нередко вы не знаете:  

  

* *кто, где, когда учил или валидиновал модель,*,
* *класс модели* — например, какой структурой она является: нейросетью, деревом или лесом.
* *какие данные нужно подавать на вход, что получается на выходе*,
* *из чего модель состоит* — например, сколько у нее слоев или функций активации,
* *как работать с моделью* — как при помощи сущности, описывающей модель, выгрузить данные с внешнего источника (например, с S3 или SFS) и перевести их в нужный формат для работы с PyTorch, TensorFlow или Transformers,
* *полный набор* *гиперпараметров* *модели*,
* *отличия модели от предшественников*.

  

### Как ускорить работу с моделями

  

Рассмотрим в качестве примеров два сервиса, с помощью которых можно ускорить работу с моделями.  

  

#### Пример 1: Hugging Face

  

Как в случае с датасетами, модели в Hugging Face разделены по задачам и типам данных, у них есть описания и авторы. Точно так же можно посмотреть, как менялась модель, изучить детали по работе с ней.  

  

![](https://habrastorage.org/r/w1560/getpro/habr/post_images/fc0/89a/c81/fc089ac8133eec678198346f9374a1cf.png)  

#### Пример 2: Jupyter и ClearML

  

Поскольку нам нужно не только хранить информацию о моделях, но и дообучать их, нужна «рабочая станция» с помощью которой мы подгрузим веса и начнем с ними работать. Приемлемый вариант — использовать связку Jupyter и ClearML. Она уже имплементирована в нашу ML-платформу.  

  

Пользователю не нужно настраивать среду разработки и работу с GPU, все уже готово. Вся информация об экспериментах собрана в ClearML. Можно посмотреть автора модели, сохранять логи и другие данные по экспериментам.  

  

![](https://habrastorage.org/r/w1560/getpro/habr/post_images/660/b3d/a7a/660b3da7a2736d8a38d4d5a6838e12f9.png)  

Кроме того, все параметры модели можно легко переносить из Jupyter в ClearML, а после — использовать для проверок гипотез: варьировать значения и смотреть на результаты.  

  

![](https://habrastorage.org/r/w1560/getpro/habr/post_images/91c/418/835/91c418835620331eca0f794abbc9bbda.png)  

*Перенос параметров модели из Jupyter в ClearML.*  

  

После дообучения ClearML автоматически выведет информацию по утилизации ресурсов GPU и процессу обучения (последнее приходится логировать в Tensorboard-логи, но на этом все — ClearML сам подтягивает данные в свою панель):  

  

![](https://habrastorage.org/r/w1560/getpro/habr/post_images/169/968/d75/169968d7500b7fea98980d878f4d4bd0.png)  

### Аспекты ускорения

  

#### Технические аспекты

  

→ Не нужно тратить время на воссоздание рабочего окружения.  

  

→ Код можно повторно использовать для дообучения модели.  

  

#### Процессные аспекты

  

→ Информация по экспериментам организована и сведена в одну точку.  

  

Деплой модели
-------------

  

Датасет загрузили, проверили гипотезы и дообучили модель — осталось довести ее до конечного пользователя. То есть поднять некий сервис, через который наши друзья, коллеги и простые незнакомцы смогут общаться с моделью.  

  

**Проблемы:**  

  

* *писать веб-обвязку для модели может быть затруднительно,*
* *нужно настроить логирование,*
* *модели должны масштабироваться под нагрузкой и обновляться до актуальных версий.*

  

### Как ускорить деплой

  

Для начала посмотрим, какими средствами можно упростить и ускорить деплой в Hugging Face.  

  

#### Пример 1: Hugging Face

  

Прямо внутри платформы вы можете создать свой спейс, набросать пользовательский интерфейс и расшарить к нему доступ. Так пользователи смогут общаться с инференс-инстансом вашей модели.  

  

![](https://habrastorage.org/r/w1560/getpro/habr/post_images/b81/d5f/a68/b81d5fa6801fd989a89b52b9b93cc151.png)  

*Hugging Face, форма создания спейса.*  

  

Эта фича доступна для всех пользователей Hugging Face — посмотреть готовые варианты можно по [ссылке](https://huggingface.co/spaces).  

  

![](https://habrastorage.org/r/w1560/getpro/habr/post_images/6f8/d13/a01/6f8d13a010d27ef61b01afa9ccefbdd6.png)  

*Hugging Face, раздел с пользовательскими* *спейсами**.*  

  

У Hugging Face есть также сервис [Inference Endpoints](https://ui.endpoints.huggingface.co/welcome), но он платный.  

  

#### Пример 2: KServe

  

Альтернативный вариант, который мы используем в ML-платформе Selectel, — KServe. Необходимо опубликовать новую версию инференс-сервиса, закинув в KServe идентификатор дообученной модели, и пользователи смогут отправлять по ссылкам запросы, а модель — отвечать на них. Если быть технически более точным, то:  

  

* идентификатор модели достается из ClearML-эксперимента и кладется в кастомный инференс-скрипт,
* кастомный инференс-скрипт запекается в Docker-образ,
* полное имя Docker-образа указывается в манифесте инференс-сервиса KServe,
* манифест инференс-сервиса публикуется в KServe.

  

Поскольку Dockerfile, манифест инференс-сервиса и сам кастомный инференс-скрипт почти что всегда остаются без изменений, на их повторном использовании можно сэкономить много времени.  

  

![](https://habrastorage.org/r/w1560/getpro/habr/post_images/b1e/c58/a04/b1ec58a04f0e21b2fa36b95bb4468d16.png)  

*KServe**, ссылки для работы с* *инференс**-инстансом модели.*  

  

Так пользователь может, например, отправить запрос модели с промтом в качестве POST-параметра. А инференс-инстанс ответит текстом или ссылкой на изображение (зависит от модели, которую вы используете). Есть разные протоколы, в соответствии с которыми формируются тела запросов, ссылки и другие «сущности». Более подробно можно прочитать в [документации KServe](https://kserve.github.io/website/0.10/modelserving/data_plane/v1_protocol/).  

  

### Аспекты ускорения

  

#### Технические аспекты

  

→ Пишем только логику обработки входных-выходных данных.  

  

→ Веб-обвязку, логирование и масштабирование забирает на себя KServe.  

  

#### Процессные аспекты

  

→ Создание нового инференса свелось к замене идентификатора на новую модель.  

  

Как организовывать эксперименты без навыков MLOps
-------------------------------------------------

  

Мы рассмотрели проведение ML-экспериментов от подготовки данных до деплоя инференс-инстансов. И выяснили, что ускорение экспериментов можно свести к трем пунктам:  

  

* **Введению ряда подробно описанных соглашений.** Когда не только вы, но и ваши коллеги знают, как устроены датасеты и модели внутри эксперимента, какие гиперпараметры и параметры влияют на результат, проверка гипотез значительно ускоряется. Поэтому важно вводить соглашения, унифицировать типы данных и описания датасетов с моделями.
* **Повторному использованию рабочих окружений и кода для подготовки данных и обучения моделей.** Если вы единожды написали скрипт, который можно многократно переиспользовать для новых ML-экспериментов, меняя только параметры, вы сэкономили время. Принцип DRY работает не только в классической разработке.
* **Использованию инструментов, с помощью которых можно абстрагироваться от мелких технических деталей.** Конфигурирование инструментов вроде Nvidia-драйверов, CUDA Toolkit и рабочих окружений часто занимает много времени. Ситуация усложняется, когда нужно настроить шеринг ресурсов GPU, чтобы сразу несколько специалистов могли работать на одной машине. Если все сконфигурировано за вас — профит, вы сэкономили много времени.

  

С последним мы можем помочь. В сентябре вышла из беты ML-платформа Selectel — облачное решение с преднастроенными аппаратными и программными компонентами для обучения и развертывания ML-моделей.  

  

Мы разворачиваем платформу индивидуально для каждого клиента и можем включить в сборку такие open source-инструменты, как ClearML или Kubeflow. В общем, все для того, чтобы вы смогли организовать полный цикл обучения и тестирования ML-моделей.  

  

Тем, кто заинтересуется платформой, мы предоставим [бесплатный двухнедельный тестовый период](https://slc.tl/s26vs). В рамках тестирования вы сможете выбрать интересующие модели GPU и провести несколько экспериментов. На это время мы создадим чат в Telegram для срочных вопросов по работе платформы.  

  


> **Остались вопросы?** Задавайте их в комментариях, присоединяйтесь к обсуждениям, а также — смотрите полную версию [доклада про ускорение ML-экспериментов](https://www.youtube.com/live/J5drdpS-KS4?feature=shared&t=10619).

  

  

Материалы по теме
-----------------

  

* [Как развернуть нейросеть в облаке за 5 минут? Обзор библиотеки Diffusers](https://slc.tl/7wa4w)
* [Подойдет ли PostgreSQL вообще всем проектам или нужны альтернативы](https://slc.tl/7oqtv)
* [Как упростить анализ данных? Запуск и сценарии использования готовой виртуальной машины для аналитики](https://slc.tl/4yinb)