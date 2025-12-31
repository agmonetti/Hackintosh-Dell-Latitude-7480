# Hackintosh Dell Latitude 7480 - macOS Ventura

Este repositorio contiene la configuración EFI (OpenCore) necesaria para ejecutar macOS Ventura en una Dell Latitude 7480.

> **Estado:** Funcional (Gráficos, Audio, Teclado, Touchpad con gestos).
> **Bootloader:** OpenCore

## 💻 Especificaciones de Hardware

| Componente | Detalle | Notas |
| :--- | :--- | :--- |
| **Modelo** | Dell Latitude 7480 | |
| **CPU** | Intel Core i7-7600U | Kaby Lake |
| **GPU** | Intel HD Graphics 620 | Aceleración gráfica completa |
| **RAM** | 16 GB] | |
| **Almacenamiento** | SK hynix SC308 S | |
| **Audio** | Realtek ALC3246 (ALC256) | Layout ID: 11 |
| **Ethernet** | Intel I219-LM | |
| **Touchpad** | ALPS I2C | Requiere configuración especial AlpsHID |

## ⚙️ Configuración de BIOS

Para arrancar correctamente, la BIOS debe estar configurada así:

* **SATA Operation:** AHCI
* **Secure Boot:** Disabled
* **Touchpad/Mouse:** "Touchpad/PS-2 Mouse" (Crítico para que el touchpad funcione en modo I2C/ALPS)
* **Virtualization (VT-d):** Disabled (o usar `DisableIoMapper` en config.plist)
* **Fast Boot:** Minimal o Disabled

## 📂 Estructura y Kexts Críticos

### Orden de Carga del Kernel (Crucial)
El orden de los Kexts en `config.plist` -> `Kernel` -> `Add` es estricto para evitar Kernel Panics con el Touchpad ALPS:

1.  **Lilu.kext**
2.  **VirtualSMC.kext**
3.  **WhateverGreen.kext**
4.  **AppleALC.kext**
5.  **VoodooPS2Controller.kext** (Teclado)
6.  **VoodooPS2Keyboard.kext**
7.  **VoodooI2CServices.kext**
8.  **VoodooGPIO.kext**
9.  **VoodooInput.kext** (Versión de VoodooI2C - Enabled: True)
10. **VoodooI2C.kext**
11. **VoodooI2CHID.kext** (Versión modificada para compatibilidad ALPS)
12. **AlpsHID.kext** (Driver satélite específico para Dell ALPS)

> **Nota:** `VoodooPS2Trackpad.kext` y el `VoodooInput` de PS2 deben estar **Desactivados (False)**.

### Parches ACPI (SSDTs)
Ubicados en `EFI/OC/ACPI`:

* `SSDT-EC-USBX-LAPTOP.aml` (Gestión de energía embebida)
* `SSDT-PLUG-DRTNIA.aml` (Gestión de energía CPU)
* `SSDT-PNLF.aml` (Brillo de pantalla)
* `SSDT-XOSI.aml` (Simulación de Windows para activar I2C)
* *(Desactivado)* `SSDT-GPI0.aml` (Genera conflictos en este modelo específico)

### Argumentos de Arranque (Boot-Args)
`NVRAM` -> `Add` -> `7C436110-AB2A-4BBB-A880-FE41995C9F82`:

* `-v`: Modo verbose (texto de arranque).
* `keepsyms=1 debug=0x100`: Depuración de pánicos.
* `alcid=11`: Habilita el audio (altavoces y micrófono).
* `-vi2c-force-polling`: **Obligatorio** actualmente para que el cursor funcione, ya que el modo interrupción (GPIO) es inestable en este panel ALPS.

## 📝 To Do 

Lista de tareas pendientes para perfeccionar el sistema:

### 🔧 Prioridad Alta
- [ ] **Eliminar logs de arranque (Verbose):**
    - Quitar `-v` y `debug=0x100` de `boot-args`.
    - En `Misc -> Debug`, desactivar `Target` (poner a 3 o 0) y `ApplePanic`.
- [ ] **Solucionar "Lagging" del Touchpad:**
    - El cursor funciona pero con retraso debido al modo "Polling".
    - *Posible solución:* Investigar parcheo manual de Pinning GPIO o probar downgrade de `VoodooI2C` a versión 2.8 para mejorar compatibilidad con ALPS.
- [ ] **Autoinicio directo (Skip OpenCore Menu):**
    - En `config.plist` -> `Misc` -> `Boot`:
        - `ShowPicker`: **False** (Oculta el menú).
        - `Timeout`: **5** (Espera 5 seg y arranca automático).
        - `PollAppleHotKeys`: **True** (Para poder mostrar el menú manteniendo presionada la tecla `Esc` o `Option` al arrancar si se necesita emergencia).

- [ ] **Gestión de Energía Avanzada (CPUFriend):** Generar `CPUFriendDataProvider.kext` para optimizar las frecuencias del i7-7600U, mejorando la duración de batería y bajando la temperatura.
- [ ] **Interfaz Gráfica (OpenCanopy):** Si decides volver a activar el menú de arranque, instalar el recurso `OpenCanopy` para tener iconos visuales tipo Mac real en lugar de texto simple.
- [ ] **Hibernación:** Desactivar hibernación profunda (`sudo pmset -a hibernatemode 0`) para evitar corrupción de datos en Hackintosh.
