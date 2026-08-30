**Теги:** OSINR, Web.

**Сложность:** Easy.
### 1. Загружаем изображение брошюры
В архиве был файл **thebrochure.png**. 

### 2. Находим скрытый аккаунт Instagram
**Команда:**  
Смотрим на изображение брошюры  — там есть текст «Find us on Instagram».

**Результат:**  
Аккаунт — https://www.instagram.com/thebytelotusresort

### 3. Ищем скрытую связь
В этом аккаунте единственный аккаунт, на который подписаны — https://www.instagram.com/veratheconcierge 

### 4. Извлекаем флаг с помощью igviewer.net
Открываем профиль через сервис https://igviewer.net/profile/veratheconcierge

**Результат:**  
Только 3 поста.  
Во всех трёх постах в описаниях — части **Base64** (закодировано по кусочкам)

### 5. Склеиваем и декодируем B64
```
echo "VEhNe1YzckBzX2FDQzB1bnRfaDRzX2IzM25fZjB1bmQhfQ==" | base64 -d
```

**Вывод команды:**
```
THM{V3r@s_aCC0unt_h4s_b33n_f0und!}
```

**Флаг найден!**
