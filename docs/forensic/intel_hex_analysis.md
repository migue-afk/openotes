---
title: Intel Hex analysis
layout: home
parent: Forensic
nav_order: 3
tags: [analysis, intel, hex, bin, strings, ofsset]
---

# Intel Hex Analysis
-----

Microcontrollers may contain sensitive information, this can be obtained by analyzing the .hex file extracted from the microcontroller this is possible if the microcontroller is not locked.

---
### Extract the .hex file from the microcontroller

This extraction process is performed on microcontrollers using the Arduino platform. However, the specific method of obtaining the data may vary depending on the environment; what is truly important is having the resulting .hex file.

```bash
 avrdude -c arduino -p m328p -P /dev/ttyACM0 -b 115200 -U flash:r:extractfirmware.hex:i
```

---

### Hex to binary

Now it is necessary to convert `.hex` to `.bin` to analyze with the tool **strings**

```bash
objcopy -I ihex -O binary file_hex.hex file_bin.bin
```


---

### `strings` (extraction of text strings)

There are text strings that may contain useful information, such as passwords, commands, error messages, usernames, among others. We can extract this information using the `strings` command, which can be used with images, memory dumps, executable files, among others.

```bash
strings file_bin.bin | grep -i "passw"
```

This result for file is 

```bash
password123
```

Another example could be

```bash
strings file_bin.bin | grep URL -A 5
```

It is obtained

```bash
AT+HTTPPARA="URL","DIRECCION URL
&dato=
&bateria=
&memoria=
AT+HTTPPARA="REDIR",1
password123
```

> You can find the hexadecimal file in
`https://github.com/migue-afk/Envio_de_datos_a_la_nube_SIM900`

