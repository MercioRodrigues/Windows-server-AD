# Simulação de Cadeia de Ataque - Laboratório de Segurança Ofensiva
<br>
<br>

## Objetivo

Esta parte do projeto é a continuação do projecto anterior Windows Server AD em que foi construido em laboratório um ambiente Active Directory, foram também usadas ferramentas que ja tinham sido preparadas num outro projecto que realizei anteriormente, SOC SOAR Project. <br> Esta parte tem como objetivo simular uma cadeia de ataque realista em um ambiente de domínio Windows e sem deteção por parte do Windows Defender, explorando diferentes etapas de comprometimento típicas de atacantes reais. A simulação foi conduzida em um laboratório controlado, permitindo a análise detalhada de cada fase da intrusão.

A cadeia de ataque envolve as seguintes etapas:

1. **Acesso inicial**: comprometimento de uma estação de trabalho através da execução de uma macro maliciosa em um documento do Word.
2. **Escalada de privilégios local**: obtenção de privilégios SYSTEM explorando uma tarefa agendada mal configurada.
3. **Exfiltração de credenciais**: extração da memória do processo LSASS para capturar credenciais em texto claro e hashes NTLM.
4. **Movimentação lateral**: acesso ao controlador de domínio (Domain Controller) utilizando técnicas de Pass-the-Hash.

**Análise pós-ataque**: utilização de ferramentas de monitorização e deteção como **Wazuh** e **Wireshark** para investigar a intrusão e compreender os rastros deixados nos logs do sistema e na rede.

O objetivo final é obter não apenas uma shell no controlador de domínio, mas também documentar detalhadamente os indicadores de comprometimento (IoCs), comportamentos suspeitos e evidências forenses que podem ser usadas por equipes de defesa para deteção precoce e resposta a incidentes.

---

⚠️ *Aviso: Este projeto foi realizado exclusivamente para fins educacionais e em ambiente isolado. Nenhuma das técnicas aqui descritas deve ser utilizada em ambientes reais sem autorização explícita.*
