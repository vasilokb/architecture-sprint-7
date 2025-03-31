
# Инструкция по выполнению задания 5: Управление трафиком внутри кластера Kubernetes

**Цель:** Развернуть четыре сервиса в одном namespace с образом Nginx, изолировать трафик к одному из них и настроить сетевые политики для разделения трафика между парами API и UI.  

---

## Требования
1. **Развернуть четыре сервиса в namespace `system`:**
    - Имена: `front-end-app`, `back-end-api-app`, `admin-front-end-app`, `admin-back-end-api-app`.
    - Метки: `role=front-end`, `role=back-end-api`, `role=admin-front-end`, `role=admin-back-end-api`.
    - Использовать команду: `kubectl run <name> --image=nginx --labels role=<role> --expose --port 80`.
2. **Изолировать `admin-back-end-api-app`:**
    - Запретить трафик от всех подов, кроме `admin-front-end-app`.
3. **Настроить сетевые политики для пар:**
    - Разрешить двусторонний трафик между `front-end-app` и `back-end-api-app`.
    - Разрешить двусторонний трафик между `admin-front-end-app` и `admin-back-end-api-app`.
    - Запретить остальной трафик между сервисами.

---

## Структура директорий
Создаём следующую структуру на диске `D:\Practicum\sprint-7\propdevelopment\`:
```
D:\Practicum\sprint-7\propdevelopment\
├── manifests\
│   ├── services\
│   │   ├── isolate-admin-back-end-api.yaml
│   │   └── non-admin-api-allow.yaml
├── scripts\
│   ├── deploy-services.bat
│   └── verify-traffic.bat
```

- **Описание структуры:**
    - `manifests\services\` — директория для YAML-файлов с сетевыми политиками:
        - `isolate-admin-back-end-api.yaml` — изоляция сервиса `admin-back-end-api-app`.
        - `non-admin-api-allow.yaml` — настройка трафика между `front-end-app` и `back-end-api-app`.
    - `scripts\` — директория для скриптов Windows:
        - `deploy-services.bat` — развёртывание сервисов и применение сетевых политик.
        - `verify-traffic.bat` — создание тестового pod для проверки трафика.

---

## Содержимое файлов

### 1. Манифесты для сетевых политик

#### `D:\Practicum\sprint-7\propdevelopment\manifests\services\isolate-admin-back-end-api.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-admin-back-end-api
  namespace: system
spec:
  podSelector:
    matchLabels:
      role: admin-back-end-api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: admin-front-end
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: admin-front-end
    ports:
    - protocol: TCP
      port: 80
```
- **Описание:**
    - Применяется к pod с меткой `role=admin-back-end-api` (т.е. `admin-back-end-api-app`).
    - Разрешает входящий трафик (ingress) только от pod с меткой `role=admin-front-end` (т.е. `admin-front-end-app`) на порт 80.
    - Разрешает исходящий трафик (egress) только к pod с меткой `role=admin-front-end` на порт 80.
    - Запрещает весь остальной трафик к/от `admin-back-end-api-app`.

#### `D:\Practicum\sprint-7\propdevelopment\manifests\services\non-admin-api-allow.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: non-admin-api-allow
  namespace: system
spec:
  podSelector:
    matchLabels:
      role: back-end-api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: front-end
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: front-end
    ports:
    - protocol: TCP
      port: 80
```
- **Описание:**
    - Применяется к pod с меткой `role=back-end-api` (т.е. `back-end-api-app`).
    - Разрешает входящий трафик (ingress) только от pod с меткой `role=front-end` (т.е. `front-end-app`) на порт 80.
    - Разрешает исходящий трафик (egress) только к pod с меткой `role=front-end` на порт 80.
    - Запрещает весь остальной трафик к/от `back-end-api-app`.

### 2. Скрипты для Windows

#### `D:\Practicum\sprint-7\propdevelopment\scripts\deploy-services.bat`
```bat
@echo off
ECHO Deploying services in system namespace...

ECHO Step 1: Creating front-end-app...
kubectl run front-end-app --image=nginx --labels role=front-end --expose --port 80 -n system

ECHO Step 2: Creating back-end-api-app...
kubectl run back-end-api-app --image=nginx --labels role=back-end-api --expose --port 80 -n system

