**Теги:** Windows, Forensic, Persistence, Reverse Engineering.
**Сложность:** Medium.

## Описание задания

В ходе расследования был получен дамп репозитория WMI Windows:

```
INDEX.BTR
OBJECTS.DATA
MAPPING1.MAP
MAPPING2.MAP
MAPPING3.MAP
```

Цель исследования — определить, присутствует ли в репозитории механизм постоянного закрепления (WMI Persistence), и восстановить вредоносную полезную нагрузку (payload).

---

# Что такое WMI Repository?

Windows Management Instrumentation (WMI) хранит свои объекты в специальной базе данных — **Repository**.

Именно здесь находятся:

- пространства имён (Namespaces);
- классы WMI;
- экземпляры классов;
- постоянные подписки на события (Permanent Event Subscriptions).

Файлы репозитория:

```
INDEX.BTR
OBJECTS.DATA
MAPPING*.MAP
```

не являются текстовыми и не открываются обычными средствами. Для их анализа необходим специализированный парсер.

---

# Первичный анализ

Сначала определим тип файлов.

```bash
file INDEX.BTR
```

Получаем

```
INDEX.BTR: data
```

Попробуем найти печатные строки.

```bash
strings INDEX.BTR | head
```

Получаем только несколько внутренних идентификаторов:

```
CI_41C53E6DB1ACF2453CEFD41398198E613F10DFF47709ECAB1D7F037756AC8CE7
CI_FD1C1D414B71B5082C266A650E31EF6D5019382724244B685F217C0AAE00A921
CR_E3B0C44298FC1C149AFBF4C8996FB92427AE41E4649B934CA495991B7852B855
```

Полезной информации здесь нет.

---

# Попытка использовать готовый инструмент

Первоначально был использован проект **WMI_Forensics**.

```bash
git clone https://github.com/davidpany/WMI_Forensics
```

Запуск:

```bash
python3 PyWMIPersistenceFinder.py ../OBJECTS.DATA
```

завершился ошибкой

```
TypeError:
expected str instance, bytes found
```

Причина оказалась достаточно простой.

Проект написан под **Python 2**, а современные версии Python работают с байтовыми строками иначе. Вместо того чтобы исправлять старый код, было решено исследовать репозиторий самостоятельно.

---

# Использование библиотеки dissect.cim

Для работы с репозиторием была установлена библиотека:

```bash
python3 -m venv venv
source venv/bin/activate

pip install dissect.cim
```

Она предоставляет Python API для чтения WMI Repository.

Репозиторий открывается следующим образом:

```python
from dissect.cim import CIM

repo = CIM(
    open("INDEX.BTR","rb"),
    open("OBJECTS.DATA","rb"),
    [
        open("MAPPING1.MAP","rb"),
        open("MAPPING2.MAP","rb"),
        open("MAPPING3.MAP","rb")
    ]
)
```

Репозиторий успешно загрузился.

---

# Перечисление пространств имён

Первым делом необходимо понять структуру базы.

Выполнив перечисление Namespace, получили:

```
root
root\subscription
root\DEFAULT
root\CIMV2
root\SecurityCenter2
root\WMI
...
```

Особое внимание сразу привлекает

```
root\subscription
```

Именно здесь Windows хранит механизм **Permanent Event Subscription**, который злоумышленники очень любят использовать для закрепления в системе.

---

# Поиск экземпляров классов

Далее были перечислены все классы, имеющие экземпляры.

В пространстве

```
root\subscription
```

обнаружены:

```
__EventFilter
CommandLineEventConsumer
__FilterToConsumerBinding
NTEventLogEventConsumer
```

Это очень характерный набор объектов для WMI Persistence.

---

# Анализ Event Filter

В репозитории найдено два фильтра событий:

```
EngineTelemetryFilter

SCM Event Log Filter
```

Фильтр определяет, **при каком событии** будет запускаться полезная нагрузка.

---

# Анализ Event Consumer

Также найден объект

```
EngineTelemetryConsumer
```

типа

```
CommandLineEventConsumer
```

У данного класса присутствуют свойства

```
CommandLineTemplate
ExecutablePath
WorkingDirectory
```

Именно свойство **CommandLineTemplate** определяет команду, которая будет выполнена Windows.

По названию объекта уже можно заметить попытку маскировки под системную телеметрию.

---

# Анализ Filter Binding

Связь фильтра с потребителем событий хранится в классе

```
__FilterToConsumerBinding
```

