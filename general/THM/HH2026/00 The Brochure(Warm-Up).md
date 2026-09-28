**Теги:** OSINT, Web.

**Складність:** Easy.

### 1. Завантажуємо зображення брошури
В архіві був файл **thebrochure.png**. 

### 2. Знаходимо прихований акаунт Instagram
**Команда:**  
Дивимося на зображення брошури — там є текст «Find us on Instagram».

**Результат:**  
Акаунт — https://www.instagram.com/thebytelotusresort

### 3. Шукаємо прихований зв’язок
У цього акаунта єдина підписка — https://www.instagram.com/veratheconcierge 

### 4. Витягуємо прапор за допомогою igviewer.net
Відкриваємо профіль через сервіс https://igviewer.net/profile/veratheconcierge

**Результат:**  
Лише 3 дописи.  
В описах усіх трьох дописів — частини **Base64** (закодовано частинами)

### 5. Об’єднуємо та декодуємо B64
```
echo "VEhNe1YzckBzX2FDQzB1bnRfaDRzX2IzM25fZjB1bmQhfQ==" | base64 -d
```

**Вивід команди:**
```
THM{V3r@s_aCC0unt_h4s_b33n_f0und!}
```

**Прапор знайдено!**
