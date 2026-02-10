# Добавление нового Telegram-бота (Bot Factory)

Пошаговая инструкция по добавлению нового бота в кластер через GitOps.

**Время:** ~10 минут  
**Требования:** `gh`, `git`, `kubectl`, `kubeseal`, доступ к GitHub

---

## 1. Создать бота в Telegram

1. Открой Telegram → `@BotFather`
2. Отправь `/newbot`
3. Задай имя и username
4. Сохрани **токен** (формат: `123456789:ABCdef...`)

> Токен НЕ коммитить в открытом виде. Только через SealedSecret.

---

## 2. Создать app-repo из шаблона

```bash
# Создать repo из template на GitHub
gh repo create gabela626-coder/MY_BOT \
  --template gabela626-coder/bot-template \
  --public --clone

cd MY_BOT

# Заменить placeholder на реальное имя
find . -type f -not -path './.git/*' \
  -exec sed -i 's/BOT_NAME/MY_BOT/g' {} +

# Проверить замену
grep -r "BOT_NAME" --include="*.py" --include="*.yml" .
# Должно быть пусто (все заменены)

# Коммит и push → CI автоматически соберёт image
git add -A
git commit -m "feat: init MY_BOT from template"
git push
```

Дождись завершения CI (GitHub Actions). Запомни **short SHA** коммита — это image tag.

```bash
git rev-parse HEAD | cut -c1-8
# Например: a1b2c3d4
```

---

## 3. Сгенерировать SealedSecret

```bash
# Создать plaintext secret (dry-run, НЕ применяем в кластер)
kubectl create secret generic MY_BOT-secret \
  -n infra \
  --from-literal=BOT_TOKEN=<ТОКЕН_ОТ_BOTFATHER> \
  --dry-run=client -o yaml > /tmp/MY_BOT-secret.yaml

# Зашифровать через kubeseal
kubeseal \
  --cert infra/keys/sealed-secrets-public.pem \
  -o yaml < /tmp/MY_BOT-secret.yaml \
  > /tmp/MY_BOT-sealedsecret.yaml

# Удалить plaintext!
shred -u /tmp/MY_BOT-secret.yaml
```

---

## 4. Добавить в GitOps repo

```bash
cd /path/to/boti-gitops

# 4a. Скопировать шаблон
cp -r apps/_template apps/MY_BOT

# 4b. Заменить placeholder
cd apps/MY_BOT
sed -i 's/BOT_NAME/MY_BOT/g' deployment.yaml sealedsecret.yaml application.yaml

# 4c. Установить image tag (short SHA из шага 2)
sed -i 's/CHANGE_ME/a1b2c3d4/g' deployment.yaml

# 4d. Скопировать настоящий SealedSecret
cp /tmp/MY_BOT-sealedsecret.yaml apps/MY_BOT/sealedsecret.yaml

# Удалить временный файл
rm /tmp/MY_BOT-sealedsecret.yaml

cd ../..
```

---

## 5. Зарегистрировать в ArgoCD (через app-definitions)

```bash
# 5a. Скопировать Application в app-definitions
cp apps/MY_BOT/application.yaml argocd/app-definitions/MY_BOT.yaml

# 5b. Добавить в kustomization.yaml
# Открыть argocd/app-definitions/kustomization.yaml и добавить строку:
#   - MY_BOT.yaml

# 5c. Коммит и push
git add apps/MY_BOT \
       argocd/app-definitions/MY_BOT.yaml \
       argocd/app-definitions/kustomization.yaml

git commit -m "feat: add MY_BOT to GitOps"
git push
```

---

## 6. Проверка в ArgoCD

1. Открой ArgoCD UI: `https://192.168.31.28:30080`
2. Дождись sync `root-app` (подхватит новый Application)
3. Проверь `MY_BOT`:
   - Status: **Synced**
   - Health: **Healthy**

```bash
# Или через kubectl:
kubectl get pod -n infra -l app=MY_BOT
kubectl logs -n infra -l app=MY_BOT --tail=10
```

Если в логах видишь `Application started` и `getUpdates` — бот работает.

---

## Чек-лист

- [ ] Бот создан в @BotFather
- [ ] App-repo создан из template, BOT_NAME заменён
- [ ] CI прошёл, image в GHCR
- [ ] SealedSecret сгенерирован, plaintext удалён
- [ ] GitOps манифесты добавлены (apps/MY_BOT/)
- [ ] Application зарегистрирован в app-definitions
- [ ] ArgoCD: Synced / Healthy
- [ ] Бот отвечает в Telegram на /start

---

## Именование

| Что | Формат | Пример |
|---|---|---|
| Repo | `MY_BOT` | `bot-weather` |
| Image | `ghcr.io/gabela626-coder/MY_BOT:TAG` | `ghcr.io/gabela626-coder/bot-weather:a1b2c3d4` |
| Deployment | `MY_BOT` | `bot-weather` |
| Secret | `MY_BOT-secret` | `bot-weather-secret` |
| ArgoCD App | `MY_BOT` | `bot-weather` |
| GitOps path | `apps/MY_BOT/` | `apps/bot-weather/` |
| Label | `app: MY_BOT` | `app: bot-weather` |
