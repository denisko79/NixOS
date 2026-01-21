# Настройка Git и публикация конфигурации NixOS на GitHub

## 1. Настройка Git-идентификации

Выполни в терминале:

```bash
git config --global user.email "kumachev.denis@yandex.ru"
git config --global user.name "denisko79"
```

> ⚠️ Все твои коммиты в `/etc/nixos` (и других репозиториях) теперь будут подписаны этим именем и почтой.  
> Убедись, что `kumachev.denis@gmail.com` добавлена в [GitHub → Settings → Emails](https://github.com/settings/emails), иначе коммиты не привяжутся к твоему профилю.

---

## 2. Создание репозитория на GitHub

1. Зайди на [github.com](https://github.com) и войди в аккаунт.
2. Нажми **New repository** (или через меню `+` → **New repository**).
3. Заполни:
   - **Repository name**: например `nixos-x230i`, `nix-config` (без пробелов)
   - **Description**: `My NixOS configuration for ThinkPad X230i`
   - **Visibility**: `Private` или `Public` — по желанию
4. **Сними все галочки**:
   
   - ☐ Add a README file  
   - ☐ Add .gitignore  
   - ☐ Choose a license  
   
   > Это важно! Иначе репозиторий будет не пустым, и `git push` без `--force` не сработает.
5. Нажми **Create repository**.

---

## 3. Добавление remote и отправка кода

### Шаг 1: Скопируй URL репозитория
Пример HTTPS-URL:  
`https://github.com/denisko79/nixos-x230i.git`

### Шаг 2: Добавь remote в локальный репозиторий
Выполни в `/etc/nixos`:

```bash
git remote add origin https://github.com/denisko79/nixos-x230i.git
```

Проверь:

```bash
git remote -v
```

Должно быть:

```
origin  https://github.com/denisko79/nixos-x230i.git (fetch)
origin  https://github.com/denisko79/nixos-x230i.git (push)
```

### Шаг 3: Отправь изменения

```bash
git push -u origin main
```

> Если используется ветка `master` (старый стандарт), замени `main` на `master`.

---

## 4. Аутентификация при push

### Вариант A: HTTPS + Personal Access Token (PAT)

1. **Создай токен**:
   - Перейди в [GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)](https://github.com/settings/tokens)
   - Нажми **Generate new token** → **Generate new token (classic)**
   - Заполни:
     - **Note**: `nixos-x230i push`
     - **Expiration**: 30–90 дней или `No expiration`
     - **Scopes**: ✅ `repo` (полный доступ к репозиториям)
   - Нажми **Generate token** и **скопируй его** (показывается один раз!).

2. **При первом `git push`**:
   - **Username**: `denisko79`
   - **Password**: вставь **токен** (не пароль от аккаунта!)

3. *(Опционально)* Сохрани токен локально:

```bash
git config --global credential.helper store
```

> Токен сохранится в `~/.git-credentials` в открытом виде (допустимо для домашнего ноутбука).

---

### Вариант B: SSH (рекомендуется для долгосрочного использования)

1. **Сгенерируй SSH-ключ** (если нет):

```bash
ssh-keygen -t ed25519 -C "kumachev.denis@yandex.ru"
# Нажимай Enter на всех вопросах (или задай passphrase)
```

2. **Добавь публичный ключ в GitHub**:

```bash
cat ~/.ssh/id_ed25519.pub
```

Скопируй вывод → зайди в [GitHub → Settings → SSH and GPG keys → New SSH key](https://github.com/settings/keys)  
- **Title**: `X230i`  
- **Key**: вставь содержимое `id_ed25519.pub`  
- Нажми **Add SSH key**

3. **Измени remote на SSH**:

```bash
git remote set-url origin git@github.com:denisko79/nixos-x230i.git
```

4. **Проверь подключение**:

```bash
ssh -T git@github.com
```

Должно ответить: `Hi denisko79! You've successfully authenticated...`

5. **Пушь без паролей**:

```bash
git push -u origin main
```

> Теперь аутентификация работает автоматически!

---

## Что выбрать?

| Способ          | Плюсы                                        | Минусы                                                       |
| --------------- | -------------------------------------------- | ------------------------------------------------------------ |
| **HTTPS + PAT** | Быстро (2 минуты)                            | Нужно хранить токен; возможны запросы при каждом push (если нет credential helper) |
| **SSH**         | Удобно навсегда, безопасно, без ввода данных | Требует однократной настройки (5 минут)                      |

---

## Возможные ошибки

### ❌ `refusing to merge unrelated histories` или `non-fast-forward`
**Причина**: в репозитории на GitHub есть файлы (README, .gitignore и т.п.).

**Решение**:
- Лучше: удалить репозиторий и создать заново **пустым**.
- Или (осторожно!):  
  ```bash
  git push -u origin main --force-with-lease
  ```

---

## Краткий чек-лист для будущих изменений

```bash
cd /etc/nixos
git add .
git commit -m "Added TLP and brightnessctl"
git push
```

✅ Готово! Твой конфиг теперь в безопасности, под версионным контролем и доступен с любой машины.
```