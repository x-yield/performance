## Базовые знания методики

Нужно понимать и помнить какую задачу вы решаете в процессе нагрузочного
тестирования. В большинстве случаев задача заключается в том, чтобы
провести эксперимент, в результате которого мы проверим выдержит ли
система требуемую нагрузку, удовлетворяя требованиям.

При этом стоит понимать, что нагрузка бывает разная и зависит
от тестируемого сервиса, а точнее - от тех пользователей,
которые пользуются тестируемым сервисом. Можно классифицировать
нагрузку "по характеру" на два основных типа. Называют их все
по-разному ("инстансами" и "рпсами", "VU" vs "RPS", "закрытая"
и "открытая" модель нагрузки и т.д.), поэтому проще пояснить чем
один отличается от другого.

### Закрытая модель нагрузки / RPS
Количество пользователей системы исчислимо.
Интенсивность нагрузки зависит от того, как быстро отвечает сервис.
И чем быстрее отвечает тестируемый сервис, тем бОльшую нагрузку создают
пользователи. И процесс, который происходит в течение работы сервиса,
подчиняется множеству пользователей, использующих его. Говоря проще,
чем медленнее отвечает сервис, тем меньшую нагрузку пользователи создают
(в запросах в секунду). Для того, чтобы визуально представить как это
работает, вспомните любой гипермаркет и представьте, что это наш сервис.
Кассы в данном случае - это обработчики клиентов, их количество
ограничено. Чем медленнее они будут работать, тем меньше клиентов через
них пройдёт. Их производительность ограничена временем работы, а
нагрузка на условный сервис - количеством касс.

### Открытая модель нагрузки / instances
Количество пользователей неисчислимо. Интенсивность нагрузки не зависит
от того, как быстро отвечает сервис (до определенного ощутимого предела).
При замедлении времени ответа, нагрузка на условный сервис начинает
расти, растёт количество параллельно обрабатываемых запросов. Именно
так ведут себя пользователи типичного веб-сервиса в интернете.


## Какую модель использовать?

В подавляющем большинстве тестов используется открытая модель
нагрузки или RPS, количество "пользователей" при этом автоматически
регулируется инструментом по анализу результатов стрельбы за предыдущую
секунду. Запросы подаются в инстансы (сущность, которая делает один
запрос и ждёт ответ) в соответствии с расписанием теста, если вдруг
сервис начинает отвечать медленнее и текущего количества инстансов
становится недостаточно для создания нагрузки в соответствии со схемой
нагрузки, стрелялка подключает новые инстансы, параллельных запросов
становится больше. И наоборот.

Про это хорошо рассказывал на яндексовом митапе
Андрей Похилько (https://github.com/undera) в видео.