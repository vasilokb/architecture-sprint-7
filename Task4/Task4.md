# Защита доступа к кластеру Kubernetes


## Введение
PropDevelopment активно использует Kubernetes для развёртывания бизнес-сервисов. Для обеспечения безопасности кластера требовалось настроить ролевой доступ (RBAC) с учётом организационной структуры компании и защиты привилегированных действий (например, просмотра секретов). Задание выполнялось на пустом Minikube в среде Windows, что потребовало адаптации подхода из-за отсутствия интеграции с Active Directory.

---

## Контекст и требования
- **Требования:**
    - Ограничить доступ к управлению кластером для различных групп пользователей.
    - Защитить доступ к секретам, выделив привилегированную группу.
    - Создать минимум две дополнительные группы: с правами только на просмотр и для настройки кластера.
    - Разграничить доступ по организационной структуре компании.
- **Ограничения:** Пустой Minikube без Active Directory, работа в Windows Terminal.

---

## Выполнение задания

### 1. Определение ролей и групп пользователей
На основе требований и структуры компании (задание 1) были определены четыре роли и соответствующие группы:

| Роль            | Права роли                                                                                     | Группы пользователей         |
|-----------------|------------------------------------------------------------------------------------------------|------------------------------|
| **ClusterAdmin** | Полный доступ ко всем ресурсам кластера: управление подами, секретами, namespaces, RBAC, нодами | ClusterAdmins (ИБ, админы)  |
| **ReadOnly**     | Просмотр ресурсов (поды, сервисы, деплойменты, логи) в namespaces без права изменения и доступа к секретам | Managers, Analysts          |
| **Developer**    | Создание, редактирование, удаление подов, деплойментов, сервисов в своих namespaces (без секретов) | SalesDevs, SmartHomeDevs    |
| **SystemOperator** | Управление ресурсами в `system` (api-gateway, auth-service-1), доступ к секретам в `system`, настройка интеграций | SystemOps (DevOps, ИБ)     |

- **Пространства имён:** `sales`, `utilities`, `smarthome`, `finance`, `data`, `system` для изоляции ресурсов по доменам (задания 1 и 3).

### 2. Разворачивание Minikube
- **Команда:**
  ```
  minikube start --driver=docker --memory=4096 --cpus=2
  ```
- **Результат:** Пустой кластер запущен в Windows с использованием Docker как драйвера.

### 3. Структура директорий
Создана структура для хранения манифестов и скриптов:
```
D:\Practicum\sprint-7\propdevelopment\
├── manifests\
│   ├── namespaces\
│   │   ├── sales.yaml
│   │   ├── utilities.yaml
│   │   ├── smarthome.yaml
│   │   ├── finance.yaml
│   │   ├── data.yaml
│   │   └── system.yaml
│   ├── rbac\
│   │   ├── cluster-roles.yaml
│   │   ├── roles.yaml
│   │   └── bindings.yaml
│   │   └── test-bindings.yaml (временный для Minikube)
├── scripts\
│   ├── deploy-namespaces.bat
│   ├── setup-rbac.bat
│   └── verify-access.bat
```

### 4. Настройка RBAC

#### Манифесты
- **`cluster-roles.yaml`:**
    - `cluster-admin-role`: полный доступ ко всем ресурсам.
    - `readonly-cluster-role`: просмотр подов, сервисов, деплойментов.
- **`roles.yaml`:**
    - `developer-role` в `sales` и `smarthome`: управление подами, сервисами, деплойментами.
    - `system-operator-role` в `system`: управление ресурсами и секретами.
    - `readonly-role` в `utilities` и `data`: только просмотр.
- **`bindings.yaml`:**
    - Привязка ролей к группам (`ClusterAdmins`, `SalesDevs`, и т.д.) для реальной среды.
- **`test-bindings.yaml`:**
    - Временная привязка ролей к пользователю `minikube` для проверки в пустом Minikube.

