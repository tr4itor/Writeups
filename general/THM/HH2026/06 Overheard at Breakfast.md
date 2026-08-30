**Теги:** OSINT, Hashing, Social Media.
**Сложность:** Easy.

## 1. Получение файла комнаты

Для начала получаем архив:

```text
overheard-at-breakfast-1784259780309.zip
```

Внутри архива находится изображение:

```text
conversation.png
```

На изображении представлена переписка между **Понци** и **Ламбо**.

---

## 2. Анализ переписки

В ходе разговора Понци пытается узнать контакт Ламбо в социальных сетях.

Ламбо сообщает, что сейчас почти не пользуется социальными сетями, однако раньше использовал бесплатный инструмент, который позволял загружать профиль и связывать с ним другие социальные аккаунты.

Он также говорит:

> «Начиналось, если я правильно помню, с буквы G.»

Кроме того, Ламбо оставляет свой email:

```text
lambobytelotushotel@gmail.com
```

Подсказка с буквой `G` и возможностью связывать профиль с другими аккаунтами позволяет предположить, что речь идёт о **Gravatar**.

---

## 3. Поиск Gravatar по email

Gravatar позволяет обращаться к аватару пользователя через MD5-хеш его email.

В логе используется следующий формат:

```text
https://www.gravatar.com/avatar/ХЭШ_ОТ_EMAIL
```

Следовательно, сначала необходимо получить MD5-хеш адреса:

```text
lambobytelotushotel@gmail.com
```

Для этого выполняем:

```bash
echo -n "lambobytelotushotel@gmail.com" | md5sum
```

Получаем хеш:

```text
d4a5fc5d3128890778667e24617d7cc0
```

После этого URL Gravatar будет иметь вид:

```text
https://www.gravatar.com/avatar/d4a5fc5d3128890778667e24617d7cc0
```

---

## 4. Поиск профиля

Используя полученную почту, через holehe находим профиль Gravatar:

```text
https://gravatar.com/cheerfullysongf28e3c3716
```

Профиль принадлежит:

```text
Lambo
```

Также в профиле указано:

```text
Lam-boh · Byte Lotus Hotel
```

Таким образом, подсказка из переписки действительно привела к профилю Ламбо.

---

## 5. Получение флага

На найденном профиле присутствует сообщение:

```text
Funny thing about email hashes, they follow you places you didn't expect. Glad you found the right corner of the internet! Here is your prize: VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9
```

Полученная строка имеет характерный формат Base64.

Декодируем:

```text
VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9
```

После декодирования получаем:

```text
THM{S3creT_Pr0fil3_H4s_b33n_Ident1fi3d}
```

---

## Итог

Цепочка решения:

```text
conversation.png
       │
       ▼
Переписка Понци и Ламбо
       │
       ▼
Email:
lambobytelotushotel@gmail.com
       │
       ▼
Подсказка "начиналось с G"
       │
       ▼
Gravatar
       │
       ▼
MD5 email
d4a5fc5d3128890778667e24617d7cc0
       │
       ▼
Поиск профиля Lambo
       │
       ▼
Base64-строка
VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9
       │
       ▼
Base64 decode
       │
       ▼
THM{S3creT_Pr0fil3_H4s_b33n_Ident1fi3d}
```
