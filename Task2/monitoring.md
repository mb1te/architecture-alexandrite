# Выбор и настройка мониторинга в системе

### Мотивация
Текущие проблемы:
- Неизвестные причины потери заказов
- Нет информации о производительности API, непонятно как заказы партнеров влияют на систему
- Жалобы на медленную загрузку главной страницы MES

Плюсы после внедрения мониторинга:
- Снижение времени диагностики проблем
- Раннее обнаружение деградации системы

### Выбор подхода к мониторингу

Для Shop API, CRM API, MES будет использован RED (Rate, Errors, Duration)
Для Shop DB, Messages Queue, MES DB будет использован USE (Utilization, Saturation, Errors)

Список метрик:

**MES API** (подход RED)
- RPS for MES API — для контроля нагрузки. Ярлыки: endpoint, method, status
- HTTP 500 for MES API — 5xx могут приводить к потере заказов. Ярлыки: endpoint, service
- Response time (latency) for MES API — операторы жалуются на медленную главную страницу. Ярлыки: endpoint, percentile (p50, p95, p99)
- CPU % for MES API — может влиять на производительность. Ярлыки: instance
- Memory Utilisation for MES API — OOM может вызвать потерю доступности. Ярлыки: instance

**Shop API** (подход RED)
- RPS for Shop API — контроль нагрузки. Ярлыки: endpoint, method, status
- HTTP 500 for Shop API — 5xx могут приводить к потере заказов. Ярлыки: endpoint, service
- Response time (latency) for Shop API — медленные ответы могут быть признаком инцидента с перегруженным сервисом. Ярлыки: endpoint, percentile (p50, p95, p99)

**CRM API** (подход RED)
- RPS for CRM API — контроль нагрузки. Ярлыки: endpoint, method, status
- HTTP 500 for CRM API — 5xx могут приводить к потере заказов. Ярлыки: endpoint, service
- Response time (latency) for CRM API — медленные ответы могут быть признаком инцидента с перегруженным сервисом, также влияет на скорость работы менеджеров. Ярлыки: endpoint, percentile (p50, p95, p99)

**RabbitMQ** (подход USE)
- Dead-letter-exchange — потеря сообщений ведет к потере заказов. Ярлыки: queue_name
- Messages in flight — больше очередь приводит к задержкам заказов. Ярлыки: queue_name

**MES DB** (подход USE)
- Memory Utilisation for MES DB — повышенная нагрузка при росте числа заказов. Ярлыки: instance
- Number of connections for MES DB — пул соединений может исчерпаться при росте числа заказов. Ярлыки: instance
- Size of MES DB instance — рост занимаемого места данными при увеличении числа заказов. Ярлыки: instance

**S3 Storage**
- Size of S3 storage — рост занимаемого места файлами 3D-моделей от клиентов. Ярлыки: bucket_name

### План действий

1. Развернуть Prometheus и Grafana (Yandex Cloud)
2. Добавить экспортеры:
    - rabbitmq_exporter для RabbitMQ
    - postgres_exporter для баз данных
    - Spring Boot Actuator для Shop и CRM (Java)
    - Prometheus middleware для MES (C#)
3. Настроить Grafana с дашбордами: API, RabbitMQ, DB
4. Настроить Alertmanager с уведомлениями в мессенджер или на почту

### Пороговые значения насыщенности

| Метрика | Порог | Действие |
|---------|-------|----------|
| CPU MES API | >80% в течение 5 мин | Горит алерт, DevOps думает о горизонтальном масштабировании |
| Dead-letter messages | >0 | Горит алерт, заводится тикет разработчикам на разбор |
| Messages in flight | >1000 | Горит алерт, DevOps проверяет, что консюмеры живы |
| MES API p95 latency | >5s | Горит алерт, заводится тикет разработчикам на оптимизацию |
| HTTP 500 rate | >1% от RPS | Горит алерт, разработчик оперативно разбирается в причинах |
| DB connections | >80% от max | Горит алерт, DevOps проверяет connection pool |