# Windows Server Active Directory

Em construção!!!!.......

## Índice
[Diagrama da Rede](#Diagrama-da-Rede)

[Objectivo do Projecto](#Objectivo-do-Projecto)
- [Competências Adquiridas](#Competências-Adquiridas)

[Passo a Passo](#Passo-a-Passo)
- [Criação e configuração Inícial da Maquina Virtual](#Criação-da-Maquina-virtual)   
- [Iniciar a Maquina e instalação do SO](#Iniciar-a-Maquina-e-instalação-do-SO)
- [Configuração de um RAID 5](#Configuração-de-um-RAID)
- [Configuração do Servidor](#Configuração-do-Servidor)
  
  - [Instalação e Configuração Active Directory](#Instalação-Active-Directory)
  - [Configuração DNS](#Configuração-do-DNS)
  - [Instalação e Configuração DHCP](#-Instalação-e-Configuração-do-DHCP)
  - [NIC Teaming](#NIC-Teaming)


## Diagrama da Rede

## Objectivo do Projecto


### Competências Adquiridas

## Passo a Passo

### Criação da Maquina virtual
 <br/>
    <br/>
Vamos começar com a criação de uma máquina virtual utilizando o Virtualbox. 
Para adicionar uma máquina nova a partir de uma ISO previamente descarregada, clicamos em "New".  


<p align="center">
  <br/>
  Machine -> New
  <br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/371b4728-a7cc-48fd-9b1b-72ab5378b7dd" height="80%" width="80%"/>
    <br/>
    <br/>
  <p/>
    
De seguida damos o nome a máquina, escolhemos a pasta para onde serão guardados todos os ficheiros que são criados durante a criação da máquina, tal como o disco virtual, e indicamos o ficheiro ISO que vamos usar para a criação. Clique `Next`  


**Nota:** Selecionei a opção `Skip unattended installation` porque é do meu interesse proceder a instalação manual do sistema operativo. 
<br/>
    <br/>

<p align="center">
  <br/>
  Seleção do Sistema Operativo
  <br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/1fc6b161-515b-43e1-b585-5c50e073c211" height="60%" width="60%"/>
<p/> 
  <br/>
  <br/>

Nas janelas seguintes escolhi as especificações de Hardware da máquina virtual. Para este projeto **50GB** de disco, **4GB** de RAM e **4** CPUs são suficientes.  


<p align="center">
  <br/>
  Tamanho do Disco
  <br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/f3f9b919-3dcd-41b2-8bbc-60e28c61e927" height="60%" width="60%"/>
  <br/>
  <br/>
<p/> 
  
  Na janela de **Hardware** Selecionei a opção **EFI**, mais em baixo explicarei esta opção bem como Secure Boot que irei selecionar e porquê.
  <br/>
  <br/>
<p align="center">
  Hardware
  <br/><br/>
  <img src="https://github.com/user-attachments/assets/c3cadb82-1964-46eb-acf7-0b593280ff53" height="60%" width="60%"/>
  <br/><br/>
  Resumo final das opções selecionadas.
  <br/><br/>
  <img src="https://github.com/user-attachments/assets/cefccaaa-a482-4882-b614-8a3884f5ee10" height="60%" width="60%"/>
<p/>  
<br/>
<br/>
  
### Settings
Antes de Iniciar a máquina e proceder a instalação do Windows Server 2019 a partir da ISO previamente montada, existem algumas opções e configurações que irei selecionar.  
É possível aceder as configurações da maquina clicando em `Settings`com a máquina selecionada. 
<p align="center">
  <br/>
  <img src="https://github.com/user-attachments/assets/2886d5c6-f69b-4b6f-be79-d0985594675b" height="60%" width="60%"/>
<p/> 
  <br/> <br/>
 Dentro do Menu `Settings` activei duas opções que irão aumentar a performance e a segurança da maquina. 

 **EFI (Extensible Firmware Interface):** É uma interface de firmware que substitui a BIOS tradicional, oferecendo um boot mais rápido, suporte para discos grandes (GPT) e recursos avançados de segurança e compatibilidade.

**Secure Boot:** É uma funcionalidade do EFI/UEFI que impede a execução de software não assinado durante a inicialização do sistema, ajudando a proteger contra malware e rootkits ao garantir que apenas código confiável seja carregado.
 
 <br/> <br/>
 <p align="center">
  <img src="https://github.com/user-attachments/assets/e6696933-611d-4390-8009-c886d3f96af8" height="60%" width="60%"/>
  <br/> <br/>
 <p/> 
 Na aba processador, ativei a opção `PAE/NX`
  
 **PAE (Physical Address Extension):** Permite que processadores de 32 bits acessem mais de 4 GB de RAM, estendendo o endereçamento para até 64 GB. 

**O NX (No eXecute)** protege contra ataques que tentam executar código malicioso em áreas de memória que deveriam conter apenas dados. Os principais tipos de ataques que ele mitiga incluem:
- **Buffer Overflow com Execução de Código** – Um atacante pode tentar sobrescrever áreas da memória com código malicioso e executá-lo. O NX impede essa execução em regiões de dados, como a pilha (stack) e o monte (heap).

- **Return-to-libc** – Em vez de injetar código, o atacante tenta reutilizar funções legítimas da biblioteca C para executar comandos maliciosos. Embora o NX não impeça diretamente esse ataque, ele força os atacantes a usar técnicas mais avançadas, como ROP (Return-Oriented Programming).

- **Ataques de Shellcode** – O NX evita que código arbitrário injetado na memória seja executado, dificultando a execução de shellcodes maliciosos.
<br/> <br/>


<p align="center">  
  <br/> <br/>
  <img src="https://github.com/user-attachments/assets/857746a6-d3b0-4ce5-bbc9-8cf7a41933d1" height="60%" width="60%"/>
<p/>
 <br/> <br/>
 
 #### Configuração de placa de rede

 De seguida configurei dois adaptadores de rede, 1 ligado a rede externa, em NAT, e outro ligado a rede interna. 
 <br/> <br/>
 <p align="center"> 
  <img src="https://github.com/user-attachments/assets/0033b34b-f3dd-4085-8ef8-1b1cc047768a" height="60%" width="60%"/>
   <br/> <br/>
  <img src="https://github.com/user-attachments/assets/fa3c3a84-4bf2-4982-bf92-65237cf9ba0a" height="60%" width="60%"/>
<p/> 
<br/>
<br/>
  
### Iniciar a Maquina e instalação do SO  
De seguida iniciamos a máquina, procedendo para a instalação do sistema operativo. Basta seguir os passos que mostra nas imagens.  
    <br/> <br/>
<p align="center">
  <img src="https://github.com/user-attachments/assets/9dbb0776-27b6-4a15-8d3d-f544cf8371c7" height="60%" width="60%"/>
  <img src="https://github.com/user-attachments/assets/ec85779a-212d-4894-8c35-cdea23cf0acb" height="60%" width="60%"/>
  <img src="https://github.com/user-attachments/assets/c0d27ba3-4884-4dae-92d1-a32a18909c2c" height="60%" width="60%"/>
  <img src="https://github.com/user-attachments/assets/c5f56a00-a933-4768-877f-c443914d3289" height="60%" width="60%"/>
  <img src="https://github.com/user-attachments/assets/3f64bd6e-5b4b-46ad-aedd-bb8fe6dfe09e" height="60%" width="60%"/>
  <img src="https://github.com/user-attachments/assets/510c6116-9d86-4bdc-af6c-1b6a49605afc" height="60%" width="60%"/>
<p/> 
<br/>
<br/>

### Configuração de um RAID
<br/><br/>
<p align="center">  
<img src="https://github.com/user-attachments/assets/5e293f1a-040b-44e5-b3bf-f46d3a56f059" height="60%" width="60%"/><br/><br/>
<img src="https://github.com/user-attachments/assets/351d9381-47ba-4a83-bec1-0e605a22acec" height="60%" width="60%"/><br/><br/>
<img src="https://github.com/user-attachments/assets/c38e12b9-107b-44f5-9e2f-34c488f3e62b" height="60%" width="60%"/><br/><br/>
<img src="https://github.com/user-attachments/assets/5c3c4824-a2d6-43e0-8b50-c01d5a5d2031" height="60%" width="60%"/><br/><br/>
<img src="https://github.com/user-attachments/assets/fac7a11e-1681-46db-bb13-75c27ec8273b" height="60%" width="60%"/><br/><br/>
<img src="https://github.com/user-attachments/assets/bb6dae46-70e1-4a24-9b45-83bcb9fbb603" height="60%" width="60%"/><br/><br/>
<img src="https://github.com/user-attachments/assets/0733e75a-149b-48e5-a2b7-e8eb03a70095" height="60%" width="60%"/><br/><br/>
<img src="https://github.com/user-attachments/assets/7d9943f7-4ee4-4364-9cea-13a04fcfb450" height="60%" width="60%"/><br/><br/>
<img src="https://github.com/user-attachments/assets/67bd7e88-145f-4165-aa04-8829c99ae0b3" height="60%" width="60%"/><br/><br/>
<img src="https://github.com/user-attachments/assets/e5318114-950b-4248-954e-22875be3fd88" height="60%" width="60%"/><br/><br/>
<img src="https://github.com/user-attachments/assets/a9d61723-8e47-4a86-a621-cab5f8227c86" height="60%" width="60%"/><br/><br/>
<img src="https://github.com/user-attachments/assets/b6d1ffd5-d92e-4c99-bea7-9e1dbcd5ff64" height="60%" width="60%"/><br/><br/>
<img src="https://github.com/user-attachments/assets/033c4265-7e84-43b8-ab4c-3aa0c752ebce" height="60%" width="60%"/><br/><br/>
<img src="https://github.com/user-attachments/assets/cd284cd7-d01a-4870-ae7b-5bcc21becdf4" height="60%" width="60%"/><br/><br/>
<img src="https://github.com/user-attachments/assets/7ac1e68b-f5de-4ae7-bea1-19cc1b0c76eb" height="60%" width="60%"/><br/><br/>
<img src="https://github.com/user-attachments/assets/b72fdd18-e166-4650-903b-1d899c9db030" height="60%" width="60%"/><br/><br/>
<p/>
<br/><br/>
  
### Configuração do Servidor

Primeira coisa a fazer é mudar o nome do Servidor, e atribuir um sufixo DNS. No meu caso dei nome ao Servidor Marcio e o sufixo DNS pilao.pt que irá ser o nome da minha rede.
<br/>
<p align="center">  
  <img src="https://github.com/user-attachments/assets/72fe12bf-4eea-4d43-ab64-dc7b2c7fe1dd" height="60%" width="60%"/>
  <img src="https://github.com/user-attachments/assets/9028201a-3669-43c1-8c15-f86ddcb06c56" height="60%" width="60%"/>
  <img src="https://github.com/user-attachments/assets/2a4584ec-7609-4f2a-8842-08f767db9b1e" height="60%" width="60%"/>
  <img src="https://github.com/user-attachments/assets/8bd19136-3846-46d5-bacb-1327d4605fce" height="60%" width="60%"/>
 <p/>
<br/>
 <br/>
 De seguida é necessário reiniciar a máquina para as alterações serem aplicadas. 
 <br/>
 <br/>
 <p align="center"> 
  <img src="https://github.com/user-attachments/assets/652dcc53-b215-47f3-8d47-119c6ee73c19" height="60%" width="60%"/>
<p/>
 
  Para melhor identificar qual o adaptador que ira ser configurado de seguida mudei o nome dos adaptadores para externo e interno tal como mostra nas imagens.
  <br/><br/>
  <p align="center"> 
   <img src="https://github.com/user-attachments/assets/d334ac62-5249-4754-882e-7fd4335abfa9" height="60%" width="60%"/>
   <img src="https://github.com/user-attachments/assets/8683d05b-cd24-4288-ab65-dfc0afb72a0c" height="60%" width="60%"/>
  <p/>
   <br/><br/>
 
 #### Atribuição de IP da rede interna.
De seguida irei atribuir um IP fixo a minha rede interna configurando o adaptador ao qual chamei “interno”. 
<br/><br/>
Clicando em “Properties” e de seguida escolhendo a opção IPV4 e clicando em “Properties”, abrirá a janela seguinte onde introduzi o IP que desejo e a subnet mask da rede, isto significa que as máquinas que fizerem parte da rede interna e estiverem ligadas a este servidor irão ser atribuídos IPs com base neste mask. 
<p align="center"> 
 <br/><br/>
 <img src="https://github.com/user-attachments/assets/84ca6b0e-9a06-449d-8b76-cf9b34997ba9" height="60%" width="60%"/>
 <br/><br/>
 <img src="https://github.com/user-attachments/assets/3d9873f8-c0f0-4bc3-845b-70ef52b650d5" height="60%" width="60%"/>
<p/>
 <br/><br/>
Para finalizar clicando em “advanced” irei atribuir a prioridade 1 para que este adaptador tenha a maior prioridade de ligação na máquina.
Na adaptador externo coloquei metric 20. 
 <br/><br/>
 <p align="center"> 
 <img src="https://github.com/user-attachments/assets/d9fd26ab-a4e0-42e4-9110-12053403afe8" height="60%" width="60%"/>
<p/>
 <br/><br/>
Testando a configuração usando o comando ´ping´ vejo que o IP esta a responder. 
Usando o comando ´ipconfig´ vejo as definições dos meus adaptadores de rede.
 
 <br/><br/>
 <p align="center"> 
  Ping
  <br/><br/>
 <img src="https://github.com/user-attachments/assets/462c4022-e8fc-4a6c-ba0d-ef4a301c1d37" height="60%" width="60%"/>
  <br/><br/>
  Ipconfig
  <br/><br/>
 <img src="https://github.com/user-attachments/assets/35d06919-f349-4198-bc08-d8cc5ebd2e73" height="60%" width="60%"/>
<p/>


### Instalação Active Directory
 De seguida instalei Active Directory no servidor.
 <br/><br/>
 **Active Directory:** É um serviço da Microsoft usado para gerenciar identidades e recursos em redes corporativas. Ele é amplamente utilizado em empresas para autenticação, autorização e gerenciamento de usuários, computadores e outros dispositivos dentro de um ambiente Windows.
 <br/><br/>
**Componentes do Active Directory**
 <br/><br/>
**Domain Controller (DC)** 🏢 – Servidor que gerencia o AD e autentica os usuários. O servidor que estou a configurar.

**Domínio** 🌍 – Um grupo de computadores, usuários e dispositivos gerenciados por um controlador de domínio.

**Floresta (Forest)** 🌳 – Um ou mais domínios que compartilham a mesma estrutura de AD.

**OU (Unidade Organizacional)** 📂 – Subdivisões dentro do domínio para organizar usuários, grupos e dispositivos.
 <br/><br/>
 No canto superior direito da janela do "Server Manager" clicamos em `Manage`e depois `Add Roles and Features`.
 Isto abre o Add Roles and Features Wizard, o assistente que nos permite instalar novos serviços no servidor.
<br/><br/>
 <p align="center"> 
  <img src="https://github.com/user-attachments/assets/d3231b47-bb78-4821-9793-44d581ac0b1c" height="60%" width="60%"/> 
    <br/><br/>
  <img src="https://github.com/user-attachments/assets/d96248e3-eac4-45e9-bf10-8b417aa99751" height="60%" width="60%"/> 
    <br/><br/>
<p/>      
  
  Clicando "Next" ate chegar a esta janela onde escolhemos o servidor que irá funcionar como **Controlador de Dominio**
<p align="center">  
<br/><br/>
  <img src="https://github.com/user-attachments/assets/b3fe96f3-d332-4b2e-a8c9-c3f9b1a93862" height="60%" width="60%"/>
 <p/> 
   <br/><br/>
  
   Na janela seguinte então escolhemos que queremos instalar `Active Directory Domain Service`.
   
<br/><br/>
 <p align="center">
  <img src="https://github.com/user-attachments/assets/d29de415-9c80-4d11-b6bd-ed5764458c2f" height="60%" width="60%"/>  
 <p/>
 <br/><br/>
 Clicando sempre em "Next" finalizamos a instalação.
  <br/><br/>
 <p align="center">
   <img src="https://github.com/user-attachments/assets/507e8ffc-9504-43e8-8806-3722a1c5dfe7" height="60%" width="60%"/> 
    <br/><br/>
  <img src="https://github.com/user-attachments/assets/be410ffe-0438-467a-9cd4-3f883438cf03" height="60%" width="60%"/> 
    <br/><br/>
<p/>

Quando um servidor tem o AD DS instalado pode ser promovido a **Controlador de Domínio (DC)**, ele assume o papel de gerenciar a autenticação e autorização na rede.
<br/>
<br/>
**O que acontece ao instalar o AD DS?**

**O Servidor pode se tornar um Controlador de Domínio** 🌍
  - Ele armazenará e gerenciará a Base de Dados do Active Directory.
  - Fará a autenticação dos usuários e computadores na rede.

**Criação de um Domínio ou Adição a um Domínio Existente** 🏢
  - Pode criar um novo domínio ou integrar-se a um domínio já existente.
  - Exemplo: Se criar um domínio chamado `pilao.pt`, todos os usuários e computadores pertencentes à rede usarão esse domínio.

**Gerenciamento de Políticas e Permissões** 🔐
  - Através do Group Policy (GPO), poderá definir regras para todos os computadores e usuários.
  - Exemplo: Impedir que os usuários alterem configurações do sistema ou definir senhas complexas.

**Criação da Estrutura de Diretórios do AD** 📂
  - O AD DS organiza os recursos em Unidades Organizacionais (OU’s) para facilitar a administração.
  - Exemplo: Separar funcionários por departamentos (RH, TI, Financeiro).
<br/><br/>  
#### Configuração do Domain Service
 <br/><br/>
No Server Manager clicamos em AD DS e o servidor tem um alerta que é necessário configurar o Domain service. Clicando em `more` abre um wizard.

   <p align="center">
   <img src="https://github.com/user-attachments/assets/a8c9e2fd-d15b-4a36-9f0c-df7df9288eb9" height="60%" width="60%"/>
 <p/>
 <br/><br/>
   
   O nosso objectivo é adicionar uma **"forest"** nova.
   <br/><br/>
<p align="center">
   <img src="https://github.com/user-attachments/assets/a43b00c7-a3c3-4b09-89bb-4c4fb3052b50" height="60%" width="60%"/>
<p/>
  <br/><br/>
  
  No ecrã seguinte escolhemos as capacidades do Domain Controller e escolhemos a password do modo **DSRM**, que é o modo de restauração do AD, caso seja necessário manutenção e recuperação da base de dados do AD. 
  <br/><br/>
  <p align="center">
    <img src="https://github.com/user-attachments/assets/6a393e22-b8ce-40e7-89fc-17c8ca113165" height="60%" width="60%"/>
    <br/><br/>
 <p/>
   
   Nos ecrãs seguintes basta seguir as imagens e carregando em **"Next"** até chegar a última janela onde clicamos em instalar.
   
   <br/><br/>
   <p align="center">
    <img src="https://github.com/user-attachments/assets/9755f87a-a125-4474-825c-c5369a46dc6e" height="60%" width="60%"/>
    <br/><br/>
    <img src="https://github.com/user-attachments/assets/2a4460f4-74c9-4fcd-88fa-b6524db0f4ef" height="60%" width="60%"/>
    <br/><br/>
    <img src="https://github.com/user-attachments/assets/51d9df26-b9a0-4fbf-89e9-096e7da9e689" height="60%" width="60%"/>
    <br/><br/>
  <br/><br/>
 <p/>
   
### Configuração do DNS   
<br/><br/> 
O Active Directory depende do DNS para localizar controladores de domínio e autenticar usuários.
<br/>
Uma resolução de nomes correta garante a estabilidade da rede, permitindo que clientes encontrem serviços facilmente.
<br/>
Configurações incorretas de DNS podem causar falhas de login, problemas na replicação do AD e dificuldades na comunicação da rede.
<br/><br/> 

Por defeito a Forward Lookup Zone foi criada automaticamente a quando da instalação do AD DS, agora vamos configurar a **Reverse Lookup Zone**
<br/>
A **Reverse Lookup Zone** tem como principal objetivo permitir que o **DNS** traduza **endereços IP em nomes de domínio**, funcionando de forma oposta à **Forward Lookup Zone**, que traduz **nomes de domínio em endereços IP**.
<br/><br/>
Não é obrigatório criar uma **Reverse Lookup Zone**, mas é altamente recomendado para redes empresariais, especialmente se o servidor DNS for usado para serviços internos e autenticação de rede.
<br/><br/> 

No Server Manager, clicamos em Tools e selecionamos DNS para abrir o DNS Manager.

No painel esquerdo, clicamos com o botão direito em Reverse Lookup Zones e selecionamos New Zone.
<p align="center">
  <img src="https://github.com/user-attachments/assets/ff537070-47dd-4c8c-8a0b-ce2637bb0d7b" height="20%" width="20%"/>
  <img src="https://github.com/user-attachments/assets/7dd22b88-298a-46ef-80f7-d6046a35d835" height="60%" width="60%"/>
  <br/><br/> 
  <img src="https://github.com/user-attachments/assets/e3cc732b-9c50-4726-9f5d-d491cea76ac1" height="60%" width="60%"/>
  <br/><br/> 
 <p/>
   
  Selecionamos **Primary Zone**, marquei `Store the zone in Active Directory (AD)` e depois em **"Next"**.
  
  Selecionar **Primary Zone** significa que este servidor DNS será responsável por manter e gerenciar a zona de forma principal. Isso quer dizer que:
  - Ele armazenará os registros DNS da zona.
  - Será o servidor autorizado para resolver consultas dentro dessa zona.
<br/><br/>
<p align="center">
  <img src="https://github.com/user-attachments/assets/9c476774-c407-4324-9d86-548c560a326c" height="60%" width="60%"/>
  <br/><br/>
<p/> 
  
  Escolhemos opção, `To all DNS servers running on domain controllers in this domain: pilao.pt` clicamos em "next"
  
  <br/>
  Esta opção replica a zona de DNS apenas para os servidores DNS que estão em execução em controladores de domínio dentro do domínio "pilao.pt". Isso significa que apenas os controladores de domínio que pertencem ao domínio      "pilao.pt" e que estão executando o serviço DNS terão uma cópia desta zona.
  
 <br/><br/>
 <p align="center"> 
  <img src="https://github.com/user-attachments/assets/e7fbc716-6213-461d-9876-fe0d00415c87" height="60%" width="60%"/>
  <br/><br/>
  <img src="https://github.com/user-attachments/assets/cc26a3c2-c55c-4620-9265-477225ccb31a" height="60%" width="60%"/>
  <br/><br/>
<p/>
  
  Inseri a parte da rede do endereço IP da rede interna (por exemplo, para a minha `192.168.1.0/24`, inseri `192.168.1`) e cliquei em Next.
  
<br/><br/>
<p align="center"> 
  <img src="https://github.com/user-attachments/assets/2bac9f17-28b5-449f-8ca4-d8a85e0a1469" height="60%" width="60%"/>
<p/>
   <br/><br/>
  
  Selecionei `Allow only secure dynamic updates`, clicando de seguida em Next, depois em Finish.
  
   <br/><br/>
<p align="center">   
  <img src="https://github.com/user-attachments/assets/4b707c6a-ed56-4ec7-836c-d525ef4c1634" height="60%" width="60%"/>
  <br/><br/>
  <img src="https://github.com/user-attachments/assets/28ea3fc6-9a10-4188-b4e3-b152804d79fe" height="60%" width="60%"/>
  <br/><br/>
 <p/>
   
Para concluir a configuração do **Reverse Lookup**, tenho de indicar a "quem" perguntar pelo nome do domínio associado a um Ip.
<br/>
Para isso é necessário criar um novo `Pointer` e indicar que quero que o meu servidor seja aquele que faz essa conversão, para isso indico o seu IP e o seu **hostname**.
<br/><br/>
<p align="center"> 
  <img src="https://github.com/user-attachments/assets/85656f97-9f88-4dfe-b8f3-b7be883bf7bc" height="60%" width="60%"/>
  <br/><br/>
  <img src="https://github.com/user-attachments/assets/d4847689-e6fc-427b-9c37-7a2b08fe5bf8" height="60%" width="60%"/>
  <br/><br/>
  <img src="https://github.com/user-attachments/assets/bedc2545-dc2f-46e1-aaf0-ec9e0b6b8904" height="60%" width="60%"/>
  <br/><br/>
<p/>
Por fim testamos tanto o forward lookup bem como o reverse lookup e confirmamos que o servidor está corretamente converter ip em nome e nome em ip  
  <br/><br/>
<p align="center"> 
  <img src="https://github.com/user-attachments/assets/c20210f3-8284-4ca7-ab43-4ecbb99501f1" height="60%" width="60%"/>
  <br/><br/>
<p/>

### 🧭 Instalação e Configuração do DHCP
<br/><br/> 
Para começar a instalação do serviço DHCP, abrimos o Server Manager, clicamos em Manage no canto superior direito e escolhemos a opção Add Roles and Features.
<br/><br/>
Isto abre o Add Roles and Features Wizard, o assistente que nos permite instalar novos serviços no servidor.
<br/><br/>
<p align="center">
  <img src="https://github.com/user-attachments/assets/79948978-a2bc-4681-8cd6-11839f2ae972" height="60%" width="60%"/>
<br/><br/>  

Seguimos os passos do assistente até chegarmos à secção **Select server roles**. Aqui, selecionamos a opção **DHCP Server** e clicamos em **"Next"** para continuar.
<br/>  
Ao instalar esta role, permitimos que o servidor atribua endereços IP automaticamente aos dispositivos da rede, bem como outras configurações como gateway e DNS.
<br/>
Continuamos a clicar **"Next"** e por fim **"Install"**
<p/>
<p align="center">
  <br/><br/> 
   <img src="https://github.com/user-attachments/assets/c4374bd4-e1de-40db-8229-0fbd1e43ee8e" height="60%" width="60%"/>
  <br/><br/>  
  <img src="https://github.com/user-attachments/assets/0ebf926e-4869-4dc5-9a84-02c3c8c6d360" height="60%" width="60%"/>
  <br/><br/>
<p/> 

<br/>

Após a instalação da role, no **Server Manager**, surge um aviso no canto superior indicando que a role **DHCP** precisa de ser configurada. Clicamos em **Complete DHCP configuration** para iniciar o assistente de pós-instalação.

<br/><br/>
<br/>
<p align="center">
  <img src="https://github.com/user-attachments/assets/e17c0f0c-7c7c-4b08-99f4-92647358bec3" height="60%" width="60%"/>
  <br/><br/>
  <img src="https://github.com/user-attachments/assets/7bea8be6-e939-4521-8309-4fdd63297cbd" height="60%" width="60%"/>
  <br/><br/>
<p/>

Durante a configuração, o assistente pede para autorizarmos o servidor DHCP no domínio. Esta autorização é necessária para garantir que apenas servidores confiáveis atribuem endereços IP na rede.

Confirmamos o **nome do domínio** e a **conta de utilizador** (normalmente já preenchida) e clicamos em Commit para concluir.
<p align="center">
<br/><br/>
  <img src="https://github.com/user-attachments/assets/82c2167d-b7fa-4593-9689-5d7cb1041aba" height="60%" width="60%"/>
  <br/><br/>
<p/>  
Se tudo estiver correto, vemos uma mensagem a indicar que a configuração foi concluída com sucesso. O servidor DHCP está agora autorizado e pronto para que criemos escopos de IP e comecemos a atribuir configurações aos clientes da rede.
<br/><br/>
  
Clicamos em **"Close"** para terminar.
<p align="center">
<br/><br/>
  <img src="https://github.com/user-attachments/assets/f342e044-630b-45a8-98bc-bbb5690ecfc9" height="60%" width="60%"/>
  <br/><br/>
<p/> 

#### 📦 Criar Escopo DHCP  

No **Server Manager**, clicamos em **Tools** e depois em **DHCP** para abrir a consola de gestão do serviço DHCP.<br/><br/>

Na estrutura à esquerda, expandimos o servidor e clicamos com o botão direito em IPv4. A seguir, escolhemos a opção `New Scope`.<br/><br/>

Iniciamos assim o **New Scope Wizard**, que nos vai guiar na criação do escopo de endereços IP.
<br/><br/>
<p align="center">
  <img src="https://github.com/user-attachments/assets/a67ea8c5-34b0-4d08-8256-479b764dfb1e" height="20%" width="20%"/>
  <img src="https://github.com/user-attachments/assets/5e000412-cc24-45c4-b853-83bfbd3af3d4" height="30%" width="30%"/>
  <br/><br/>
<p/>
  
No primeiro passo do assistente, atribuímos um nome descritivo ao escopo. Por exemplo: `Aulas`.<br/><br/>

Este nome serve apenas para organização interna, não afeta o funcionamento da rede.<br/><br/>

Clicamos em **"Next"** para continuar. <br/><br/> 

<p align="center">
  <img src="https://github.com/user-attachments/assets/df3bf7d5-8659-4781-b26f-7c98c650126d" height="60%" width="60%"/>
<p/> 
  <br/><br/>
Neste passo, indicamos o intervalo de endereços IP que o DHCP irá distribuir na rede.
<br/><br/>
Este intervalo define os IPs disponíveis para dispositivos nesta rede específica.
  <br/><br/>
<p align="center">  
  <img src="https://github.com/user-attachments/assets/d47c4685-e650-490f-a76c-e1504e12df0a" height="60%" width="60%"/>
<p/>
  <br/><br/>
Podemos agora definir IPs dentro do intervalo que não queremos que sejam atribuídos automaticamente — por exemplo, IPs reservados para impressoras, servidores, ou outros dispositivos fixos.<br/><br/>

Se não tivermos exclusões, deixamos em branco e clicamos em **"Next"**.
  <br/><br/>
<p align="center">  
  <img src="https://github.com/user-attachments/assets/13f07384-d1c3-4cd6-aa00-7dacc3737023" height="60%" width="60%"/>
<p/>  
  <br/><br/>
Aqui definimos o tempo que um endereço IP permanece atribuído a um dispositivo antes de ser libertado.<br/><br/>

Por padrão, o valor é de **8 dias**. Podemos ajustá-lo conforme as necessidades da rede.
  <br/><br/>
<p align="center">  
  <img src="https://github.com/user-attachments/assets/a00331c2-3aa1-4764-8358-e633ebff442f" height="60%" width="60%"/>
<p/>  
  <br/><br/>
O assistente pergunta se queremos configurar as opções adicionais agora, como:

- Default Gateway

- DNS Servers

- Domain Name

Escolhemos **Yes, I want to configure these options now** e clicamos em **"Next"**.
  <br/><br/>
<p align="center">  
  <img src="https://github.com/user-attachments/assets/e675859c-d81c-43ef-af29-9b5a5cfe30c7" height="60%" width="60%"/>
<p/>  
  <br/><br/>
Aqui indicamos o endereço IP do nosso servidor (ou gateway) da rede, normalmente o dispositivo que faz a ligação à Internet. <br/><br/>

Exemplo no nosso cenário: `192.168.1.200` <br/><br/>

Clicamos em **Add**, confirmamos o IP na lista e depois em **Next**. 
  <br/><br/>
<p align="center">
  <img src="https://github.com/user-attachments/assets/31d40dbe-32bc-4faf-8c6d-2411a8110955" height="60%" width="60%"/>
<p/>
  <br/><br/>
Adicionamos o IP do nosso servidor DNS (normalmente o próprio servidor Windows que está a correr o DHCP e Active Directory).

- **Parent domain:** preenchido automaticamente (`pilao.pt`)

- **IP address:** adicionamos o IP do nosso servidor, `192.168.1.200`
<br/><br/>
<p align="center">
  <img src="https://github.com/user-attachments/assets/d91a2530-38d8-464f-8004-43e23ce92cb8" height="60%" width="60%"/>
 <p/>  
  <br/><br/>
   
Se não utilizarmos WINS na rede (Dispositivos com SO antigos), que é o nosso caso, deixamos em branco e clicamos em **Next**.
  <br/><br/>
  <br/>
O assistente pergunta se queremos ativar o escopo imediatamente.<br/><br/>

Selecionamos **Yes, I want to activate this scope now** e clicamos em **"Next"** e finalizamos a configuração do DHCP.
<br/><br/>
<p align="center">
  <img src="https://github.com/user-attachments/assets/9033d5ee-436e-4016-9be7-bf4d3d89fb7a" height="60%" width="60%"/>
<p/>  
<br/><br/>
  <p/>

### NIC Teaming
<br/><br/>
O **NIC Teaming** permite combinar duas ou mais placas de rede físicas numa única interface lógica, garantindo redundância **(failover)** e/ou maior performance **(load balancing)**. Configurei a redundância para garantir que, se uma das interfaces falhar, a outra mantém a ligação de rede ativa.
<br/><br/>
Primeiro é necessário acrescentar mais 2 placas de rede a nossa máquina virtual, 1 placa externa (NAT) e outra interna.
<p align="center">
 <img src="https://github.com/user-attachments/assets/3371843c-9507-45e9-a5b8-0d0e5d35c504" height="60%" width="60%"/><br/><br/>
 <img src="https://github.com/user-attachments/assets/fcb14b21-b292-492b-8906-3621231b000b" height="60%" width="60%"/><br/><br/>
<p/> 
Mudei os nomes das placas para ser mais fácil indentifica-las  
<br/><br/>  
<p align="center">  
 <img src="https://github.com/user-attachments/assets/07923bdb-dcd7-48f9-bea3-370fb34f0d68" height="80%" width="80%"/><br/><br/>
 <p/>
   
Abrimos o **Server Manager** e no menu lateral direito clicamos em **Local Server**.

Na secção **NIC Teaming**, clicamos na palavra `Disabled` para abrir a consola de gestão.

<br/><br/>  
<p align="center"> 
 <img src="https://github.com/user-attachments/assets/79d620d7-f3f6-4251-bf26-bd6f4149155b" height="60%" width="60%"/><br/><br/>
<p/>

  Na janela de NIC Teaming, clicamos em cima de uma das placas que queremos que forme a Teaming e escolhemos a opção `New Team`.
  
<br/><br/>  
<p align="center"> 
 <img src="https://github.com/user-attachments/assets/031b302e-ed4a-41fa-bf1b-bbe5bdc7cf36" height="60%" width="60%"/><br/><br/>
<p/>

Damos um nome ao nosso Team e selecionamos as duas interfaces de rede físicas que queremos incluir.<br/><br/> 

Clicamos em `Additional properties` para configurar:

- **Teaming mode:** Escolhemos Switch Independent (não depende de configuração no switch)

- **Load balancing mode:** Escolhemos Address Hash ou Dynamic. No caso de estarmos a usar VirtualBox usamos Address Hash.

- **Standby adapter:** deixamos em branco se quisermos que ambas as interfaces estejam ativas, ou escolhemos uma como reserva (standby)

Clicamos em **OK** para criar o Team.
  
<br/><br/>  
<p align="center">    
 <img src="https://github.com/user-attachments/assets/2e95ad5f-2dcd-4992-91df-969c3bdbb3e3" height="60%" width="60%"/><br/><br/>
<p/>

Após alguns segundos, o novo adaptador lógico é criado e aparece na lista de interfaces. 

<br/><br/>  
<p align="center">  
  <img src="https://github.com/user-attachments/assets/abb5a15a-80f5-42b3-8b43-be13f1eb7390" height="60%" width="60%"/><br/><br/>
<p/>

No pequeno Video em baixo, demonstro o NIC Teaming em ação quando um dos adaptadores falha. É possível verificar que o acesso a internet não é interrompido
<br/><br/>

https://github.com/user-attachments/assets/85e7c3b5-6a67-44a9-9fb2-a75299de3740

<br/><br/>

O mesmo pode ser feito com os adaptadores da rede interna, oferecendo uma redundância contra falhas. Para efeitos de simplificação não irei mostrar com imagens como fazer visto ser o mesmo que fiz em cima com os adaptadores externos. 