В репозитории присутствовала связь

```
SCM Event Log Filter
        ↓
NTEventLogEventConsumer
```

Она выглядит вполне легитимной.

Однако объект

```
EngineTelemetryConsumer
```

никакой привязки не имел.

Это означало, что вредоносная логика спрятана внутри самого Consumer.

---

# Извлечение команды PowerShell

При анализе структуры экземпляра удалось получить содержимое свойства

```
CommandLineTemplate
```

В нём находилась команда

```powershell
cmd /C powershell.exe -Sta -Nop -Window Hidden -enc <Base64>
```

Используются сразу несколько типичных признаков вредоносного PowerShell:

* `-Nop` — запуск без пользовательского профиля;
* `-Window Hidden` — скрытое окно;
* `-enc` — команда передаётся в Base64.

---

# Декодирование PowerShell

Base64 оказался закодирован в UTF-16LE.

После декодирования получаем следующий сценарий:

```powershell
$file = ([WmiClass]'ROOT\cimv2:Win32_HardwareTelemetry').Properties['ConfigData'].Value;

$o = New-Object IO.MemoryStream;

$d = New-Object IO.Compression.DeflateStream(
    [IO.MemoryStream][Convert]::FromBase64String($file),
    [IO.Compression.CompressionMode]::Decompress
);

...

[Reflection.Assembly]::Load($o.ToArray()).EntryPoint.Invoke(...)
```

---

# Что делает этот PowerShell?

### Шаг 1

Получает свойство

```
ConfigData
```

из класса

```
Win32_HardwareTelemetry
```

То есть сама полезная нагрузка **не хранится в PowerShell**.

PowerShell лишь извлекает её из WMI.

---

### Шаг 2

Содержимое свойства представляет собой

```
Base64
```

---

### Шаг 3

Полученная строка распаковывается алгоритмом

```
Deflate
```

---

### Шаг 4

Полученный массив байт загружается как

```
.NET Assembly
```

через

```powershell
Reflection.Assembly.Load()
```

---

### Шаг 5

После загрузки вызывается

```
EntryPoint
```

то есть исполняемый файл запускается прямо из памяти.

На диск он не записывается.

---

# Поиск класса Win32_HardwareTelemetry

Следующим этапом необходимо было найти класс

```
Win32_HardwareTelemetry
```

в пространстве

```
root\CIMV2
```

Экземпляров класса обнаружено не было.

Однако удалось исследовать **описание класса (ClassDefinition)**.

Внутри него находилось большое бинарное поле.

После анализа структуры было найдено свойство

```
ConfigData
```

с типом

```
string
```

После него располагалась огромная Base64-строка.

Именно она и является полезной нагрузкой.

---

# Извлечение полезной нагрузки

Из бинарной структуры класса была извлечена строка длиной примерно

```
2212 символов
```

Она была сохранена:

```bash
payload.b64
```

После чего декодирована:

```bash
base64 -d payload.b64 > payload.deflate
```

Полученный файл сжат алгоритмом Deflate.

---

# Распаковка Deflate

Небольшой Python-скрипт распаковал поток:

```python
import zlib

data = open("payload.deflate","rb").read()

out = zlib.decompress(data,-15)

open("payload.bin","wb").write(out)
```

После распаковки появился файл

```
payload.bin
```

---

# Анализ PE-файла

Определяем тип файла.

```bash
file payload.bin
```

Получаем

```
PE32 executable
Intel 80386
Mono/.NET assembly
```

То есть внутри WMI действительно хранилась полноценная .NET-сборка.

Проверка первых байтов подтверждает формат PE:

```
4D 5A
```

или

```
MZ
```

---

# Дизассемблирование

Для просмотра IL-кода использовалась утилита

```bash
monodis payload.bin > payload.il
```

В результате была получена промежуточная сборка (Intermediate Language), пригодная для дальнейшего статического анализа.

 После всех этапов, результат: THM{P4tch_op3ned_th3_BacKd00r}
 
---

# Итоговая схема атаки

Вся цепочка работы вредоносного механизма выглядит следующим образом:

```
WMI Event
      │
      ▼
Event Filter
      │
      ▼
CommandLineEventConsumer
      │
      ▼
PowerShell
      │
      ▼
Получение ConfigData из WMI
      │
      ▼
Base64
      │
      ▼
Deflate
      │
      ▼
.NET Assembly
      │
      ▼
Reflection.Assembly.Load()
      │
      ▼
Запуск EntryPoint
```

