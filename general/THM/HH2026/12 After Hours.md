**Tags:** Windows, Forensic, Persistence, Reverse Engineering.

**Difficulty:** Medium.

## Task Description

During an investigation, a dump of a Windows WMI repository was obtained:

```
INDEX.BTR
OBJECTS.DATA
MAPPING1.MAP
MAPPING2.MAP
MAPPING3.MAP
```

The goal of the investigation is to determine whether a persistence mechanism (WMI Persistence) is present in the repository and recover the malicious payload.

---

# What is the WMI Repository?

Windows Management Instrumentation (WMI) stores its objects in a special database — **Repository**.

This is where the following are stored:

* namespaces;
* WMI classes;
* class instances;
* Permanent Event Subscriptions.

Repository files:

```
INDEX.BTR
OBJECTS.DATA
MAPPING*.MAP
```

are not text files and cannot be opened with standard tools. A specialized parser is required for analysis.

---

# Initial Analysis

First, let's determine the file type.

```bash
file INDEX.BTR
```

We get:

```
INDEX.BTR: data
```

Let's try to find printable strings.

```bash
strings INDEX.BTR | head
```

We only get a few internal identifiers:

```
CI_41C53E6DB1ACF2453CEFD41398198E613F10DFF47709ECAB1D7F037756AC8CE7
CI_FD1C1D414B71B5082C266A650E31EF6D5019382724244B685F217C0AAE00A921
CR_E3B0C44298FC1C149AFBF4C8996FB92427AE41E4649B934CA495991B7852B855
```

There is no useful information here.

---

# Attempt to Use an Existing Tool

Initially, the **WMI_Forensics** project was used.

```bash
git clone https://github.com/davidpany/WMI_Forensics
```

Running:

```bash
python3 PyWMIPersistenceFinder.py ../OBJECTS.DATA
```

resulted in an error:

```
TypeError:
expected str instance, bytes found
```

The reason turned out to be quite simple.

The project was written for **Python 2**, while modern Python versions handle byte strings differently. Instead of fixing the old code, it was decided to investigate the repository manually.

---

# Using the dissect.cim Library

The following library was installed to work with the repository:

```bash
python3 -m venv venv
source venv/bin/activate

pip install dissect.cim
```

It provides a Python API for reading the WMI Repository.

The repository can be opened as follows:

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

The repository was successfully loaded.

---

# Enumerating Namespaces

First, we need to understand the structure of the database.

Enumerating the namespaces produced:

```
root
root\subscription
root\DEFAULT
root\CIMV2
root\SecurityCenter2
root\WMI
...
```

The following immediately stands out:

```
root\subscription
```

This is where Windows stores the **Permanent Event Subscription** mechanism, which attackers commonly use for persistence.

---

# Searching for Class Instances

Next, all classes with instances were enumerated.

The following were found in the

```
root\subscription
```

namespace:

```
__EventFilter
CommandLineEventConsumer
__FilterToConsumerBinding
NTEventLogEventConsumer
```

This is a very characteristic set of objects for WMI Persistence.

---

# Event Filter Analysis

Two event filters were found in the repository:

```
EngineTelemetryFilter

SCM Event Log Filter
```

The filter determines **when** the payload will be executed.

---

# Event Consumer Analysis

An object named

```
EngineTelemetryConsumer
```

of type

```
CommandLineEventConsumer
```

was also found.

This class contains the following properties:

```
CommandLineTemplate
ExecutablePath
WorkingDirectory
```

The **CommandLineTemplate** property determines the command that Windows will execute.

The object name itself already suggests an attempt to disguise it as system telemetry.

---

# Filter Binding Analysis

The relationship between the filter and event consumer is stored in the

```
__FilterToConsumerBinding
```

class.

The repository contained the following relationship:

```
SCM Event Log Filter
        ↓
NTEventLogEventConsumer
```

It looks completely legitimate.

However, the

```
EngineTelemetryConsumer
```

object had no binding.

This meant that the malicious logic was hidden inside the Consumer itself.

---

# Extracting the PowerShell Command

While analyzing the instance structure, the contents of the

```
CommandLineTemplate
```

property were obtained.

It contained the following command:

```powershell
cmd /C powershell.exe -Sta -Nop -Window Hidden -enc <Base64>
```

Several typical indicators of malicious PowerShell are used at once:

* `-Nop` — starts without the user profile;
* `-Window Hidden` — hides the window;
* `-enc` — the command is passed as Base64.

---

# Decoding PowerShell

The Base64 was encoded using UTF-16LE.

After decoding, we get the following script:

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

# What Does This PowerShell Do?

### Step 1

It retrieves the

```
ConfigData
```

property from the

```
Win32_HardwareTelemetry
```

class.

This means the payload itself **is not stored in PowerShell**.

PowerShell only extracts it from WMI.

---

### Step 2

The contents of the property are represented as

```
Base64
```

---

### Step 3

The resulting string is decompressed using

```
Deflate
```

---

### Step 4

The resulting byte array is loaded as a

```
.NET Assembly
```

using:

```powershell
Reflection.Assembly.Load()
```

---

### Step 5

After loading, the

```
EntryPoint
```

is invoked, meaning the executable is launched directly from memory.

It is not written to disk.

---

# Searching for the Win32_HardwareTelemetry Class

The next step was to find the

```
Win32_HardwareTelemetry
```

class in the

```
root\CIMV2
```

namespace.

No instances of the class were found.

However, it was possible to analyze the **ClassDefinition**.

Inside it was a large binary field.

After analyzing the structure, the following property was found:

```
ConfigData
```

with type

```
string
```

After it, there was a huge Base64 string.

This is the payload.

---

# Extracting the Payload

The string was extracted from the binary class structure. It was approximately

```
2212 characters
```

long.

It was saved as:

```bash
payload.b64
```

and then decoded:

```bash
base64 -d payload.b64 > payload.deflate
```

The resulting file is compressed using Deflate.

---

# Decompressing Deflate

A small Python script was used to decompress the stream:

```python
import zlib

data = open("payload.deflate","rb").read()

out = zlib.decompress(data,-15)

open("payload.bin","wb").write(out)
```

After decompression, the following file appeared:

```
payload.bin
```

---

# PE File Analysis

Let's determine the file type.

```bash
file payload.bin
```

We get:

```
PE32 executable
Intel 80386
Mono/.NET assembly
```

This confirms that a complete .NET assembly was actually stored inside WMI.

Checking the first bytes also confirms the PE format:

```
4D 5A
```

or

```
MZ
```

---

# Disassembly

The `monodis` utility was used to view the IL code:

```bash
monodis payload.bin > payload.il
```

As a result, an Intermediate Language (IL) representation was obtained, suitable for further static analysis.

After all the steps, the result is:

```
THM{P4tch_op3ned_th3_BacKd00r}
```

---

# Final Attack Chain

The entire chain of the malicious mechanism looks like this:

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
Retrieving ConfigData from WMI
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
EntryPoint Execution
```
