# DevOps Lab 9 — Kubernetes basics (Flask + Redis) + RollingUpdate

> **Репозиторий:** `torenDM/DevOpsLab9`  
> **Задание:** Flask+Redis в k8s (minikube), сервис Flask `LoadBalancer`, Redis `ClusterIP`, проверка работы, **RollingUpdate**. fileciteturn4file1

---

## 1) Цель работы
1. Развернуть **minikube** на VM1.  
2. “Куберизировать” стек **Flask + Redis**:
   - Flask: `Deployment` (5 replicas) + `Service` типа `LoadBalancer` (порт **8000 → 5000**, `externalIPs: [10.0.2.15]`) fileciteturn4file19
   - Redis: `Deployment` (1 replica) + `Service` типа `ClusterIP`  
3. Обеспечить доступ из Windows через `minikube tunnel` и проброс порта VirtualBox. fileciteturn4file7  
4. Выполнить **RollingUpdate** на новую версию образа Flask (`flask:v2`) через `kubectl set image ...`. fileciteturn4file5

---

## 2) Стенд
| Узел | Роль | IP | Подключение |
|---|---|---:|---|
| VM1 | minikube / kubectl | `10.0.2.15` | `ssh student@127.0.0.1 -p 2222` |
| VM2 | (не требуется для ЛР9) | `10.0.2.16` | `ssh student@127.0.0.1 -p 2223` |

**Доступ к приложению из Windows:** `http://127.0.0.1:18000` (проброс `host 18000 → guest 8000` на VM1). fileciteturn4file7

---

## 3) Структура репозитория
Минимум, который требуется положить в репо: `k8s/`, `README.md`, `screens/`. fileciteturn4file1

Пример дерева:
```
DevOpsLab9/
  k8s/
    flask-deploy.yaml
    flask-svc.yaml
    redis-deploy.yaml
    redis-svc.yaml
  flask_redis/            # исходники и Dockerfile (если используете их из Lab8)
  screens/
    01_nodes.png
    02_images.png
    03_pods.png
    04_services.png
    05_browser_v1.png
    06_rollout.png
    07_browser_v2.png
    08_tunnel.png
  README.md
```

---

## 4) Запуск и проверка (VM1)

### 4.1 Проверить minikube
```bash
minikube status
kubectl get nodes -o wide
```

### 4.2 Собрать образ Flask внутри minikube
> Билд делаем **внутри окружения minikube**, чтобы образ был доступен кластеру.

```bash
cd ~/DevOpsLab9
eval $(minikube docker-env)

# Если Dockerfile лежит в ./flask_redis
docker build -t flask:v1 ./flask_redis

minikube image ls | grep flask
```

### 4.3 Применить манифесты k8s
```bash
kubectl apply -f k8s/
kubectl get pods -o wide
kubectl get svc -o wide
```

Ожидаемо:
- 5 pod’ов Flask в `Running`
- 1 pod Redis в `Running`
- сервис Flask: `LoadBalancer`, порт **8000 → 5000**, `externalIPs: 10.0.2.15` fileciteturn4file19

---

## 5) Доступ с Windows (обязательно tunnel)
В **отдельном терминале на VM1** (не закрывать):
```bash
sudo minikube tunnel --bind-address 10.0.2.15
```
Затем:
- VirtualBox → VM1 → Port Forwarding: `host 18000 → guest 8000`
- Открыть в Windows: `http://127.0.0.1:18000` fileciteturn4file7

---

## 6) RollingUpdate (Flask v2)

### 6.1 Изменить текст в приложении (чтобы видеть обновление)
На VM1:
```bash
cd ~/DevOpsLab9
grep -RIn --exclude-dir=.git "DevOps Lab" flask_redis
```

Замените, например, `DevOps Lab 8` на `DevOps Lab 9 (v2)`:
```bash
FILE="./flask_redis/app.py"   # <-- замените на ваш путь из grep
cp -a "$FILE" "${FILE}.bak"
sed -i 's/DevOps Lab 8/DevOps Lab 9 (v2)/g' "$FILE"
grep -n "DevOps Lab" "$FILE"
```

### 6.2 Собрать новый образ `flask:v2`
```bash
cd ~/DevOpsLab9
eval $(minikube docker-env)
docker build -t flask:v2 ./flask_redis
minikube image ls | grep flask
```

### 6.3 Обновить Deployment
```bash
kubectl get deploy -o name
kubectl set image deployment/flask-app flask=flask:v2
kubectl rollout status deployment/flask-app
kubectl get pods -o wide
```

### 6.4 Проверка
На VM1:
```bash
curl -s http://10.0.2.15:8000 | head -n 5
```
В Windows:
- открыть `http://127.0.0.1:18000` и увидеть новый текст **(v2)**.

---

## 7) Скриншоты (screens/)
По методичке нужны скриншоты “pods 5 реплик, services, работающий endpoint”. fileciteturn4file1  
Ниже — подробный список “что именно должно быть видно”:

1. **`01_nodes.png`** — вывод:
   ```bash
   minikube status
   kubectl get nodes -o wide
   ```
   Должно быть видно `Ready` и что кластер `Running`.

2. **`02_images.png`** — вывод:
   ```bash
   minikube image ls | grep flask
   ```
   Должны быть видны теги `flask:v1` (и позже `flask:v2`).

3. **`03_pods.png`** — вывод:
   ```bash
   kubectl get pods -o wide
   ```
   Должно быть **5 pod’ов Flask** + **1 pod Redis**, статусы `Running`.

4. **`04_services.png`** — вывод:
   ```bash
   kubectl get svc -o wide
   ```
   Должно быть видно:
   - Flask-service = `LoadBalancer`
   - порт `8000/TCP`
   - `EXTERNAL-IP` / `externalIPs` = `10.0.2.15` fileciteturn4file19

5. **`05_browser_v1.png`** — Windows браузер `http://127.0.0.1:18000` **до обновления**.

6. **`06_rollout.png`** — вывод:
   ```bash
   kubectl rollout status deployment/flask-app
   kubectl get pods
   ```
   Должно быть видно успешное завершение rollout.

7. **`07_browser_v2.png`** — Windows браузер `http://127.0.0.1:18000` **после обновления**, текст содержит `DevOps Lab 9 (v2)`.

8. *(опционально, но полезно)* **`08_tunnel.png`** — окно терминала с:
   ```bash
   sudo minikube tunnel --bind-address 10.0.2.15
   ```
   где видно `Tunnel successfully started`. fileciteturn4file7

### Встраивание скриншотов в README (чтобы отображались на GitHub)
> Важно: имена файлов должны совпадать.

#### Проверки кластера
![](screens/01_nodes.png)

#### Образы
![](screens/02_images.png)

#### Pods (5 реплик)
![](screens/03_pods.png)

#### Services (LoadBalancer)
![](screens/04_services.png)

#### Браузер до/после RollingUpdate
![](screens/05_browser_v1.png)

![](screens/07_browser_v2.png)

#### Rollout
![](screens/06_rollout.png)

---

## 8) Push в GitHub (VM1)
```bash
cd ~/DevOpsLab9
git init
git add .
git commit -m "Lab9: k8s manifests + rolling update"
git branch -M main
git remote remove origin 2>/dev/null || true
git remote add origin https://github.com/torenDM/DevOpsLab9.git
git push -u origin main
```

---

## 9) Что считается “готово”
- В репо есть `k8s/`, `README.md`, `screens/` fileciteturn4file1  
- На скринах видно: **5 pod’ов Flask**, сервис **LoadBalancer**, работающий endpoint и успешный RollingUpdate fileciteturn4file1turn4file5
