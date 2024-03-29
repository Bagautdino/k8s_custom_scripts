# Руководство по обновлению MongoDB в Kubernetes с использованием Bitnami/MongoDB

Это руководство описывает процесс обновления MongoDB в среде Kubernetes, в частности, с использованием чарта Bitnami/MongoDB как зависимости. Важно обеспечить совместимость версий `mongodump` и `mongorestore`, чтобы избежать ошибок во время процесса обновления.

## Предварительные требования

- Кластер Kubernetes
- kubectl, настроенный для целевого кластера
- Доступ к подам MongoDB в кластере Kubernetes

## Шаг 1: Создание дампа базы данных

Перед началом обновления крайне важно сделать резервную копию вашей базы данных. Ниже приведены шаги для экспорта базы данных на локальное хранилище с использованием `mongodump`.

1. Откройте проброс порта, чтобы получить доступ к MongoDB с вашего локального компьютера:

```bash
kubectl port-forward pod/<pod-name> 20000:27017 -n <namespace>
```

2. Используйте `mongodump` для экспорта базы данных:

```bash
mongodump --port="20000" --archive=<database-name> -v --username=<username> --password=<password>
```

Убедитесь, что версия `mongodump` совпадает с версией `mongorestore`, которая будет использоваться позже, чтобы избежать проблем совместимости.

## Шаг 2: Последовательное обновление

Обновляйте версию MongoDB последовательно, чтобы избежать резких переходов на новые версии, которые могут вызвать проблемы совместимости.

1. Начните с обновления с версии `10.0.0` до `12.16.0`, при этом версия приложения MongoDB обновится до `6.0.0` (Debian).

2. Проверьте, что обновленная версия функционирует корректно, прежде чем продолжать.

## Шаг 3: Устранение ошибки nonCompatibleVersion

Если при обновлении с `12.16.0` до `13.0.0` появляется ошибка `nonCompatibleVersion`, выполните следующие шаги:

1. Масштабируйте развертывание до 0:

```bash
kubectl scale deployment <deployment-name> --replicas=0 -n <namespace>
```

2. Удалите существующий запрос на постоянный том (PVC), чтобы удалить старые данные и разрешить новую инициализацию:

```bash
kubectl get pvc -n <namespace>
kubectl delete persistentvolumeclaim <pvc-name> --namespace <namespace>
```

3. Масштабируйте развертывание обратно, что может не сработать напрямую. Вместо этого используйте задание (job) или аналогичный метод для повторного развертывания службы.

## Шаг 4: Восстановление базы данных

После очистки базы данных и обновления MongoDB до версии 6 восстановите ранее сделанный дамп базы данных:

1. Снова инициируйте проброс порта:

```bash
kubectl port-forward pod/<pod-name> 20000:27017 -n <namespace>
```

2. Перейдите в папку, содержащую дамп базы данных, и используйте `mongorestore` для импорта данных:

```bash
mongorestore --archive=<database-name> --drop -u root --authenticationDatabase admin --verbose=5 --port=20000
```

## Шаг 5: Целевая мажорная версия достигнута, теперь обновляем минорную

Ставим самую последнюю минорную версию.
Если возникают ошибки ставим ниже.