#### Скрипты
1. **`deploy-namespaces.bat`:**
   ```bat
   @echo off
   ECHO Deploying namespaces...
   kubectl apply -f ..\manifests\namespaces\sales.yaml
   kubectl apply -f ..\manifests\namespaces\utilities.yaml
   kubectl apply -f ..\manifests\namespaces\smarthome.yaml
   kubectl apply -f ..\manifests\namespaces\finance.yaml
   kubectl apply -f ..\manifests\namespaces\data.yaml
   kubectl apply -f ..\manifests\namespaces\system.yaml
   ECHO Namespaces deployed.
   pause
   ```

2. **`setup-rbac.bat`:**
   ```bat
   @echo off
   ECHO Setting up RBAC...
   if exist ..\manifests\rbac\cluster-roles.yaml (
       kubectl apply -f ..\manifests\rbac\cluster-roles.yaml
   ) else (
       ECHO Error: cluster-roles.yaml not found!
   )
   if exist ..\manifests\rbac\roles.yaml (
       kubectl apply -f ..\manifests\rbac\roles.yaml
   ) else (
       ECHO Error: roles.yaml not found!
   )
   if exist ..\manifests\rbac\bindings.yaml (
       kubectl apply -f ..\manifests\rbac\bindings.yaml
   ) else (
       ECHO Error: bindings.yaml not found!
       exit /b 1
   )
   ECHO RBAC setup complete.
   pause
   ```

3. **`verify-access.bat` (адаптирован для Minikube):**
   ```bat
   @echo off
   ECHO Verifying access...
   ECHO Checking ClusterAdmin access (should have full permissions):
   kubectl auth can-i get pods --all-namespaces
   kubectl auth can-i get secrets --all-namespaces
   ECHO Checking SalesDevs access (should edit in sales, not elsewhere):
   kubectl auth can-i create pods -n sales
   kubectl auth can-i create pods -n utilities
   ECHO Checking SmartHomeDevs access (should edit in smarthome):
   kubectl auth can-i create deployments -n smarthome
   ECHO Checking SystemOps access (should manage system with secrets):
   kubectl auth can-i get secrets -n system
   kubectl auth can-i get pods -n sales
   ECHO Checking Managers access (should only view utilities):
   kubectl auth can-i get pods -n utilities
   kubectl auth can-i create pods -n utilities
   ECHO Checking Analysts access (should view utilities and data):
   kubectl auth can-i get pods -n utilities
   kubectl auth can-i get pods -n data
   kubectl auth can-i delete pods -n data
   ECHO Verification complete.
   pause
   ```

#### Последовательность применения
1. **Запуск Minikube:**
   ```
   minikube start --driver=docker --memory=4096 --cpus=2
   ```
2. **Создание пространств имён:**
   ```
   cd D:\Practicum\sprint-7\propdevelopment\scripts
   deploy-namespaces.bat
   ```
3. **Настройка RBAC:**
   ```
   setup-rbac.bat
   ```
4. **Применение тестовых привязок:**
   ```
   kubectl apply -f ..\manifests\rbac\test-bindings.yaml
   ```
5. **Проверка доступа:**
   ```
   verify-access.bat
   ```

### 5. Результаты
- **Вывод `verify-access.bat`:**
  ```
  Verifying access...
  Checking ClusterAdmin access (should have full permissions):
  yes
  yes
  Checking SalesDevs access (should edit in sales, not elsewhere):
  yes
  no
  Checking SmartHomeDevs access (should edit in smarthome):
  yes
  Checking SystemOps access (should manage system with secrets):
  yes
  no
  Checking Managers access (should only view utilities):
  yes
  no
  Checking Analysts access (should view utilities and data):
  yes
  yes
  no
  Verification complete.
  ```
- **Интерпретация:** Права соответствуют таблице ролей, RBAC работает корректно для пользователя `minikube`.

---

## Связь с предыдущими заданиями
- **Задание 1 (безопасность данных):** `SystemOperator` защищает секреты в `system`, `ReadOnly` ограничивает доступ к конфиденциальным данным.
- **Задание 2 (проверочный лист):** RBAC соответствует требованиям управления доступом и мониторинга.
- **Задание 3 (внешние интеграции):** `Developer` в `smarthome` поддерживает новые сервисы, `SystemOperator` управляет `api-gateway`.

---