ECHO Step 3: Creating admin-front-end-app...
kubectl run admin-front-end-app --image=nginx --labels role=admin-front-end --expose --port 80 -n system

ECHO Step 4: Creating admin-back-end-api-app...
kubectl run admin-back-end-api-app --image=nginx --labels role=admin-back-end-api --expose --port 80 -n system

ECHO Step 5: Applying network policies...
kubectl apply -f ..\manifests\services\isolate-admin-back-end-api.yaml
kubectl apply -f ..\manifests\services\non-admin-api-allow.yaml

ECHO All services deployed and network policies applied.
pause
```
- **Описание:**
    - Создаёт четыре pod и соответствующие им сервисы с помощью `kubectl run`.
    - Применяет две сетевые политики для управления трафиком.

#### `D:\Practicum\sprint-7\propdevelopment\scripts\verify-traffic.bat`
```bat
@echo off
ECHO Starting traffic verification...

ECHO Creating test pod with random name...
set POD_NAME=test-pod-%RANDOM%
kubectl run %POD_NAME% --rm -i -t --image=alpine -n system -- sh

ECHO When inside the pod, run these commands:
ECHO   apk add curl
ECHO   curl --connect-timeout 2 http://front-end-app:80
ECHO   curl --connect-timeout 2 http://back-end-api-app:80
ECHO   curl --connect-timeout 2 http://admin-front-end-app:80
ECHO   curl --connect-timeout 2 http://admin-back-end-api-app:80
ECHO Type 'exit' to close the pod.

