## Резюме
Репозиторий содержит выполнение дипломного проекта "Облачное хранилище” по курсу "Java программист".
## Постановка задачи
Разработать приложение - REST-сервис. Сервис должен предоставить REST интерфейс для возможности загрузки файлов и 
вывода списка уже загруженных файлов пользователя. Все запросы к сервису должны быть авторизованы. Заранее 
подготовленное веб-приложение (FRONT) должно подключаться к разработанному сервису без доработок, а также использовать 
функционал FRONT для авторизации, загрузки и вывода списка файлов пользователя.


## Описание:
Приложение построено по стандартной трех-слойной схеме, включающей три слоя:
- <b>Контроллеры.</b> Обеспечивают взаимодействие с пользователем или внешним API по протоколу HTTP. Они экспонируют
необходимый набор [энд-поинтов](https://github.com/mrRicochet/Cloud/blob/master/cloud-fileservice/src/main/resources/CloudServiceSpecification.yaml),
обеспечивают извлечение данных из поступающих запросов, формирование ответов на запросы, включая и ответы на запросы в
случае возникновения ошибок. Реализация указанных функций в классах пакетов controller и exceptionhandlers.
- <b>Сервисы</b>. Реализуют логику обработки данных, полученных от контроллеров. Реализация в классах пакета service.
- <b>Репозиторий</b>. Выполняет операции взаимодействия с базой данных, такие как извлечение и сохранение данных в БД. 
Реализация в интерфейсах Spring data JPA в пакете repository.<br><br>
Отдельным блоком выделены компоненты, выполняющие аутентификацию и авторизацию входящих запросов (security). В приложении
реализована аутентификация и авторизация с помощью jwt токенов. Все энд-поинты приложения требуют аутентификации за 
исключением /login, который доступен всем. Обратившись к /login пользователь получает в ответ jwt токен, который ему
необходимо прилагать в заголовке каждого запроса (кроме /login).<br>
Поскольку jwt токен является "самодостаточным" для
аутентификации и авторизации пользователя и никак не связан с сессией пользователя, дополнительно реализован механизм
учета токенов пользователей, выполнивших logout - черный список токенов. При выполнении logout, токен обратившегося
пользователя помещается в черный список. При аутентификации пользователя, специальный фильтр в security chain проверяет 
токен, в том числе на наличие его в черном списке и запрещает дальнейшие операции, если токен присутствует в черном
списке. Сам черный список реализован на базе библиотеки net.jodah.expiringmap, которая позволяет удобно реализовать
сохранение токенов в черном списке и автоматическое удаление токена из списка при наступлении момента времени, когда токен
прекращает свое действие. Для выполнения операций с токенами использована библиотека io.jsonwebtoken. Реализация классов
обеспечивающих security приложения находится в пакете security, настройка в config/WebSecurityConfig.<br><br>
Для хранения данных использована база данных PostgreSQL. Структура базы данных создается с помощью liquibase. 
Миграционные скрипты liquibase написаны так, чтобы можно было использовать любую реляционную базу данных. В случае,
если liquibase определяет наличие PostgreSQL, то он создает специфичные для этой БД некоторые типы данных (bytea для 
хранения файлов), а также некоторые специфичные constraints (проверка формата e-mail).<br>
Создание и администрирование пользователей приложения находится за рамками настоящего проекта. Три учетных записи
пользователей создаются в БД при старте приложения, если их там еще нет (пакет init).<br>
Настройки приложения находятся в файлах application.properties. Текстовые сообщения в messages.properties.<br>


## Тестирование приложения
Все написанные тесты формально являются интеграционными тестами. Они "поднимают" контекст приложения (@SpringBootTest).
С точки зрения логики тестирования и назначения тестов, условно можно разделить тесты приложения на две группы:
- <b>Unit test</b>. Эти тесты, выполняют тестирование методов сервисов и контролеров максимально обособленно 
(насколько это возможно, учитывая что контекст приложения все равно "поднимается") от других частей приложения. То есть 
в рамках этой группы тестов для каждого компонента, тестируется поведение его методов при максимально возможном 
mock-ировании (@MockBean) остальных сервисов. База данных используется та же, которая указана в профайле dev. Для того
чтобы минимизировать число инициализаций контекста приложения использован подход с объединением тестов в Suite, который
позволяет инициализировать контекст один раз на весь Suite.
Эти тесты собраны в src\test\java\com\humga\cloudservice\unittests.<br>
- <b>Full user stories test.</b> Эти тесты, выполняют тестирование типичного пользовательского сценария работы с приложением.
Эти тесты создают тест-контейнер приложения, тест-контейнер пустой базы данных и тест-контейнер самого теста 
(имитирует пользователя). Приложение при запуске инициализирует БД. Затем тест-котейнер (пользователя) выполняет ряд
операций типичного пользовательского сценария (логин, создание файлов, отображение списка, выход из приложения). Эти тесты
находятся в src/test/java/com/humga/cloudservice/TestContainersTests.java<br>
Для ручного создана коллекция postman src/test/Cloud API collection.postman_collection.json.<br>

## Сборка и запуск
Все скрипты для построения docker образов и запуска приложений в docker и docker-compose находятся в директории docker.
Скрипты необходимо запускать из корневой папки проекта. Скрипты для построения образов имеют префикс `docker-build`, 
скрипты для запуска `docker-run`. Для контейнера БД используется официальный latest образ PostgresSQL, который можно
запустить как контейнер с параметрами, указанными в файле docker-compose.yml в разделе db. Готовый к развертыванию
дистрибутив FRONTEND приложения включен в проект и находится в src/main/resources/frontend/dist. Для запуска его в
контейнере используется образ nginx:latest.  
Для сборки самого приложения в jar файл необходимо выполнить `mvn clean package`.

Для запуска приложения (все команды построения/запуска или скрипты .cmd нужно запускать из корневой папки проекта):
- Скачать latest docker образ postgreSQL
- Запустить docker контейнер postgreSQL командой из файла `docker/docker-run-db.cmd`
- Подключиться к postgreSQL в контейнере с помощью любого средcтва работы с БД, например DBeaver
- Создать в контейнере базу данных с именем `cloudstoragetest` и пользователем `postgres` c паролем `postgres` 
- Собрать бэкенд-приложение с помощью команды `mvn clean package`.
- Собрать docker-образ приложения с помощью команды из `docker/docker-build-backend.cmd`
- Скачать latest docker образ nginx
- Собрать docker-образ FRONTEND приложения с помощью команды из `docker/docker-build-frontend.cmd`
- Остановить контейнер базы данных
- Запустить все три контейнера (БД, backend, FRONTEND) с помощью команды из `docker/docker-compose-up.cmd`
- Зайти на http://localhost:80 и залогиниться в приложение под пользователем `alex@email.com` c паролем `passAlex`




