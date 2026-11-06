# runedrive-gitops

Репозиторій конфігурації розгортання RuneDrive за моделлю GitOps (DevOps ЛР-5).
Бажаний стан кластера описаний тут; Argo CD сам приводить кластер до цього опису.
Ручного `helm upgrade --install` більше немає — єдиний застосований вручну об'єкт це `bootstrap/root.yaml`.

## Структура

| Шлях | Призначення |
| --- | --- |
| `bootstrap/root.yaml` | Кореневий Application (App of Apps). Стежить за каталогом `argocd-apps/`. Застосовується вручну один раз. |
| `argocd-apps/runedrive.yaml` | Application застосунку. Його `metadata.name` стає назвою Helm-релізу. |
| `charts/runedrive/` | Helm-чарт, перенесений з ЛР-4. |
| `charts/runedrive/values-minikube.yaml` | Значення для локального кластера minikube. |

## Як це працює

`root` (namespace `argocd`) читає `argocd-apps/` і створює звідти дочірні Application-и.
Дочірній `runedrive` рендерить чарт `charts/runedrive` зі значеннями `values-minikube.yaml`
і розгортає результат у namespace `runedrive`. Helm тільки рендерить маніфести,
у кластері після цього не асоціюється з жодним релізом (`helm list -n runedrive` — порожньо).

Новий компонент кластера додається одним файлом у `argocd-apps/`.

## Значення для minikube

| Параметр | Значення | Навіщо |
| --- | --- | --- |
| `image.repository` | `ghcr.io/kaziksuper/runedrive` | Публічний образ, кластер тягне сам, без `minikube image load` |
| `image.tag` | `1.0.0` | Конкретний тег, не `latest`; рухається только через Git |
| `image.pullPolicy` | `IfNotPresent` | Не тягти образ щоразу |
| `replicaCount` | `2` | Кількість реплік API |
| `existingSecret` | `runedrive-secret` | Secret створено вручну в кластері, у Git його значень немає |
| `ingress.enabled` | `false` | Доступ через `kubectl port-forward` |

## Секрети

Значення паролей токенів у цьому репозиторії відсутні. Якщо `existingSecret` задано,
шаблон Secret не рендериться, а Deployment посилається на Secret, створений вручну.
Argo CD не видаляє такий Secret навіть з `prune`, бо він не створений з Git.

```sh
kubectl create namespace runedrive
kubectl -n runedrive create secret generic runedrive-secret \
  --from-literal=POSTGRES_PASSWORD=... \
  --from-literal=JWT_SECRET_KEY=...
```

## Версія застосунку

Чарт передає контейнеру змінну оточення `APP_VERSION`, рівну `image.tag`.
Ендпоінт `/version` повертає її, тому зміна тега в Git одразу видно в відповіді застосунку.

```sh
kubectl -n runedrive port-forward svc/runedrive-runedrive-api 8081:80
curl -s localhost:8081/version
```

## Перший деплой з нуля

```sh
minikube start
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd port-forward svc/argocd-server 8080:443
kubectl apply -f bootstrap/root.yaml
argocd app list
```

## Див. також

`lab5_demo_commands.md` у папці лабораторної — розбір помилок і сценарій захисту.