pause
```
- **Описание:**
    - Создаёт временный pod с уникальным именем (например, `test-pod-12345`) для проверки трафика.
    - Открывает оболочку внутри pod и выводит инструкции для тестирования.

---

## Пошаговая инструкция

### Шаг 1: Подготовка окружения
1. **Проверка инструментов:**
    - Откройте Windows Terminal.
    - Выполните команды для проверки установленных инструментов:
      ```
      minikube version
      docker --version
      kubectl version --client
      ```
    - Если что-то отсутствует:
        - **Minikube:** Скачайте с [официального сайта](https://minikube.sigs.k8s.io/docs/start/), установите, добавьте в PATH (например, `C:\Program Files\Minikube`).
        - **Docker:** Установите Docker Desktop с [официального сайта](https://www.docker.com/products/docker-desktop/), запустите.
        - **kubectl:** Скачайте с [Kubernetes сайта](https://kubernetes.io/docs/tasks/tools/), добавьте в PATH (например, `C:\kubectl`).

2. **Запуск Minikube с Calico:**
    - Остановите и удалите существующий кластер (если есть):
      ```
      minikube stop
      minikube delete
      ```
    - Запустите новый кластер с поддержкой сетевых политик:
      ```
      minikube start --driver=docker --memory=4096 --cpus=2 --network-plugin=cni --cni=calico
      ```
    - Ожидаемый вывод:
      ```
      * minikube vX.X.X on Microsoft Windows
      * Using the docker driver
      * Starting control plane node minikube in cluster minikube
      * Creating docker container (CPUs=2, Memory=4096MB) ...
      * Done! kubectl is now configured to use "minikube" cluster
      ```

### Шаг 2: Создание структуры директорий и файлов
1. **Создание директорий:**
    - Выполните в Windows Terminal:
      ```
      mkdir D:\Practicum\sprint-7\propdevelopment\manifests\services
      mkdir D:\Practicum\sprint-7\propdevelopment\scripts
      ```

2. **Создание манифестов:**
    - **Файл `isolate-admin-back-end-api.yaml`:**
        - Откройте Notepad.
        - Скопируйте код:
          ```yaml
          apiVersion: networking.k8s.io/v1
          kind: NetworkPolicy
          metadata:
            name: isolate-admin-back-end-api
            namespace: system
          spec:
            podSelector:
              matchLabels:
                role: admin-back-end-api
            policyTypes:
            - Ingress
            - Egress
            ingress:
            - from:
              - podSelector:
                  matchLabels:
                    role: admin-front-end
              ports:
              - protocol: TCP
                port: 80
            egress:
            - to:
              - podSelector:
                  matchLabels:
                    role: admin-front-end
              ports:
              - protocol: TCP
                port: 80
          ```
        - Сохраните как `D:\Practicum\sprint-7\propdevelopment\manifests\services\isolate-admin-back-end-api.yaml`.

    - **Файл `non-admin-api-allow.yaml`:**
        - Откройте Notepad.
        - Скопируйте код:
          ```yaml
          apiVersion: networking.k8s.io/v1
          kind: NetworkPolicy
          metadata:
            name: non-admin-api-allow
            namespace: system
          spec:
            podSelector:
              matchLabels:
                role: back-end-api
            policyTypes:
            - Ingress
            - Egress
            ingress:
            - from:
              - podSelector:
                  matchLabels:
                    role: front-end
              ports:
              - protocol: TCP
                port: 80
            egress:
            - to:
              - podSelector:
                  matchLabels:
                    role: front-end
              ports:
              - protocol: TCP
                port: 80
          ```
        - Сохраните как `D:\Practicum\sprint-7\propdevelopment\manifests\services\non-admin-api-allow.yaml`.

3. **Создание скриптов:**
    - **Файл `deploy-services.bat`:**
        - Откройте Notepad.
        - Скопируйте код:
          ```bat
          @echo off
          ECHO Deploying services in system namespace...
   
          ECHO Step 1: Creating front-end-app...
          kubectl run front-end-app --image=nginx --labels role=front-end --expose --port 80 -n system
   
          ECHO Step 2: Creating back-end-api-app...
          kubectl run back-end-api-app --image=nginx --labels role=back-end-api --expose --port 80 -n system
   
          ECHO Step 3: Creating admin-front-end-app...
          kubectl run admin-front-end-app --image=nginx --labels role=admin-front-end --expose --port 80 -n system
   
          ECHO Step 4: Creating admin-back-end-api-app...
          kubectl run admin-back-end-api-app --image=nginx --labels role=admin-back-end-api --expose --port 80 -n system
   
          ECHO Step 5: Applying network policies...
          kubectl apply -f ..\manifests\services\isolate-admin-back-end-api.yaml
          kubectl apply -f ..\manifests\services\non-admin-api-allow.yaml
   
          ECHO All services deployed and network policies applied.
          pause
          ```
        - Сохраните как `D:\Practicum\sprint-7\propdevelopment\scripts\deploy-services.bat`.

    - **Файл `verify-traffic.bat`:**
        - Откройте Notepad.
        - Скопируйте код:
          ```bat
          @echo off
          ECHO Starting traffic verification...
   
          ECHO Creating test pod with random name...
          set POD_NAME=test-pod-%RANDOM%
          kubectl run %POD_NAME% --rm -i -t --image=alpine -n system -- sh
   
          ECHO When inside the pod, run these commands:
          ECHO   apk add curl
          ECHO   curl --connect-timeout 2 http://front-end-app:80
          ECHO   curl --connect-timeout 2 http://back-end-api-app:80
          ECHO   curl --connect-timeout 2 http://admin-front-end-app:80
          ECHO   curl --connect-timeout 2 http://admin-back-end-api-app:80
          ECHO Type 'exit' to close the pod.
   
          pause
          ```
        - Сохраните как `D:\Practicum\sprint-7\propdevelopment\scripts\verify-traffic.bat`.

### Шаг 3: Создание namespace
- Выполните в Windows Terminal:
  ```
  kubectl create namespace system
  ```
- Ожидаемый вывод:
  ```
  namespace/system created
  ```

### Шаг 4: Развёртывание сервисов
1. **Перейдите в директорию скриптов:**
   ```
   cd D:\Practicum\sprint-7\propdevelopment\scripts
   ```

2. **Запустите скрипт для развёртывания:**
   ```
   deploy-services.bat
   ```
    - Ожидаемый вывод:
      ```
      Deploying services in system namespace...
      Step 1: Creating front-end-app...
      pod/front-end-app created
      service/front-end-app created
      Step 2: Creating back-end-api-app...
      pod/back-end-api-app created
      service/back-end-api-app created
      Step 3: Creating admin-front-end-app...
      pod/admin-front-end-app created
      service/admin-front-end-app created
      Step 4: Creating admin-back-end-api-app...
      pod/admin-back-end-api-app created
      service/admin-back-end-api-app created
      Step 5: Applying network policies...
      networkpolicy.networking.k8s.io/isolate-admin-back-end-api created
      networkpolicy.networking.k8s.io/non-admin-api-allow created
      All services deployed and network policies applied.
      Для продолжения нажмите любую клавишу . . .
      ```

### Шаг 5: Проверка трафика
1. **Запустите скрипт для проверки:**
   ```
   verify-traffic.bat
   ```
    - Ожидаемый вывод:
      ```
      Starting traffic verification...
      Creating test pod with random name...
      ```
    - Вы окажетесь внутри pod (например, `test-pod-12345`):
      ```
      / #
      ```

2. **Установите `curl` и выполните проверку:**
    - Внутри pod выполните команды:
      ```
      / # apk add curl
      ```
        - Ожидаемый вывод:
          ```
          fetch http://dl-cdn.alpinelinux.org/alpine/v3.XX/main/x86_64/APKINDEX.tar.gz
          ...
          (1/1) Installing curl (X.X.X-rX)
          OK: XX MiB in XX packages
          ```

      ```
      / # curl --connect-timeout 2 http://front-end-app:80
      ```
        - Ожидаемый результат: HTML-страница Nginx.

      ```
      / # curl --connect-timeout 2 http://back-end-api-app:80
      ```
        - Ожидаемый результат: HTML-страница Nginx.

      ```
      / # curl --connect-timeout 2 http://admin-front-end-app:80
      ```
        - Ожидаемый результат: HTML-страница Nginx.

      ```
      / # curl --connect-timeout 2 http://admin-back-end-api-app:80
      ```
        - Ожидаемый результат: ошибка, например:
          ```
          curl: (28) Connection timed out after 2001 ms
          ```

3. **Проверка парного взаимодействия:**
    - **Из `front-end-app`:**
      ```
      kubectl exec -it front-end-app -n system -- /bin/bash
      ```
        - Установите `curl` (Nginx-образ на основе Debian):
          ```
          root@front-end-app:/# apt-get update
          root@front-end-app:/# apt-get install -y curl
          ```
        - Проверьте:
          ```
          root@front-end-app:/# curl --connect-timeout 2 http://back-end-api-app:80
          ```
            - Ожидаемый результат: HTML-страница Nginx.
          ```
          root@front-end-app:/# curl --connect-timeout 2 http://admin-back-end-api-app:80
          ```
            - Ожидаемый результат: ошибка (трафик заблокирован).
        - Выйдите:
          ```
          root@front-end-app:/# exit
          ```

    - **Из `admin-front-end-app`:**
      ```
      kubectl exec -it admin-front-end-app -n system -- /bin/bash
      ```
        - Установите `curl`:
          ```
          root@admin-front-end-app:/# apt-get update
          root@admin-front-end-app:/# apt-get install -y curl
          ```
        - Проверьте:
          ```
          root@admin-front-end-app:/# curl --connect-timeout 2 http://admin-back-end-api-app:80
          ```
            - Ожидаемый результат: HTML-страница Nginx.
          ```
          root@admin-front-end-app:/# curl --connect-timeout 2 http://back-end-api-app:80
          ```
            - Ожидаемый результат: ошибка (трафик заблокирован).
        - Выйдите:
          ```
          root@admin-front-end-app:/# exit
          ```

4. **Выход из тестового pod:**
   ```
   / # exit
   ```
    - Pod удалится автоматически из-за `--rm`.

---

## Итог
- **Сервисы:** Развёрнуты `front-end-app`, `back-end-api-app`, `admin-front-end-app`, `admin-back-end-api-app` с метками `role`.
- **Сетевые политики:**
    - `admin-back-end-api-app` изолирован от всех, кроме `admin-front-end-app`.
    - Трафик разрешён между парами: `front-end-app` ↔ `back-end-api-app`, `admin-front-end-app` ↔ `admin-back-end-api-app`.
- **Проверка:** Трафик протестирован через тестовый pod и прямые подключения.

---

### Выполнение и проверка
1. Выполните все шаги с начала до конца.
2. Убедитесь, что трафик работает только между разрешёнными парами, как описано.
